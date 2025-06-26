"""


import re
import json
import subprocess

# --- CONFIG ---
raw_path = "pipeline_raw.txt"     # input: raw (Kibana-style) pipeline
clean_path = "pipeline_clean.json"  # output: valid JSON
es_url = "http://localhost:9200/_ingest/pipeline/my-pipeline-id"

# --- STEP 1: Read raw text (not necessarily valid JSON) ---
with open(raw_path, "r", encoding="utf-8") as f:
    raw_text = f.read()

# --- STEP 2: Replace multiline script source blocks: source: """ ... """ → escaped string ---
def escape_script_block(match):
    script_content = match.group(1)
    escaped = (
        script_content
        .replace("\\", "\\\\")  # escape backslashes first
        .replace("\"", "\\\"")  # escape quotes
        .replace("\r", "")      # normalize line endings
        .replace("\n", "\\n")   # escape newlines
    )
    return f'"source": "{escaped}"'

# Find: source: """(multiline content)""" (Kibana-style triple-quoted script)
fixed_text = re.sub(
    r'source\s*:\s*"""\s*(.*?)\s*"""',
    escape_script_block,
    raw_text,
    flags=re.DOTALL
)

# --- STEP 3: Try parsing as strict JSON ---
try:
    data = json.loads(fixed_text)
except json.JSONDecodeError as e:
    print(f"❌ JSON parsing failed after cleaning: {e}")
    print(f"👉 Suggest checking line {e.lineno}: {e.msg}")
    exit(1)

# --- STEP 4: Save as valid JSON ---
with open(clean_path, "w", encoding="utf-8", newline="\n") as f:
    json.dump(data, f, indent=2)
print(f"✅ Clean JSON saved to: {clean_path}")

# --- STEP 5: Send to Elasticsearch using curl ---
curl_cmd = [
    "curl", "-X", "PUT", es_url,
    "-H", "Content-Type: application/json",
    "--data-binary", f"@{clean_path}"
]

print("🚀 Sending to Elasticsearch...")
result = subprocess.run(curl_cmd, capture_output=True, text=True)

print("✅ Curl STDOUT:")
print(result.stdout)
if result.stderr:
    print("⚠️ Curl STDERR:")
    print(result.stderr)

"""
