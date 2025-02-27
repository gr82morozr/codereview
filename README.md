# codereview
code review

```
#!/usr/bin/env python3
"""
This script filters a large log file by removing any lines that match one or more
regular expression patterns. Static parameters (like chunk size and CPU cores)
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

def parse_config():
    """
    Reads the current script file (__file__) to extract configuration settings.
    The configuration is expected to be placed at the bottom of the script between two lines
    that exactly match "===========================".
    
    Configuration format example:
      output_folder = D:\
      chunk_size = 1000
      num_cpus = 2
      regexp_pattern =
      ^.*Begin: Fetch for sql Cursor.*$
      ^.*Another pattern to filter.*$
      
    Returns:
      A dictionary with keys for static parameters (e.g. "output_folder", "chunk_size", "num_cpus")
      and a key "patterns" containing a list of regex strings.
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
    
    in_patterns = False  # Flag indicating that following lines are regex patterns.
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

compiled_patterns = None
"""
Global variable to store compiled regular expressions. Each worker process
initializes this once to speed up filtering.
"""

def init_worker(patterns):
    """
    Worker initializer function.
    Compiles each regex pattern from the configuration and stores them in the global variable.
    
    Args:
      patterns: A list of regex pattern strings.
    """
    global compiled_patterns
    compiled_patterns = [re.compile(p) for p in patterns]

def process_chunk_wrapper(data):
    """
    Worker function to filter a chunk of log lines.
    
    Args:
      data: A tuple (lines, chunk_bytes) where 'lines' is a list of strings read from the log file.
    
    Returns:
      A list of lines that do NOT match any of the regex patterns.
    """
    lines, _ = data
    filtered = []
    for line in lines:
        skip = False
        for pattern in compiled_patterns:
            if pattern.search(line):
                skip = True  # Mark line for removal if any pattern matches.
                break
        if not skip:
            filtered.append(line)
    return filtered

def read_chunks(file_obj, chunk_size):
    """
    Generator function to read a file in chunks of a given number of lines.
    
    Args:
      file_obj: An open file object for reading.
      chunk_size: Number of lines per chunk.
    
    Yields:
      A tuple (list_of_lines, total_bytes) where total_bytes is the approximate number
      of bytes read in that chunk.
    """
    while True:
        lines = []
        bytes_count = 0
        for _ in range(chunk_size):
            line = file_obj.readline()
            if not line:
                break  # End-of-file reached.
            lines.append(line)
            bytes_count += len(line.encode('utf-8'))
        if not lines:
            break
        yield (lines, bytes_count)

def update_progress(processed, total):
    """
    Updates and prints a progress bar (in percentage) on the same console line.
    
    Args:
      processed: Number of bytes processed so far.
      total: Total number of bytes in the file.
    """
    percent = (processed / total) * 100
    sys.stdout.write(f"\rProgress: {percent:.2f}%")
    sys.stdout.flush()

def main():
    """
    Main function that:
      - Parses command-line arguments.
      - Loads configuration from this script.
      - Sets up multiprocessing to filter the log file in chunks.
      - Writes the filtered log to the specified output folder while preserving line order.
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
    
    # Retrieve static parameters from configuration.
    output_folder = config.get("output_folder", ".")
    chunk_size = config.get("chunk_size", 1000)  # Number of lines per chunk.
    num_cpus = config.get("num_cpus", 2)           # Number of CPU cores to use.
    
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
    
    # Create a multiprocessing pool with num_cpus worker processes.
    pool = multiprocessing.Pool(processes=num_cpus, initializer=init_worker, initargs=(patterns,))
    
    tasks = []      # List to hold pending async tasks.
    batch_size = 10 # Process up to 10 chunks concurrently.
    # We also maintain the order by processing tasks in the same sequence they were submitted.
    
    try:
        with open(input_file, 'r', encoding='utf-8') as in_f:
            chunk_generator = read_chunks(in_f, chunk_size)
            for chunk_data in chunk_generator:
                # Submit the chunk for asynchronous processing.
                task = pool.apply_async(process_chunk_wrapper, (chunk_data,))
                tasks.append((chunk_data[1], task))
                
                # Once a batch is collected, process them in the order of submission.
                if len(tasks) >= batch_size:
                    for bytes_count, async_result in tasks:
                        filtered_lines = async_result.get()  # Waits for the result in order.
                        out_f.writelines(filtered_lines)
                        processed_bytes += bytes_count
                        update_progress(processed_bytes, total_size)
                    tasks = []  # Clear the batch.
            
            # Process any remaining tasks.
            for bytes_count, async_result in tasks:
                filtered_lines = async_result.get()
                out_f.writelines(filtered_lines)
                processed_bytes += bytes_count
                update_progress(processed_bytes, total_size)
    finally:
        pool.close()
        pool.join()
        out_f.close()
    
    sys.stdout.write("\nFiltering complete. Output saved to: " + output_file + "\n")

if __name__ == '__main__':
    main()

"""
===========================
output_folder = D:\
chunk_size = 1000
num_cpus = 2
regexp_pattern =
^.*Begin: Fetch for sql Cursor.*$
^.*Another pattern to filter.*$
===========================
"""



```
