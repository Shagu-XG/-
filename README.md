# -
用AI写了一个很唐氏的脚本，这个脚本开了之后可以拦截截图，然后可以自定义截图前置的名字按序号输出到指定文件夹，图片格式也可以自定义
<#
Screenshot Monitor Script
Monitors clipboard for screenshots and saves them automatically
#>

# Set encoding to UTF-8
$OutputEncoding = [System.Text.Encoding]::UTF8
[Console]::OutputEncoding = [System.Text.Encoding]::UTF8

$ErrorActionPreference = "Continue"

Write-Host "========================================" -ForegroundColor Green
Write-Host "        Screenshot Monitor v1.0" -ForegroundColor Green
Write-Host "========================================" -ForegroundColor Green

# Import necessary .NET classes
Add-Type -AssemblyName System.Windows.Forms
Add-Type -AssemblyName System.Drawing
Add-Type -AssemblyName System.IO

# Get save folder
Write-Host "Save folder (default: current directory\Screenshots):"
$global:saveFolder = Read-Host
if ([string]::IsNullOrEmpty($global:saveFolder)) {
    $global:saveFolder = Join-Path -Path $PSScriptRoot -ChildPath "Screenshots"
} else {
    # Remove quotes from path
    $global:saveFolder = $global:saveFolder.Trim('"')
}

# Create save folder
if (-not (Test-Path -Path $global:saveFolder)) {
    try {
        New-Item -Path $global:saveFolder -ItemType Directory -Force | Out-Null
        Write-Host "Created folder: $global:saveFolder" -ForegroundColor Green
    } catch {
        Write-Host "  Failed to create folder: $($_.Exception.Message)" -ForegroundColor Red
        Write-Host "Press Enter to exit..."
        Read-Host
        exit
    }
}

# Get start number
Write-Host "Start number (default: 1):"
$startNumber = Read-Host
if ([string]::IsNullOrEmpty($startNumber) -or -not ($startNumber -match '^\d+$')) {
    $startNumber = 1
}
$global:currentNumber = [int]$startNumber

# Get file prefix
Write-Host "File prefix (default: screenshot):"
$global:filePrefix = Read-Host
if ([string]::IsNullOrEmpty($global:filePrefix)) {
    $global:filePrefix = "screenshot"
}

# Get screenshot format
Write-Host "File format (jpg/png/bmp, default: jpg):"
$global:format = Read-Host
if ([string]::IsNullOrEmpty($global:format)) {
    $global:format = "jpg"
} else {
    $global:format = $global:format.ToLower()
    if ($global:format -notin @("jpg", "png", "bmp")) {
        $global:format = "jpg"
    }
}

Write-Host "" -ForegroundColor White
Write-Host "========================================" -ForegroundColor Green
Write-Host "Settings completed:" -ForegroundColor Green
Write-Host "Save folder: $global:saveFolder" -ForegroundColor Cyan
Write-Host "Start number: $global:currentNumber" -ForegroundColor Cyan
Write-Host "File prefix: $global:filePrefix" -ForegroundColor Cyan
Write-Host "File format: $global:format" -ForegroundColor Cyan
Write-Host "========================================" -ForegroundColor Green
Write-Host "" -ForegroundColor White
Write-Host "Starting screenshot monitor..." -ForegroundColor Green
Write-Host "Use PrintScreen or other screenshot tools" -ForegroundColor Yellow
Write-Host "Press F1 to change settings" -ForegroundColor Yellow
Write-Host "Press Ctrl+C to stop monitoring" -ForegroundColor Yellow
Write-Host "" -ForegroundColor White

# Function: Check clipboard for image
function Check-ClipboardForImage {
    try {
        if ([System.Windows.Forms.Clipboard]::ContainsImage()) {
            $image = [System.Windows.Forms.Clipboard]::GetImage()
            return $image
        }
        return $null
    } catch {
        return $null
    }
}

