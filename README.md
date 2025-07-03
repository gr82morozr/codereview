"""

~~~
function Write-GreenSeparator {
    param (
        [int]$Length = 100,
        [char]$Char = '█'
    )

    $steps = [Math]::Floor($Length / 2)
    for ($i = 0; $i -lt $Length; $i++) {
        # Calculate gradient: 128 → 255 → 128
        if ($i -lt $steps) {
            $g = 128 + [Math]::Round(127 * ($i / $steps))
        } else {
            $g = 255 - [Math]::Round(127 * (($i - $steps) / $steps))
        }

        # Write character with RGB green color
        Write-Host -NoNewline "`e[38;2;0;${g};0m$Char"
    }

    # Reset color and write newline
    Write-Host "`e[0m"
}





~~~

"""
