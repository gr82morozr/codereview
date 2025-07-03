"""


$flagPath = "C:\Path\To\flag.txt"
$scriptToRun = "C:\Path\To\TaskScript.ps1"

Write-Host "Monitoring started... (Press Ctrl+C to stop)"

while ($true) {
    if (Test-Path $flagPath) {
        Write-Host "Flag detected. Running task script..."

        try {
            & $scriptToRun
            Write-Host "Task script completed."
        } catch {
            Write-Host "Error running script: $_"
        }

        # Optional: remove flag to prevent re-triggering
        Remove-Item $flagPath -Force
        Write-Host "Flag file removed. Resuming monitoring..."
    }

    Start-Sleep -Seconds 5  # Adjust interval as needed
}



"""
