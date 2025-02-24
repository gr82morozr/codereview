# codereview
code review

```
function GetDeepChildBySpec(Inputs, Outputs) {

/*************************************************************
 * Function: GetDeepChildBySpec
 * Author: [Your Name]
 * Purpose:
 *   This function searches a given ParentPropertySet (at any depth) for child nodes
 *   that match a specified search specification provided in the ChildType input.
 *   The ChildType input can be:
 *     1. A simple string representing the name of the child type.
 *        (Equivalent to searching for "//ChildType" at any level.)
 *     2. A path string (e.g., "0/1/ChildType") where numeric tokens represent
 *        child indices at successive levels.
 *     3. A path string with wildcards (e.g., "0/*/1/ChildType") where "*" matches
 *        any child at that level.
 *   Additionally, the ChildValue input is used to filter on the child's "Value" property.
 *   Wildcards ("", "*") in ChildValue indicate that any value is acceptable.
 *   Note: Both ChildType (if simple) and ChildValue cannot be wildcards simultaneously.
 *
 * Usage:
 *   Input Arguments:
 *   --------------------------
 *     1. ParentPropertySet (Mandatory):
 *        The root PropertySet object to search within.
 *
 *     2. ChildType (Mandatory):
 *        Either a simple string (e.g., "ChildTypeA") or a path specification
 *        (e.g., "0/1/ChildTypeA" or "0/*/1/ChildTypeA") to locate the target child.
 *
 *     3. ChildValue (Mandatory):
 *        The value to match on the child's "Value" property (e.g., "ChildValueA").
 *        If "*" or empty, then any value is acceptable.
 *
 *     4. Output_Delimiter (Optional):
 *        The delimiter used to join output strings (default is ".").
 *
 *   Output Arguments:
 *   --------------------------
 *     1. ChildExists:
 *        "Y" if one or more matching child nodes are found; "N" otherwise.
 *
 *     2. ChildCount:
 *        The total count of matching child nodes.
 *
 *     3. ChildConcatedTypes:
 *        A concatenated string of matching child types, joined by the Output_Delimiter.
 *
 *     4. ChildConcatedValues:
 *        A concatenated string of matching child values, joined by the Output_Delimiter.
 *
 * Example Usage:
 *   Input:
 *     ParentPropertySet = A PropertySet object,
 *     ChildType = "0/*/1/ChildTypeA",
 *     ChildValue = "ChildValueA",
 *     Output_Delimiter = "|" (optional).
 *   Output:
 *     ChildExists = "Y",
 *     ChildCount = "2",
 *     ChildConcatedTypes = "ChildTypeA|ChildTypeA",
 *     ChildConcatedValues = "ChildValueA|ChildValueA".
 *
 * Notes:
 *   - When ChildType is provided as a simple string (without "/"), the function performs a full
 *     depth-first search for any descendant whose type equals ChildType (unless ChildType is "*" which
 *     indicates a wildcard on type).
 *   - When ChildType is provided as a path, the tokens in the path are processed in order.
 *     Numeric tokens select a child by index, "*" selects all children at that level, and literal tokens
 *     match the child's type.
 *   - Both ChildType (if simple) and ChildValue cannot be wildcards ("", "*") at the same time.
 *
 * Update History:
 *   Initial Version: [Current Date] by [Your Name]
 **************************************************************/

  try {
    /*************************************************************
     * Input Extraction
     *************************************************************/
    // Retrieve input parameters using the sample pattern.
    var childTypeInput  = Inputs.GetProperty("ChildType");       // Retrieve ChildType
    var childValueInput = Inputs.GetProperty("ChildValue");      // Retrieve ChildValue
    var outputDelimiter = Inputs.GetProperty("Output_Delimiter");  // Retrieve Output_Delimiter

    // Create a new PropertySet for ParentPropertySet.
    var parentPS = TheApplication().NewPropertySet();
    // Loop through Inputs' children to locate the ParentPropertySet.
    for (var i = 0; i < Inputs.GetChildCount(); i++) {
      var tempChild = Inputs.GetChild(i);
      if (tempChild.GetType() === "ParentPropertySet") {
        parentPS = tempChild;
        break;
      }
    }

    /*************************************************************
     * Input Validation and Default Setting
     *************************************************************/
    // Validate that ParentPropertySet is provided.
    if (parentPS == null) {
      TheApplication().RaiseErrorText("Error: ParentPropertySet is not provided in the inputs.");
    }
    // Set default delimiter if not provided.
    if (!outputDelimiter || outputDelimiter === "") {
      outputDelimiter = ".";
    }
    // Determine if ChildValue is a wildcard (null, empty string, or "*").
    var isChildValueWildcard = (childValueInput === null || childValueInput === "" || childValueInput === "*");

    // Determine if ChildType input is a path specification.
    // If it contains a "/" then it's treated as a path; otherwise, it's a simple search string.
    var isPathSpec = (childTypeInput != null && childTypeInput.indexOf("/") !== -1);
    // For simple search, determine if ChildType is a wildcard.
    var isChildTypeWildcard = (!isPathSpec && (childTypeInput === null || childTypeInput === "" || childTypeInput === "*"));

    // Validate that both ChildType (if simple wildcard) and ChildValue are not wildcards simultaneously.
    if (isChildTypeWildcard && isChildValueWildcard) {
      TheApplication().RaiseErrorText("Error: Both ChildType and ChildValue cannot be empty or wildcard.");
    }

    /*************************************************************
     * Initialize Result Variables
     *************************************************************/
    var childCount = 0;                         // Count of matching child nodes.
    var childConcatenatedTypes = "";              // Accumulated matching child types.
    var childConcatenatedValues = "";             // Accumulated matching child values.
    var matchingCandidates = [];                  // Array to store matching candidate PropertySets.

    /*************************************************************
     * Search Logic: Simple Search vs. Path Specification
     *************************************************************/
    if (!isPathSpec) {
      // Simple Search: search the entire tree for nodes that match the type and value criteria.
      // Use an iterative depth-first search (DFS) using a stack.
      var stack = [];
      // Initialize the stack with all immediate children of the ParentPropertySet.
      for (var i = 0; i < parentPS.GetChildCount(); i++) {
        stack.push(parentPS.GetChild(i));
      }
      // Iterate while there are nodes in the stack.
      while (stack.length > 0) {
        var currentNode = stack.pop();
        // Determine if the current node matches the type criteria.
        var typeMatches = false;
        if (isChildTypeWildcard) {
          // Wildcard: match any type.
          typeMatches = true;
        } else {
          if (currentNode.GetType() === childTypeInput) {
            typeMatches = true;
          }
        }
        // Determine if the current node matches the value criteria.
        var valueMatches = false;
        var currentValue = currentNode.GetProperty("Value");
        if (isChildValueWildcard) {
          // Wildcard: match any value.
          valueMatches = true;
        } else {
          if (currentValue === childValueInput) {
            valueMatches = true;
          }
        }
        // If both criteria are met, add the node to matching candidates.
        if (typeMatches && valueMatches) {
          matchingCandidates.push(currentNode);
        }
        // Push all children of the current node onto the stack for further search.
        for (var j = 0; j < currentNode.GetChildCount(); j++) {
          stack.push(currentNode.GetChild(j));
        }
      }
    } else {
      // Path Specification: ChildType is provided as a path (e.g., "0/*/1/ChildTypeA").
      // Split the path into tokens.
      var tokens = childTypeInput.split("/");
      // Initialize candidate list with the ParentPropertySet.
      var candidates = [parentPS];
      // Process each token in the path sequentially.
      for (var i = 0; i < tokens.length; i++) {
        var token = tokens[i];
        var nextCandidates = [];
        // Process numeric tokens, wildcards, and literal type tokens.
        if (!isNaN(token)) {
          // Token is numeric: select the child at the given index.
          var index = parseInt(token, 10);
          for (var j = 0; j < candidates.length; j++) {
            var candidate = candidates[j];
            if (candidate.GetChildCount() > index) {
              nextCandidates.push(candidate.GetChild(index));
            }
          }
        } else if (token === "*") {
          // Token is a wildcard: add all children of the current candidates.
          for (var j = 0; j < candidates.length; j++) {
            var candidate = candidates[j];
            for (var k = 0; k < candidate.GetChildCount(); k++) {
              nextCandidates.push(candidate.GetChild(k));
            }
          }
        } else {
          // Token is a literal: select children whose type matches the token.
          for (var j = 0; j < candidates.length; j++) {
            var candidate = candidates[j];
            for (var k = 0; k < candidate.GetChildCount(); k++) {
              var child = candidate.GetChild(k);
              if (child.GetType() === token) {
                nextCandidates.push(child);
              }
            }
          }
        }
        // Update the candidate list for the next token.
        candidates = nextCandidates;
        // If no candidates remain, exit early.
        if (candidates.length === 0) {
          break;
        }
      }
      // After processing all tokens, the remaining candidates match the ChildType path.
      // Apply the ChildValue filter to each candidate.
      for (var i = 0; i < candidates.length; i++) {
        var candidate = candidates[i];
        var candidateValue = candidate.GetProperty("Value");
        var valueMatches = false;
        if (isChildValueWildcard) {
          valueMatches = true;
        } else {
          if (candidateValue === childValueInput) {
            valueMatches = true;
          }
        }
        if (valueMatches) {
          matchingCandidates.push(candidate);
        }
      }
    }

    /*************************************************************
     * Aggregate Results from Matching Candidates
     *************************************************************/
    for (var i = 0; i < matchingCandidates.length; i++) {
      var candidate = matchingCandidates[i];
      childCount++;
      // Accumulate the candidate's type.
      var candidateType = candidate.GetType();
      if (childConcatenatedTypes !== "") {
        childConcatenatedTypes = childConcatenatedTypes + outputDelimiter + candidateType;
      } else {
        childConcatenatedTypes = candidateType;
      }
      // Accumulate the candidate's value.
      var candidateValue = candidate.GetProperty("Value");
      if (childConcatenatedValues !== "") {
        childConcatenatedValues = childConcatenatedValues + outputDelimiter + candidateValue;
      } else {
        childConcatenatedValues = candidateValue;
      }
    }

    /*************************************************************
     * Prepare and Set Output Results
     *************************************************************/
    var childExists = (childCount > 0) ? "Y" : "N";
    Outputs.SetProperty("ChildExists", childExists);
    Outputs.SetProperty("ChildCount", childCount.toString());
    Outputs.SetProperty("ChildConcatedTypes", childConcatenatedTypes);
    Outputs.SetProperty("ChildConcatedValues", childConcatenatedValues);

  } catch (e) {
    /*************************************************************
     * Error Handling
     *************************************************************/
    TheApplication().RaiseErrorText("Error in GetDeepChildBySpec: " + e.toString());
  }

  return;
}

```
