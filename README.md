# codereview
code review

```
import os
import re
from datetime import datetime

# Regular expression to extract timestamp from the start of the line.
# Adjust the regex pattern and the datetime format as needed.
TIMESTAMP_REGEX = re.compile(r'^(\d{4}-\d{2}-\d{2} \d{2}:\d{2}:\d{2})')

def parse_timestamp(ts_str):
    """Convert a timestamp string to a datetime object."""
    return datetime.strptime(ts_str, "%Y-%m-%d %H:%M:%S")

def extract_timestamp(line):
    """Extract and parse the timestamp from a log line (decoded string)."""
    match = TIMESTAMP_REGEX.match(line)
    if match:
        try:
            return parse_timestamp(match.group(1))
        except ValueError:
            return None
    return None

def get_line_start(file, pos):
    """
    Given a file (opened in binary mode) and an arbitrary byte offset pos,
    find the byte offset corresponding to the start of the line that
    contains pos. Lines are assumed to be delimited by the carriage return (b'\r').
    """
    if pos == 0:
        return 0
    # Start at pos and move backwards until we find a b'\r'
    cur = pos
    while cur > 0:
        file.seek(cur - 1)
        byte = file.read(1)
        if byte == b'\r':
            break
        cur -= 1
    return cur

def get_line_info(file, pos):
    """
    Assuming pos is at the beginning of a line,
    read bytes until the next b'\r' (or EOF) is encountered.
    Returns a tuple: (line_start, line_end, line_bytes)
    where line_bytes does not include the trailing b'\r'.
    """
    file.seek(pos)
    line_bytes = bytearray()
    while True:
        char = file.read(1)
        # Stop if we hit the delimiter or EOF.
        if not char or char == b'\r':
            break
        line_bytes.extend(char)
    line_end = file.tell()  # This is right after the delimiter (or EOF)
    return pos, line_end, bytes(line_bytes)

def binary_search_lower_bound(file, target_time):
    """
    Find the byte offset of the first log line whose timestamp is >= target_time.
    The file must be sorted by timestamp.
    """
    file.seek(0, os.SEEK_END)
    file_size = file.tell()
    left = 0
    right = file_size
    candidate = None

    while left < right:
        mid = (left + right) // 2
        # Align mid to the beginning of a line.
        line_start = get_line_start(file, mid)
        # Read the full line
        _, line_end, line_bytes = get_line_info(file, line_start)
        try:
            line_str = line_bytes.decode('utf-8', errors='ignore').strip()
        except Exception:
            # If decoding fails, skip this line.
            left = line_end
            continue

        ts = extract_timestamp(line_str)
        if ts is None:
            # Skip lines that don't parse.
            left = line_end
            continue

        if ts >= target_time:
            # Candidate found; try to search left for an earlier matching line.
            candidate = line_start
            right = line_start
        else:
            # Move to the next line.
            left = line_end

    # If no candidate was found, candidate remains None. In that case, return the file_size.
    return candidate if candidate is not None else file_size

def binary_search_upper_bound(file, target_time):
    """
    Find the byte offset of the start of the last log line whose timestamp is <= target_time.
    The file must be sorted by timestamp.
    """
    file.seek(0, os.SEEK_END)
    file_size = file.tell()
    left = 0
    right = file_size
    candidate = None

    while left < right:
        mid = (left + right) // 2
        # Align mid to the beginning of a line.
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
            # Candidate line qualifies; record it and search right.
            candidate = line_start
            left = line_end
        else:
            right = line_start

    # If no candidate was found, candidate remains None. In that case, return 0.
    return candidate if candidate is not None else 0

def extract_log_range(input_file, output_file, from_timestamp, to_timestamp):
    """
    Extract all log lines (as bytes) between from_timestamp and to_timestamp (inclusive)
    using binary search to find the boundaries. The result is written to output_file.
    """
    with open(input_file, 'rb') as f:
        # Find the lower and upper boundaries by binary search.
        lower_bound = binary_search_lower_bound(f, from_timestamp)
        upper_bound = binary_search_upper_bound(f, to_timestamp)

        if lower_bound is None or lower_bound >= f.tell():
            print("No log lines found for the given start time.")
            return

        # Get the full last line (to include its entire content).
        f.seek(upper_bound)
        _, upper_line_end, _ = get_line_info(f, upper_bound)

        # Read all bytes between the lower_bound and the end of the last qualifying line.
        f.seek(lower_bound)
        bytes_to_read = upper_line_end - lower_bound
        data = f.read(bytes_to_read)

    with open(output_file, 'wb') as out:
        out.write(data)
    print(f"Extracted log range written to: {output_file}")

# Example usage:
if __name__ == '__main__':
    # Set your time range here.
    from_timestamp = datetime.strptime("2025-03-01 12:00:00", "%Y-%m-%d %H:%M:%S")
    to_timestamp   = datetime.strptime("2025-03-01 13:00:00", "%Y-%m-%d %H:%M:%S")
    input_file = "large_log.txt"
    output_file = "filtered_log.txt"
    extract_log_range(input_file, output_file, from_timestamp, to_timestamp)




```
