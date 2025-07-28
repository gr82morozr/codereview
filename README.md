"""

~~~

$sw = [System.Diagnostics.Stopwatch]::StartNew()

curl.exe -X PUT "http://localhost:9200/dataload-index/_doc/ABC123" `
  -H "Content-Type: application/json" `
  --data-binary "@C:\path\to\file.json"

$sw.Stop()
Write-Output "Time taken: $($sw.Elapsed.TotalMilliseconds) ms"


~~~

"""
