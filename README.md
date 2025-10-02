"""

~~~


# SSD Endurance Test Script - Constrained Language Mode Compatible
# Uses only cmdlets and core operations

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

Write-Host "=== SSD Endurance Test ===" -ForegroundColor Cyan
Write-Host "File Size: $FileSizeMB MB" -ForegroundColor Yellow
Write-Host "Test Path: $TestPath" -ForegroundColor Yellow
Write-Host "Cycles: $(if ($Cycles -eq 0) { 'Infinite (Ctrl+C to stop)' } else { $Cycles })" -ForegroundColor Yellow
Write-Host "`nGenerating initial random data file..." -ForegroundColor Green

function Format-Bytes {
    param([int64]$Bytes)
    
    if ($Bytes -ge 1PB) { return "{0:N2} PB" -f ($Bytes / 1PB) }
    if ($Bytes -ge 1TB) { return "{0:N2} TB" -f ($Bytes / 1TB) }
    if ($Bytes -ge 1GB) { return "{0:N2} GB" -f ($Bytes / 1GB) }
    if ($Bytes -ge 1MB) { return "{0:N2} MB" -f ($Bytes / 1MB) }
    if ($Bytes -ge 1KB) { return "{0:N2} KB" -f ($Bytes / 1KB) }
    return "$Bytes Bytes"
}

# Generate initial random data file using fsutil (fastest method in constrained mode)
Write-Host "Creating random data file..." -ForegroundColor White
$StartTime = Get-Date

# Use fsutil to create a file with random data (Windows built-in, very fast)
$null = & fsutil file createnew $SourceFile $FileSizeBytes 2>&1

# Fill with pseudo-random data by copying random content
# We'll use Get-Random to create varied content and write it in chunks
$ChunkSizeMB = 10
$ChunkSizeBytes = $ChunkSizeMB * 1MB
$TempChunkFile = Join-Path $TestPath "temp_chunk.bin"

# Create a chunk of random data
$RandomChunk = -join (1..[Math]::Min($ChunkSizeBytes, 10MB) | ForEach-Object { [char](Get-Random -Minimum 0 -Maximum 256) })
Set-Content -Path $TempChunkFile -Value $RandomChunk -Encoding Byte -NoNewline

# Overwrite the file with random patterns using copy
$BytesWritten = 0
$OutStream = [System.IO.File]::OpenWrite($SourceFile)
$ChunkBytes = Get-Content -Path $TempChunkFile -Encoding Byte -ReadCount 0

while ($BytesWritten -lt $FileSizeBytes) {
    $BytesToWrite = [Math]::Min($ChunkBytes.Length, $FileSizeBytes - $BytesWritten)
    $OutStream.Write($ChunkBytes, 0, $BytesToWrite)
    $BytesWritten += $BytesToWrite
    
    $PercentComplete = [Math]::Floor(($BytesWritten / $FileSizeBytes) * 100)
    Write-Progress -Activity "Generating Random Data" -Status "$PercentComplete% Complete" -PercentComplete $PercentComplete
}

$OutStream.Close()
Remove-Item -Path $TempChunkFile -Force
Write-Progress -Activity "Generating Random Data" -Completed

$Duration = (Get-Date) - $StartTime
Write-Host "Data generated in $($Duration.TotalSeconds.ToString('F2')) seconds" -ForegroundColor Gray
Write-Host "`nStarting write cycles...`n" -ForegroundColor Green

$GlobalStartTime = Get-Date

try {
    do {
        $CycleCount++
        $CycleStartTime = Get-Date
        Write-Host "--- Cycle $CycleCount ---" -ForegroundColor Magenta
        
        # Write 1: Copy to copy1
        Write-Host "[1/4] Copying to copy1..." -ForegroundColor White
        $StartTime = Get-Date
        Copy-Item -Path $SourceFile -Destination $CopyFile1 -Force
        $TotalBytesWritten += $FileSizeBytes
        $Duration = (Get-Date) - $StartTime
        $Speed = ($FileSizeMB) / $Duration.TotalSeconds
        Write-Host "    Written in $($Duration.TotalSeconds.ToString('F2'))s @ $($Speed.ToString('F2')) MB/s" -ForegroundColor Gray
        
        # Delete copy1
        Write-Host "[2/4] Deleting copy1..." -ForegroundColor White
        Remove-Item -Path $CopyFile1 -Force
        
        # Write 2: Copy to copy2
        Write-Host "[3/4] Copying to copy2..." -ForegroundColor White
        $StartTime = Get-Date
        Copy-Item -Path $SourceFile -Destination $CopyFile2 -Force
        $TotalBytesWritten += $FileSizeBytes
        $Duration = (Get-Date) - $StartTime
        $Speed = ($FileSizeMB) / $Duration.TotalSeconds
        Write-Host "    Written in $($Duration.TotalSeconds.ToString('F2'))s @ $($Speed.ToString('F2')) MB/s" -ForegroundColor Gray
        
        # Delete copy2
        Write-Host "[4/4] Deleting copy2..." -ForegroundColor White
        Remove-Item -Path $CopyFile2 -Force
        
        # Cycle summary
        $CycleDuration = (Get-Date) - $CycleStartTime
        $CycleSpeed = (($FileSizeMB * 2)) / $CycleDuration.TotalSeconds
        Write-Host "    Cycle completed in $($CycleDuration.TotalSeconds.ToString('F2'))s @ $($CycleSpeed.ToString('F2')) MB/s average" -ForegroundColor Cyan
        
        # Display total data written
        $TotalElapsed = ((Get-Date) - $GlobalStartTime).TotalSeconds
        $OverallSpeed = ($TotalBytesWritten / 1MB) / $TotalElapsed
        Write-Host "`n>>> TOTAL DATA WRITTEN: $(Format-Bytes $TotalBytesWritten) <<<" -ForegroundColor Green
        Write-Host ">>> OVERALL WRITE SPEED: $($OverallSpeed.ToString('F2')) MB/s <<<`n" -ForegroundColor Green
        
        # Check if we should continue
        if ($Cycles -gt 0 -and $CycleCount -ge $Cycles) {
            break
        }
        
    } while ($true)
    
} catch {
    Write-Host "`nError occurred: $_" -ForegroundColor Red
    Write-Host $_.Exception.Message -ForegroundColor Red
} finally {
    # Cleanup
    Write-Host "`nCleaning up..." -ForegroundColor Yellow
    if (Test-Path $SourceFile) { Remove-Item -Path $SourceFile -Force }
    if (Test-Path $CopyFile1) { Remove-Item -Path $CopyFile1 -Force }
    if (Test-Path $CopyFile2) { Remove-Item -Path $CopyFile2 -Force }
    
    Write-Host "`n=== Test Complete ===" -ForegroundColor Cyan
    Write-Host "Total Cycles: $CycleCount" -ForegroundColor Yellow
    Write-Host "Total Data Written: $(Format-Bytes $TotalBytesWritten)" -ForegroundColor Yellow
}


~~~

"""
