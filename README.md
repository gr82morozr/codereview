# codereview
code review

```
#!/usr/bin/env python3
"""
Version: 1.0.4

This script filters log files by removing lines based on two criteria:
  1. Removal regex patterns (specified in the config)
  2. Timestamp filtering – only lines whose timestamp (extracted via a regex)
     fall within a configured FROM_TIMESTAMP and TO_TIMESTAMP range are kept.

It is a sequential (single-process) script, building on Version 1.0.3b, with the following enhancements:
  - Multiple input folders (specified in 'input_folders' in the config).
  - For each file in each folder:
      * Skip if the file's last modified time is older than FROM_TIMESTAMP.
      * Print the file name being processed.
      * Apply the same "buffer until valid timestamp found" logic as in 1.0.3b.
      * If after filtering, the output is empty, do not create an output file.
      * Finally, print the original line count and the filtered line count for each file.

Usage:
  file_log.py

Configuration:
  The config is embedded at the bottom of this script (between "===========================" lines).
  You can specify:
    input_folders = X:\temp, Y:\logs
    output_folder = D:\output
    FROM_TIMESTAMP = 2025-02-27 15:35:30
    TO_TIMESTAMP   = 2025-02-27 22:00:00
    regexp_pattern =
    ^.*Begin: Fetch for sql Cursor.*$
    ^.*\[KEY\].*$
    ...
"""

import os
import sys
import re
from datetime import datetime

def parse_config():
    """
    Reads this script file (__file__) to extract configuration settings.
    The configuration is placed at the bottom of the script between two lines that exactly match
    "===========================".

    In addition to the old parameters, you can specify:
      input_folders = X:\temp, Y:\logs

    Example config lines:
      input_folders = X:\temp, Y:\logs
      output_folder = D:\output
      FROM_TIMESTAMP = 2025-02-27 15:35:30
      TO_TIMESTAMP = 2025-02-27 23:59:59
      regexp_pattern =
      ^.*Begin: Fetch for sql Cursor.*$
      ^.*\[KEY\].*$
      ...

    Returns:
      A dictionary with:
        - "input_folders": list of folders
        - "output_folder": folder path for filtered logs
        - "FROM_TIMESTAMP": str or empty
        - "TO_TIMESTAMP": str or empty
        - "patterns": list of regex pattern strings
        - possibly other numeric values like "chunk_size", "num_cpus" if you choose to keep them
    """
    config = {}
    patterns = []
    try:
        with open(__file__, 'r', encoding='utf-8') as f:
            lines = f.readlines()
    except Exception as e:
        print("Error reading script file for configuration:", e)
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

    in_patterns = False  # indicates that subsequent lines are regex patterns
    for line in config_section:
        if not line:
            continue
        if not in_patterns:
            if '=' in line:
                key, value = line.split('=', 1)
                key = key.strip()
                value = value.strip()
                if key == "regexp_pattern":
                    in_patterns = True  # Switch to reading regex patterns.
                elif key == "input_folders":
                    # Parse multiple folders, split by comma
                    raw_folders = value.split(',')
                    folder_list = [fld.strip() for fld in raw_folders if fld.strip()]
                    config["input_folders"] = folder_list
                else:
                    # Attempt to parse int
                    try:
                        config[key] = int(value)
                    except ValueError:
                        config[key] = value
            else:
                continue
        else:
            patterns.append(line)
    config["patterns"] = patterns
    # Ensure input_folders is at least an empty list if not provided
    if "input_folders" not in config:
        config["input_folders"] = []
    return config

def update_progress(processed, total):
    """
    Prints the progress percentage on the same line based on bytes processed.
    """
    percent = (processed / total) * 100
    sys.stdout.write(f"\rProgress: {percent:.2f}%")
    sys.stdout.flush()

# Timestamp extraction regex from version 1.0.3b
# Based on sample line: "ObjMgr Debug 5 123422222234c34f:0 2025-02-26 03:02:22"
# Format: <text> <text> <digit> <16 hex digits>:<digit> <timestamp YYYY-MM-DD HH:MM:SS>
import re
timestamp_pattern = re.compile(r"^\S+\s+\S+\s+\d\s+[0-9A-Fa-f]{16}:\d\s+(\d{4}-\d{2}-\d{2}\s\d{2}:\d{2}:\d{2})")

def extract_timestamp(line):
    """
    Returns a datetime object if the line has a valid timestamp, otherwise None.
    """
    m = timestamp_pattern.search(line)
    if m:
        ts_str = m.group(1)
        try:
            return datetime.strptime(ts_str, "%Y-%m-%d %H:%M:%S")
        except Exception:
            return None
    return None

def process_file_sequential(input_file, config):
    """
    Based on version 1.0.3b:
      - If file's last-modified time < FROM_TIMESTAMP => skip
      - Otherwise, read lines, apply the 'buffer until valid timestamp found' logic
      - Keep track of how many lines were read (original_count)
      - Keep track of how many lines are written (filtered_count)
      - If filtered_count == 0, do not create the file in the output folder
      - Return (original_count, filtered_count)
    """
    from_ts = None
    to_ts = None
    try:
        if config.get("FROM_TIMESTAMP", "").strip():
            from_ts = datetime.strptime(config["FROM_TIMESTAMP"].strip(), "%Y-%m-%d %H:%M:%S")
    except Exception:
        pass
    try:
        if config.get("TO_TIMESTAMP", "").strip():
            to_ts = datetime.strptime(config["TO_TIMESTAMP"].strip(), "%Y-%m-%d %H:%M:%S")
    except Exception:
        pass

    # 1) Check file last modified time
    mtime = os.path.getmtime(input_file)
    file_dt = datetime.fromtimestamp(mtime)
    if from_ts and file_dt < from_ts:
        # skip
        return (0, 0, True)  # (original_count=0, filtered_count=0, skip_entirely=True)

    # 2) We do the sequential approach from 1.0.3b
    output_folder = config.get("output_folder", ".")
    if not output_folder.endswith("\\") and not output_folder.endswith("/"):
        output_folder += os.sep
    base_name = os.path.basename(input_file)
    output_file = os.path.join(output_folder, base_name + ".filtered.log")

    patterns = config.get("patterns", [])
    # Compile removal patterns
    removal_patterns = [re.compile(p) for p in patterns]

    # We'll track how many lines read, how many lines are eventually written
    original_count = 0
    filtered_count = 0

    # We'll track total file size for progress
    total_size = os.path.getsize(input_file)
    processed_bytes = 0

    # The same buffer logic as 1.0.3b
    buffer = []
    started_output = False
    last_line_empty = False

    try:
        # We'll open the output file only if we actually write lines
        # so we do a 2-step approach: write to a temp list, then if not empty, we create the file
        lines_to_write = []

        with open(input_file, 'r', encoding='utf-8') as in_f:
            for line in in_f:
                original_count += 1
                processed_bytes += len(line.encode('utf-8'))
                update_progress(processed_bytes, total_size)

                # Check removal patterns first
                skip_line = False
                for pat in removal_patterns:
                    if pat.search(line):
                        skip_line = True
                        break
                if skip_line:
                    continue

                # Attempt to extract a timestamp
                ts = extract_timestamp(line)

                if not started_output:
                    # Buffer the line
                    buffer.append(line)
                    if ts is not None:
                        # Found a valid timestamp
                        if from_ts and ts < from_ts:
                            # discard entire buffer
                            buffer = []
                            continue
                        if to_ts and ts > to_ts:
                            # discard buffer, break => no lines in range
                            buffer = []
                            break
                        # otherwise flush the buffer
                        for buf_line in buffer:
                            if buf_line.strip() == "":
                                if last_line_empty:
                                    continue
                                else:
                                    lines_to_write.append(buf_line)
                                    last_line_empty = True
                            else:
                                lines_to_write.append(buf_line)
                                last_line_empty = False
                        buffer = []
                        started_output = True
                else:
                    # already in output mode
                    if ts is not None and to_ts and ts > to_ts:
                        # stop processing
                        break
                    # write line
                    if line.strip() == "":
                        if last_line_empty:
                            continue
                        else:
                            lines_to_write.append(line)
                            last_line_empty = True
                    else:
                        lines_to_write.append(line)
                        last_line_empty = False

        # End of file
        # If we never started output and buffer is non-empty => flush
        if not started_output and buffer:
            for buf_line in buffer:
                if buf_line.strip() == "":
                    if last_line_empty:
                        continue
                    else:
                        lines_to_write.append(buf_line)
                        last_line_empty = True
                else:
                    lines_to_write.append(buf_line)
                    last_line_empty = False

        filtered_count = len(lines_to_write)

        # If no lines, do not create the file
        if filtered_count == 0:
            return (original_count, 0, False)
        else:
            # write to output file
            os.makedirs(output_folder, exist_ok=True)
            with open(output_file, 'w', encoding='utf-8') as out_f:
                out_f.writelines(lines_to_write)
            return (original_count, filtered_count, False)

    finally:
        # progress line break
        sys.stdout.write("\n")

def main():
    """
    Main entry point.
    1) Parse config
    2) read input_folders
    3) for each folder, list files (non-recursive). For each file:
        - print the file name
        - process it
        - if skip_entirely is True => continue
        - if filtered_count == 0 => no file created
        - else => print the original_count, filtered_count
    """
    config = parse_config()

    # If no input_folders specified, we do nothing
    input_folders = config.get("input_folders", [])
    if not input_folders:
        print("No input_folders specified in config. Exiting.")
        return

    for folder in input_folders:
        if not os.path.isdir(folder):
            print(f"Warning: '{folder}' is not a valid directory. Skipping.")
            continue
        # List files in this folder (non-recursive)
        for fname in os.listdir(folder):
            full_path = os.path.join(folder, fname)
            if os.path.isfile(full_path):
                # Print the file name
                print(f"\nProcessing file: {full_path}")
                original_count, filtered_count, skip_entirely = process_file_sequential(full_path, config)
                if skip_entirely:
                    print(f"  Skipped because file last modified < FROM_TIMESTAMP.")
                    continue
                if filtered_count == 0:
                    print("  All lines filtered out => No output file created.")
                else:
                    print(f"  Original lines: {original_count}, Filtered lines: {filtered_count}")

if __name__ == '__main__':
    main()

r"""
===========================
input_folders = X:\temp, Y:\logs
output_folder = D:\temp\log
chunk_size = 5000
num_cpus = 2
FROM_TIMESTAMP = 2025-02-27 15:35:30
TO_TIMESTAMP = 2025-02-27 22:00:00
regexp_pattern =
^.*Begin: Fetch for sql Cursor.*$
^.*in cache \[LOV\].*$
^.*\[KEY\].*$
===========================
"""

"""




```
