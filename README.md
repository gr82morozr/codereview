# codereview
code review

```
def get_line_ts(file, pos, file_size, direction):
    """
    Searches from a given byte offset (pos) in a file (opened in binary mode)
    for the first complete line that contains a valid timestamp (based on a
    regular expression). Uses Windows CRLF (b'\\r\\n') as the line delimiter.

    :param file:      File handle (binary mode).
    :param pos:       Starting byte offset in the file.
    :param file_size: Total bytes in the file (e.g., via f.seek(0,2); f.tell()).
    :param direction: "LEFT" or "RIGHT", indicating the search direction.

    :return: (line_start, line_end, line_str, ts) if a valid timestamped line is found, else None
       - line_start: The byte offset where this line begins
       - line_end:   The byte offset right after the line's CRLF delimiter
       - line_str:   The decoded line content (stripped of trailing CRLF)
       - ts:         The extracted timestamp string from the line
    """
    import re

    # Regex for a timestamp at the start of the line: YYYY-MM-DD HH:MM:SS
    TIMESTAMP_REGEX = re.compile(r'^(\d{4}-\d{2}-\d{2} \d{2}:\d{2}:\d{2})')

    direction = direction.upper()
    current_pos = pos

    while True:
        # ---------------------------
        # 1) Check bounds by direction
        # ---------------------------
        if direction == "RIGHT":
            if current_pos >= file_size:
                break
        elif direction == "LEFT":
            if current_pos <= 0:
                break
        else:
            raise ValueError("Invalid direction. Use 'LEFT' or 'RIGHT'.")

        # -----------------------------------------------------
        # 2) Find the start of the line by scanning backward
        #    until we see CRLF (b'\r\n') or reach file start.
        # -----------------------------------------------------
        line_start = current_pos
        if line_start > 1:
            while line_start > 1:
                file.seek(line_start - 2)
                if file.read(2) == b'\r\n':
                    # Found the CRLF for the previous line, so our line starts here
                    break
                line_start -= 1
        else:
            # If near the start of the file, just set to 0
            line_start = 0

        # -----------------------
        # 3) Read the complete line
        # -----------------------
        file.seek(line_start)
        line_bytes = file.readline()  # Reads until b'\n'
        line_end = file.tell()

        # -------------------------
        # 4) Strip trailing CRLF (or LF)
        # -------------------------
        if line_bytes.endswith(b'\r\n'):
            trimmed_line_bytes = line_bytes[:-2]
        elif line_bytes.endswith(b'\n'):
            trimmed_line_bytes = line_bytes[:-1]
        else:
            trimmed_line_bytes = line_bytes

        # 5) Decode to string (ignore errors)
        try:
            line_str = trimmed_line_bytes.decode('utf-8', errors='ignore').strip()
        except:
            line_str = ""

        # 6) Check for timestamp at start of line
        match = TIMESTAMP_REGEX.match(line_str)
        if match:
            ts = match.group(1)
            return (line_start, line_end, line_str, ts)

        # -------------------------------------
        # 7) Move to the next iteration
        # -------------------------------------
        if direction == "RIGHT":
            # Jump forward: next search position is right after this line
            current_pos = line_end
        else:  # direction == "LEFT"
            # Jump backward: next search position is just before this line
            current_pos = line_start - 1

    # If we exit the loop, no valid timestamped line was found in that direction
    return None





```
