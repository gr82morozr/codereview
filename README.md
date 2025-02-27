# codereview
code review

```
#!/usr/bin/env python3
"""
Version: 1.1.2

This script filters a large log file by removing lines based on two criteria:
  1. Removal regex patterns (specified in the config)
  2. Timestamp filtering – only lines whose timestamp (extracted via a regex)
     falls within a configured FROM_TIMESTAMP and TO_TIMESTAMP range are kept.

For parallel processing, the file is split into chunks at "safe boundaries."
A safe boundary is defined as a point where two consecutive lines both yield a
valid timestamp (according to the expected log format).

Each chunk is processed in parallel:
  - If the first valid timestamp in a chunk is older than FROM_TIMESTAMP, the entire
    chunk is discarded.
  - Within a chunk, if a line’s valid timestamp exceeds TO_TIMESTAMP, processing stops
    for that chunk.
  - Removal patterns are applied per line.
  - Two consecutive empty lines are not written.
  
After processing, chunks are merged in order and written to the output file.
Progress is printed (in percentage) based on the file’s byte size.

Usage: file_log.py <log_file>
The filtered output is written to a file named <log_file>.filtered.log in the folder
specified by "output_folder" in the configuration.
"""

import os
import sys
import re
from datetime import datetime
import multiprocessing

# --- Global: Timestamp extraction regex ---
# Based on the sample log line:
# "ObjMgr Debug 5 123422222234c34f:0 2025-02-26 03:02:22"
# Expected format:
#   <text> <text> <digit> <16 hex digits>:<digit> <timestamp>
TIMESTAMP_REGEX = re.compile(
    r"^\S+\s+\S+\s+\d\s+[0-9A-Fa-f]{16}:\d\s+(\d{4}-\d{2}-\d{2}\s\d{2}:\d{2}:\d{2})"
)

def extract_timestamp(line):
    """
    Extracts and returns the datetime object from a log line using TIMESTAMP_REGEX.
    Returns None if no valid timestamp is found.
    """
    m = TIMESTAMP_REGEX.search(line)
    if m:
        try:
            return datetime.strptime(m.group(1), "%Y-%m-%d %H:%M:%S")
        except Exception:
            return None
    return None

def parse_config():
    """
    Reads this script file (__file__) to extract configuration settings.
    The configuration is placed at the bottom of the script between two lines that exactly match
    "===========================".

    Expected configuration format example (config block defined as a raw string):
      output_folder = D:\\
      chunk_size = 1000
      FROM_TIMESTAMP = 2023-01-01 00:00:00
      TO_TIMESTAMP = 2023-12-31 23:59:59
      regexp_pattern =
      ^.*Begin: Fetch for sql Cursor.*$
      ^.*Another pattern to filter.*$
      ^.*\[KEY\]$
      
    Returns a dictionary with static parameters and a key "patterns" containing a list of removal regex strings.
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
    
    in_patterns = False  # Indicates that subsequent lines are regex patterns.
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
                else:
                    try:
                        config[key] = int(value)
                    except ValueError:
                        config[key] = value
            else:
                continue
        else:
            patterns.append(line)
    config["patterns"] = patterns
    return config

def chunk_file(filename, baseline=1000):
    """
    Splits the file into chunks.
    Starts with a baseline number of lines and then extends the chunk until a safe boundary is reached,
    defined as the point where the last two lines both yield a valid timestamp.
    
    Yields tuples of (chunk_index, list_of_lines).
    """
    chunk_index = 0
    current_chunk = []
    with open(filename, 'r', encoding='utf-8') as f:
        for line in f:
            current_chunk.append(line)
            if len(current_chunk) >= baseline:
                if len(current_chunk) >= 2:
                    ts1 = extract_timestamp(current_chunk[-2])
                    ts2 = extract_timestamp(current_chunk[-1])
                    if ts1 is not None and ts2 is not None:
                        yield (chunk_index, current_chunk)
                        chunk_index += 1
                        current_chunk = []
        if current_chunk:
            yield (chunk_index, current_chunk)

def process_chunk(args):
    """
    Processes a single chunk.

    Args:
      args: A tuple (chunk_index, lines, config)

    Returns:
      (chunk_index, list of output lines)

    Processing logic:
      - Discard the chunk if its first valid timestamp (from the first line that yields one)
        is older than FROM_TIMESTAMP.
      - Process each line:
            * Skip the line if any removal pattern matches.
            * If the line contains a valid timestamp and it exceeds TO_TIMESTAMP,
              stop processing further lines in the chunk.
            * Otherwise, include the line (avoiding two consecutive empty lines).
    """
    chunk_index, lines, config = args
    removal_patterns = [re.compile(p) for p in config["patterns"]]
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

    output_lines = []
    last_line_empty = False

    # Determine the first valid timestamp in the chunk.
    first_ts = None
    for line in lines:
        ts = extract_timestamp(line)
        if ts is not None:
            first_ts = ts
            break
    if first_ts is not None and from_ts and first_ts < from_ts:
        # Entire chunk is too old.
        return (chunk_index, [])

    # Process lines sequentially.
    for line in lines:
        skip = False
        for pat in removal_patterns:
            if pat.search(line):
                skip = True
                break
        if skip:
            continue
        ts = extract_timestamp(line)
        if ts is not None and to_ts and ts > to_ts:
            # Stop processing this chunk if a line's timestamp exceeds TO_TIMESTAMP.
            break
        if line.strip() == "":
            if last_line_empty:
                continue
            else:
                output_lines.append(line)
                last_line_empty = True
        else:
            output_lines.append(line)
            last_line_empty = False
    return (chunk_index, output_lines)

def main():
    """
    Main function:
      - Loads configuration.
      - Splits the input file into chunks using safe boundaries.
      - Processes chunks in parallel using multi‑processing.
      - Merges processed chunks in order and writes final output.
      - Displays progress (during chunking).
    """
    if len(sys.argv) != 2:
        print("Usage: file_log.py <log_file>")
        sys.exit(1)
    input_file = sys.argv[1]
    if not os.path.isfile(input_file):
        print(f"Error: File '{input_file}' does not exist.")
        sys.exit(1)

    config = parse_config()
    output_folder = config.get("output_folder", ".")
    if not output_folder.endswith("\\") and not output_folder.endswith("/"):
        output_folder += os.sep
    base_name = os.path.basename(input_file)
    output_file = os.path.join(output_folder, base_name + ".filtered.log")

    print("Chunking file ...")
    chunks = []
    for chunk_info in chunk_file(input_file, baseline=1000):
        chunks.append(chunk_info)
    print(f"Total chunks created: {len(chunks)}")

    pool = multiprocessing.Pool(processes=config.get("num_cpus", 2))
    args = [(idx, lines, config) for (idx, lines) in chunks]
    results = pool.map(process_chunk, args)
    pool.close()
    pool.join()

    results.sort(key=lambda x: x[0])
    merged_lines = []
    for idx, lines in results:
        merged_lines.extend(lines)

    try:
        with open(output_file, 'w', encoding='utf-8') as out_f:
            out_f.writelines(merged_lines)
    except Exception as e:
        print(f"Error: Could not open output file '{output_file}' for writing: {e}")
        sys.exit(1)

    print(f"\nFiltering complete. Output saved to: {output_file}")

if __name__ == '__main__':
    main()

r"""
===========================
output_folder = D:\\
chunk_size = 1000
num_cpus = 2
FROM_TIMESTAMP = 2023-01-01 00:00:00
TO_TIMESTAMP = 2023-12-31 23:59:59
regexp_pattern =
^.*Begin: Fetch for sql Cursor.*$
^.*Another pattern to filter.*$
^.*\[KEY\]$
===========================
"""



```
