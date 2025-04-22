"""
function FindLastActiveEmployment(InputPS, OutputPS) {
    var workerId = InputPS.GetProperty("Worker Id"); // Get input Worker Id
    var bo = TheApplication().GetBusObject("NQSC Worker");
    var bc = bo.GetBusComp("NQSC Worker");

    var lastRowId = "";             // Variable to store the final Row Id
    var lastDate = "";              // To store latest modification date

    with (bc) {
        ActivateField("Status");            // Needed to filter on this field
        ActivateField("Last Updated");      // Used to find latest record
        ClearToQuery();
        SetSearchSpec("Worker Id", workerId);   // Match Worker Id
        SetSearchSpec("Status", "Active");      // Only Active ones
        ExecuteQuery(ForwardOnly);

        var isRecord = FirstRecord();
        while (isRecord) {
            var currentDate = GetFieldValue("Last Updated"); // e.g., "2024-05-21 10:03:00"
            if (lastDate === "" || currentDate > lastDate) {
                lastDate = currentDate;
                lastRowId = GetFieldValue("Id");
            }
            isRecord = NextRecord();
        }
    }

    // Set result into output propset
    OutputPS.SetProperty("Last Work Employment Id", lastRowId);

    // Clean up references
    bc = null;
    bo = null;
}
"""
