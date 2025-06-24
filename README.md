# Set your Elasticsearch endpoint
$esUrl = "http://localhost:9200/my-index/_search"

# Set your JSON query (single-line)
$jsonQuery = '{"query":{"match_all":{}}}'

# Run the curl command and save the response to result.json
curl -X POST $esUrl `
     -H "Content-Type: application/json" `
     -d $jsonQuery `
     -o result.json

# Notify the user
Write-Host "Query sent to Elasticsearch. Output saved to result.json"
