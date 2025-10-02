"""

~~~


# SSD Endurance Test Script - Optimized for Write Performance
# Keeps data in RAM and focuses on maximizing write throughput

param(
    [int64]$FileSizeMB = 100,
    [string]$TestPath = "C:\SSDTest",
    [int]$Cycles = 0  # 0 = infinite, otherwise specify number of cycles
)

# Convert MB to bytes
$FileSizeBytes = $FileSizeMB * 1MB

# Create test directory if it doesn't exist
if (-not (Test-Path $TestPath)) {
    New-Item -ItemType Directory -Path $TestPath | Out-Null
}

$SourceFile = Join-Path $TestPath "random_source.bin"
$CopyFile1 = Join-Path $TestPath "random_copy1.bin"
$CopyFile2 = Join-Path $TestPath "random_copy2.bin"

# Initialize counters
$TotalBytesWritten = 0
$CycleCount = 0

Write-Host "=== SSD Endurance Test (Write-Optimized) ===" -ForegroundColor Cyan
Write-Host "File Size: $FileSizeMB MB ($FileSizeBytes bytes)" -ForegroundColor Yellow
Write-Host "Test Path: $TestPath" -ForegroundColor Yellow
Write-Host "Cycles: $(if ($Cycles -eq 0) { 'Infinite (Ctrl+C to stop)' } else { $Cycles })" -ForegroundColor Yellow
Write-Host "`nPre-generating random data in RAM..." -ForegroundColor Green

function Format-Bytes {
    param([int64]$Bytes)
    
    if ($Bytes -ge 1PB) { return "{0:N2} PB" -f ($Bytes / 1PB) }
    if ($Bytes -ge 1TB) { return "{0:N2} TB" -f ($Bytes / 1TB) }
    if ($Bytes -ge 1GB) { return "{0:N2} GB" -f ($Bytes / 1GB) }
    if ($Bytes -ge 1MB) { return "{0:N2} MB" -f ($Bytes / 1MB) }
    if ($Bytes -ge 1KB) { return "{0:N2} KB" -f ($Bytes / 1KB) }
    return "$Bytes Bytes"
}

# Pre-generate random data in memory once
Write-Host "Generating $(Format-Bytes $FileSizeBytes) of random data..." -ForegroundColor White
$StartTime = Get-Date

# Use larger buffer for better performance
$BufferSize = [Math]::Min(64MB, $FileSizeBytes)
$RandomData = New-Object byte[] $FileSizeBytes
$RNG = [System.Security.Cryptography.RNGCryptoServiceProvider]::new()

$BytesGenerated = 0
while ($BytesGenerated -lt $FileSizeBytes) {
    $ChunkSize = [Math]::Min($BufferSize, $FileSizeBytes - $BytesGenerated)
    $TempBuffer = New-Object byte[] $ChunkSize
    $RNG.GetBytes($TempBuffer)
    [Array]::Copy($TempBuffer, 0, $RandomData, $BytesGenerated, $ChunkSize)
    $BytesGenerated += $ChunkSize
    
    # Progress indicator
    $PercentComplete = [Math]::Floor(($BytesGenerated / $FileSizeBytes) * 100)
    Write-Progress -Activity "Generating Random Data" -Status "$PercentComplete% Complete" -PercentComplete $PercentComplete
}

$RNG.Dispose()
Write-Progress -Activity "Generating Random Data" -Completed
$Duration = (Get-Date) - $StartTime
Write-Host "Data generated in $($Duration.TotalSeconds.ToString('F2')) seconds" -ForegroundColor Gray
Write-Host "`nStarting write cycles...`n" -ForegroundColor Green

try {
    do {
        $CycleCount++
        $CycleStartTime = Get-Date
        Write-Host "--- Cycle $CycleCount ---" -ForegroundColor Magenta
        
        # Write 1: Original file
        Write-Host "[1/3] Writing source file..." -ForegroundColor White
        $StartTime = Get-Date
        [System.IO.File]::WriteAllBytes($SourceFile, $RandomData)
        $TotalBytesWritten += $FileSizeBytes
        $Duration = (Get-Date) - $StartTime
        $Speed = ($FileSizeBytes / 1MB) / $Duration.TotalSeconds
        Write-Host "    Written in $($Duration.TotalSeconds.ToString('F2'))s @ $($Speed.ToString('F2')) MB/s" -ForegroundColor Gray
        
        # Write 2: First copy
        Write-Host "[2/3] Writing copy1..." -ForegroundColor White
        $StartTime = Get-Date
        [System.IO.File]::WriteAllBytes($CopyFile1, $RandomData)
        $TotalBytesWritten += $FileSizeBytes
        $Duration = (Get-Date) - $StartTime
        $Speed = ($FileSizeBytes / 1MB) / $Duration.TotalSeconds
        Write-Host "    Written in $($Duration.TotalSeconds.ToString('F2'))s @ $($Speed.ToString('F2')) MB/s" -ForegroundColor Gray
        
        # Delete copy1
        Remove-Item -Path $CopyFile1 -Force
        
        # Write 3: Second copy
        Write-Host "[3/3] Writing copy2..." -ForegroundColor White
        $StartTime = Get-Date
        [System.IO.File]::WriteAllBytes($CopyFile2, $RandomData)
        $TotalBytesWritten += $FileSizeBytes
        $Duration = (Get-Date) - $StartTime
        $Speed = ($FileSizeBytes / 1MB) / $Duration.TotalSeconds
        Write-Host "    Written in $($Duration.TotalSeconds.ToString('F2'))s @ $($Speed.ToString('F2')) MB/s" -ForegroundColor Gray
        
        # Delete copy2
        Remove-Item -Path $CopyFile2 -Force
        
        # Cycle summary
        $CycleDuration = (Get-Date) - $CycleStartTime
        $CycleSpeed = (($FileSizeBytes * 3) / 1MB) / $CycleDuration.TotalSeconds
        Write-Host "    Cycle completed in $($CycleDuration.TotalSeconds.ToString('F2'))s @ $($CycleSpeed.ToString('F2')) MB/s average" -ForegroundColor Cyan
        
        # Display total data written
        Write-Host "`n>>> TOTAL DATA WRITTEN: $(Format-Bytes $TotalBytesWritten) <<<" -ForegroundColor Green
        Write-Host ">>> AVERAGE WRITE SPEED: $(($TotalBytesWritten / 1MB / ((Get-Date) - $StartTime).TotalSeconds).ToString('F2')) MB/s <<<`n" -ForegroundColor Green
        
        # Check if we should continue
        if ($Cycles -gt 0 -and $CycleCount -ge $Cycles) {
            break
        }
        
    } while ($true)
    
} catch {
    Write-Host "`nError occurred: $_" -ForegroundColor Red
} finally {
    # Cleanup
    Write-Host "`nCleaning up..." -ForegroundColor Yellow
    if (Test-Path $SourceFile) { Remove-Item -Path $SourceFile -Force }
    if (Test-Path $CopyFile1) { Remove-Item -Path $CopyFile1 -Force }
    if (Test-Path $CopyFile2) { Remove-Item -Path $CopyFile2 -Force }
    
    Write-Host "`n=== Test Complete ===" -ForegroundColor Cyan
    Write-Host "Total Cycles: $CycleCount" -ForegroundColor Yellow
    Write-Host "Total Data Written: $(Format-Bytes $TotalBytesWritten)" -ForegroundColor Yellow
    
    # Clear random data from memory
    $RandomData = $null
    [System.GC]::Collect()
}


~~~

"""
