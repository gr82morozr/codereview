# codereview
code review

```
#!/usr/bin/env python3
"""
Version: 1.0.0

This script filters a large log file by removing lines based on two criteria:
  1. Removal regex patterns (specified in the config)
  2. Timestamp filtering – only lines whose timestamp (extracted via a regex)
     falls within a configured FROM_TIMESTAMP and TO_TIMESTAMP range are kept.

For lines that cannot be parsed for a timestamp, the script buffers them and
waits until a line with a valid timestamp is found. Then:
  - If the valid timestamp is older than FROM_TIMESTAMP, the entire buffered block
    (including lines with no timestamp) is discarded.
  - If the valid timestamp is later than TO_TIMESTAMP, processing stops.
  - Otherwise, the buffered block is flushed (i.e. written to output) and further
    lines are processed in “output mode.”

The output file is written in the same order as the input and no two consecutive empty
lines are written.

Usage: file_log.py <log_file>
The filtered output is written to a file named <log_file>.filtered.log in the folder
specified by "output_folder" in the configuration.
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

    Expected configuration format example:
      output_folder = D:\
      chunk_size = 1000
      FROM_TIMESTAMP = 2023-01-01 00:00:00
      TO_TIMESTAMP = 2023-12-31 23:59:59
      HIGHLIGHT_KEYWORDS = ERROR, WARNING, CRITICAL
      regexp_pattern =
      ^.*Begin: Fetch for sql Cursor.*$
      ^.*Another pattern to filter.*$
      ^.*[KEY].*$
      
    Returns:
      A dictionary with static parameters and a key "patterns" containing a list of regex strings
      (removal patterns).
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

def main():
    """
    Main function that:
      - Parses command-line arguments and loads configuration.
      - Converts FROM_TIMESTAMP and TO_TIMESTAMP (if provided) into datetime objects.
      - Processes the input log file sequentially:
          * For each line, first checks removal regex patterns.
          * If a line does not match any removal pattern, it is buffered.
          * The script attempts to extract a timestamp from each line.
             - If no valid timestamp is found, the line remains in the buffer.
             - When a valid timestamp is found:
                 + If not in output mode:
                     - If the timestamp is older than FROM_TIMESTAMP (if set), discard the entire buffer.
                     - If the timestamp is later than TO_TIMESTAMP (if set), stop processing.
                     - Otherwise, flush the buffer (write all buffered lines) and mark output mode as started.
                 + If already in output mode:
                     - If a valid timestamp exceeds TO_TIMESTAMP, stop processing.
                     - Otherwise, write the line.
          * The script ensures no two consecutive empty lines are written.
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

    # Compile the timestamp extraction regex.
    # This regex should capture a timestamp in the format "YYYY-MM-DD hh:mm:ss" in group(1).
    timestamp_pattern = re.compile(r"^\w+\s+\w+\s+\d+\w{16}\:\s+(\d{4}-\d{2}-\d{2}\s\d{2}:\d{2}:\d{2})")

    try:
        out_f = open(output_file, 'w', encoding='utf-8')
    except Exception as e:
        print(f"Error: Could not open output file '{output_file}' for writing: {e}")
        sys.exit(1)

    buffer = []         # Buffer to hold lines until a valid timestamp is found.
    started_output = False  # Flag indicating whether output mode has started.
    last_line_empty = False # Flag to avoid writing two consecutive empty lines.

    with open(input_file, 'r', encoding='utf-8') as in_f:
        for line in in_f:
            # First, check if the line should be removed based on removal patterns.
            remove_line = False
            for pat in removal_patterns:
                if pat.search(line):
                    remove_line = True
                    break
            if remove_line:
                continue  # Skip this line entirely.

            # Attempt to extract a timestamp.
            m = timestamp_pattern.search(line)
            if m:
                try:
                    line_ts = datetime.strptime(m.group(1), "%Y-%m-%d %H:%M:%S")
                except Exception:
                    line_ts = None
            else:
                line_ts = None

            if not started_output:
                # Buffer the line if output hasn't started.
                buffer.append(line)
                if line_ts is not None:
                    # A valid timestamp has been found in the current block.
                    # Check against FROM_TIMESTAMP and TO_TIMESTAMP.
                    if from_timestamp and line_ts < from_timestamp:
                        # The block is too old – discard the entire buffer.
                        buffer = []
                        continue  # Move on to the next line.
                    if to_timestamp and line_ts > to_timestamp:
                        # The block is too new – stop processing.
                        buffer = []
                        break
                    # Otherwise, the block is in range.
                    # Flush the buffered lines to the output.
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
                # Output mode has started.
                # If this line has a valid timestamp, check TO_TIMESTAMP.
                if line_ts is not None and to_timestamp and line_ts > to_timestamp:
                    # Stop processing if the line is too new.
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
        # If output mode never started and buffer is non-empty, flush the buffer.
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

r"""
===========================
output_folder = D:\
chunk_size = 1000
num_cpus = 2
FROM_TIMESTAMP = 2023-01-01 00:00:00
TO_TIMESTAMP = 2023-12-31 23:59:59
HIGHLIGHT_KEYWORDS = ERROR, WARNING, CRITICAL
regexp_pattern =
^.*Begin: Fetch for sql Cursor.*$
^.*Another pattern to filter.*$
^.*[KEY].*$
===========================
"""



```