# Function: Save image
function Save-Image {
    param(
        [System.Drawing.Image]$image,
        [string]$savePath
    )
    
    try {
        $image.Save($savePath)
        return $true
    } catch {
        Write-Host "  Save failed: $($_.Exception.Message)" -ForegroundColor Red
        return $false
    }
}

# Function: Change settings
function Change-Settings {
    Write-Host "" -ForegroundColor White
    Write-Host "========================================" -ForegroundColor Green
    Write-Host "Change Settings" -ForegroundColor Green
    Write-Host "========================================" -ForegroundColor Green
    Write-Host "" -ForegroundColor White
    
    # Change save folder
    Write-Host "Save folder (current: $global:saveFolder):"
    $newFolder = Read-Host
    if (-not [string]::IsNullOrEmpty($newFolder)) {
        $newFolder = $newFolder.Trim('"')
        if (-not (Test-Path -Path $newFolder)) {
            try {
                New-Item -Path $newFolder -ItemType Directory -Force | Out-Null
                Write-Host "Created folder: $newFolder" -ForegroundColor Green
            } catch {
                Write-Host "  Failed to create folder: $($_.Exception.Message)" -ForegroundColor Red
                Write-Host "Keeping current folder" -ForegroundColor Yellow
                $newFolder = $global:saveFolder
            }
        }
        $global:saveFolder = $newFolder
    }
    
    # Change start number
    Write-Host "Start number (current: $global:currentNumber):"
    $newNumber = Read-Host
    if (-not [string]::IsNullOrEmpty($newNumber) -and $newNumber -match '^\d+$') {
        $global:currentNumber = [int]$newNumber
    }
    
    # Change file prefix
    Write-Host "File prefix (current: $global:filePrefix):"
    $newPrefix = Read-Host
    if (-not [string]::IsNullOrEmpty($newPrefix)) {
        $global:filePrefix = $newPrefix
    }
    
    # Change file format
    Write-Host "File format (current: $global:format):"
    $newFormat = Read-Host
    if (-not [string]::IsNullOrEmpty($newFormat)) {
        $newFormat = $newFormat.ToLower()
        if ($newFormat -in @("jpg", "png", "bmp")) {
            $global:format = $newFormat
        }
    }
    
    Write-Host "" -ForegroundColor White
    Write-Host "========================================" -ForegroundColor Green
    Write-Host "New Settings:" -ForegroundColor Green
    Write-Host "Save folder: $global:saveFolder" -ForegroundColor Cyan
    Write-Host "Start number: $global:currentNumber" -ForegroundColor Cyan
    Write-Host "File prefix: $global:filePrefix" -ForegroundColor Cyan
    Write-Host "File format: $global:format" -ForegroundColor Cyan
    Write-Host "========================================" -ForegroundColor Green
    Write-Host "" -ForegroundColor White
    Write-Host "Resuming monitoring..." -ForegroundColor Green
    Write-Host "" -ForegroundColor White
}

# Start monitoring loop
while ($true) {
    # Check for hotkey (F1 to change settings)
    if ([Console]::KeyAvailable) {
        $key = [Console]::ReadKey($true)
        if ($key.Key -eq "F1") {
            Change-Settings
        }
    }
    
    # Check clipboard for new image
    $image = Check-ClipboardForImage
    if ($image) {
        # Generate filename
        $fileName = "$global:filePrefix`_$($global:currentNumber.ToString('0000')).$global:format"
        $savePath = Join-Path -Path $global:saveFolder -ChildPath $fileName
        
        # Save image
        Write-Host "Detected new screenshot, saving as: $fileName" -ForegroundColor White
        if (Save-Image -image $image -savePath $savePath) {
            Write-Host "  Saved successfully: $savePath" -ForegroundColor Green
            $global:currentNumber++
            
            # Clear clipboard to avoid duplicate processing
            [System.Windows.Forms.Clipboard]::Clear()
        }
    }
    
    # Short pause to avoid high system resource usage
    Start-Sleep -Milliseconds 500
}
