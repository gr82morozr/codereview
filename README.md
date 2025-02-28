# codereview
code review

```
#!/usr/bin/env python3
"""
Version: 1.3.0

This script processes log files from multiple input folders and applies both file‑level
and content‑level filtering according to configuration parameters and optional command‑line
overrides.

Configuration parameters (defined in the config block at the end) include:
  INPUT_FOLDERS             = X:\ | W:\
  OUTPUT_FOLDER             = D:\temp\log

  FROM_TIMESTAMP            = 2025-02-28 15:35:00
  TO_TIMESTAMP              = 2025-02-28 15:40:00

  FILE_NAME_INCLUDE_REGEXP_PATTERN =
    JMSOutboundder.*\.log
    EAIAdapter.*\.XML

  FILE_INCLUDE_REGEXP_PATTERN =
    ^.*my critical line here.*$
    ^.*my 2nd critical line here.*$

  CONTENT_EXCLUDE_REGEXP_PATTERN =
    ^.*Begin: Fetch the line.*$
    ^.*\[KEY\].*$
    
Usage:
  - When no command‑line parameters are supplied, the script uses the config values.
  - Command‑line overrides:
      > "YYYY-MM-DD HH:MM:SS"   or   > -200    (meaning FROM_TIMESTAMP = now - 200s)
      < "YYYY-MM-DD HH:MM:SS"   or   < 200     (meaning TO_TIMESTAMP = FROM_TIMESTAMP + 200s)
  - Example:
      python script.py > -200 < 10
        sets FROM_TIMESTAMP to (now - 200s) and TO_TIMESTAMP to (FROM_TIMESTAMP + 10s).

File‑level filtering:
  1. Skip file if its last modified time is older than FROM_TIMESTAMP.
  2. Skip file if its name does not match any pattern from FILE_NAME_INCLUDE_REGEXP_PATTERN.
  3. Skip file if it does not contain at least one line matching FILE_INCLUDE_REGEXP_PATTERN.

Content‑level filtering:
  - Discard all lines before the first line with a valid timestamp ≥ FROM_TIMESTAMP.
  - Stop processing lines once a line with a valid timestamp > TO_TIMESTAMP is encountered.
  - Remove lines matching any CONTENT_EXCLUDE_REGEXP_PATTERN.

After processing each file, the script prints the file name and the original vs. filtered line counts.
If no content remains after filtering, no output file is created.
Finally, the script prints the total processing time.
"""

import os
import sys
import re
import time
from datetime import datetime, timedelta

# -----------------------
# Command-line override parsing
# -----------------------
def parse_command_line_args():
    """
    Parses command-line arguments to override FROM_TIMESTAMP and TO_TIMESTAMP.
    Recognizes arguments starting with '>' for FROM_TIMESTAMP and '<' for TO_TIMESTAMP.
    Returns a tuple (from_override, to_override) as strings (or None if not provided).
    """
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
    """
    Reads configuration from this script file between "===========================" lines.
    Supports keys:
      INPUT_FOLDERS, OUTPUT_FOLDER, FROM_TIMESTAMP, TO_TIMESTAMP,
      FILE_NAME_INCLUDE_REGEXP_PATTERN, FILE_INCLUDE_REGEXP_PATTERN,
      CONTENT_EXCLUDE_REGEXP_PATTERN.
    Returns a dictionary.
    """
    config = {}
    # Lists for multi-line values:
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
                # Split by '|' or comma
                folders = re.split(r'\s*[,\|]\s*', value)
                config["input_folders"] = [fld.strip() for fld in folders if fld.strip()]
            elif key == "OUTPUT_FOLDER":
                config["output_folder"] = value
            elif key in ("FROM_TIMESTAMP", "TO_TIMESTAMP"):
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
            # Continuation line for multi-line keys
            if current_key in ("file_name_include", "file_include", "content_exclude"):
                config[current_key].append(line)
    for key in ("input_folders", "file_name_include", "file_include", "content_exclude"):
        if key not in config:
            config[key] = []
    return config

# -----------------------
# Timestamp Extraction
# -----------------------
# Using version 1.0.3b's regex:
timestamp_pattern = re.compile(r"^\S+\s+\S+\s+\d\s+[0-9A-Fa-f]{16}:\d\s+(\d{4}-\d{2}-\d{2}\s\d{2}:\d{2}:\d{2})")

def extract_timestamp(line):
    """
    Returns a datetime object if the line has a valid timestamp, else None.
    """
    m = timestamp_pattern.search(line)
    if m:
        try:
            return datetime.strptime(m.group(1), "%Y-%m-%d %H:%M:%S")
        except Exception:
            return None
    return None

# -----------------------
# Content Filtering
# -----------------------
def process_file_content(input_file, from_ts, to_ts, config):
    """
    Processes one file's content.
      - Reads all lines and counts original lines.
      - Buffers lines until a valid timestamp is found.
      - Discards all lines before the first line with a valid timestamp >= from_ts.
      - Stops processing when a line with a timestamp > to_ts is encountered.
      - Removes lines matching any CONTENT_EXCLUDE_REGEXP_PATTERN.
    Returns (original_line_count, filtered_lines as list).
    """
    original_count = 0
    filtered_lines = []
    last_line_empty = False
    buffer = []
    started_output = False

    content_exclude_patterns = [re.compile(p) for p in config.get("content_exclude", [])]

    with open(input_file, 'r', encoding='utf-8') as f:
        for line in f:
            original_count += 1
            ts = extract_timestamp(line)
            if not started_output:
                buffer.append(line)
                if ts is not None:
                    if from_ts and ts < from_ts:
                        buffer = []
                        continue
                    # Flush the buffer
                    for buf_line in buffer:
                        skip = any(p.search(buf_line) for p in content_exclude_patterns)
                        if skip:
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
                skip = any(p.search(line) for p in content_exclude_patterns)
                if skip:
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
            skip = any(p.search(buf_line) for p in content_exclude_patterns)
            if skip:
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
# File-Level Filtering
# -----------------------
def file_level_filter(input_file, config):
    """
    Performs file-level filtering:
      - Checks file's last modified time.
      - Checks if file name matches any pattern from FILE_NAME_INCLUDE_REGEXP_PATTERN (case-insensitive).
      - Checks if file contains at least one line matching FILE_INCLUDE_REGEXP_PATTERN.
    Returns True if the file should be processed; otherwise False.
    """
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
# Compute Timestamps with Overrides
# -----------------------
def compute_overridden_timestamps(from_override, to_override, config):
    """
    Computes final FROM and TO timestamps based on config and command-line overrides.
    Override values may be absolute timestamps or relative (numeric, in seconds).
    If TO is empty, defaults to now.
    Returns (from_ts, to_ts) as datetime objects.
    Exits if TO < FROM.
    """
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
                print("Error: Invalid FROM_TIMESTAMP override.")
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
                print("Error: Invalid TO_TIMESTAMP override.")
                sys.exit(1)
    if to_ts is None:
        to_ts = now
    if from_ts is not None and to_ts < from_ts:
        print("Error: TO_TIMESTAMP is older than FROM_TIMESTAMP.")
        sys.exit(1)
    return from_ts, to_ts

# -----------------------
# Main Function
# -----------------------
def main():
    start_time = time.time()
    config = parse_config()
    # Compute final FROM and TO timestamps with command-line overrides.
    from_ts, to_ts = compute_overridden_timestamps(*parse_command_line_args(), config)
    config["FROM_TIMESTAMP"] = from_ts.strftime("%Y-%m-%d %H:%M:%S") if from_ts else ""
    config["TO_TIMESTAMP"] = to_ts.strftime("%Y-%m-%d %H:%M:%S") if to_ts else ""
    
    input_folders = config.get("input_folders", [])
    if not input_folders:
        print("No INPUT_FOLDERS specified in config. Exiting.")
        sys.exit(0)
    
    for folder in input_folders:
        if not os.path.isdir(folder):
            print(f"Warning: '{folder}' is not a valid directory. Skipping.")
            continue
        for fname in os.listdir(folder):
            full_path = os.path.join(folder, fname)
            if os.path.isfile(full_path):
                if not file_level_filter(full_path, config):
                    continue
                # Check file modified time (already done in file_level_filter)
                print(f"\nProcessing file: {full_path}")
                orig_count, filtered_lines = process_file_content(full_path, from_ts, to_ts, config)
                if len(filtered_lines) == 0:
                    print("  File content filtered out completely; no output file created.")
                else:
                    out_folder = config.get("output_folder", ".")
                    if not out_folder.endswith("\\") and not out_folder.endswith("/"):
                        out_folder += os.sep
                    os.makedirs(out_folder, exist_ok=True)
                    out_file = os.path.join(out_folder, os.path.basename(full_path) + ".filtered.log")
                    with open(out_file, 'w', encoding='utf-8') as f_out:
                        f_out.writelines(filtered_lines)
                    print(f"  {os.path.basename(full_path)}: {orig_count} => {len(filtered_lines)}")
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
