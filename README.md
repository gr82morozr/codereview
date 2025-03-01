# codereview
code review

```
#!/usr/bin/env python3
"""
Version: 1.4.5

This script processes log files from multiple input folders in parallel using a worker pool.
Configuration is loaded from an external JSON file (the first command-line argument or defaults to "config.json").
All configuration keys are uppercase, for example:

{
  "INPUT_FOLDERS": ["X:\\", "W:\\"],
  "OUTPUT_FOLDER": "D:\\temp\\log",
  "FROM_TIMESTAMP": "",
  "TO_TIMESTAMP": "",
  "CPU_CORES": "4",
  "FILE_NAME_INCLUDE_REGEXP_PATTERN": [...],
  "FILE_INCLUDE_REGEXP_PATTERN": [...],
  "CONTENT_EXCLUDE_REGEXP_PATTERN": [...]
}

File-level filtering (in each worker):
  - Skips a file if its last modified time is older than FROM_TIMESTAMP.
  - If FILE_NAME_INCLUDE_REGEXP_PATTERN is non-empty, the file name must match at least one pattern.

Time range + content filtering (in each worker, single pass + in-memory):
  1. Read file line-by-line, skipping lines whose valid timestamp is outside [FROM_TIMESTAMP, TO_TIMESTAMP].
  2. Collect in-range lines in memory.
  3. Remove lines matching CONTENT_EXCLUDE_REGEXP_PATTERN (only once per line).
  4. If FILE_INCLUDE_REGEXP_PATTERN is non-empty, at least one line must match; otherwise, skip.
  5. If lines remain, write them to OUTPUT_FOLDER (as "<file>.filtered.log").

Before processing, OUTPUT_FOLDER is validated and recursively cleaned.
Workers update a shared progress dictionary (one slot per CPU core) as they read lines,
and the manager process refreshes a fixed display every 2 seconds.

Usage:
    python script.py [config.json] [> FROM_OVERRIDE] [< TO_OVERRIDE]

Example:
    python script.py config.json > -200 < 10
sets FROM_TIMESTAMP to (now - 200s) and TO_TIMESTAMP to (FROM_TIMESTAMP + 10s).

On Windows 10, ANSI escape sequences are enabled so that the progress display updates on fixed lines.
"""

import os
import sys
import re
import json
import time
import shutil
from datetime import datetime, timedelta
from multiprocessing import Process, Manager, Queue

# Enable ANSI escape sequences on Windows 10, if possible.
if os.name == 'nt':
    import ctypes
    kernel32 = ctypes.windll.kernel32
    handle = kernel32.GetStdHandle(-11)  # STD_OUTPUT_HANDLE = -11
    mode = ctypes.c_uint32()
    kernel32.GetConsoleMode(handle, ctypes.byref(mode))
    mode.value |= 0x0004  # ENABLE_VIRTUAL_TERMINAL_PROCESSING
    kernel32.SetConsoleMode(handle, mode.value)

def load_config():
    """
    Loads configuration from an external JSON file.
    If the first argument ends with ".json", that is used as the config file name;
    otherwise "config.json" is used. Keys are normalized to uppercase.
    """
    config_file = "config.json"
    if len(sys.argv) > 1 and sys.argv[1].lower().endswith(".json"):
        config_file = sys.argv[1]
        sys.argv.pop(1)
    try:
        with open(config_file, 'r', encoding='utf-8') as f:
            raw = json.load(f)
    except Exception as e:
        print(f"Error loading config file '{config_file}':", e)
        sys.exit(1)
    normalized = {}
    for k, v in raw.items():
        normalized[k.upper()] = v
    if "OUTPUT_FOLDERS" in normalized:
        normalized["OUTPUT_FOLDER"] = normalized["OUTPUT_FOLDERS"]
        del normalized["OUTPUT_FOLDERS"]
    return normalized

def check_output_folder(config):
    """
    Validates OUTPUT_FOLDER from config, ensures it's non-empty,
    is a directory, is writable, not a disallowed extension, then recursively cleans it.
    """
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
    # Clean it recursively
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

def parse_command_line_args():
    from_override = None
    to_override = None
    args = sys.argv[1:]
    for arg in args:
        if arg.startswith('>'):
            from_override = arg[1:].strip().strip('"').strip("'")
        elif arg.startswith('<'):
            to_override = arg[1:].strip().strip('"').strip("'")
    return from_override, to_override

def compute_overridden_timestamps(from_override, to_override, config):
    """
    Merges command-line overrides with config values for FROM_TIMESTAMP and TO_TIMESTAMP,
    supporting relative offsets (like '-200') or absolute timestamps.
    """
    now = datetime.now()
    config_from = config.get("FROM_TIMESTAMP", "").strip()
    config_to = config.get("TO_TIMESTAMP", "").strip()
    from_ts = None
    to_ts = None

    # Parse config_from if it's not an integer offset
    try:
        if config_from and not re.fullmatch(r"-?\d+", config_from):
            from_ts = datetime.strptime(config_from, "%Y-%m-%d %H:%M:%S")
    except Exception:
        pass
    # Parse config_to if it's not an integer offset
    try:
        if config_to and not re.fullmatch(r"-?\d+", config_to):
            to_ts = datetime.strptime(config_to, "%Y-%m-%d %H:%M:%S")
    except Exception:
        pass

    # Now parse command line overrides
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

    # If TO is still None, default to now
    if to_ts is None:
        to_ts = now
    # Validate
    if from_ts is not None and to_ts < from_ts:
        print("Error: TO_TIMESTAMP is older than FROM_TIMESTAMP.")
        sys.exit(1)

    return from_ts, to_ts

def extract_timestamp(line):
    """
    Extracts a timestamp from a line using a function attribute regex (no global).
    """
    if not hasattr(extract_timestamp, "_pattern"):
        extract_timestamp._pattern = re.compile(r"\d\s+[0-9A-Fa-f]{16}:\d\s+(\d{4}-\d{2}-\d{2}\s\d{2}:\d{2}:\d{2})")
    m = extract_timestamp._pattern.search(line)
    if m:
        try:
            return datetime.strptime(m.group(1), "%Y-%m-%d %H:%M:%S")
        except Exception:
            return None
    return None

def file_level_filter(input_file, config):
    """
    Worker-side file-level filtering:
      - Skip if last modified < FROM_TIMESTAMP
      - If FILE_NAME_INCLUDE_REGEXP_PATTERN is non-empty, file name must match at least one pattern.
    """
    # Last-modified time check
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

    # File name check
    fn_patterns = config.get("FILE_NAME_INCLUDE_REGEXP_PATTERN", [])
    if fn_patterns:
        if not any(re.search(p, os.path.basename(input_file), re.IGNORECASE) for p in fn_patterns):
            return False

    return True

def process_file_content_mp(input_file, from_ts, to_ts, config, progress_callback):
    """
    Single-pass time range filtering:
      - read lines once, skip lines whose timestamp < from_ts or > to_ts
      - store in memory
    Then in memory:
      - remove lines matching CONTENT_EXCLUDE_REGEXP_PATTERN
      - if FILE_INCLUDE_REGEXP_PATTERN is non-empty, ensure at least one line matches
    Returns (original_line_count, final_filtered_lines)
    """
    original_count = 0
    in_range_lines = []
    total_size = os.path.getsize(input_file)
    processed_bytes = 0

    with open(input_file, 'r', encoding='utf-8') as f:
        for line in f:
            original_count += 1
            processed_bytes += len(line.encode('utf-8'))
            progress_callback((processed_bytes / total_size) * 100)

            # Extract timestamp
            ts = extract_timestamp(line)
            # If line has a valid timestamp, check if outside [from_ts, to_ts]
            if ts is not None:
                if (from_ts and ts < from_ts) or (to_ts and ts > to_ts):
                    continue
            # Keep lines in memory
            in_range_lines.append(line)

    # Now in memory we do the exclude check, empty-line check, etc.
    # We'll remove consecutive empty lines in a single pass, then remove lines matching exclude patterns.
    content_exclude = config.get("CONTENT_EXCLUDE_REGEXP_PATTERN", [])
    compiled_excludes = [re.compile(p) for p in content_exclude]
    # We'll produce final_filtered_lines
    final_filtered_lines = []
    last_line_empty = False
    for line in in_range_lines:
        if line.strip() == "":
            if last_line_empty:
                # skip consecutive empty
                continue
            else:
                candidate = line
                last_line_empty = True
        else:
            candidate = line
            last_line_empty = False

        # Now apply exclude patterns once
        if any(p.search(candidate) for p in compiled_excludes):
            continue

        final_filtered_lines.append(candidate)

    # Finally, if FILE_INCLUDE_REGEXP_PATTERN is non-empty, at least one line must match
    fi_patterns = config.get("FILE_INCLUDE_REGEXP_PATTERN", [])
    if fi_patterns:
        # If none of final_filtered_lines matches any pattern, skip file
        if not any(any(re.search(p, line) for p in fi_patterns) for line in final_filtered_lines):
            return (original_count, [])

    return (original_count, final_filtered_lines)

def worker_process(slot_id, file_queue, progress_dict, config, from_ts, to_ts):
    """
    Worker process function:
      - Repeatedly gets a file from file_queue
      - file_level_filter => skip if fails
      - time range + content filter => single pass in memory
      - if lines remain => write output
      - update progress_dict for manager display
    """
    while True:
        try:
            file = file_queue.get_nowait()
        except Exception:
            break

        # File-level filtering
        if not file_level_filter(file, config):
            progress_dict[slot_id] = f"Core {slot_id}: {os.path.basename(file)} - Skipped (File-level)"
            continue

        # Show initial 0% progress
        progress_dict[slot_id] = f"Core {slot_id}: {os.path.basename(file)} - 0.00%"

        def progress_callback(percentage):
            progress_dict[slot_id] = f"Core {slot_id}: {os.path.basename(file)} - {percentage:.2f}%"

        orig_count, filtered_lines = process_file_content_mp(file, from_ts, to_ts, config, progress_callback)

        out_folder = config.get("OUTPUT_FOLDER", "").strip()
        if out_folder and not out_folder.endswith(("\\", "/")):
            out_folder += os.sep

        if len(filtered_lines) > 0:
            # Write output
            os.makedirs(out_folder, exist_ok=True)
            out_file = os.path.join(out_folder, os.path.basename(file) + ".filtered.log")
            with open(out_file, 'w', encoding='utf-8') as f_out:
                f_out.writelines(filtered_lines)
            progress_dict[slot_id] = f"Core {slot_id}: {os.path.basename(file)}: {orig_count} => {len(filtered_lines)}"
        else:
            progress_dict[slot_id] = f"Core {slot_id}: {os.path.basename(file)}: {orig_count} => 0 (Filtered out)"

        time.sleep(0.2)

def main():
    start_time = time.time()

    # Load config from JSON
    config = load_config()

    # Merge command-line overrides for FROM_TIMESTAMP, TO_TIMESTAMP
    from_ts, to_ts = compute_overridden_timestamps(None, None, config)
    config["FROM_TIMESTAMP"] = from_ts.strftime("%Y-%m-%d %H:%M:%S") if from_ts else ""
    config["TO_TIMESTAMP"] = to_ts.strftime("%Y-%m-%d %H:%M:%S") if to_ts else ""

    # Validate & clean output folder
    check_output_folder(config)

    # Build file list from config["INPUT_FOLDERS"]
    file_list = []
    input_folders = config.get("INPUT_FOLDERS", [])
    for folder in input_folders:
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

    # Start manager, progress dict, queue
    manager = Manager()
    progress_dict = manager.dict()
    file_queue = Queue()
    for file in file_list:
        file_queue.put(file)

    # CPU cores
    try:
        cpu_cores = int(config.get("CPU_CORES", "4"))
    except Exception:
        cpu_cores = 4

    # Initialize progress
    for i in range(cpu_cores):
        progress_dict[i] = f"Core {i}: Idle"

    # Spawn workers
    workers = []
    for i in range(cpu_cores):
        p = Process(target=worker_process, args=(i, file_queue, progress_dict, config, from_ts, to_ts))
        p.start()
        workers.append(p)

    # Manager loop: refresh the same fixed lines (cpu_cores) every 2s
    for _ in range(cpu_cores):
        print("")
    while any(p.is_alive() for p in workers):
        sys.stdout.write(f"\033[{cpu_cores}A")  # Move cursor up
        for i in range(cpu_cores):
            status = progress_dict.get(i, f"Core {i}: Idle")
            sys.stdout.write("\033[K" + status + "\n")
        sys.stdout.flush()
        time.sleep(2)

    # Final refresh
    sys.stdout.write(f"\033[{cpu_cores}A")
    for i in range(cpu_cores):
        status = progress_dict.get(i, f"Core {i}: Idle")
        sys.stdout.write("\033[K" + status + "\n")
    sys.stdout.flush()

    # Join workers
    for p in workers:
        p.join()

    end_time = time.time()
    print(f"\nTotal time consumed: {end_time - start_time:.2f} seconds")

if __name__ == '__main__':
    main()




```
