# codereview
code review

```
#!/usr/bin/env python3
"""
Version: 1.2.0

This script filters a large log file by removing lines based on two criteria:
  1. Removal regex patterns (specified in the config)
  2. Timestamp filtering – only lines whose timestamp (extracted via a regex)
     falls within a configured FROM_TIMESTAMP and TO_TIMESTAMP range are kept.

The script uses multi‑processing by processing the file in blocks (chunks). Each block is:
  - Split using a safe boundary (a baseline of 1000 lines extended until two consecutive
    lines both yield a valid timestamp, or forced after 2×baseline lines).
  - Then, for each block, the minimum and maximum valid timestamps are computed.
      * If the block is entirely outside the range, it’s discarded.
      * Otherwise, the block is trimmed:
            - Remove all lines before the first line with a valid timestamp ≥ FROM_TIMESTAMP.
            - Remove all lines after the last line with a valid timestamp ≤ TO_TIMESTAMP.
  - Finally, each trimmed block is processed line‑by‑line:
            * Removal regexes are applied.
            * If a line’s timestamp exceeds TO_TIMESTAMP, processing stops for that block.
            * Two consecutive empty lines are avoided.
After processing, all blocks are merged in order to produce the final output.

Progress is printed (as percentage based on bytes processed) during chunking.

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
# Expected sample log line:
# "ObjMgr Debug 5 123422222234c34f:0 2025-02-26 03:02:22"
# Format: <text> <text> <digit> <16 hex digits>:<digit> <timestamp>
TIMESTAMP_REGEX = re.compile(
    r"^\S+\s+\S+\s+\d\s+[0-9A-Fa-f]{16}:\d\s+(\d{4}-\d{2}-\d{2}\s\d{2}:\d{2}:\d{2})"
)

def extract_timestamp(line):
    """
    Extracts and returns a datetime object from a log line using TIMESTAMP_REGEX.
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
    The configuration is placed at the bottom of the script between two lines that exactly match "===========================".

    Expected configuration format (raw string block):
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
    
    in_patterns = False  # Indicates subsequent lines are regex patterns.
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
    Starts with a baseline number of lines and extends the chunk until a safe boundary is reached,
    defined as the point where the last two lines both yield a valid timestamp.
    If no safe boundary is found by 2×baseline lines, the chunk is yielded anyway.
    
    Yields tuples of (chunk_index, list_of_lines).
    """
    chunk_index = 0
    current_chunk = []
    with open(filename, 'r', encoding='utf-8') as f:
        for line in f:
            current_chunk.append(line)
            if len(current_chunk) >= baseline:
                safe_boundary = False
                if len(current_chunk) >= 2:
                    ts1 = extract_timestamp(current_chunk[-2])
                    ts2 = extract_timestamp(current_chunk[-1])
                    if ts1 is not None and ts2 is not None:
                        safe_boundary = True
                if safe_boundary or len(current_chunk) >= 2 * baseline:
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
    
    Processing logic (block-level smart trimming):
      1. Build an array of valid timestamps with their line indices.
      2. If the block has valid timestamps:
          - Compute block_min and block_max.
          - If block_max < FROM_TIMESTAMP or block_min > TO_TIMESTAMP, discard the block.
          - Otherwise, determine:
              * top_index: the first index in the block where a valid timestamp ≥ FROM_TIMESTAMP occurs.
              * bottom_index: the last index where a valid timestamp ≤ TO_TIMESTAMP occurs.
          - Trim the block to lines[top_index : bottom_index+1].
      3. Process the (trimmed) block line-by-line:
          - Skip lines matching any removal pattern.
          - If a line's valid timestamp exceeds TO_TIMESTAMP, stop processing further lines.
          - Avoid writing two consecutive empty lines.
      4. If the block has no valid timestamp, output the block as-is.
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

    # Build list of (index, timestamp) for lines with valid timestamp.
    valid_indices = []
    for i, line in enumerate(lines):
        ts = extract_timestamp(line)
        if ts is not None:
            valid_indices.append((i, ts))
    
    if valid_indices:
        block_min = min(ts for i, ts in valid_indices)
        block_max = max(ts for i, ts in valid_indices)
        # Discard block if completely outside the range.
        if from_ts and block_max < from_ts:
            return (chunk_index, [])
        if to_ts and block_min > to_ts:
            return (chunk_index, [])
        # Determine top boundary.
        top_index = 0
        if from_ts:
            for i, ts in valid_indices:
                if ts >= from_ts:
                    top_index = i
                    break
        # Determine bottom boundary.
        bottom_index = len(lines) - 1
        if to_ts:
            for i, ts in reversed(valid_indices):
                if ts <= to_ts:
                    bottom_index = i
                    break
        trimmed_lines = lines[top_index: bottom_index + 1]
    else:
        trimmed_lines = lines

    # Process the trimmed block line-by-line.
    output_lines = []
    last_line_empty = False
    for line in trimmed_lines:
        skip = False
        for pat in removal_patterns:
            if pat.search(line):
                skip = True
                break
        if skip:
            continue
        ts = extract_timestamp(line)
        if ts is not None and to_ts and ts > to_ts:
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
      - Processes chunks in parallel using multi-processing.
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
