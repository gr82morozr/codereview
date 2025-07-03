"""

~~~
def extract_named_dsls(dsl):
    """
    Extracts all `_name`-tagged queries from the DSL and returns:
    {
      "name1": { ...DSL containing only name1... },
      "name2": { ...DSL containing only name2... },
      ...
    }
    """
    import copy

    names = set()

    def collect_names(node):
        if isinstance(node, dict):
            for key, value in node.items():
                if key == "_name" and isinstance(value, str):
                    names.add(value)
                else:
                    collect_names(value)
        elif isinstance(node, list):
            for item in node:
                collect_names(item)

    def filter_dsl(node, target_name):
        if isinstance(node, dict):
            if "_name" in node and node["_name"] != target_name:
                return None

            result = {}
            for key, value in node.items():
                child = filter_dsl(value, target_name)
                if child is not None:
                    result[key] = child
            return result if result else None

        elif isinstance(node, list):
            new_list = []
            for item in node:
                child = filter_dsl(item, target_name)
                if child is not None:
                    new_list.append(child)
            return new_list if new_list else None

        else:
            return node

    # Step 1: collect all names
    collect_names(dsl)

    # Step 2: build result dict
    result = {}
    for name in names:
        filtered = filter_dsl(copy.deepcopy(dsl), name)
        if filtered:
            result[name] = filtered

    return result



~~~

"""
