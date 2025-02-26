# codereview
code review

```
import java.text.SimpleDateFormat

// Define the date format
def dateFormat = new SimpleDateFormat("MM/dd/yyyy HH:mm:ss")

// Get the existing RandomTimestamp property
def randomTimestampStr = context.expand('${#TestCase#RandomTimestamp}')

if (randomTimestampStr) {
    def date = dateFormat.parse(randomTimestampStr) // Convert to Date
    def epochTime = date.getTime() / 1000 // Convert to epoch time (seconds)

    // Generate new timestamps in epoch (subtract/add 10 seconds)
    def epochB4 = (epochTime - 10) * 1000 // Convert back to milliseconds
    def epochAF = (epochTime + 10) * 1000 // Convert back to milliseconds

    // Convert back to formatted timestamps
    def randomTimestampB4 = dateFormat.format(new Date(epochB4))
    def randomTimestampAF = dateFormat.format(new Date(epochAF))

    // Store values as test case properties
    testRunner.testCase.setPropertyValue("RandomTimestamp_B4", randomTimestampB4)
    testRunner.testCase.setPropertyValue("RandomTimestamp_AF", randomTimestampAF)

    // Log results
    log.info("RandomTimestamp: " + randomTimestampStr)
    log.info("RandomTimestamp_B4: " + randomTimestampB4)
    log.info("RandomTimestamp_AF: " + randomTimestampAF)
} else {
    log.error("RandomTimestamp property is missing or empty.")
}






```
