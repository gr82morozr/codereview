"""

~~~
import copy

def extract_named_dsls(dsl):
    """
    Extracts all `_name`-tagged entries from `query.bool.should` and returns:
    {
      "searchA": { DSL with only _name == "searchA" kept in should list },
      "searchB": { DSL with only _name == "searchB" kept in should list },
      ...
    }
    """
    result = {}

    # Step 1: locate the should array
    try:
        should_list = dsl["query"]["bool"]["should"]
    except (KeyError, TypeError):
        return {}  # Invalid structure

    # Step 2: collect all unique _name values
    name_to_clause = {}
    for clause in should_list:
        if isinstance(clause, dict) and "_name" in clause:
            name = clause["_name"]
            name_to_clause[name] = clause  # assumes one per _name (or overwrite last one)

    # Step 3: for each name, build a filtered DSL keeping only that clause
    for name, clause in name_to_clause.items():
        new_dsl = copy.deepcopy(dsl)
        try:
            new_dsl["query"]["bool"]["should"] = [clause]
            result[name] = new_dsl
        except (KeyError, TypeError):
            continue  # skip if structure isn't valid

    return result




~~~

"""
