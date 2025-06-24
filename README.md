"""




pattern = re.compile(r"result\.(\d{3})\.json")

# List matching files
files = [f for f in os.listdir('.') if pattern.match(f)]

if not files:
    next_filename = "result.001.json"
else:
    # Extract numeric parts and find max
    max_n = max(int(pattern.match(f).group(1)) for f in files)
    next_filename = f"result.{max_n + 1:03d}.json"



"""
