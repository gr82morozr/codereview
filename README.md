"""

~~~


#!/usr/bin/env python3
# -*- coding: utf-8 -*-

# ======================= USER-CONFIGURABLE CONSTANTS =======================
TEMPLATE_PATH = "template.xml"        # XML template file that contains the placeholder
OUTPUT_PATH   = "out.xml"             # Where the filled XML will be written
PLACEHOLDER   = "{{BASE64_DATA}}"     # Placeholder token to be replaced in the template
# ==========================================================================

import sys           # to read the single CLI argument (MB)
import os            # to generate random bytes with os.urandom
import base64        # to convert the random bytes to Base64
from pathlib import Path  # for simple, robust file I/O

def main():
    # --- 1) validate & read the single argument (integer MB) ---
    if len(sys.argv) != 2:  # require exactly one argument: MB
        print(f"Usage: {Path(sys.argv[0]).name} <MB>", file=sys.stderr)  # show usage if wrong
        sys.exit(2)  # exit with error code

    try:
        mb = int(sys.argv[1])  # convert the provided argument to integer MB
    except ValueError:
        print("Error: <MB> must be an integer.", file=sys.stderr)  # inform user on invalid input
        sys.exit(2)  # exit with error

    if mb <= 0:
        print("Error: <MB> must be > 0.", file=sys.stderr)  # forbid zero/negative sizes
        sys.exit(2)

    # --- 2) compute total bytes to generate (MB -> bytes) ---
    total_bytes = mb * 1024 * 1024  # 1 MB = 1,048,576 bytes (MiB style)

    # --- 3) generate exactly that many cryptographically random bytes ---
    raw = os.urandom(total_bytes)  # returns a bytes object of length total_bytes

    # --- 4) base64-encode the bytes in one shot (simple & fine up to ~50MB) ---
    b64_bytes = base64.b64encode(raw)  # convert binary data to Base64 bytes

    # --- 5) decode Base64 bytes to a regular text string for XML insertion ---
    b64_text = b64_bytes.decode("ascii")  # Base64 is ASCII-safe; decode to str

    # --- 6) load the XML template (UTF-8) ---
    template_path = Path(TEMPLATE_PATH)  # wrap string path with Path for convenience
    if not template_path.is_file():      # ensure the template exists
        print(f"Error: template not found: {template_path}", file=sys.stderr)
        sys.exit(1)

    template_str = template_path.read_text(encoding="utf-8")  # read entire template as text

    # --- 7) replace ALL placeholder occurrences with the Base64 string ---
    if PLACEHOLDER not in template_str:  # warn if placeholder is missing
        print(f"Warning: placeholder '{PLACEHOLDER}' not found in template.", file=sys.stderr)
    filled_str = template_str.replace(PLACEHOLDER, b64_text)  # simple global replace

    # --- 8) write the filled XML to the output file (UTF-8) ---
    Path(OUTPUT_PATH).write_text(filled_str, encoding="utf-8")  # save to disk

    # --- 9) print a tiny success message ---
    print(f"[OK] Generated {mb} MB random data, filled into '{OUTPUT_PATH}'.")

# standard boilerplate to run main() when invoked as script
if __name__ == "__main__":
    main()





~~~

"""
