"""

~~~
$job_1 = { ...job code 1... }
$job_2 = { ...job code 2... }
$job_3 = { ...job code 3... }

# 1. Collect all variables named $job_*
$jobVars = Get-Variable -Name 'job_*' | Sort-Object Name

# 2. Start all jobs and collect job handles
$jobs = @()
foreach ($var in $jobVars) {
    $jobs += Start-Job -ScriptBlock $var.Value
}

# 3. Wait for all jobs to finish
Wait-Job -Job $jobs

# 4. (Optional) Receive output/results
$results = $jobs | ForEach-Object { Receive-Job -Job $_ }
Remove-Job -Job $jobs




~~~

"""
