"""

~~~

import os
import fnmatch

folder_path = "/your/folder/path"  # replace with your actual folder
pattern = "output*.json"

float_values = []

# Iterate all files matching the pattern
for filename in os.listdir(folder_path):
    if fnmatch.fnmatch(filename, pattern):
        file_path = os.path.join(folder_path, filename)
        if os.path.isfile(file_path):
            with open(file_path, 'r') as f:
                try:
                    value = float(f.read().strip())
                    float_values.append(value)
                except ValueError:
                    print(f"Invalid float in file: {filename}")

# Compute and print results
if float_values:
    average = sum(float_values) / len(float_values)
    max_val = max(float_values)
    min_val = min(float_values)

    print(f"Files matched: {len(float_values)}")
    print(f"Average: {average}")
    print(f"Max: {max_val}")
    print(f"Min: {min_val}")
else:
    print("No valid float files found.")

~~~

"""
