"""

~~~


def recursive_lookup_unique(d, key):
    return list({v for k, v in _flatten(d) if k == key})

def _flatten(d):
    if isinstance(d, dict):
        for k, v in d.items():
            if isinstance(v, (dict, list)):
                yield from _flatten(v)
            else:
                yield (k, v)
    elif isinstance(d, list):
        for item in d:
            yield from _flatten(item)




~~~

"""
