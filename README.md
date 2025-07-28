"""

~~~


# Prepare results
if float_values:
    count = len(float_values)
    total = sum(float_values)
    average = total / count
    max_val = max(float_values)
    min_val = min(float_values)

    result = (
        f"Files matched: {count}\n"
        f"Average: {average}\n"
        f"Max: {max_val}\n"
        f"Min: {min_val}\n"
    )

    print(result)

    # Save to result.txt
    result_path = os.path.join(folder_path, "result.txt")
    with open(result_path, 'w') as out_file:
        out_file.write(result)
else:
    print("No valid float files found.")
~~~

"""
