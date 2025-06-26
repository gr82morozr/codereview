"""


def safe_write(filepath, html, retries=3, delay=1):
    attempt = 0
    while attempt < retries:
        try:
            with open(filepath, "w") as file:
                file.write(html)
            return True  # success
        except PermissionError as e:
            print(f"[Attempt {attempt + 1}] PermissionError: {e}")
            time.sleep(delay)
            attempt += 1
    print("Failed to write file after several attempts.")
    return False  # failed

"""
