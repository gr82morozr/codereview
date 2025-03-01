# codereview
code review

```
#!/usr/bin/env python3
"""
Version: 1.4.4

This script processes log files from multiple input folders in parallel using a worker pool.
It loads configuration from an external JSON file (specified as the first command‑line argument or defaults to "config.json").
Configuration keys (all uppercase) include:
  - INPUT_FOLDERS (array of folder paths)
  - OUTPUT_FOLDER (destination folder for filtered logs)
  - FROM_TIMESTAMP, TO_TIMESTAMP (absolute timestamp strings or empty; may be overridden via command‑line)
  - CPU_CORES (number of worker processes)
  - FILE_NAME_INCLUDE_REGEXP_PATTERN (array of regex strings; if empty, all filenames are accepted)
  - FILE_INCLUDE_REGEXP_PATTERN (array of regex strings; if empty, all files are accepted)
  - CONTENT_EXCLUDE_REGEXP_PATTERN (array of regex strings; lines matching these are removed)

File-level filtering (performed in each worker):
  - Skip a file if its last modified time is older than FROM_TIMESTAMP.
  - If FILE_NAME_INCLUDE_REGEXP_PATTERN is non-empty, the file name must match at least one pattern.
  (FILE_INCLUDE_REGEXP_PATTERN is now applied during content filtering.)

Content-level filtering (performed sequentially within each worker):
  - Discard all lines before the first line with a valid timestamp ≥ FROM_TIMESTAMP.
  - Stop processing when a line with a valid timestamp > TO_TIMESTAMP is encountered.
  - For each line (or buffered line), first apply empty-line filtering, then perform content exclusion (each line is checked exactly once).
  - Finally, if FILE_INCLUDE_REGEXP_PATTERN is provided, at least one line in the filtered content must match; otherwise, the file is skipped.

Before processing begins, OUTPUT_FOLDER is validated and recursively cleaned.

Worker processes get files from a shared queue and update a shared progress dictionary.
A manager process refreshes the same fixed set of display lines (one per core) every 2 seconds.

Command‑line overrides (using “>” and “<”) allow dynamic timestamp settings.

Usage:
    python script.py [config.json] [> FROM_OVERRIDE] [< TO_OVERRIDE]
Example:
    python script.py config.json > -200 < 10

On Windows 10, ANSI escape sequences are enabled so that the progress display updates on the same lines.
"""

import os
import sys
import re
import json
import time
from datetime import datetime, timedelta
from multiprocessing import Process, Manager, Queue
import shutil

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
# Validate and Clean OUTPUT_FOLDER
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
    # Recursively remove all files and subdirectories in OUTPUT_FOLDER.
    for item in os.listdir(output_folder):
        item_path = os.path.join(output_folder, item)
        try:
            if os.path.isfile(item_path) or os.path.islink(item_path):
                os.unlink(item_path)
            elif os.path.isdir(item_path):
                shutil.rmtree(item_path)
        except Exception as e:
            print(f"Error cleaning output folder '{output_folder}': {e}")
            sys.exit(1)

# -----------------------
# Command-line Override Parsing
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
    fn_patterns = config.get("FILE_NAME_INCLUDE_REGEXP_PATTERN", [])
    if fn_patterns:
        if not any(re.search(p, os.path.basename(input_file), re.IGNORECASE) for p in fn_patterns):
            return False
    return True

# -----------------------
# Content Filtering with Progress Callback for MP (and file-include check)
# -----------------------
def process_file_content_mp(input_file, from_ts, to_ts, config, progress_callback):
    """
    Processes one file's content as follows:
      1. Reads lines sequentially and counts original lines.
      2. Buffers lines until the first line with a valid timestamp ≥ from_ts is encountered.
      3. For each line (once output has started):
            - If a line has a valid timestamp > to_ts, stop processing further.
            - Then, if the line is empty (with consecutive empty check), add it.
            - Finally, perform the content_exclude check (once per line).
      4. After processing, if FILE_INCLUDE_REGEXP_PATTERN is provided (non-empty), at least one line in the filtered output must match; otherwise, the result is empty.
    Progress (percentage of file bytes processed) is reported via progress_callback.
    Returns a tuple: (original_line_count, filtered_lines as list)
    """
    original_count = 0
    filtered_lines = []
    last_line_empty = False
    buffer = []
    started_output = False
    total_size = os.path.getsize(input_file)
    processed_bytes = 0
    content_exclude_patterns = config.get("CONTENT_EXCLUDE_REGEXP_PATTERN", [])
    compiled_excludes = [re.compile(p) for p in content_exclude_patterns]
    with open(input_file, 'r', encoding='utf-8') as f:
        for line in f:
            original_count += 1
            processed_bytes += len(line.encode('utf-8'))
            progress_callback((processed_bytes / total_size) * 100)
            ts = extract_timestamp(line)
            # First, if line has a valid timestamp outside [from_ts, to_ts], skip it.
            if ts is not None:
                if (from_ts and ts < from_ts) or (to_ts and ts > to_ts):
                    continue
            # Buffer lines until output starts.
            if not started_output:
                buffer.append(line)
                if ts is not None and (from_ts is None or ts >= from_ts):
                    # Flush the buffer.
                    for buf_line in buffer:
                        # First, perform empty-line filtering.
                        if buf_line.strip() == "":
                            if last_line_empty:
                                continue
                            else:
                                candidate = buf_line
                                last_line_empty = True
                        else:
                            candidate = buf_line
                            last_line_empty = False
                        # Now perform content exclusion check once.
                        if any(p.search(candidate) for p in compiled_excludes):
                            continue
                        filtered_lines.append(candidate)
                    buffer = []
                    started_output = True
            else:
                if ts is not None and to_ts and ts > to_ts:
                    break
                # Process current line: first empty-line check.
                if line.strip() == "":
                    if last_line_empty:
                        continue
                    else:
                        candidate = line
                        last_line_empty = True
                else:
                    candidate = line
                    last_line_empty = False
                # Now perform content exclusion check.
                if any(p.search(candidate) for p in compiled_excludes):
                    continue
                filtered_lines.append(candidate)
    if not started_output and buffer:
        for buf_line in buffer:
            if buf_line.strip() == "":
                if last_line_empty:
                    continue
                else:
                    candidate = buf_line
                    last_line_empty = True
            else:
                candidate = buf_line
                last_line_empty = False
            if any(p.search(candidate) for p in compiled_excludes):
                continue
            filtered_lines.append(candidate)
    # File-Include Check: if FILE_INCLUDE_REGEXP_PATTERN is provided, ensure at least one line matches.
    fi_patterns = config.get("FILE_INCLUDE_REGEXP_PATTERN", [])
    if fi_patterns:
        if not any(any(re.search(p, line) for p in fi_patterns) for line in filtered_lines):
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
        # Worker does its own file-level filtering.
        if not file_level_filter(file, config):
            progress_dict[slot_id] = f"Core {slot_id}: {os.path.basename(file)} - Skipped (File-level)"
            continue
        # Update progress: show file name.
        progress_dict[slot_id] = f"Core {slot_id}: {os.path.basename(file)} - 0.00%"
        def progress_callback(p):
            progress_dict[slot_id] = f"Core {slot_id}: {os.path.basename(file)} - {p:.2f}%"
        orig_count, filtered_lines = process_file_content_mp(file, from_ts, to_ts, config, progress_callback)
        out_folder = config.get("OUTPUT_FOLDER", "").strip()
        if out_folder:
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
    # Build file list from INPUT_FOLDERS.
    file_list = []
    for folder in config.get("INPUT_FOLDERS", []):
        if not os.path.isdir(folder):
            print(f"Warning: '{folder}' is not a valid directory. Skipping.")
            continue
        for fname in os.listdir(folder):
            full_path = os.path.join(folder, fname)
            if os.path.isfile(full_path):
                file_list.append(full_path)
    if not file_list:
        print("No files found. Exiting.")
        sys.exit(0)
    manager = Manager()
    progress_dict = manager.dict()
    file_queue = Queue()
    for file in file_list:
        file_queue.put(file)
    try:
        cpu_cores = int(config.get("CPU_CORES", "4"))
    except Exception:
        cpu_cores = 4
    for i in range(cpu_cores):
        progress_dict[i] = f"Core {i}: Idle"
    workers = []
    for i in range(cpu_cores):
        p = Process(target=worker_process, args=(i, file_queue, progress_dict, config, from_ts, to_ts))
        p.start()
        workers.append(p)
    # Manager loop: refresh the same fixed lines every 2 seconds.
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
# End of File - External JSON config is used.
# -----------------------




```
