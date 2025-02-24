# codereview
code review

```
function GetDeepChildBySpec(Inputs, Outputs) {

/*************************************************************
 * Function: GetDeepChildBySpec
 * Author: [Your Name]
 * Purpose:
 *   This function searches a given ParentPropertySet (at any depth) for child nodes 
 *   that match a specified search specification provided in the ChildType input,
 *   and filters them by the ChildValue input. The ChildType input can be:
 *     1. A simple string representing the name of the child type (e.g., "ChildTypeA"),
 *        which is equivalent to searching for "//ChildTypeA" at any level.
 *     2. A path string (e.g., "0/1/ChildTypeA") where numeric tokens represent child 
 *        indices at successive levels.
 *     3. A path string with wildcards (e.g., "0/*/1/ChildTypeA") where "*" matches any child 
 *        at that level.
 *
 *   The ChildValue input can be:
 *     1. A simple string (e.g., "ChildValueA") to match exactly.
 *     2. A pattern specified with the prefix "RegEx:" (e.g., "RegEx:\d+\.ChildValueA") in which
 *        case the remainder is treated as a regular expression to match the child's "Value" property.
 *     3. A wildcard ("", "*") indicating that any value is acceptable.
 *
 *   Note: Both ChildType (if provided as a simple string) and ChildValue cannot be wildcards
 *         simultaneously.
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
 *        The value to match on the child's "Value" property. It can be:
 *           - A simple string (e.g., "ChildValueA"),
 *           - A regular expression pattern prefixed with "RegEx:" (e.g., "RegEx:\d+\.ChildValueA"),
 *           - Or a wildcard ("", "*") for any value.
 *
 *     4. OutputDelimiter (Optional):
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
 *        A concatenated string of matching child types, joined by the OutputDelimiter.
 *
 *     4. ChildConcatedValues:
 *        A concatenated string of matching child values, joined by the OutputDelimiter.
 *
 * Example Usage:
 *   Input:
 *     ParentPropertySet = A PropertySet object,
 *     ChildType = "0/*/1/ChildTypeA",
 *     ChildValue = "RegEx:\\d+\\.ChildValueA",
 *     OutputDelimiter = "|" (optional).
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
 *   - When ChildType is provided as a path, the tokens in the path are processed sequentially.
 *     Numeric tokens select a child by index, "*" selects all children at that level, and literal tokens
 *     match the child's type.
 *   - The ChildValue input supports a "RegEx:" prefix to enable pattern matching.
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
    var childValueInput = Inputs.GetProperty("ChildValue");        // Retrieve ChildValue
    var outputDelimiter = Inputs.GetProperty("OutputDelimiter");   // Retrieve OutputDelimiter

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

    // Determine if ChildType input is a path specification.
    // If it contains a "/" then it's treated as a path; otherwise, it's a simple search string.
    var isPathSpec = (childTypeInput != null && childTypeInput.indexOf("/") !== -1);
    // For a simple search, determine if ChildType is a wildcard.
    var isChildTypeWildcard = false;
    if (!isPathSpec) {
      isChildTypeWildcard = (childTypeInput === null || childTypeInput === "" || childTypeInput === "*");
    }

    // Process ChildValue input.
    // Check if ChildValue is specified as a regular expression (prefixed with "RegEx:").
    var isRegex = false;
    var regex = null;
    if (childValueInput != null && childValueInput.indexOf("RegEx:") === 0) {
      isRegex = true;
      var regexPattern = childValueInput.substring(6);  // Remove the "RegEx:" prefix.
      regex = new RegExp(regexPattern);
    }
    // Determine if ChildValue is a wildcard (only applicable if not a RegEx).
    var isChildValueWildcard = false;
    if (!isRegex) {
      isChildValueWildcard = (childValueInput === null || childValueInput === "" || childValueInput === "*");
    }

    // Validate that both ChildType (if simple) and ChildValue are not wildcards simultaneously.
    if (!isPathSpec && isChildTypeWildcard && isChildValueWildcard) {
      TheApplication().RaiseErrorText("Error: Both ChildType and ChildValue cannot be empty or wildcard.");
    }

    /*************************************************************
     * Initialize Result Variables
     *************************************************************/
    var childCount = 0;                // Count of matching child nodes.
    var childConcatedTypes = "";         // Accumulated matching child types.
    var childConcatedValues = "";        // Accumulated matching child values.
    var matchingCandidates = [];         // Array to store matching candidate PropertySets.

    /*************************************************************
     * Search Logic: Simple Search vs. Path Specification
     *************************************************************/
    if (!isPathSpec) {
      // Simple Search: perform an iterative depth-first search on the entire tree.
      var stack = [];
      // Initialize the stack with all immediate children of the ParentPropertySet.
      for (var i = 0; i < parentPS.GetChildCount(); i++) {
        stack.push(parentPS.GetChild(i));
      }
      // Iterate while there are nodes in the stack.
      while (stack.length > 0) {
        var currentNode = stack.pop();
        // Check if the current node matches the ChildType criteria.
        var typeMatches = false;
        if (isChildTypeWildcard) {
          typeMatches = true;  // Wildcard: match any type.
        } else {
          if (currentNode.GetType() === childTypeInput) {
            typeMatches = true;
          }
        }
        // Check if the current node matches the ChildValue criteria.
        var valueMatches = false;
        var currentValue = currentNode.GetProperty("Value");
        if (isChildValueWildcard) {
          valueMatches = true;  // Wildcard: match any value.
        } else if (isRegex) {
          // Use the regular expression to test the current value.
          valueMatches = regex.test(currentValue);
        } else {
          if (currentValue === childValueInput) {
            valueMatches = true;
          }
        }
        // If both criteria match, add the node to the matching candidates.
        if (typeMatches && valueMatches) {
          matchingCandidates.push(currentNode);
        }
        // Push all children of the current node onto the stack.
        for (var j = 0; j < currentNode.GetChildCount(); j++) {
          stack.push(currentNode.GetChild(j));
        }
      }
    } else {
      // Path Specification: process the ChildType input as a path (e.g., "0/*/1/ChildTypeA").
      var tokens = childTypeInput.split("/");
      // Start with the ParentPropertySet as the sole candidate.
      var candidates = [parentPS];
      // Process each token in sequence.
      for (var i = 0; i < tokens.length; i++) {
        var token = tokens[i];
        var nextCandidates = [];
        if (!isNaN(token)) {
          // Numeric token: select the child at the given index.
          var index = parseInt(token, 10);
          for (var j = 0; j < candidates.length; j++) {
            var candidate = candidates[j];
            if (candidate.GetChildCount() > index) {
              nextCandidates.push(candidate.GetChild(index));
            }
          }
        } else if (token === "*") {
          // Wildcard token: select all children of the current candidates.
          for (var j = 0; j < candidates.length; j++) {
            var candidate = candidates[j];
            for (var k = 0; k < candidate.GetChildCount(); k++) {
              nextCandidates.push(candidate.GetChild(k));
            }
          }
        } else {
          // Literal token: select children whose type matches the token.
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
        // Update candidates for the next token.
        candidates = nextCandidates;
        if (candidates.length === 0) {
          break;
        }
      }
      // After processing all tokens, filter the resulting candidates by ChildValue.
      for (var i = 0; i < candidates.length; i++) {
        var candidate = candidates[i];
        var candidateValue = candidate.GetProperty("Value");
        var valueMatches = false;
        if (isChildValueWildcard) {
          valueMatches = true;
        } else if (isRegex) {
          valueMatches = regex.test(candidateValue);
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
      if (childConcatedTypes !== "") {
        childConcatedTypes = childConcatedTypes + outputDelimiter + candidateType;
      } else {
        childConcatedTypes = candidateType;
      }
      // Accumulate the candidate's value.
      var candidateValue = candidate.GetProperty("Value");
      if (childConcatedValues !== "") {
        childConcatedValues = childConcatedValues + outputDelimiter + candidateValue;
      } else {
        childConcatedValues = candidateValue;
      }
    }

    /*************************************************************
     * Prepare and Set Output Results
     *************************************************************/
    var childExists = (childCount > 0) ? "Y" : "N";
    Outputs.SetProperty("ChildExists", childExists);
    Outputs.SetProperty("ChildCount", childCount.toString());
    Outputs.SetProperty("ChildConcatedTypes", childConcatedTypes);
    Outputs.SetProperty("ChildConcatedValues", childConcatedValues);

  } catch (e) {
    /*************************************************************
     * Error Handling
     *************************************************************/
    TheApplication().RaiseErrorText("Error in GetDeepChildBySpec: " + e.toString());
  }

  return;
}


```
