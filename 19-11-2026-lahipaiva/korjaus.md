Tätä tarvitaan Powershellissä. Muuttaa Windowsin rivinvaihdot (CRLF, \r\n) Linux-yhteensopiviksi (LF, \n)

```sh
Get-ChildItem -Path . -File -Recurse | ForEach-Object {
    $content = Get-Content $_.FullName -Raw
    $content = $content -replace "`r`n", "`n"
    [System.IO.File]::WriteAllText($_.FullName, $content)
}
```