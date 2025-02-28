# codereview
code review

```
#!/usr/bin/env python3
"""
This script filters a large log file by removing lines based on two criteria:
  1. Removal regex patterns (specified in the config)
  2. Timestamp filtering – only lines whose timestamp (extracted via a regex)
     falls within a configured FROM_TIMESTAMP and TO_TIMESTAMP range are kept.
     
Static parameters (such as chunk size, number of CPU cores, and timestamp bounds)
are stored in the embedded configuration section at the bottom.

Usage: file_log.py <log_file>
The filtered output is written to a file named <log_file>.filtered.log in the folder
specified by "output_folder" in the configuration.
"""

import os
import sys
import re
import multiprocessing
import time
from datetime import datetime

def parse_config():
    """
    Reads this script file (__file__) to extract configuration settings.
    The configuration is placed at the bottom of the script between two lines that exactly match
    "===========================".
    
    Expected configuration format example:
      output_folder = D:\
      chunk_size = 1000
      num_cpus = 2
      FROM_TIMESTAMP = 2023-01-01 00:00:00
      TO_TIMESTAMP = 2023-12-31 23:59:59
      regexp_pattern =
      ^.*Begin: Fetch for sql Cursor.*$
      ^.*Another pattern to filter.*$
      ^.*[KEY].*$
      
    Returns:
      A dictionary with static parameters and a key "patterns" containing a list
      of regex strings (removal patterns).
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

# Global variables for worker processes.
compiled_patterns = None
compiled_timestamp_pattern = None
global_from_timestamp = None
global_to_timestamp = None
"""
Global variables:
  - compiled_patterns: list of compiled removal regex patterns.
  - compiled_timestamp_pattern: regex to extract the timestamp.
  - global_from_timestamp / global_to_timestamp: timestamp boundaries (as datetime objects).
"""

def init_worker(patterns, from_ts, to_ts):
    """
    Worker initializer function.
    Compiles each removal regex pattern and the timestamp extraction pattern,
    and sets the global timestamp boundaries.
    
    Args:
      patterns: List of regex strings for removal.
      from_ts: A datetime object (lower bound) or None.
      to_ts: A datetime object (upper bound) or None.
    """
    global compiled_patterns, compiled_timestamp_pattern, global_from_timestamp, global_to_timestamp
    compiled_patterns = [re.compile(p) for p in patterns]
    # Compile the timestamp extraction regex.
    # This regex is expected to capture a timestamp (format "YYYY-MM-DD hh:mm:ss")
    # from the log line in group(1).
    compiled_timestamp_pattern = re.compile(r"^\w+\s+\w+\s+\d+\w{16}\:\s+(\d{4}\-\d{2}\-\d{2}\s\d{2}:\d{2}:\d{2})")
    global_from_timestamp = from_ts
    global_to_timestamp = to_ts

def process_chunk_wrapper(data):
    """
    Worker function to filter a chunk of log lines.
    
    For each line:
      1. Remove the line if any removal regex matches.
      2. Otherwise, attempt to extract a timestamp.
         - If a timestamp is found:
             * If both bounds are provided, remove the line if its timestamp is
               before FROM_TIMESTAMP or after TO_TIMESTAMP.
             * If only FROM_TIMESTAMP is provided, remove the line if its timestamp
               is before FROM_TIMESTAMP.
             * If only TO_TIMESTAMP is provided, remove the line if its timestamp
               is after TO_TIMESTAMP.
         - If no timestamp is found or parsing fails, the line is kept.
    
    Args:
      data: Tuple (lines, chunk_bytes) where 'lines' is a list of strings.
      
    Returns:
      A list of lines that pass the filters.
    """
    lines, _ = data
    filtered = []
    for line in lines:
        skip = False
        # Check removal patterns.
        for pattern in compiled_patterns:
            if pattern.search(line):
                skip = True
                break
        if not skip:
            m = compiled_timestamp_pattern.search(line)
            if m:
                ts_str = m.group(1)
                try:
                    line_ts = datetime.strptime(ts_str, "%Y-%m-%d %H:%M:%S")
                    if global_from_timestamp and global_to_timestamp:
                        if line_ts < global_from_timestamp or line_ts > global_to_timestamp:
                            skip = True
                    elif global_from_timestamp:
                        if line_ts < global_from_timestamp:
                            skip = True
                    elif global_to_timestamp:
                        if line_ts > global_to_timestamp:
                            skip = True
                except Exception:
                    # If timestamp parsing fails, keep the line.
                    pass
        if not skip:
            filtered.append(line)
    return filtered

def read_chunks(file_obj, chunk_size):
    """
    Generator that reads a file in chunks of a specified number of lines.
    
    Args:
      file_obj: Open file object.
      chunk_size: Number of lines per chunk.
      
    Yields:
      Tuple (list_of_lines, bytes_count) where bytes_count is the approximate number
      of bytes read for that chunk.
    """
    while True:
        lines = []
        bytes_count = 0
        for _ in range(chunk_size):
            line = file_obj.readline()
            if not line:
                break
            lines.append(line)
            bytes_count += len(line.encode('utf-8'))
        if not lines:
            break
        yield (lines, bytes_count)

def update_progress(processed, total):
    """
    Updates and prints a progress bar (percentage) on the same console line.
    
    Args:
      processed: Bytes processed so far.
      total: Total file size in bytes.
    """
    percent = (processed / total) * 100
    sys.stdout.write(f"\rProgress: {percent:.2f}%")
    sys.stdout.flush()

def main():
    """
    Main function that:
      - Parses command-line arguments.
      - Loads configuration from this script.
      - Parses FROM_TIMESTAMP and TO_TIMESTAMP (if provided) into datetime objects.
      - Sets up multiprocessing to filter the log file in chunks.
      - Writes the filtered output (maintaining line order and ensuring no two
        consecutive empty lines).
      - Updates a progress bar as the file is processed.
    """
    if len(sys.argv) != 2:
        print("Usage: file_log.py <log_file>")
        sys.exit(1)
    
    input_file = sys.argv[1]
    if not os.path.isfile(input_file):
        print(f"Error: File '{input_file}' does not exist.")
        sys.exit(1)
    
    config = parse_config()
    
    # Retrieve static parameters.
    output_folder = config.get("output_folder", ".")
    chunk_size = config.get("chunk_size", 1000)
    num_cpus = config.get("num_cpus", 2)
    from_timestamp_str = config.get("FROM_TIMESTAMP", "")
    to_timestamp_str = config.get("TO_TIMESTAMP", "")
    
    # Parse timestamp boundaries if provided.
    try:
        from_timestamp = datetime.strptime(from_timestamp_str, "%Y-%m-%d %H:%M:%S") if from_timestamp_str.strip() else None
    except Exception:
        from_timestamp = None
    try:
        to_timestamp = datetime.strptime(to_timestamp_str, "%Y-%m-%d %H:%M:%S") if to_timestamp_str.strip() else None
    except Exception:
        to_timestamp = None
    
    if not output_folder.endswith("\\") and not output_folder.endswith("/"):
        output_folder += os.sep
    base_name = os.path.basename(input_file)
    output_file = os.path.join(output_folder, base_name + ".filtered.log")
    
    patterns = config.get("patterns", [])
    if not patterns:
        print("Error: No regex patterns found in configuration.")
        sys.exit(1)
    
    total_size = os.path.getsize(input_file)
    processed_bytes = 0
    
    try:
        out_f = open(output_file, 'w', encoding='utf-8')
    except Exception as e:
        print(f"Error: Could not open output file '{output_file}' for writing: {e}")
        sys.exit(1)
    
    # Create a multiprocessing pool with num_cpus workers.
    pool = multiprocessing.Pool(processes=num_cpus, initializer=init_worker, initargs=(patterns, from_timestamp, to_timestamp))
    
    tasks = []      # To hold async tasks.
    batch_size = 10 # Process up to 10 chunks concurrently.
    last_line_empty = False  # To avoid writing two consecutive empty lines.
    
    try:
        with open(input_file, 'r', encoding='utf-8') as in_f:
            chunk_generator = read_chunks(in_f, chunk_size)
            for chunk_data in chunk_generator:
                task = pool.apply_async(process_chunk_wrapper, (chunk_data,))
                tasks.append((chunk_data[1], task))
                
                if len(tasks) >= batch_size:
                    for bytes_count, async_result in tasks:
                        filtered_lines = async_result.get()  # Retrieve in submission order.
                        for line in filtered_lines:
                            if line.strip() == "":
                                if last_line_empty:
                                    continue
                                else:
                                    out_f.write(line)
                                    last_line_empty = True
                            else:
                                out_f.write(line)
                                last_line_empty = False
                        processed_bytes += bytes_count
                        update_progress(processed_bytes, total_size)
                    tasks = []  # Clear the batch.
            
            # Process any remaining tasks.
            for bytes_count, async_result in tasks:
                filtered_lines = async_result.get()
                for line in filtered_lines:
                    if line.strip() == "":
                        if last_line_empty:
                            continue
                        else:
                            out_f.write(line)
                            last_line_empty = True
                    else:
                        out_f.write(line)
                        last_line_empty = False
                processed_bytes += bytes_count
                update_progress(processed_bytes, total_size)
    finally:
        pool.close()
        pool.join()
        out_f.close()
    
    sys.stdout.write("\nFiltering complete. Output saved to: " + output_file + "\n")

if __name__ == '__main__':
    main()

r"""
===========================
output_folder = D:\
chunk_size = 1000
num_cpus = 2
FROM_TIMESTAMP = 2023-01-01 00:00:00
TO_TIMESTAMP = 2023-12-31 23:59:59
regexp_pattern =
^.*Begin: Fetch for sql Cursor.*$
^.*Another pattern to filter.*$
^.*[KEY].*$
===========================
"""




```
