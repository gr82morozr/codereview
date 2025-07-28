"""

~~~


folder_path = "/your/folder/path"  # <-- change to your folder path
float_values = []

# Manual pattern match for 'output*.json'
for filename in os.listdir(folder_path):
    if filename.startswith("output") and filename.endswith(".json"):
        file_path = os.path.join(folder_path, filename)
        if os.path.isfile(file_path):
            with open(file_path, 'r') as f:
                try:
                    value = float(f.read().strip())
                    float_values.append(value)
                except ValueError:
                    print(f"Skipped invalid float in file: {filename}")

# Print results
if float_values:
    count = len(float_values)
    total = sum(float_values)
    avg = total / count
    print(f"Files matched: {count}")
    print(f"Average: {avg}")
    print(f"Max: {max(float_values)}")
    print(f"Min: {min(float_values)}")
else:
    print("No valid float files found.")
~~~

"""
