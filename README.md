"""


import json

def extract_fields(mapping, prefix=""):
    fields = []
    for field_name, field_info in mapping.items():
        full_name = f"{prefix}.{field_name}" if prefix else field_name
        if "properties" in field_info:
            # It's an object — recurse
            fields.extend(extract_fields(field_info["properties"], full_name))
        else:
            # It's a leaf field
            fields.append(full_name)
    return fields

# Load your mapping JSON (from file, or API response)
with open("mapping.json", "r") as f:
    mapping_json = json.load(f)

# Navigate to the properties section
# If this is a full API response from /index/_mapping:
index_name = list(mapping_json.keys())[0]
mapping_props = mapping_json[index_name]["mappings"]["properties"]

# Extract all fields
all_fields = extract_fields(mapping_props)

# Print each field
for field in all_fields:
    print(field)



"""
