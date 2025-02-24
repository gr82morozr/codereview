# codereview
code review

```
function GetDeepChildBySpec(Inputs, Outputs) {

/*************************************************************
 * Function: GetDeepChildBySpec
 * Author: [Your Name]
 * Purpose:
 *   This function searches a given ParentPropertySet (at any depth) for child nodes 
 *   that match specified ChildType and/or ChildValue criteria. Wildcards ("", "*") indicate 
 *   that the criterion should not filter. Note that both ChildType and ChildValue cannot be wildcards.
 *
 * Usage:
 *   Input Arguments:
 *   --------------------------
 *     1. ParentPropertySet (Mandatory):
 *        The root PropertySet object to search within.
 *
 *     2. ChildType (Mandatory):
 *        The type to filter by (e.g., "ChildTypeA"). If "*" or empty, then any type is acceptable
 *        if ChildValue is provided.
 *
 *     3. ChildValue (Mandatory):
 *        The value to filter by (e.g., "ChildValueA"). If "*" or empty, then any value is acceptable
 *        if ChildType is provided.
 *
 *     4. Output_Delimiter (Optional):
 *        The delimiter used to join output strings (default is ".").
 *
 *   Output Arguments:
 *   --------------------------
 *     1. ChildExists:
 *        "Y" if matching child node(s) are found; "N" otherwise.
 *
 *     2. ChildCount:
 *        The count of matching child nodes.
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
 *     ChildType = "ChildTypeA",
 *     ChildValue = "ChildValueA",
 *     Output_Delimiter = "|" (optional).
 *   Output:
 *     ChildExists = "Y",
 *     ChildCount = "2",
 *     ChildConcatedTypes = "ChildTypeA|ChildTypeA",
 *     ChildConcatedValues = "ChildValueA|ChildValueA".
 *
 * Notes:
 *   - Both ChildType and ChildValue cannot be wildcards ("", "*") at the same time.
 *
 * Update History:
 *   Initial Version: [Current Date] by [Your Name]
 **************************************************************/

  /*************************************************************
   * Input Extraction
   *************************************************************/
  // Retrieve input parameters using the same pattern as the sample.
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
  // Determine if ChildType or ChildValue is a wildcard (null, empty string, or "*").
  var isChildTypeWildcard  = (childTypeInput === null || childTypeInput === "" || childTypeInput === "*");
  var isChildValueWildcard = (childValueInput === null || childValueInput === "" || childValueInput === "*");
  // Validate that both ChildType and ChildValue are not wildcards simultaneously.
  if (isChildTypeWildcard && isChildValueWildcard) {
    TheApplication().RaiseErrorText("Error: Both ChildType and ChildValue cannot be empty or wildcard.");
  }

  /*************************************************************
   * Initialize Result Variables and Stack for Iterative Traversal
   *************************************************************/
  var childCount = 0;               // Count of matching child nodes.
  var childConcatedTypes = "";      // Accumulated matching child types.
  var childConcatedValues = "";     // Accumulated matching child values.
  // Create an array to serve as a stack for iterative depth-first traversal.
  var psStack = [];
  psStack.push(parentPS);

  /*************************************************************
   * Iterative Traversal of PropertySet Hierarchy
   *************************************************************/
  while (psStack.length > 0) {
    // Pop the top PropertySet from the stack.
    var currentPS = psStack.pop();
    // Loop through all immediate children of the current PropertySet.
    for (var i = 0; i < currentPS.GetChildCount(); i++) {
      var child = currentPS.GetChild(i);
      // Retrieve the child's type.
      var currentChildType = child.GetType();
      // Retrieve the child's value; assume it is stored in a property named "Value".
      var currentChildValue = child.GetProperty("Value");

      // Initialize matching flags for type and value.
      var typeMatches = false;
      var valueMatches = false;

      // Check if the child's type meets the search criteria.
      if (isChildTypeWildcard) {
        typeMatches = true;  // Wildcard: match any type.
      } else if (currentChildType === childTypeInput) {
        typeMatches = true;
      }
      // Check if the child's value meets the search criteria.
      if (isChildValueWildcard) {
        valueMatches = true;  // Wildcard: match any value.
      } else if (currentChildValue === childValueInput) {
        valueMatches = true;
      }

      // If both criteria match, update the result variables.
      if (typeMatches && valueMatches) {
        childCount++;  // Increment the match count.

        // Append the child's type using the specified delimiter.
        if (childConcatedTypes !== "") {
          childConcatedTypes = childConcatedTypes + outputDelimiter + currentChildType;
        } else {
          childConcatedTypes = currentChildType;
        }

        // Append the child's value using the specified delimiter.
        if (childConcatedValues !== "") {
          childConcatedValues = childConcatedValues + outputDelimiter + currentChildValue;
        } else {
          childConcatedValues = currentChildValue;
        }
      }

      // If the child has further descendants, push it onto the stack for further traversal.
      if (child.GetChildCount() > 0) {
        psStack.push(child);
      }
    }
  }

  /*************************************************************
   * Prepare and Set Output Results
   *************************************************************/
  // Determine if any matching child nodes were found.
  var childExists = (childCount > 0) ? "Y" : "N";
  // Set the output properties.
  Outputs.SetProperty("ChildExists", childExists);
  Outputs.SetProperty("ChildCount", childCount.toString());
  Outputs.SetProperty("ChildConcatedTypes", childConcatedTypes);
  Outputs.SetProperty("ChildConcatedValues", childConcatedValues);

  return;
}
```
