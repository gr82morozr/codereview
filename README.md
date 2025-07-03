"""

~~~
def find_key_value(d, target_key):
    if isinstance(d, dict):
        for key, value in d.items():
            if key == target_key:
                return value
            result = find_key_value(value, target_key)
            if result is not None:
                return result
    elif isinstance(d, list):
        for item in d:
            result = find_key_value(item, target_key)
            if result is not None:
                return result
    return None


~~~

"""
