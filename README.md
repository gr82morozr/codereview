# codereview
code review

```
import java.text.SimpleDateFormat

// Get current date in YYYYMMDD format
def todayDate = new SimpleDateFormat("yyyyMMdd").format(new Date())

// Property names
def dateProperty = "LastSequenceDate"
def sequenceProperty = "SequenceNumber"

// Get previous stored date
def storedDate = context.testCase.getPropertyValue(dateProperty)
def storedSequence = context.testCase.getPropertyValue(sequenceProperty)

// Initialize sequence number
if (storedDate != todayDate) {
    // New day, reset sequence
    storedSequence = "1"
    context.testCase.setPropertyValue(dateProperty, todayDate)
} else {
    // Increment sequence for the day
    storedSequence = (storedSequence.toInteger() + 1).toString()
}

// Store updated sequence number
context.testCase.setPropertyValue(sequenceProperty, storedSequence)

// Generate unique sequence for today
def uniqueSequence = todayDate + "-" + storedSequence

// Store unique sequence in a property
context.testCase.setPropertyValue("UniqueSequence", uniqueSequence)

log.info("Generated Unique Sequence: " + uniqueSequence)




```
