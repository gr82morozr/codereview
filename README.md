"""

~~~

$sw = [System.Diagnostics.Stopwatch]::StartNew()

curl.exe -X PUT "http://localhost:9200/dataload-index/_doc/ABC123" `
  -H "Content-Type: application/json" `
  --data-binary "@C:\path\to\file.json"

$sw.Stop()
Write-Output "Time taken: $($sw.Elapsed.TotalMilliseconds) ms"


$timeTaken = $sw.Elapsed.TotalMilliseconds
$logEntry = "[$(Get-Date -Format 'yyyy-MM-dd HH:mm:ss')] File: $filePath | ID: $docId | Time: ${timeTaken} ms"

Add-Content -Path $logPath -Value $logEntry

~~~

"""
