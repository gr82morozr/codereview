# codereview
code review

```
import os
import re
from datetime import datetime

# Regular expression to extract a timestamp from the beginning of a line.
# Adjust the regex pattern and datetime format if needed.
TIMESTAMP_REGEX = re.compile(r'^(\d{4}-\d{2}-\d{2} \d{2}:\d{2}:\d{2})')

def parse_timestamp(ts_str):
    """Convert a timestamp string into a datetime object."""
    return datetime.strptime(ts_str, "%Y-%m-%d %H:%M:%S")

def extract_timestamp(line):
    """Extract and parse the timestamp from a log line (string)."""
    match = TIMESTAMP_REGEX.match(line)
    if match:
        try:
            return parse_timestamp(match.group(1))
        except ValueError:
            return None
    return None

def get_line_start(file, pos):
    """
    Given a file (opened in binary mode) and an arbitrary byte offset 'pos',
    scan backwards to find the start of the line. For Windows logs, lines end
    with b'\r\n'. This function returns the byte offset immediately after the
    previous b'\r\n' (i.e. the start of the current line), or 0 if none is found.
    """
    if pos == 0:
        return 0
    cur = pos
    # We need at least two bytes to check for the CRLF sequence.
    while cur > 1:
        file.seek(cur - 2)
        chunk = file.read(2)
        if chunk == b'\r\n':
            return cur
        cur -= 1
    return 0

def get_line_info(file, pos):
    """
    Assuming 'pos' is at the start of a line, read until the next CRLF (b'\r\n')
    is encountered. Returns a tuple:
        (line_start, line_end, line_content)
    where line_content is the line's bytes without the trailing newline.
    """
    file.seek(pos)
    line_bytes = file.readline()  # Reads until b'\n'
    line_end = file.tell()
    # Remove the Windows CRLF (b'\r\n') if present; if not, remove a lone LF (b'\n')
    if line_bytes.endswith(b'\r\n'):
        line_content = line_bytes[:-2]
    elif line_bytes.endswith(b'\n'):
        line_content = line_bytes[:-1]
    else:
        line_content = line_bytes
    return pos, line_end, line_content

def binary_search_lower_bound(file, target_time):
    """
    Find the byte offset of the first log line whose timestamp is >= target_time.
    Assumes the file is sorted by timestamp.
    """
    file.seek(0, os.SEEK_END)
    file_size = file.tell()
    left = 0
    right = file_size
    candidate = None

    while left < right:
        mid = (left + right) // 2
        # Align mid to the start of a line.
        line_start = get_line_start(file, mid)
        _, line_end, line_bytes = get_line_info(file, line_start)
        try:
            line_str = line_bytes.decode('utf-8', errors='ignore').strip()
        except Exception:
            left = line_end
            continue

        ts = extract_timestamp(line_str)
        if ts is None:
            left = line_end
            continue

        if ts >= target_time:
            candidate = line_start
            right = line_start  # Search left for an earlier qualifying line.
        else:
            left = line_end  # Move right.
    return candidate if candidate is not None else file_size

def binary_search_upper_bound(file, target_time):
    """
    Find the byte offset at the start of the last log line whose timestamp is <= target_time.
    Assumes the file is sorted by timestamp.
    """
    file.seek(0, os.SEEK_END)
    file_size = file.tell()
    left = 0
    right = file_size
    candidate = None

    while left < right:
        mid = (left + right) // 2
        line_start = get_line_start(file, mid)
        _, line_end, line_bytes = get_line_info(file, line_start)
        try:
            line_str = line_bytes.decode('utf-8', errors='ignore').strip()
        except Exception:
            right = line_start
            continue

        ts = extract_timestamp(line_str)
        if ts is None:
            right = line_start
            continue

        if ts <= target_time:
            candidate = line_start
            left = line_end  # Search right for a later qualifying line.
        else:
            right = line_start  # Move left.
    return candidate if candidate is not None else 0

def extract_log_range(input_file, output_file, from_timestamp, to_timestamp):
    """
    Extracts log lines between 'from_timestamp' and 'to_timestamp' (inclusive) by using
    binary search to identify the lower and upper boundaries. The extracted bytes
    are written to the output file.
    """
    with open(input_file, 'rb') as f:
        # Find lower and upper boundaries by binary search.
        lower_bound = binary_search_lower_bound(f, from_timestamp)
        upper_bound = binary_search_upper_bound(f, to_timestamp)

        # If no valid lower bound is found, report and exit.
        if lower_bound is None or lower_bound >= f.tell():
            print("No log lines found for the given start time.")
            return

        # Ensure we include the complete last line by reading its full length.
        f.seek(upper_bound)
        _, upper_line_end, _ = get_line_info(f, upper_bound)

        # Read all bytes from the lower bound to the end of the last qualifying line.
        f.seek(lower_bound)
        bytes_to_read = upper_line_end - lower_bound
        data = f.read(bytes_to_read)

    with open(output_file, 'wb') as out:
        out.write(data)
    print(f"Extracted log range written to: {output_file}")

# Example usage:
if __name__ == '__main__':
    from_timestamp = datetime.strptime("2025-03-01 12:00:00", "%Y-%m-%d %H:%M:%S")
    to_timestamp   = datetime.strptime("2025-03-01 13:00:00", "%Y-%m-%d %H:%M:%S")
    input_file = "large_log.txt"
    output_file = "filtered_log.txt"
    extract_log_range(input_file, output_file, from_timestamp, to_timestamp)





```
