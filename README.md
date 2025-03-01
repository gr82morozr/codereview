# codereview
code review

```
#!/usr/bin/env python3
"""
Version: 1.4.2

This script processes log files from multiple input folders in parallel.
It loads its configuration from an external JSON file (passed as the first command‑line argument or defaults to "config.json").
Configuration keys (all uppercase) include:
  - INPUT_FOLDERS (array of folder paths)
  - OUTPUT_FOLDER (destination folder for filtered logs)
  - FROM_TIMESTAMP, TO_TIMESTAMP (absolute timestamp strings or empty; may be overridden via command‑line)
  - CPU_CORES (number of worker processes)
  - FILE_NAME_INCLUDE_REGEXP_PATTERN (array of regexes; if empty, all filenames are accepted)
  - FILE_INCLUDE_REGEXP_PATTERN (array of regexes; if empty, all files are accepted)
  - CONTENT_EXCLUDE_REGEXP_PATTERN (array of regexes; lines matching these are removed)

File‑level filtering (performed in each worker):
  - Skip file if its last‑modified time is older than FROM_TIMESTAMP.
  - If FILE_NAME_INCLUDE_REGEXP_PATTERN is non‑empty, the file name must match at least one pattern.

Content‑level filtering (performed in each worker):
  - Read the file and “filter out” lines whose valid timestamp is outside the time range.
  - After that, if FILE_INCLUDE_REGEXP_PATTERN is provided, require that at least one line in the filtered content matches one of these patterns; if not, skip the file.
  - Finally, remove lines matching any CONTENT_EXCLUDE_REGEXP_PATTERN.
  
A worker process (assigned to a CPU core) repeatedly gets a file from a shared queue and processes it:
  - It updates a shared progress dictionary with its status (file name and progress %) while processing.
  - When finished with a file, if filtered content remains, an output file is written to OUTPUT_FOLDER.
  
A manager process refreshes the same fixed set of display lines (one per core) every 2 seconds.

Before processing begins, OUTPUT_FOLDER is validated:
  - It must be non‑empty.
  - It must be an accessible, writable directory.
  - It must not appear to be a file (unless its extension is one of: .log, .xml, .txt).

Command‑line overrides (using arguments starting with “>” for FROM_TIMESTAMP and “<” for TO_TIMESTAMP) let you dynamically set these values.
For example:
    python script.py config.json > -200 < 10
sets FROM_TIMESTAMP to (now – 200s) and TO_TIMESTAMP to (FROM_TIMESTAMP + 10s).

Usage:
    python script.py [config.json] [> FROM_OVERRIDE] [< TO_OVERRIDE]
"""

import os
import sys
import re
import json
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
# Load External JSON Configuration
# -----------------------
def load_config():
    config_file = "config.json"
    if len(sys.argv) > 1 and sys.argv[1].lower().endswith(".json"):
        config_file = sys.argv[1]
        sys.argv.pop(1)
    try:
        with open(config_file, 'r', encoding='utf-8') as f:
            config = json.load(f)
    except Exception as e:
        print(f"Error loading config file '{config_file}':", e)
        sys.exit(1)
    normalized = {}
    for k, v in config.items():
        normalized[k.upper()] = v
    if "OUTPUT_FOLDERS" in normalized:
        normalized["OUTPUT_FOLDER"] = normalized["OUTPUT_FOLDERS"]
        del normalized["OUTPUT_FOLDERS"]
    return normalized

# -----------------------
# Validate OUTPUT_FOLDER
# -----------------------
def check_output_folder(config):
    output_folder = config.get("OUTPUT_FOLDER", "").strip()
    if not output_folder:
        print("Error: OUTPUT_FOLDER is empty in config.")
        sys.exit(1)
    base, ext = os.path.splitext(output_folder)
    allowed = {".log", ".xml", ".txt"}
    if ext and ext.lower() not in allowed:
        print(f"Error: OUTPUT_FOLDER '{output_folder}' appears to be a file (extension '{ext}' not allowed).")
        sys.exit(1)
    if not os.path.exists(output_folder):
        try:
            os.makedirs(output_folder, exist_ok=True)
        except Exception as e:
            print(f"Error: OUTPUT_FOLDER '{output_folder}' is not accessible or writable: {e}")
            sys.exit(1)
    else:
        if not os.path.isdir(output_folder):
            print(f"Error: OUTPUT_FOLDER '{output_folder}' is not a directory.")
            sys.exit(1)
        if not os.access(output_folder, os.W_OK):
            print(f"Error: OUTPUT_FOLDER '{output_folder}' is not writable.")
            sys.exit(1)

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
# Timestamp Extraction (No Globals)
# -----------------------
def extract_timestamp(line):
    # Use a function attribute to compile the regex once.
    if not hasattr(extract_timestamp, "_pattern"):
        extract_timestamp._pattern = re.compile(r"\d\s+[0-9A-Fa-f]{16}:\d\s+(\d{4}-\d{2}-\d{2}\s\d{2}:\d{2}:\d{2})")
    m = extract_timestamp._pattern.search(line)
    if m:
        try:
            return datetime.strptime(m.group(1), "%Y-%m-%d %H:%M:%S")
        except Exception:
            return None
    return None

# -----------------------
# File-Level Filtering (Worker-side)
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
    # Check only file name pattern here.
    fn_patterns = config.get("FILE_NAME_INCLUDE_REGEXP_PATTERN", [])
    if fn_patterns:
        if not any(re.search(p, os.path.basename(input_file), re.IGNORECASE) for p in fn_patterns):
            return False
    return True

# -----------------------
# Content Filtering with Progress Callback for MP (and file-include check)
# -----------------------
def process_file_content_mp(input_file, from_ts, to_ts, config, progress_callback):
    original_count = 0
    filtered_lines = []
    last_line_empty = False
    buffer = []
    started_output = False
    total_size = os.path.getsize(input_file)
    processed_bytes = 0
    # Compile content_exclude patterns locally.
    content_exclude_patterns = config.get("CONTENT_EXCLUDE_REGEXP_PATTERN", [])
    compiled_excludes = [re.compile(p) for p in content_exclude_patterns]
    # Collect all filtered lines.
    with open(input_file, 'r', encoding='utf-8') as f:
        for line in f:
            original_count += 1
            processed_bytes += len(line.encode('utf-8'))
            progress_callback((processed_bytes / total_size) * 100)
            ts = extract_timestamp(line)
            # 3.1 Filter out lines with valid timestamps outside [from_ts, to_ts].
            if ts is not None:
                if (from_ts and ts < from_ts) or (to_ts and ts > to_ts):
                    continue
            # Buffer lines until we see a valid timestamp (if not started yet).
            if not started_output:
                buffer.append(line)
                if ts is not None:
                    # Now flush buffer.
                    for buf_line in buffer:
                        if any(p.search(buf_line) for p in compiled_excludes):
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
                # Already started: if a line's timestamp > to_ts, stop.
                if ts is not None and to_ts and ts > to_ts:
                    break
                if any(p.search(line) for p in compiled_excludes):
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
            if any(p.search(buf_line) for p in compiled_excludes):
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
    # 3.2 File-Include Check: if FILE_INCLUDE_REGEXP_PATTERN is provided, then at least one line must match.
    fi_patterns = config.get("FILE_INCLUDE_REGEXP_PATTERN", [])
    if fi_patterns:
        if not any(any(re.search(p, line) for p in fi_patterns) for line in filtered_lines):
            # No matching line found; skip file.
            return original_count, []
    return original_count, filtered_lines

# -----------------------
# Worker Process Function
# -----------------------
def worker_process(slot_id, file_queue, progress_dict, config, from_ts, to_ts):
    while True:
        try:
            file = file_queue.get_nowait()
        except Exception:
            break
        # Do file-level filtering in worker.
        if not file_level_filter(file, config):
            progress_dict[slot_id] = f"Core {slot_id}: {os.path.basename(file)} - Skipped (File-level)"
            continue
        # Update progress with file name.
        progress_dict[slot_id] = f"Core {slot_id}: {os.path.basename(file)} - 0.00%"
        def progress_callback(p):
            progress_dict[slot_id] = f"Core {slot_id}: {os.path.basename(file)} - {p:.2f}%"
        orig_count, filtered_lines = process_file_content_mp(file, from_ts, to_ts, config, progress_callback)
        out_folder = config.get("OUTPUT_FOLDER", "").strip()
        if not out_folder:
            progress_dict[slot_id] = f"Core {slot_id}: {os.path.basename(file)}: {orig_count} => 0 (No OUTPUT_FOLDER)"
        else:
            if not out_folder.endswith("\\") and not out_folder.endswith("/"):
                out_folder += os.sep
            if len(filtered_lines) > 0:
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
    config = load_config()
    from_ts, to_ts = compute_overridden_timestamps(*parse_command_line_args(), config)
    config["FROM_TIMESTAMP"] = from_ts.strftime("%Y-%m-%d %H:%M:%S") if from_ts else ""
    config["TO_TIMESTAMP"] = to_ts.strftime("%Y-%m-%d %H:%M:%S") if to_ts else ""
    check_output_folder(config)
    # Build file list from INPUT_FOLDERS (file-level filtering except file_include).
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
    # Manager loop: refresh the same fixed number of lines every 2 seconds.
    for _ in range(cpu_cores):
        print("")
    while any(p.is_alive() for p in workers):
        sys.stdout.write(f"\033[{cpu_cores}A")
        for i in range(cpu_cores):
            status = progress_dict.get(i, f"Core {i}: Idle")
            sys.stdout.write("\033[K" + status + "\n")
        sys.stdout.flush()
        time.sleep(2)
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

# -----------------------
# Load External JSON Configuration
# -----------------------
def load_config():
    config_file = "config.json"
    if len(sys.argv) > 1 and sys.argv[1].lower().endswith(".json"):
        config_file = sys.argv[1]
        sys.argv.pop(1)
    try:
        with open(config_file, 'r', encoding='utf-8') as f:
            config = json.load(f)
    except Exception as e:
        print(f"Error loading config file '{config_file}':", e)
        sys.exit(1)
    normalized = {}
    for k, v in config.items():
        normalized[k.upper()] = v
    if "OUTPUT_FOLDERS" in normalized:
        normalized["OUTPUT_FOLDER"] = normalized["OUTPUT_FOLDERS"]
        del normalized["OUTPUT_FOLDERS"]
    return normalized

if __name__ == '__main__':
    main()



```
