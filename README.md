"""

~~~
import copy

def get_search_name_list(dsl):
    """
    Recursively extract all '_name' values from the DSL.
    """
    names = set()

    def recurse(node):
        if isinstance(node, dict):
            for key, value in node.items():
                if key == "_name" and isinstance(value, str):
                    names.add(value)
                else:
                    recurse(value)
        elif isinstance(node, list):
            for item in node:
                recurse(item)

    recurse(dsl)
    return list(names)


def get_named_dsl(dsl, target_name):
    """
    Return a copy of the DSL with only clauses that contain _name == target_name.
    All other named clauses are removed.
    """
    def recurse_filter(node):
        if isinstance(node, dict):
            # If this is a named clause but not the target, remove it
            if "_name" in node and node["_name"] != target_name:
                return None

            filtered = {}
            for key, value in node.items():
                child = recurse_filter(value)
                if child is not None:
                    filtered[key] = child
            return filtered if filtered else None

        elif isinstance(node, list):
            new_list = []
            for item in node:
                child = recurse_filter(item)
                if child is not None:
                    new_list.append(child)
            return new_list if new_list else None

        else:
            return node

    dsl_copy = copy.deepcopy(dsl)
    return recurse_filter(dsl_copy)



~~~

"""
