# codereview
code review

```
#!/usr/bin/env python3
"""
Version: 1.4.0

This script processes log files from multiple input folders in parallel using a worker pool.
The number of CPU cores (workers) is read from the configuration via the key CPU_CORES.
The manager process builds a file queue (after file-level filtering) and spawns worker processes.
Each worker processes assigned files (applying file‐ and content‑level filtering) and writes an output
file (if filtered content remains) to OUTPUT_FOLDER.

File-level filtering:
  - Skips a file if its last modified time is older than FROM_TIMESTAMP.
  - If FILE_NAME_INCLUDE_REGEXP_PATTERN is provided (non‑empty), the file name must match at least one pattern.
  - If FILE_INCLUDE_REGEXP_PATTERN is provided (non‑empty), the file must contain at least one matching line.
  
Content-level filtering (performed sequentially within each worker):
  - Discards all lines before the first line with a valid timestamp ≥ FROM_TIMESTAMP.
  - Stops processing lines when a line with a valid timestamp > TO_TIMESTAMP is encountered.
  - Removes lines matching any CONTENT_EXCLUDE_REGEXP_PATTERN.

During processing, each worker (assigned a core slot) updates a shared progress dictionary.
The manager process refreshes the same fixed display (one line per worker) every 2 seconds.
After a file is processed, its output is written (if not empty) and a summary (original vs. filtered line counts)
is reported. Finally, the total processing time is printed.

Usage:
  - With no command-line parameters, the script uses the config values.
  - Command-line overrides:
       > "YYYY-MM-DD HH:MM:SS"   or   > -200   (i.e. FROM_TIMESTAMP = now - 200s)
       < "YYYY-MM-DD HH:MM:SS"   or   < 200    (i.e. TO_TIMESTAMP = FROM_TIMESTAMP + 200s)
  - Example:
       python script.py > -200 < 10

On Windows 10, ANSI escape sequences are enabled to refresh the same lines.
"""

import os
import sys
import re
import time
from datetime import datetime, timedelta
from multiprocessing import Process, Manager, Queue

# Enable ANSI escape sequences on Windows 10.
if os.name == 'nt':
    import ctypes
    kernel32 = ctypes.windll.kernel32
    handle = kernel32.GetStdHandle(-11)  # STD_OUTPUT_HANDLE = -11
    mode = ctypes.c_uint32()
    kernel32.GetConsoleMode(handle, ctypes.byref(mode))
    mode.value |= 0x0004  # ENABLE_VIRTUAL_TERMINAL_PROCESSING
    kernel32.SetConsoleMode(handle, mode.value)

# -----------------------
# Command-line override parsing
# -----------------------
def parse_command_line_args():
    from_override = None
    to_override = None
    args = sys.argv[1:]
    for arg in args:
        arg = arg.strip()
        if arg.startswith('>'):
            from_override = arg[1:].strip().strip('"').strip("'")
        elif arg.startswith('<'):
            to_override = arg[1:].strip().strip('"').strip("'")
    return from_override, to_override

# -----------------------
# Config Parsing
# -----------------------
def parse_config():
    config = {}
    file_name_include = []
    file_include = []
    content_exclude = []
    current_key = None
    try:
        with open(__file__, 'r', encoding='utf-8') as f:
            lines = f.readlines()
    except Exception as e:
        print("Error reading config from script file:", e)
        sys.exit(1)
    config_section = []
    inside_config = False
    for line in lines:
        if line.strip() == "===========================":
            if not inside_config:
                inside_config = True
                continue
            else:
                break
        if inside_config:
            config_section.append(line.strip())
    for line in config_section:
        if not line:
            continue
        if '=' in line:
            key, value = line.split('=', 1)
            key = key.strip().upper()
            value = value.strip()
            if key == "INPUT_FOLDERS":
                folders = re.split(r'\s*[,\|]\s*', value)
                config["input_folders"] = [fld.strip() for fld in folders if fld.strip()]
            elif key == "OUTPUT_FOLDER":
                config["OUTPUT_FOLDER"] = value
            elif key in ("FROM_TIMESTAMP", "TO_TIMESTAMP", "CPU_CORES"):
                config[key] = value
            elif key == "FILE_NAME_INCLUDE_REGEXP_PATTERN":
                current_key = "file_name_include"
                config[current_key] = [value] if value else []
            elif key == "FILE_INCLUDE_REGEXP_PATTERN":
                current_key = "file_include"
                config[current_key] = [value] if value else []
            elif key == "CONTENT_EXCLUDE_REGEXP_PATTERN":
                current_key = "content_exclude"
                config[current_key] = [value] if value else []
            else:
                config[key.lower()] = value
        else:
            if current_key in ("file_name_include", "file_include", "content_exclude"):
                config[current_key].append(line)
    for key in ("input_folders", "file_name_include", "file_include", "content_exclude"):
        if key not in config:
            config[key] = []
    return config

# -----------------------
# Timestamp Extraction
# -----------------------
# Updated regex: match a digit, a space, then 16 hex digits, a colon, a digit, a space,
# then capture the timestamp in format "YYYY-MM-DD HH:MM:SS"
timestamp_pattern = re.compile(r"\d\s+[0-9A-Fa-f]{16}:\d\s+(\d{4}-\d{2}-\d{2}\s\d{2}:\d{2}:\d{2})")

def extract_timestamp(line):
    m = timestamp_pattern.search(line)
    if m:
        try:
            return datetime.strptime(m.group(1), "%Y-%m-%d %H:%M:%S")
        except Exception:
            return None
    return None

# -----------------------
# File-Level Filtering
# -----------------------
def file_level_filter(input_file, config):
    mtime = os.path.getmtime(input_file)
    file_dt = datetime.fromtimestamp(mtime)
    from_ts = None
    try:
        if config.get("FROM_TIMESTAMP", "").strip():
            from_ts = datetime.strptime(config["FROM_TIMESTAMP"].strip(), "%Y-%m-%d %H:%M:%S")
    except Exception:
        pass
    if from_ts and file_dt < from_ts:
        return False
    fn_patterns = config.get("file_name_include", [])
    if fn_patterns:
        if not any(re.search(p, os.path.basename(input_file), re.IGNORECASE) for p in fn_patterns):
            return False
    fi_patterns = config.get("file_include", [])
    if fi_patterns:
        found = False
        try:
            with open(input_file, 'r', encoding='utf-8') as f:
                for line in f:
                    if any(re.search(p, line) for p in fi_patterns):
                        found = True
                        break
        except Exception:
            return False
        if not found:
            return False
    return True

# -----------------------
# Content Filtering with Progress Callback for MP
# -----------------------
def process_file_content_mp(input_file, from_ts, to_ts, config, progress_callback):
    original_count = 0
    filtered_lines = []
    last_line_empty = False
    buffer = []
    started_output = False
    total_size = os.path.getsize(input_file)
    processed_bytes = 0
    content_exclude_patterns = [re.compile(p) for p in config.get("content_exclude", [])]
    with open(input_file, 'r', encoding='utf-8') as f:
        for line in f:
            original_count += 1
            processed_bytes += len(line.encode('utf-8'))
            percentage = (processed_bytes / total_size) * 100
            progress_callback(percentage)
            ts = extract_timestamp(line)
            if not started_output:
                buffer.append(line)
                if ts is not None:
                    if from_ts and ts < from_ts:
                        buffer = []
                        continue
                    for buf_line in buffer:
                        if any(p.search(buf_line) for p in content_exclude_patterns):
                            continue
                        if buf_line.strip() == "":
                            if last_line_empty:
                                continue
                            else:
                                filtered_lines.append(buf_line)
                                last_line_empty = True
                        else:
                            filtered_lines.append(buf_line)
                            last_line_empty = False
                    buffer = []
                    started_output = True
            else:
                if ts is not None and to_ts and ts > to_ts:
                    break
                if any(p.search(line) for p in content_exclude_patterns):
                    continue
                if line.strip() == "":
                    if last_line_empty:
                        continue
                    else:
                        filtered_lines.append(line)
                        last_line_empty = True
                else:
                    filtered_lines.append(line)
                    last_line_empty = False
    if not started_output and buffer:
        for buf_line in buffer:
            if any(p.search(buf_line) for p in content_exclude_patterns):
                continue
            if buf_line.strip() == "":
                if last_line_empty:
                    continue
                else:
                    filtered_lines.append(buf_line)
                    last_line_empty = True
            else:
                filtered_lines.append(buf_line)
                last_line_empty = False
    return original_count, filtered_lines

# -----------------------
# Compute Timestamps with Overrides
# -----------------------
def compute_overridden_timestamps(from_override, to_override, config):
    now = datetime.now()
    config_from = config.get("FROM_TIMESTAMP", "").strip()
    config_to = config.get("TO_TIMESTAMP", "").strip()
    from_ts = None
    to_ts = None
    try:
        if config_from and not re.fullmatch(r"-?\d+", config_from):
            from_ts = datetime.strptime(config_from, "%Y-%m-%d %H:%M:%S")
    except Exception:
        pass
    try:
        if config_to and not re.fullmatch(r"-?\d+", config_to):
            to_ts = datetime.strptime(config_to, "%Y-%m-%d %H:%M:%S")
    except Exception:
        pass
    from_override, to_override = parse_command_line_args()
    if from_override is not None:
        if re.fullmatch(r"-?\d+", from_override):
            from_ts = now + timedelta(seconds=int(from_override))
        else:
            try:
                from_ts = datetime.strptime(from_override, "%Y-%m-%d %H:%M:%S")
            except Exception:
                print("Error: Invalid FROM_TIMESTAMP override format.")
                sys.exit(1)
    if to_override is not None:
        if re.fullmatch(r"-?\d+", to_override):
            if from_ts is None:
                print("Error: Relative TO_TIMESTAMP override provided but FROM_TIMESTAMP is not set.")
                sys.exit(1)
            to_ts = from_ts + timedelta(seconds=int(to_override))
        else:
            try:
                to_ts = datetime.strptime(to_override, "%Y-%m-%d %H:%M:%S")
            except Exception:
                print("Error: Invalid TO_TIMESTAMP override format.")
                sys.exit(1)
    if to_ts is None:
        to_ts = now
    if from_ts is not None and to_ts < from_ts:
        print("Error: TO_TIMESTAMP is older than FROM_TIMESTAMP.")
        sys.exit(1)
    return from_ts, to_ts

# -----------------------
# Worker Process Function
# -----------------------
def worker_process(slot_id, file_queue, progress_dict, config, from_ts, to_ts):
    """
    Worker process: repeatedly gets a file from file_queue, processes it (both file- and content-level filtering),
    writes an output file (if content remains), and updates progress_dict for its slot.
    """
    while True:
        try:
            file = file_queue.get_nowait()
        except Exception:
            break
        # Update progress: print file name initially.
        progress_dict[slot_id] = f"Core {slot_id}: {os.path.basename(file)} - 0.00%"
        def progress_callback(p):
            progress_dict[slot_id] = f"Core {slot_id}: {os.path.basename(file)} - {p:.2f}%"
        orig_count, filtered_lines = process_file_content_mp(file, from_ts, to_ts, config, progress_callback)
        # Write output file if filtered_lines exist.
        if len(filtered_lines) > 0:
            out_folder = config.get("OUTPUT_FOLDER", ".")
            if not out_folder.endswith("\\") and not out_folder.endswith("/"):
                out_folder += os.sep
            os.makedirs(out_folder, exist_ok=True)
            out_file = os.path.join(out_folder, os.path.basename(file) + ".filtered.log")
            with open(out_file, 'w', encoding='utf-8') as f_out:
                f_out.writelines(filtered_lines)
            progress_dict[slot_id] = f"Core {slot_id}: {os.path.basename(file)}: {orig_count} => {len(filtered_lines)}"
        else:
            progress_dict[slot_id] = f"Core {slot_id}: {os.path.basename(file)}: {orig_count} => 0 (Filtered out)"
        time.sleep(0.2)

# -----------------------
# Main Function
# -----------------------
def main():
    start_time = time.time()
    config = parse_config()
    from_ts, to_ts = compute_overridden_timestamps(*parse_command_line_args(), config)
    config["FROM_TIMESTAMP"] = from_ts.strftime("%Y-%m-%d %H:%M:%S") if from_ts else ""
    config["TO_TIMESTAMP"] = to_ts.strftime("%Y-%m-%d %H:%M:%S") if to_ts else ""
    # Build file list from input folders.
    file_list = []
    for folder in config.get("INPUT_FOLDERS", []):
        if not os.path.isdir(folder):
            print(f"Warning: '{folder}' is not a valid directory. Skipping.")
            continue
        for fname in os.listdir(folder):
            full_path = os.path.join(folder, fname)
            if os.path.isfile(full_path) and file_level_filter(full_path, config):
                file_list.append(full_path)
    if not file_list:
        print("No files to process after file-level filtering. Exiting.")
        sys.exit(0)
    try:
        cpu_cores = int(config.get("CPU_CORES", "4"))
    except Exception:
        cpu_cores = 4
    manager = Manager()
    progress_dict = manager.dict()
    file_queue = Queue()
    for file in file_list:
        file_queue.put(file)
    for i in range(cpu_cores):
        progress_dict[i] = f"Core {i}: Idle"
    workers = []
    for i in range(cpu_cores):
        p = Process(target=worker_process, args=(i, file_queue, progress_dict, config, from_ts, to_ts))
        p.start()
        workers.append(p)
    # Manager loop: update progress display on same fixed lines every 2 seconds.
    # First, print blank lines to reserve space.
    for _ in range(cpu_cores):
        print("")
    while any(p.is_alive() for p in workers):
        sys.stdout.write(f"\033[{cpu_cores}A")
        for i in range(cpu_cores):
            status = progress_dict.get(i, f"Core {i}: Idle")
            sys.stdout.write("\033[K" + status + "\n")
        sys.stdout.flush()
        time.sleep(2)
    # Final update.
    sys.stdout.write(f"\033[{cpu_cores}A")
    for i in range(cpu_cores):
        status = progress_dict.get(i, f"Core {i}: Idle")
        sys.stdout.write("\033[K" + status + "\n")
    sys.stdout.flush()
    for p in workers:
        p.join()
    end_time = time.time()
    total_time = end_time - start_time
    print(f"\nTotal time consumed: {total_time:.2f} seconds")

if __name__ == '__main__':
    main()

r"""
===========================
INPUT_FOLDERS  = X:\ | W:\
OUTPUT_FOLDER  = D:\temp\log  

FROM_TIMESTAMP = 2025-02-28 15:35:00
TO_TIMESTAMP   = 2025-02-28 15:40:00
CPU_CORES      = 4

FILE_NAME_INCLUDE_REGEXP_PATTERN =
JMSOutboundder.*\.log
EAIAdapter.*\.XML

FILE_INCLUDE_REGEXP_PATTERN = 
^.*my critical line here.*$
^.*my 2nd critical line here.*$

CONTENT_EXCLUDE_REGEXP_PATTERN = 
^.*Begin: Fetch the line.*$
^.*\[KEY\].*$
===========================
"""






```
