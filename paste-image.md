---
description: "Paste image from clipboard into the conversation. Use 'clear' argument to delete all saved clipboard images."
allowed-tools: Bash, Read
---

# Paste Image from Clipboard

The user wants to share an image from their clipboard with you.

## Instructions

**If the argument is `clear`:**
Run this PowerShell command to delete all clipboard images:
```
powershell -Command "Remove-Item 'C:\Users\shimo\.claude\clipboard-images\*' -Force -ErrorAction SilentlyContinue; Write-Host 'Cleared all clipboard images'"
```
Then confirm deletion to the user.

**Otherwise (default — paste image):**

1. Run this PowerShell command to save the clipboard image:
```
powershell -Command "Add-Type -AssemblyName System.Windows.Forms; if ([System.Windows.Forms.Clipboard]::ContainsImage()) { $dir = 'C:\Users\shimo\.claude\clipboard-images'; if (!(Test-Path $dir)) { New-Item -ItemType Directory -Path $dir -Force | Out-Null }; $ts = Get-Date -Format 'yyyyMMdd-HHmmss'; $path = Join-Path $dir \"clipboard-$ts.png\"; [System.Windows.Forms.Clipboard]::GetImage().Save($path, [System.Drawing.Imaging.ImageFormat]::Png); Write-Host $path } else { Write-Host 'NO_IMAGE' }"
```

2. If the output is `NO_IMAGE`, tell the user:
   "אין תמונה ב-clipboard. תעתיק תמונה קודם (צילום מסך, העתק תמונה, וכו')"

3. If the output is a file path, use the Read tool to read that image file path and then describe/analyze the image for the user. Ask the user what they'd like to do with it.

$ARGUMENTS
