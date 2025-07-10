"""

~~~
$job_1 = { ...job code 1... }
$job_2 = { ...job code 2... }
$job_3 = { ...job code 3... }

# Example: Collect all job scriptblocks
$jobVars = Get-Variable -Name 'job_*' | Sort-Object Name

$maxParallel = 20
$allJobs = @()

foreach ($var in $jobVars) {
    # Start the job
    $allJobs += Start-Job -ScriptBlock $var.Value

    # If we've started $maxParallel jobs, wait for any to finish before starting more
    while (@($allJobs | Where-Object { $_.State -eq 'Running' }).Count -ge $maxParallel) {
        Start-Sleep -Seconds 1  # Polling interval, adjust as needed
    }
}

# Wait for any remaining jobs
Wait-Job -Job $allJobs

# (Optional) Get results
$results = $allJobs | ForEach-Object { Receive-Job -Job $_ }
Remove-Job -Job $allJobs




~~~

"""
