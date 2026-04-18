# Tips for Czur Book Scanner

The Czur Book Scanner is not the perfect solution for scanning a book. However, it is a reasonable option if you want to scan a book by yourself without damaging it.

You may prefer to use it primarily for scanning books, despite the app supporting embedded OCR and image editing features, as these features can sometimes cause unexpected behavior, such as distorting the scanned image.

### Tips

1. It’s better to use bare fingers rather than finger cots.
2. To manually remove fingers from the scanned image:
   * Download **Photoscape**
   * Select the **Tools** tab
   * Use the **Spot Healing Brush** tool
3. To convert scanned images to EPUB format, use **Kindle Comic Converter**.
   EPUB manages images internally using HTML links, allowing for faster image rendering compared to PDF-converted images.
4. **[Kindle Comic Converter](https://github.com/ciromattia/kcc)** -> Please turn off cropping mode. If it is turned on, it may cause unintended behavior.
5. **CZUR** scanners use **ABBYY OCR** internally. However, enabling OCR may distort scanned images due to the inherent nature of OCR processing.
6. You can apply **OCR** at any time after scanning by using an **OCR** application directly.

### epub-to-pdf

- PDF files natively support anti-aliasing. If an eBook reader does not support anti-aliasing, some image-based EPUB files may appear blurred, since they internally contain only images.
- To enable anti-aliasing, use the attached PowerShell script to convert the EPUB to PDF.
- The script assumes the EPUB is image-based. The EPUB file was created using Kindle Comic Converter.


```powershell
# Set working directory
$epubDir = "<your_epub_dir>"
Set-Location -Path $epubDir

# --- Part 1: Remove cover.jpg in Parallel ---
Write-Host "Starting Parallel Cover Cleanup..." -ForegroundColor Cyan

Get-ChildItem -Filter *.epub | ForEach-Object -Parallel {
    $epubPath = $_.FullName
    $zipPath  = [System.IO.Path]::ChangeExtension($epubPath, ".zip")

    Write-Host "Processing: $($_.Name)"

    # Rename EPUB to ZIP
    Rename-Item $epubPath $zipPath

    # Extract to temp folder
    $tempFolder = Join-Path $env:TEMP ([System.IO.Path]::GetFileNameWithoutExtension($zipPath) + "_" + [guid]::NewGuid().ToString().Substring(0,8))
    Expand-Archive -Path $zipPath -DestinationPath $tempFolder -Force

    # Remove cover image if it exists
    $coverPath = Join-Path $tempFolder "OEBPS\Images\cover.jpg"
    if (Test-Path $coverPath) {
        Remove-Item $coverPath -Force
        Write-Host "  Removed cover.jpg from $($_.Name)"
    }

    # Recreate ZIP
    Remove-Item $zipPath -Force
    Compress-Archive -Path "$tempFolder\*" -DestinationPath $zipPath

    # Rename back to EPUB
    $newEpubPath = [System.IO.Path]::ChangeExtension($zipPath, ".epub")
    Rename-Item $zipPath $newEpubPath

    # Cleanup temp
    Remove-Item $tempFolder -Recurse -Force
} -ThrottleLimit 6

Write-Host "All covers processed.`n" -ForegroundColor Green

# --- Part 2: Calibre Conversion in Parallel ---
$ebookConvert = "C:\Program Files\Calibre2\ebook-convert.exe"

Write-Host "Starting Parallel Conversion..." -ForegroundColor Cyan

Get-ChildItem -Filter *.epub | ForEach-Object -Parallel {
    # Reference the variable from the parent scope using $using:
    $exe = $using:ebookConvert
    $input  = $_.FullName
    $output = [System.IO.Path]::ChangeExtension($input, ".pdf")

    Write-Host "Converting: $($_.Name)"

    & "$exe" `
        "$input" "$output" `
        --margin-top 0 `
        --margin-bottom 0 `
        --margin-left 0 `
        --margin-right 0 `
        --pdf-page-margin-top 0 `
        --pdf-page-margin-bottom 0 `
        --pdf-page-margin-left 0 `
        --pdf-page-margin-right 0 `
        --no-chapters-in-toc
} -ThrottleLimit 4

Write-Host "All EPUBs converted to PDF." -ForegroundColor Green
```
