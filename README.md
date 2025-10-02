"""

~~~


# SSD Endurance Test Script - Pure Cmdlet Version with Real Random Data
# Works in Constrained Language Mode

param(
    [int64]$FileSizeMB = 100,
    [string]$TestPath = "C:\SSDTest",
    [int]$Cycles = 0  # 0 = infinite, otherwise specify number of cycles
)

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

function Format-Bytes {
    param([int64]$Bytes)
    
    $PB = 1PB
    $TB = 1TB
    $GB = 1GB
    $MB = 1MB
    $KB = 1KB
    
    if ($Bytes -ge $PB) { return "{0:N2} PB" -f ($Bytes / $PB) }
    if ($Bytes -ge $TB) { return "{0:N2} TB" -f ($Bytes / $TB) }
    if ($Bytes -ge $GB) { return "{0:N2} GB" -f ($Bytes / $GB) }
    if ($Bytes -ge $MB) { return "{0:N2} MB" -f ($Bytes / $MB) }
    if ($Bytes -ge $KB) { return "{0:N2} KB" -f ($Bytes / $KB) }
    return "$Bytes Bytes"
}

# Generate initial random data file with truly random bytes
Write-Host "`nGenerating incompressible random data..." -ForegroundColor Green
$StartTime = Get-Date

$FileSizeBytes = $FileSizeMB * 1MB

# Generate in chunks to show progress
$ChunkSizeMB = 5
$ChunkSizeBytes = $ChunkSizeMB * 1MB
$TotalChunks = [Math]::Ceiling($FileSizeMB / $ChunkSizeMB)
$CurrentChunk = 0

# Delete existing file if present
if (Test-Path $SourceFile) {
    Remove-Item -Path $SourceFile -Force
}

# Create empty file first
$null = New-Item -Path $SourceFile -ItemType File -Force

# Generate and append random data in chunks
$BytesWritten = 0

while ($BytesWritten -lt $FileSizeBytes) {
    $CurrentChunk++
    $RemainingBytes = $FileSizeBytes - $BytesWritten
    $ThisChunkSize = [Math]::Min($ChunkSizeBytes, $RemainingBytes)
    
    Write-Progress -Activity "Generating Random Data" -Status "Chunk $CurrentChunk of $TotalChunks" -PercentComplete (($BytesWritten / $FileSizeBytes) * 100)
    
    # Generate truly random bytes for this chunk
    $RandomBytes = New-Object byte[] $ThisChunkSize
    for ($i = 0; $i -lt $ThisChunkSize; $i++) {
        $RandomBytes[$i] = Get-Random -Minimum 0 -Maximum 256
    }
    
    # Append to file
    Add-Content -Path $SourceFile -Value $RandomBytes -Encoding Byte
    
    $BytesWritten += $ThisChunkSize
}

Write-Progress -Activity "Generating Random Data" -Completed

$Duration = (Get-Date) - $StartTime
$DurationSeconds = $Duration.TotalSeconds
Write-Host "Random data generated in $($DurationSeconds.ToString('F2')) seconds" -ForegroundColor Gray

# Verify file size
$FileInfo = Get-Item $SourceFile
Write-Host "File created: $($FileInfo.Length) bytes" -ForegroundColor Gray

Write-Host "`nStarting write cycles...`n" -ForegroundColor Green

$GlobalStartTime = Get-Date

try {
    do {
        $CycleCount++
        $CycleStartTime = Get-Date
        Write-Host "--- Cycle $CycleCount ---" -ForegroundColor Magenta
        
        # Write 1: Copy to copy1
        Write-Host "[1/4] Copying to copy1..." -ForegroundColor White
        $WriteStart = Get-Date
        Copy-Item -Path $SourceFile -Destination $CopyFile1 -Force
        $TotalBytesWritten += $FileSizeBytes
        $WriteDuration = (Get-Date) - $WriteStart
        $WriteSeconds = $WriteDuration.TotalSeconds
        if ($WriteSeconds -gt 0) {
            $Speed = $FileSizeMB / $WriteSeconds
            Write-Host "    Written in $($WriteSeconds.ToString('F2'))s @ $($Speed.ToString('F2')) MB/s" -ForegroundColor Gray
        }
        
        # Delete copy1
        Write-Host "[2/4] Deleting copy1..." -ForegroundColor White
        Remove-Item -Path $CopyFile1 -Force
        
        # Write 2: Copy to copy2
        Write-Host "[3/4] Copying to copy2..." -ForegroundColor White
        $WriteStart = Get-Date
        Copy-Item -Path $SourceFile -Destination $CopyFile2 -Force
        $TotalBytesWritten += $FileSizeBytes
        $WriteDuration = (Get-Date) - $WriteStart
        $WriteSeconds = $WriteDuration.TotalSeconds
        if ($WriteSeconds -gt 0) {
            $Speed = $FileSizeMB / $WriteSeconds
            Write-Host "    Written in $($WriteSeconds.ToString('F2'))s @ $($Speed.ToString('F2')) MB/s" -ForegroundColor Gray
        }
        
        # Delete copy2
        Write-Host "[4/4] Deleting copy2..." -ForegroundColor White
        Remove-Item -Path $CopyFile2 -Force
        
        # Cycle summary
        $CycleDuration = (Get-Date) - $CycleStartTime
        $CycleSeconds = $CycleDuration.TotalSeconds
        if ($CycleSeconds -gt 0) {
            $CycleSpeed = ($FileSizeMB * 2) / $CycleSeconds
            Write-Host "    Cycle completed in $($CycleSeconds.ToString('F2'))s @ $($CycleSpeed.ToString('F2')) MB/s average" -ForegroundColor Cyan
        }
        
        # Display total data written
        $TotalElapsed = (Get-Date) - $GlobalStartTime
        $TotalSeconds = $TotalElapsed.TotalSeconds
        $FormattedTotal = Format-Bytes $TotalBytesWritten
        Write-Host "`n>>> TOTAL DATA WRITTEN: $FormattedTotal <<<" -ForegroundColor Green
        
        if ($TotalSeconds -gt 0) {
            $TotalMB = $TotalBytesWritten / 1MB
            $OverallSpeed = $TotalMB / $TotalSeconds
            Write-Host ">>> OVERALL WRITE SPEED: $($OverallSpeed.ToString('F2')) MB/s <<<`n" -ForegroundColor Green
        }
        
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
    
    $FormattedTotal = Format-Bytes $TotalBytesWritten
    Write-Host "`n=== Test Complete ===" -ForegroundColor Cyan
    Write-Host "Total Cycles: $CycleCount" -ForegroundColor Yellow
    Write-Host "Total Data Written: $FormattedTotal" -ForegroundColor Yellow
}
~~~

"""
