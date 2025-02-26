# codereview
code review

```
import java.text.SimpleDateFormat
import java.util.Random

// Define the format
def dateFormat = new SimpleDateFormat("MM/dd/yyyy HH:mm:ss")

// Get current time
def now = System.currentTimeMillis()

// Generate a random offset (between 0 and 7 days in milliseconds)
def randomOffset = new Random().nextInt(7 * 24 * 60 * 60 * 1000) // Random up to 7 days

// Apply offset
def randomTimestamp = new Date(now - randomOffset) // Subtract offset for random past date

// Format the date
def formattedTimestamp = dateFormat.format(randomTimestamp)

// Store in Test Case property
context.testCase.setPropertyValue("RandomTimestamp", formattedTimestamp)

log.info("Generated Random Timestamp: " + formattedTimestamp)





```
