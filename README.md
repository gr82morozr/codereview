# codereview
code review

```
#!/usr/bin/env python3
"""
This script filters a large log file by removing lines based on two criteria:
  1. Removal regex patterns (specified in the config)
  2. Timestamp filtering – only lines whose timestamp (extracted via a regex)
     fall within the configured FROM_TIMESTAMP and TO_TIMESTAMP range are kept.
     
For lines that cannot be parsed for a timestamp, the script buffers them and waits
until it finds a line with a valid timestamp. Then:
  - If that valid timestamp is older than FROM_TIMESTAMP, the entire buffered block is discarded.
  - If it is within range, the buffered block is flushed (i.e. output) and subsequent lines are processed.
  - If a valid timestamp is later than TO_TIMESTAMP, processing stops.
     
The output file is written in the same order as the input and no two consecutive empty lines are written.
Static parameters (such as chunk size, output folder, and timestamp bounds) are stored in the embedded configuration section.
     
Usage: file_log.py <log_file>
The filtered output is written to <log_file>.filtered.log in the folder specified by "output_folder" in the config.
"""

import os
import sys
import re
from datetime import datetime

def parse_config():
    """
    Reads this script file (__file__) to extract configuration settings.
    The configuration is placed at the bottom of the script between two lines that exactly match:
    
      ===========================
    
    Expected configuration format example:
      output_folder = D:\
      chunk_size = 1000
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
    
    in_patterns = False  # Flag indicating that subsequent lines are regex patterns.
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

def main():
    """
    Main function that:
      - Parses command-line arguments and loads configuration.
      - Converts FROM_TIMESTAMP and TO_TIMESTAMP (if provided) into datetime objects.
      - Processes the input log file sequentially:
          * For each line, first checks removal regex patterns.
          * Buffers lines that cannot be parsed for a timestamp.
          * When a line with a valid timestamp is found:
              - If the timestamp is older than FROM_TIMESTAMP, the entire buffered block is discarded.
              - Otherwise (and if within TO_TIMESTAMP if provided), the buffered block is flushed to output and output mode starts.
          * Once output mode has started, each line is checked (if it has a timestamp, compared against TO_TIMESTAMP) before being written.
          * Processing stops if a valid timestamp is found that is later than TO_TIMESTAMP.
      - Ensures that no two consecutive empty lines are written.
      - Writes the filtered output in the same order as the input.
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
    # chunk_size is kept for compatibility but is not used in sequential processing.
    from_timestamp_str = config.get("FROM_TIMESTAMP", "").strip()
    to_timestamp_str = config.get("TO_TIMESTAMP", "").strip()
    
    try:
        from_timestamp = datetime.strptime(from_timestamp_str, "%Y-%m-%d %H:%M:%S") if from_timestamp_str else None
    except Exception:
        from_timestamp = None
    try:
        to_timestamp = datetime.strptime(to_timestamp_str, "%Y-%m-%d %H:%M:%S") if to_timestamp_str else None
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
    
    # Compile removal patterns.
    removal_patterns = [re.compile(p) for p in patterns]
    """
    Each removal pattern will cause a line to be skipped (i.e. removed) if it matches.
    """
    
    # Compile the timestamp extraction regex.
    # This regex is expected to capture a timestamp in the format "YYYY-MM-DD hh:mm:ss" from the log line.
    # (It uses the same pattern as before.)
    timestamp_pattern = re.compile(r"^\w+\s+\w+\s+\d+\w{16}\:\s+(\d{4}-\d{2}-\d{2}\s\d{2}:\d{2}:\d{2})")
    
    try:
        out_f = open(output_file, 'w', encoding='utf-8')
    except Exception as e:
        print(f"Error: Could not open output file '{output_file}' for writing: {e}")
        sys.exit(1)
    
    buffer = []         # Buffer for lines before output mode is determined.
    started_output = False  # Flag to indicate that a valid in-range timestamp has been found and output has started.
    last_line_empty = False # Flag to ensure no two consecutive empty lines are written.
    
    with open(input_file, 'r', encoding='utf-8') as in_f:
        for line in in_f:
            # First, check if the line should be removed by any removal pattern.
            skip_due_to_removal = False
            for pat in removal_patterns:
                if pat.search(line):
                    skip_due_to_removal = True
                    break
            if skip_due_to_removal:
                continue  # Skip this line entirely.
            
            # Attempt to extract a timestamp from the line.
            m = timestamp_pattern.search(line)
            if m:
                try:
                    line_ts = datetime.strptime(m.group(1), "%Y-%m-%d %H:%M:%S")
                except Exception:
                    line_ts = None
            else:
                line_ts = None
            
            if not started_output:
                # Buffer the line until we can decide based on a valid timestamp.
                buffer.append(line)
                if line_ts is not None:
                    # A valid timestamp has been found in this block.
                    # Decide whether to flush (output) or discard the buffered block.
                    if from_timestamp is not None and line_ts < from_timestamp:
                        # The block is older than FROM_TIMESTAMP;
                        # discard the entire buffered block and continue.
                        buffer = []
                        continue
                    if to_timestamp is not None and line_ts > to_timestamp:
                        # The block is later than TO_TIMESTAMP;
                        # stop processing further lines.
                        buffer = []
                        break
                    # Otherwise, the block is in-range.
                    # Flush the buffered lines to the output file.
                    for buf_line in buffer:
                        if buf_line.strip() == "":
                            if last_line_empty:
                                continue
                            else:
                                out_f.write(buf_line)
                                last_line_empty = True
                        else:
                            out_f.write(buf_line)
                            last_line_empty = False
                    buffer = []  # Clear the buffer.
                    started_output = True
            else:
                # Already in output mode.
                # If this line has a valid timestamp, check if it exceeds TO_TIMESTAMP.
                if line_ts is not None and to_timestamp is not None and line_ts > to_timestamp:
                    # Stop processing further lines.
                    break
                # Write the line directly.
                if line.strip() == "":
                    if last_line_empty:
                        continue
                    else:
                        out_f.write(line)
                        last_line_empty = True
                else:
                    out_f.write(line)
                    last_line_empty = False
        # End of file reached.
        # If output mode was never started (no valid timestamp found) flush buffer.
        if not started_output and buffer:
            for buf_line in buffer:
                if buf_line.strip() == "":
                    if last_line_empty:
                        continue
                    else:
                        out_f.write(buf_line)
                        last_line_empty = True
                else:
                    out_f.write(buf_line)
                    last_line_empty = False
    
    out_f.close()
    sys.stdout.write("\nFiltering complete. Output saved to: " + output_file + "\n")

if __name__ == '__main__':
    main()

"""
===========================
output_folder = D:\
chunk_size = 1000
FROM_TIMESTAMP = 2023-01-01 00:00:00
TO_TIMESTAMP = 2023-12-31 23:59:59
regexp_pattern =
^.*Begin: Fetch for sql Cursor.*$
^.*Another pattern to filter.*$
^.*[KEY].*$
===========================
"""



```
