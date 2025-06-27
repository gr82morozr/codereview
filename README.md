"""


import time
import sys

def replace_text_in_file(file_path, target, replacement):
    retry_count = 0
    func_name = 'replace_text_in_file'

    while True:
        try:
            # Read original content
            with open(file_path, 'r', encoding='utf-8') as file:
                content = file.read()

            # Replace the text
            content = content.replace(target, replacement)

            # Write updated content back
            with open(file_path, 'w', encoding='utf-8') as file:
                file.write(content)

            # Success
            print(f"\n{func_name} - completed")
            break

        except PermissionError:
            retry_count += 1
            sys.stdout.write(f"\r{func_name} - retry - {retry_count}")
            sys.stdout.flush()
            time.sleep(0.5)  # short delay before retrying

# Example usage
replace_text_in_file("example.txt", "old_text", "new_text")






"""
