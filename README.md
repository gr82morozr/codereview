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
 *   For each matching candidate, the function returns the full path (concatenation of types from 
 *   the root ParentPropertySet down to the matching node) in the output "ChildConcatedTypes". 
 *   This full path maps one-to-one with the corresponding value in "ChildConcatedValues".
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
 *        A concatenated string of the full paths (each path is a series of type names 
 *        separated by "/") for matching child nodes, joined by the OutputDelimiter.
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
 *     ChildConcatedTypes = "RootType/Type1/Type2/ChildTypeA|RootType/Type1/Type4/ChildTypeA",
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
    var childConcatedTypes = "";         // Accumulated full paths of matching child types.
    var childConcatedValues = "";        // Accumulated matching child values.
    var matchingCandidates = [];         // Array to store objects { node, path } for matching candidates.

    /*************************************************************
     * Search Logic: Simple Search vs. Path Specification
     *************************************************************/
    if (!isPathSpec) {
      // Simple Search: perform an iterative depth-first search on the entire tree.
      // We'll use a stack that holds objects with the node and its full path.
      var stack = [];
      // Initialize the stack with all immediate children of the ParentPropertySet.
      // Full path is: ParentPropertySet.GetType() + "/" + child.GetType()
      var rootType = parentPS.GetType();
      for (var i = 0; i < parentPS.GetChildCount(); i++) {
        var child = parentPS.GetChild(i);
        stack.push({ node: child, path: rootType + "/" + child.GetType() });
      }
      // Iterate while there are nodes in the stack.
      while (stack.length > 0) {
        var currentObj = stack.pop();
        var currentNode = currentObj.node;
        var currentPath = currentObj.path;
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
          valueMatches = regex.test(currentValue);
        } else {
          if (currentValue === childValueInput) {
            valueMatches = true;
          }
        }
        // If both criteria match, add the current node and its full path to matching candidates.
        if (typeMatches && valueMatches) {
          matchingCandidates.push({ node: currentNode, path: currentPath });
        }
        // Push all children of the current node onto the stack with updated full path.
        for (var j = 0; j < currentNode.GetChildCount(); j++) {
          var childObj = currentNode.GetChild(j);
          // Update the path: currentPath + "/" + child's type.
          var newPath = currentPath + "/" + childObj.GetType();
          stack.push({ node: childObj, path: newPath });
        }
      }
    } else {
      // Path Specification: process the ChildType input as a path (e.g., "0/*/1/ChildTypeA").
      // We'll use an array of candidate objects { node, path }.
      // Initialize with the ParentPropertySet as the starting candidate.
      var candidates = [{ node: parentPS, path: parentPS.GetType() }];
      // Split the path into tokens.
      var tokens = childTypeInput.split("/");
      // Process each token sequentially.
      for (var i = 0; i < tokens.length; i++) {
        var token = tokens[i];
        var nextCandidates = [];
        if (!isNaN(token)) {
          // Numeric token: select the child at the given index.
          var index = parseInt(token, 10);
          for (var j = 0; j < candidates.length; j++) {
            var candidate = candidates[j];
            if (candidate.node.GetChildCount() > index) {
              var childNode = candidate.node.GetChild(index);
              // Update full path.
              nextCandidates.push({ node: childNode, path: candidate.path + "/" + childNode.GetType() });
            }
          }
        } else if (token === "*") {
          // Wildcard token: select all children of the current candidates.
          for (var j = 0; j < candidates.length; j++) {
            var candidate = candidates[j];
            for (var k = 0; k < candidate.node.GetChildCount(); k++) {
              var childNode = candidate.node.GetChild(k);
              nextCandidates.push({ node: childNode, path: candidate.path + "/" + childNode.GetType() });
            }
          }
        } else {
          // Literal token: select children whose type matches the token.
          for (var j = 0; j < candidates.length; j++) {
            var candidate = candidates[j];
            for (var k = 0; k < candidate.node.GetChildCount(); k++) {
              var childNode = candidate.node.GetChild(k);
              if (childNode.GetType() === token) {
                nextCandidates.push({ node: childNode, path: candidate.path + "/" + childNode.GetType() });
              }
            }
          }
        }
        // Update candidates for next iteration.
        candidates = nextCandidates;
        if (candidates.length === 0) {
          break;
        }
      }
      // After processing tokens, filter the candidates by ChildValue.
      for (var i = 0; i < candidates.length; i++) {
        var candidateObj = candidates[i];
        var candidateValue = candidateObj.node.GetProperty("Value");
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
          matchingCandidates.push(candidateObj);
        }
      }
    }

    /*************************************************************
     * Aggregate Results from Matching Candidates
     *************************************************************/
    for (var i = 0; i < matchingCandidates.length; i++) {
      var candidateObj = matchingCandidates[i];
      childCount++;
      // Use the stored full path as the candidate's type path.
      var fullPath = candidateObj.path;
      if (childConcatedTypes !== "") {
        childConcatedTypes = childConcatedTypes + outputDelimiter + fullPath;
      } else {
        childConcatedTypes = fullPath;
      }
      // Accumulate the candidate's value.
      var candidateValue = candidateObj.node.GetProperty("Value");
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
