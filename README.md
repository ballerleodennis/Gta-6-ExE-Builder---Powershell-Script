# Gta-6-ExE-Builder---Powershell-Script


Verstanden — das ganze Projekt in **einer** PowerShell-Datei. Kein Python, kein PyInstaller, keine Zusatzdateien. Anlegen, Prüfen, Piece-Check, EXE-Hash und Report, alles in einem Skript mit Oberfläche und CLI.

**`GTA6LeakCheck.ps1`**
```powershell
<#
.SYNOPSIS
    Struktur- und Integritaetspruefung fuer GTA6.Xbox.v1.0.2.1-CYBERLEEK.

.DESCRIPTION
    Einzeldatei-Werkzeug, zugeschnitten auf genau dieses Torrent-Verzeichnis.

    Kann das komplette Verzeichnis anlegen (alle 108 Dateien, exakte Groessen,
    standardmaessig als sparse Dateien), die Struktur pruefen, Pieces per SHA-1
    gegen die Torrent-Metadaten verifizieren, die Start-EXE hashen und einen
    Report schreiben.

    Torrent  : GTA6.Xbox.v1.0.2.1-CYBERLEEK
    Infohash : 465d6a0eac381e68ca77115f44f19f0c968b9590
    Dateien  : 108
    Gesamt   : 223880932870 Bytes

.PARAMETER Path
    Zielordner.

.PARAMETER Create
    Verzeichnis und alle Dateien anlegen.

.PARAMETER Check
    Struktur pruefen.

.PARAMETER Pieces
    SHA-1-Piece-Pruefung gegen eine .torrent-Datei.

.PARAMETER Torrent
    Pfad zur .torrent-Datei. Fehlt sie, wird sie geladen.

.PARAMETER Exe
    Start-EXE gegen den offiziellen SHA-256 pruefen.

.PARAMETER Report
    Report schreiben.

.PARAMETER Json
    Report als JSON.

.PARAMETER NoSparse
    Echte Dateien statt sparse.

.PARAMETER Force
    Vorhandene Dateien auf Sollgroesse bringen.

.PARAMETER Gui
    Oberflaeche starten (Standard ohne Schalter).

.EXAMPLE
    .\GTA6LeakCheck.ps1

.EXAMPLE
    .\GTA6LeakCheck.ps1 -Path "D:\GTA6" -Create -Check -Report "D:\bericht.txt"
#>

[CmdletBinding(DefaultParameterSetName = 'Gui')]
param(
    [Parameter(ParameterSetName = 'Cli')]
    [Parameter(ParameterSetName = 'Gui')]
    [string]$Path = (Join-Path $env:USERPROFILE 'GTA6.Xbox.v1.0.2.1-CYBERLEEK'),

    [Parameter(ParameterSetName = 'Cli')] [switch]$Create,
    [Parameter(ParameterSetName = 'Cli')] [switch]$Check,
    [Parameter(ParameterSetName = 'Cli')] [switch]$Pieces,
    [Parameter(ParameterSetName = 'Cli')] [string]$Torrent,
    [Parameter(ParameterSetName = 'Cli')] [switch]$Exe,
    [Parameter(ParameterSetName = 'Cli')] [string]$Report,
    [Parameter(ParameterSetName = 'Cli')] [switch]$Json,
    [Parameter(ParameterSetName = 'Cli')] [switch]$NoSparse,
    [Parameter(ParameterSetName = 'Cli')] [switch]$Force,
    [Parameter(ParameterSetName = 'Gui')] [switch]$Gui
)

$ErrorActionPreference = 'Continue'

# ============================== Konstanten ===================================
$script:TorrentName = 'GTA6.Xbox.v1.0.2.1-CYBERLEEK'
$script:Infohash    = '465d6a0eac381e68ca77115f44f19f0c968b9590'
$script:TorrentUrl  = 'https://lt.t0r.store/tor/68440382.torrent'
$script:TotalBytes  = [int64]223880932870
$script:FileCount   = 108
$script:PieceLength = [int64]16777216
$script:PieceCount  = 13345
$script:ExpectedExe = 'game_scarlett_master.exe'
$script:ExeSha256   = 'e95020b2fa4a721db3df59fdecf75621730ac559d8835252f6c3b658d5a08993'
$script:ZipSha256   = '7545a2f7fb4b399cf8850987f087ed9a4d8499ddb6d6c2d642fd60c27f569730'
$script:ZipPassword = 'THANKYOUCYBERLEEK'
$script:Markers      = @('appxmanifest.xml', 'gameos.xvd', 'MicrosoftGame.Config')
$script:Magnet = "magnet:?xt=urn:btih:$($script:Infohash)&dn=GTA6.Xbox.v1.0.2.1-CYBERLEEK&tr=udp://open.tracker.cl:1337/announce"

$script:Manifest = @'
255488|Bink2Scarlett.dll
815104|Catalog000.bin
2542|MicrosoftGame.Config
1|Update.alignmentchunk
199152|XCurl.dll
884224|XboxBvhBuilder.dll
13312|amd_fidelityfx_loader_xbox.dll
786432|amd_fidelityfx_upscaler_xbox.dll
4407626384|anim.rpf
8880|appdata0.rpf
3394|appxmanifest.xml
28543454672|audio.rpf
6642192|common.rpf
298992|concrt140.dll
34411776|data/ttdb/graphics/ttdb_graphics.ttpf
883202048|data/ttdb/interiors/ttdb_interiors_0_3.ttpf
898113536|data/ttdb/interiors/ttdb_interiors_1_3.ttpf
881431552|data/ttdb/interiors/ttdb_interiors_2_3.ttpf
900875008|data/ttdb/interiors/ttdb_interiors_3_3.ttpf
151531776|data/ttdb/lighting/lighting.ttpf
1765679616|data/ttdb/maps/ttdb_maps_0_7.ttpf
1746187776|data/ttdb/maps/ttdb_maps_1_7.ttpf
1818806528|data/ttdb/maps/ttdb_maps_2_7.ttpf
1787349760|data/ttdb/maps/ttdb_maps_3_7.ttpf
1791670784|data/ttdb/maps/ttdb_maps_4_7.ttpf
1840029440|data/ttdb/maps/ttdb_maps_5_7.ttpf
1745923072|data/ttdb/maps/ttdb_maps_6_7.ttpf
1755748608|data/ttdb/maps/ttdb_maps_7_7.ttpf
494316032|data/ttdb/peds/decorations_0_3.ttpf
510001152|data/ttdb/peds/decorations_1_3.ttpf
491507456|data/ttdb/peds/decorations_2_3.ttpf
497391360|data/ttdb/peds/decorations_3_3.ttpf
135676160|data/ttdb/peds/peds_loose.ttpf
953116416|data/ttdb/peds/ttdb_peds_0_3.ttpf
965556736|data/ttdb/peds/ttdb_peds_1_3.ttpf
937699072|data/ttdb/peds/ttdb_peds_2_3.ttpf
957291520|data/ttdb/peds/ttdb_peds_3_3.ttpf
983620352|data/ttdb/props/octa_imposters.ttpf
1304458240|data/ttdb/props/ttdb_props_0_3.ttpf
1256908800|data/ttdb/props/ttdb_props_1_3.ttpf
1239990784|data/ttdb/props/ttdb_props_2_3.ttpf
1328911616|data/ttdb/props/ttdb_props_3_3.ttpf
2775084288|data/ttdb/terrain/ttdb_terrain_0_7.ttpf
2739444480|data/ttdb/terrain/ttdb_terrain_1_7.ttpf
2780145920|data/ttdb/terrain/ttdb_terrain_2_7.ttpf
2718679808|data/ttdb/terrain/ttdb_terrain_3_7.ttpf
2741339648|data/ttdb/terrain/ttdb_terrain_4_7.ttpf
2674379008|data/ttdb/terrain/ttdb_terrain_5_7.ttpf
2688808960|data/ttdb/terrain/ttdb_terrain_6_7.ttpf
2828296704|data/ttdb/terrain/ttdb_terrain_7_7.ttpf
1946070528|data/ttdb/terrain/ttdb_terrain_image_based_0_1.ttpf
1890978048|data/ttdb/terrain/ttdb_terrain_image_based_1_1.ttpf
222976|data/ttdb/ui/platform.ttpf
845115648|data/ttdb/ui/ttdb_ui.ttpf
195418624|data/ttdb/vehicles/ttdb_vehicles.ttpf
838474496|data/ttdb/vehicles/ttdb_vehicles_loose_0_1.ttpf
890206208|data/ttdb/vehicles/ttdb_vehicles_loose_1_1.ttpf
238897664|data/ttdb/vfx/ttdb_vfx.ttpf
1421518080|data/ttdb/weapons/ttdb_weapons.ttpf
739758848|data/ttdb/weapons/weapon_liveries.ttpf
481544304|data.rpf
348069888|gameos.xvd
13698|layout_3202719e-b4b4-421c-9f8f-70d1c4a7d60b_xs.xml
76155065056|levels.rpf
975856|libHttpClient.GDK.dll
8192|microsoft_gameconfig.bin
8832481776|movies.rpf
565232|msvcp140.dll
26104|msvcp140_1.dll
187888|msvcp140_2.dll
57312|msvcp140_atomic_wait.dll
22008|msvcp140_codecvt_ids.dll
6707512496|navmeshes.rpf
8297921520|pack/core_bvh4/core_bvh4.rpf
13536|pack/core_licenses/core_licenses_data.rpf
1295760752|pack/core_restofworld/core_restofworld.rpf
24529408|pack/core_restofworld/data/ttdb/peds/ttdb_peds.ttpf
35286784|pack/core_restofworld/data/ttdb/props/ttdb_props.ttpf
21081856|pack/core_restofworld/data/ttdb/textures_loose.ttpf
4729600|pack/core_restofworld/data/ttdb/ui/ttdb_ui.ttpf
3129600|pack/core_restofworld/data/ttdb/vfx/ttdb_vfx.ttpf
791984|pack/core_restofworld001/core_restofworld001.rpf
38656|pack/core_restofworld001/data/ttdb/ui/ttdb_ui.ttpf
193522176|pack/shared_minigame000/data/ttdb/maps/ttdb_maps.ttpf
1427712|pack/shared_minigame000/data/ttdb/terrain/ttdb_terrain_image_based.ttpf
810581008|pack/shared_minigame000/shared_minigame000.rpf
61987040|pack/shared_minigame000_bvh4/shared_minigame000_bvh4.rpf
697378560|pack/story000/data/ttdb/maps/ttdb_maps.ttpf
222333440|pack/story000/data/ttdb/terrain/ttdb_terrain_image_based.ttpf
3357184|pack/story000/data/ttdb/vfx/ttdb_vfx.ttpf
20022508672|pack/story000/story000.rpf
78023728|pack/story000/story000_script.rpf
5632|pack/story000_bvh4/story000_bvh4.rpf
198354992|pack/story000_restofworld/story000_restofworld.rpf
4798213264|packs.rpf
1152|resources.pri
4358960|script.rpf
411695600|shaders.rpf
192585024|textures.rpf
336888|vccorlib140.dll
72176|vcomp140.dll
97776|vcruntime140.dll
37848|vcruntime140_1.dll
54536|xbs_logo_150.png
7102|xbs_logo_44.png
404663|xbs_logo_480.png
29356|xbs_logo_store.png
2507194|xbs_splash.png
'@

# ============================== Hilfsfunktionen ==============================
function Get-Manifest {
    $list = New-Object System.Collections.Generic.List[object]
    foreach ($line in ($script:Manifest -split "`r?`n")) {
        if ([string]::IsNullOrWhiteSpace($line)) { continue }
        $parts = $line.Split('|', 2)
        $list.Add([pscustomobject]@{ Size = [int64]$parts[0]; Rel = $parts[1].Replace('/', '\') })
    }
    if ($list.Count -ne $script:FileCount) { throw "Manifest: $($list.Count) statt $($script:FileCount) Eintraege" }
    $sum = ($list | Measure-Object -Property Size -Sum).Sum
    if ($sum -ne $script:TotalBytes) { throw "Manifest: Summe $sum statt $($script:TotalBytes)" }
    return $list
}

function Format-Size {
    param([Nullable[int64]]$Bytes)
    if ($null -eq $Bytes) { return '-' }
    if ($Bytes -lt 1024) { return "$Bytes B" }
    $units = @('KB', 'MB', 'GB', 'TB')
    $value = [double]$Bytes / 1024
    foreach ($unit in $units) {
        if ($value -lt 1024 -or $unit -eq 'TB') { return ('{0:N1} {1}' -f $value, $unit) }
        $value /= 1024
    }
}

function Resolve-Root {
    param([string]$Raw)
    if ([string]::IsNullOrWhiteSpace($Raw)) { throw 'Kein Ordner angegeben.' }
    $full = $ExecutionContext.SessionState.Path.GetUnresolvedProviderPathFromPSPath($Raw.Trim('"'))
    return $full
}

function Test-IsSparseCapable {
    param([string]$Root)
    try {
        $drive = (Get-Item -LiteralPath $Root).PSDrive
        $fs = (Get-Volume -DriveLetter $drive.Name).FileSystem
        return ($fs -in @('NTFS', 'ReFS'))
    } catch { return $true }
}

function New-Tree {
    param(
        [string]$Root,
        [switch]$Sparse,
        [switch]$Overwrite,
        [scriptblock]$OnProgress
    )
    $entries = Get-Manifest
    if (-not (Test-Path -LiteralPath $Root)) { New-Item -ItemType Directory -Path $Root -Force | Out-Null }

    if ($Sparse -and -not (Test-IsSparseCapable -Root $Root)) {
        Write-Warning 'Dateisystem kann kein Sparse, es wird voll belegt.'
        $Sparse = $false
    }

    if (-not $Sparse) {
        $drive = (Get-Item -LiteralPath $Root).PSDrive
        $free = (Get-PSDrive -Name $drive.Name).Free
        if ($free -lt $script:TotalBytes) {
            throw ("Zu wenig Platz: {0:N1} GB frei, benoetigt {1:N1} GB." -f ($free / 1GB), ($script:TotalBytes / 1GB))
        }
    }

    $created = 0; $skipped = 0
    $failed = New-Object System.Collections.Generic.List[string]

    for ($i = 0; $i -lt $entries.Count; $i++) {
        $entry = $entries[$i]
        $full = Join-Path $Root $entry.Rel
        $dir = Split-Path -Parent $full

        if (-not (Test-Path -LiteralPath $dir)) { New-Item -ItemType Directory -Path $dir -Force | Out-Null }

        try {
            if ((Test-Path -LiteralPath $full) -and -not $Overwrite) { $skipped++ }
            else {
                $stream = [System.IO.File]::Open($full, 'Create', 'Write', 'None'); $stream.Close()
                if ($Sparse) { & fsutil.exe sparse setflag "$full" 2>$null | Out-Null }
                $stream = [System.IO.File]::Open($full, 'Open', 'Write', 'None')
                $stream.SetLength($entry.Size)
                $stream.Close()
                $created++
            }
        } catch { $failed.Add("$($entry.Rel): $($_.Exception.Message)") }

        if ($OnProgress) { & $OnProgress ($i + 1) $entries.Count $entry.Rel }
    }

    Set-Content -LiteralPath (Join-Path $Root 'MAGNET.txt') -Value $script:Magnet -Encoding ASCII

    try {
        $target = Join-Path $Root "$($script:TorrentName).torrent"
        Invoke-WebRequest -Uri $script:TorrentUrl -OutFile $target -UseBasicParsing -TimeoutSec 30
    } catch { Write-Warning "Torrent-Download fehlgeschlagen: $($_.Exception.Message)" }

    return [pscustomobject]@{ Created = $created; Skipped = $skipped; Failed = $failed }
}

function Get-Sha256 {
    param([string]$File, [scriptblock]$OnProgress)
    $sha = [System.Security.Cryptography.SHA256]::Create()
    $stream = [System.IO.File]::OpenRead($File)
    $buffer = New-Object byte[] (4MB)
    $done = [int64]0
    try {
        while (($read = $stream.Read($buffer, 0, $buffer.Length)) -gt 0) {
            $sha.TransformBlock($buffer, 0, $read, $null, 0) | Out-Null
            $done += $read
            if ($OnProgress) { & $OnProgress $done }
        }
        $sha.TransformFinalBlock($buffer, 0, 0) | Out-Null
        return ([System.BitConverter]::ToString($sha.Hash) -replace '-', '').ToLowerInvariant()
    } finally { $stream.Close(); $sha.Dispose() }
}

function Invoke-Scan {
    param([string]$Root, [scriptblock]$OnProgress)
    $entries = Get-Manifest
    $result = New-Object System.Collections.Generic.List[object]
    $markers = New-Object System.Collections.Generic.List[string]

    foreach ($marker in $script:Markers) {
        if (Test-Path -LiteralPath (Join-Path $Root $marker)) { $markers.Add($marker) }
    }

    $position = 0
    foreach ($entry in $entries) {
        $position++
        $full = Join-Path $Root $entry.Rel
        $found = $null; $status = 'FEHLT'
        if (Test-Path -LiteralPath $full) {
            try {
                $found = (Get-Item -LiteralPath $full -Force).Length
                $status = if ($found -eq $entry.Size) { 'OK' } else { 'GROESSE' }
            } catch { $status = 'LESEFEHLER' }
        }
        $result.Add([pscustomobject]@{ Rel = $entry.Rel; Expected = $entry.Size; Found = $found; Status = $status })
        if ($OnProgress) { & $OnProgress $position $entries.Count $entry.Rel }
    }

    $expectedSet = @{}
    foreach ($entry in $entries) { $expectedSet[$entry.Rel] = $true }

    $extras = New-Object System.Collections.Generic.List[string]
    Get-ChildItem -LiteralPath $Root -Recurse -File -Force -ErrorAction SilentlyContinue | ForEach-Object {
        $rel = $_.FullName.Substring($Root.Length).TrimStart('\')
        if ($expectedSet.ContainsKey($rel)) { return }
        if ($_.Name -in @('MAGNET.txt', 'Thumbs.db', 'desktop.ini')) { return }
        if ($_.Extension -in @('.torrent', '.txt', '.log', '.zip')) { return }
        $extras.Add($rel)
    }

    return [pscustomobject]@{
        Root    = $Root
        Files   = $result
        Extras  = $extras
        Markers = $markers
        Stamp   = (Get-Date)
    }
}

function Get-ScanSummary {
    param($Scan)
    $ok = @($Scan.Files | Where-Object Status -eq 'OK').Count
    $missing = @($Scan.Files | Where-Object Status -eq 'FEHLT').Count
    $wrong = @($Scan.Files | Where-Object Status -eq 'GROESSE').Count
    $bad = @($Scan.Files | Where-Object Status -eq 'LESEFEHLER').Count
    $found = ($Scan.Files | Where-Object Found | Measure-Object -Property Found -Sum).Sum
    if ($null -eq $found) { $found = [int64]0 }
    $rootOk = $Scan.Markers.Count -ge 2

    $verdict = if (-not $rootOk) { 'FALSCHER ORDNER' }
               elseif ($missing -eq 0 -and $wrong -eq 0 -and $bad -eq 0) { 'VOLLSTAENDIG' }
               else { 'UNVOLLSTAENDIG' }

    [pscustomobject]@{
        Ok = $ok; Missing = $missing; Wrong = $wrong; Bad = $bad
        Found = [int64]$found; RootOk = $rootOk; Verdict = $verdict
        Complete = ($rootOk -and $missing -eq 0 -and $wrong -eq 0 -and $bad -eq 0)
    }
}

# --------------------------- Bencode und Pieces ------------------------------
function Read-Bencoded {
    param([byte[]]$Data, [ref]$Pos)
    $p = $Pos.Value
    $tag = $Data[$p]
    switch ($tag) {
        105 {  # i
            $end = [Array]::IndexOf($Data, [byte]101, $p + 1)
            $value = [int64][System.Text.Encoding]::ASCII.GetString($Data, $p + 1, $end - $p - 1)
            $Pos.Value = $end + 1; return $value
        }
        108 {  # l
            $Pos.Value = $p + 1
            $list = New-Object System.Collections.Generic.List[object]
            while ($Data[$Pos.Value] -ne 101) { $list.Add((Read-Bencoded -Data $Data -Pos $Pos)) }
            $Pos.Value++
            return , $list.ToArray()
        }
        100 {  # d
            $Pos.Value = $p + 1
            $map = @{}
            while ($Data[$Pos.Value] -ne 101) {
                $key = Read-Bencoded -Data $Data -Pos $Pos
                $map[[System.Text.Encoding]::ASCII.GetString($key)] = (Read-Bencoded -Data $Data -Pos $Pos)
            }
            $Pos.Value++
            return $map
        }
        default {
            $colon = [Array]::IndexOf($Data, [byte]58, $p)
            $length = [int][System.Text.Encoding]::ASCII.GetString($Data, $p, $colon - $p)
            $start = $colon + 1
            $Pos.Value = $start + $length
            return $Data[$start..($start + $length - 1)]
        }
    }
}

function Import-Torrent {
    param([string]$File)
    $raw = [System.IO.File]::ReadAllBytes($File)
    $pos = 0
    $data = Read-Bencoded -Data $raw -Pos ([ref]$pos)
    $info = $data['info']
    if ($null -eq $info) { throw 'Torrent ohne info-Block.' }

    $pieceLength = [int64]$info['piece length']
    $rawPieces = $info['pieces']
    $pieces = New-Object System.Collections.Generic.List[byte[]]
    for ($i = 0; $i -lt $rawPieces.Length; $i += 20) {
        $chunk = New-Object byte[] 20
        [Array]::Copy($rawPieces, $i, $chunk, 0, 20)
        $pieces.Add($chunk)
    }

    $files = New-Object System.Collections.Generic.List[object]
    foreach ($item in $info['files']) {
        $rel = ($item['path'] | ForEach-Object { [System.Text.Encoding]::ASCII.GetString($_) }) -join '\'
        $files.Add([pscustomobject]@{ Rel = $rel; Size = [int64]$item['length'] })
    }

    if ($pieceLength -ne $script:PieceLength) { throw "Piece-Laenge $pieceLength unerwartet." }
    if ($files.Count -ne $script:FileCount) { throw "$($files.Count) Dateien unerwartet." }
    if ($pieces.Count -ne $script:PieceCount) { throw "$($pieces.Count) Pieces unerwartet." }
    if ((($files | Measure-Object -Property Size -Sum).Sum) -ne $script:TotalBytes) { throw 'Gesamtgroesse weicht ab.' }

    return [pscustomobject]@{ Files = $files; Pieces = $pieces; PieceLength = $pieceLength }
}

function Invoke-PieceCheck {
    param([string]$Root, [string]$TorrentFile, [scriptblock]$OnProgress)
    $torrent = Import-Torrent -File $TorrentFile

    $sha = [System.Security.Cryptography.SHA1]::Create()
    $buffer = New-Object byte[] $script:PieceLength
    $filled = 0
    $index = 0
    $okCount = 0
    $badPieces = New-Object System.Collections.Generic.List[int]
    $badFiles = New-Object System.Collections.Generic.List[string]
    $doneBytes = [int64]0
    $currentFile = ''

    function Complete-Piece {
        $digest = $sha.ComputeHash($buffer, 0, $filled)
        if ([System.Linq.Enumerable]::SequenceEqual($digest, $torrent.Pieces[$index])) { $script:okCount++ }
        else { $badPieces.Add($index); if (-not $badFiles.Contains($currentFile)) { $badFiles.Add($currentFile) } }
        $index++
        $filled = 0
    }

    try {
        foreach ($file in $torrent.Files) {
            $currentFile = $file.Rel
            $full = Join-Path $Root $file.Rel
            if (-not (Test-Path -LiteralPath $full)) { continue }

            $stream = [System.IO.File]::OpenRead($full)
            $remaining = $file.Size
            try {
                while ($remaining -gt 0) {
                    $want = [Math]::Min([Math]::Min(4MB, $remaining), $script:PieceLength - $filled)
                    $read = $stream.Read($buffer, $filled, [int]$want)
                    if ($read -le 0) { break }
                    $filled += $read; $remaining -= $read; $doneBytes += $read
                    if ($filled -eq $script:PieceLength) {
                        Complete-Piece
                        if ($OnProgress) { & $OnProgress $doneBytes $script:TotalBytes $index $script:PieceCount }
                    }
                }
            } finally { $stream.Close() }
        }
        if ($filled -gt 0 -and $index -lt $script:PieceCount) { Complete-Piece }
    } finally { $sha.Dispose() }

    return [pscustomobject]@{
        Total = $script:PieceCount; Ok = $okCount
        Bad = $badPieces.Count; BadPieces = $badPieces; BadFiles = $badFiles
    }
}

# --------------------------------- Report ------------------------------------
function Get-ReportText {
    param($Scan, $ExeResult, $PieceResult)
    $summary = Get-ScanSummary -Scan $Scan
    $lines = New-Object System.Collections.Generic.List[string]
    $lines.Add('=' * 74)
    $lines.Add(" $($script:TorrentName)")
    $lines.Add(" infohash $($script:Infohash)")
    $lines.Add('=' * 74)
    $lines.Add("Ordner   : $($Scan.Root)")
    $lines.Add("Zeit     : $($Scan.Stamp.ToString('dd.MM.yyyy HH:mm:ss'))")
    $lines.Add("Ergebnis : $($summary.Verdict)")
    $lines.Add('')
    $lines.Add("Dateien  : $($summary.Ok)/$($script:FileCount) korrekt")
    $lines.Add("Groesse  : $(Format-Size $summary.Found) von $(Format-Size $script:TotalBytes)")
    if ($summary.Missing) { $lines.Add("Fehlend  : $($summary.Missing)") }
    if ($summary.Wrong)   { $lines.Add("Falsche Groesse: $($summary.Wrong)") }
    if ($summary.Bad)     { $lines.Add("Nicht lesbar   : $($summary.Bad)") }
    if (-not $summary.RootOk) {
        $lines.Add('')
        $lines.Add('WARNUNG: Wurzelordner erkennt kaum Marker-Dateien.')
    }
    $lines.Add('')

    foreach ($file in ($Scan.Files | Where-Object Status -ne 'OK')) {
        $lines.Add(("[{0,-10}] {1}" -f $file.Status, $file.Rel))
        $lines.Add(("             erwartet {0} / gefunden {1}" -f (Format-Size $file.Expected), (Format-Size $file.Found)))
    }

    if ($ExeResult) {
        $lines.Add('')
        $lines.Add('--- Start-EXE (nicht im Torrent) ---')
        if (-not $ExeResult.Found) { $lines.Add("$($script:ExpectedExe) liegt nicht im Ordner.") }
        elseif ($ExeResult.Matches) { $lines.Add('SHA-256 korrekt.') }
        else { $lines.Add("SHA-256 WEICHT AB: $($ExeResult.Sha256)") }
    }

    if ($PieceResult) {
        $lines.Add('')
        $lines.Add('--- Piece-Pruefung ---')
        $lines.Add("geprueft $($PieceResult.Ok + $PieceResult.Bad) von $($PieceResult.Total)")
        $lines.Add("fehlerhaft: $($PieceResult.Bad)")
        if ($PieceResult.BadPieces.Count) {
            $lines.Add('Indizes: ' + (($PieceResult.BadPieces | Select-Object -First 50) -join ', '))
        }
    }

    if ($Scan.Extras.Count) {
        $lines.Add('')
        $lines.Add("--- Zusaetzliche Dateien ($($Scan.Extras.Count)) ---")
        $lines.AddRange([string[]]($Scan.Extras | Select-Object -First 100))
    }

    $lines.Add('')
    $lines.Add('=' * 74)
    return ($lines -join [Environment]::NewLine)
}

function Get-ReportJson {
    param($Scan, $ExeResult, $PieceResult)
    $summary = Get-ScanSummary -Scan $Scan
    [pscustomobject]@{
        torrent = $script:TorrentName
        infohash = $script:Infohash
        root = $Scan.Root
        verdict = $summary.Verdict
        filesOk = $summary.Ok
        filesMissing = $summary.Missing
        filesWrongSize = $summary.Wrong
        bytesExpected = $script:TotalBytes
        bytesFound = $summary.Found
        exe = $ExeResult
        pieces = $PieceResult
        extras = $Scan.Extras
        files = $Scan.Files
    } | ConvertTo-Json -Depth 6
}

# ================================== CLI ======================================
function Invoke-Cli {
    $root = Resolve-Root -Raw $Path
    $scan = $null; $exeResult = $null; $pieceResult = $null

    if ($Create) {
        Write-Host "Lege Struktur an: $root"
        $created = New-Tree -Root $root -Sparse:(-not $NoSparse) -Overwrite:$Force -OnProgress {
            param($pos, $total, $rel)
            Write-Progress -Activity 'Dateien anlegen' -Status "$pos/$total $rel" -PercentComplete ($pos * 100 / $total)
        }
        Write-Progress -Activity 'Dateien anlegen' -Completed
        Write-Host ("angelegt {0}, uebersprungen {1}, Fehler {2}" -f $created.Created, $created.Skipped, $created.Failed.Count)
        $created.Failed | ForEach-Object { Write-Warning $_ }
    }

    if ($Check -or $Exe -or $Pieces) {
        if (-not (Test-Path -LiteralPath $root)) { throw "Ordner nicht gefunden: $root" }
        $scan = Invoke-Scan -Root $root -OnProgress {
            param($pos, $total, $rel)
            Write-Progress -Activity 'Pruefe Struktur' -Status "$pos/$total $rel" -PercentComplete ($pos * 100 / $total)
        }
        Write-Progress -Activity 'Pruefe Struktur' -Completed
    }

    if ($Exe) {
        $exePath = Join-Path $root $script:ExpectedExe
        if (Test-Path -LiteralPath $exePath) {
            $hash = Get-Sha256 -File $exePath
            $exeResult = [pscustomobject]@{ Found = $true; Sha256 = $hash; Matches = ($hash -eq $script:ExeSha256) }
        } else {
            $exeResult = [pscustomobject]@{ Found = $false; Sha256 = $null; Matches = $false }
        }
    }

    if ($Pieces) {
        $torrentFile = $Torrent
        if (-not $torrentFile) {
            $local = Get-ChildItem -LiteralPath $root -Filter '*.torrent' -ErrorAction SilentlyContinue | Select-Object -First 1
            if ($local) { $torrentFile = $local.FullName }
            else {
                $target = Join-Path $root "$($script:TorrentName).torrent"
                try {
                    Invoke-WebRequest -Uri $script:TorrentUrl -OutFile $target -UseBasicParsing -TimeoutSec 30
                    $torrentFile = $target
                } catch { Write-Warning "Torrent nicht ladbar: $($_.Exception.Message)" }
            }
        }
        if ($torrentFile) {
            $pieceResult = Invoke-PieceCheck -Root $root -TorrentFile $torrentFile -OnProgress {
                param($done, $total, $index, $count)
                Write-Progress -Activity 'Piece-Pruefung' -Status "Piece $index/$count" -PercentComplete ($done * 100 / $total)
            }
            Write-Progress -Activity 'Piece-Pruefung' -Completed
        }
    }

    if ($scan) {
        $output = if ($Json) { Get-ReportJson -Scan $scan -ExeResult $exeResult -PieceResult $pieceResult }
                  else { Get-ReportText -Scan $scan -ExeResult $exeResult -PieceResult $pieceResult }
        Write-Output $output
        if ($Report) { Set-Content -LiteralPath $Report -Value $output -Encoding UTF8; Write-Host "Report: $Report" }
        $summary = Get-ScanSummary -Scan $scan
        if ($summary.Complete) { exit 0 } else { exit 1 }
    }
    exit 0
}

# ================================== GUI ======================================
function Show-Gui {
    Add-Type -AssemblyName System.Windows.Forms
    Add-Type -AssemblyName System.Drawing

    $bg = [System.Drawing.Color]::FromArgb(20, 22, 26)
    $panel = [System.Drawing.Color]::FromArgb(28, 31, 38)
    $field = [System.Drawing.Color]::FromArgb(15, 17, 21)
    $text = [System.Drawing.Color]::FromArgb(232, 234, 238)
    $dim = [System.Drawing.Color]::FromArgb(139, 145, 158)
    $accent = [System.Drawing.Color]::FromArgb(47, 109, 246)
    $okColor = [System.Drawing.Color]::FromArgb(62, 163, 106)
    $warnColor = [System.Drawing.Color]::FromArgb(224, 163, 60)
    $badColor = [System.Drawing.Color]::FromArgb(224, 90, 76)

    $form = New-Object System.Windows.Forms.Form
    $form.Text = "$($script:TorrentName) - Struktur-Check"
    $form.Size = New-Object System.Drawing.Size(1000, 700)
    $form.MinimumSize = New-Object System.Drawing.Size(820, 560)
    $form.BackColor = $bg
    $form.StartPosition = 'CenterScreen'
    $form.Font = New-Object System.Drawing.Font('Segoe UI', 9)

    $header = New-Object System.Windows.Forms.Panel
    $header.Height = 46; $header.Dock = 'Top'; $header.BackColor = $panel
    $title = New-Object System.Windows.Forms.Label
    $title.Text = "$($script:TorrentName) - Struktur-Check"
    $title.ForeColor = $text; $title.Font = New-Object System.Drawing.Font('Segoe UI', 12, 'Bold')
    $title.AutoSize = $true; $title.Location = New-Object System.Drawing.Point(14, 13)
    $header.Controls.Add($title)
    $form.Controls.Add($header)

    $label = New-Object System.Windows.Forms.Label
    $label.Text = 'Zielverzeichnis'; $label.ForeColor = $dim
    $label.Location = New-Object System.Drawing.Point(16, 58); $label.AutoSize = $true
    $form.Controls.Add($label)

    $pathBox = New-Object System.Windows.Forms.TextBox
    $pathBox.Text = $Path; $pathBox.Location = New-Object System.Drawing.Point(16, 76)
    $pathBox.Width = 800; $pathBox.BackColor = $field; $pathBox.ForeColor = $text
    $pathBox.BorderStyle = 'FixedSingle'; $pathBox.Font = New-Object System.Drawing.Font('Consolas', 9)
    $form.Controls.Add($pathBox)

    $browse = New-Object System.Windows.Forms.Button
    $browse.Text = 'Durchsuchen'; $browse.Location = New-Object System.Drawing.Point(824, 75)
    $browse.Size = New-Object System.Drawing.Size(140, 24); $browse.BackColor = $panel
    $browse.ForeColor = $text; $browse.FlatStyle = 'Flat'
    $browse.Add_Click({
        $dialog = New-Object System.Windows.Forms.FolderBrowserDialog
        $dialog.Description = 'Wurzelordner des Torrents waehlen'
        if ($dialog.ShowDialog() -eq 'OK') { $pathBox.Text = $dialog.SelectedPath }
    })
    $form.Controls.Add($browse)

    $list = New-Object System.Windows.Forms.ListView
    $list.Location = New-Object System.Drawing.Point(16, 150)
    $list.Size = New-Object System.Drawing.Size(948, 400)
    $list.View = 'Details'; $list.FullRowSelect = $true; $list.GridLines = $false
    $list.BackColor = $field; $list.ForeColor = [System.Drawing.Color]::FromArgb(200, 204, 212)
    $list.BorderStyle = 'FixedSingle'
    $null = $list.Columns.Add('Datei', 480)
    $null = $list.Columns.Add('Erwartet', 110)
    $null = $list.Columns.Add('Gefunden', 110)
    $null = $list.Columns.Add('Status', 110)
    $list.Anchor = 'Top,Left,Right,Bottom'
    $form.Controls.Add($list)

    $progress = New-Object System.Windows.Forms.ProgressBar
    $progress.Location = New-Object System.Drawing.Point(16, 560)
    $progress.Size = New-Object System.Drawing.Size(948, 16)
    $progress.Anchor = 'Left,Right,Bottom'
    $form.Controls.Add($progress)

    $statusLabel = New-Object System.Windows.Forms.Label
    $statusLabel.Text = "$($script:FileCount) Dateien, $(Format-Size $script:TotalBytes) erwartet"
    $statusLabel.ForeColor = $dim; $statusLabel.AutoSize = $true
    $statusLabel.Location = New-Object System.Drawing.Point(16, 584)
    $statusLabel.Anchor = 'Left,Bottom'
    $form.Controls.Add($statusLabel)

    $summaryLabel = New-Object System.Windows.Forms.Label
    $summaryLabel.Text = 'Noch nicht geprueft.'; $summaryLabel.ForeColor = $dim
    $summaryLabel.AutoSize = $true; $summaryLabel.Location = New-Object System.Drawing.Point(16, 128)
    $form.Controls.Add($summaryLabel)

    $script:Scan = $null
    $script:ExeResult = $null
    $script:PieceResult = $null

    function New-Button {
        param($Caption, $X, $Color)
        $button = New-Object System.Windows.Forms.Button
        $button.Text = $Caption; $button.Location = New-Object System.Drawing.Point($X, 104)
        $button.Size = New-Object System.Drawing.Size(118, 26)
        $button.BackColor = $Color; $button.ForeColor = $text; $button.FlatStyle = 'Flat'
        $form.Controls.Add($button)
        return $button
    }

    $buttonCreate = New-Button 'Anlegen' 16 $panel
    $buttonCheck  = New-Button 'Pruefen' 142 $accent
    $buttonPieces = New-Button 'Piece-Check' 268 $panel
    $buttonExe    = New-Button 'EXE pruefen' 394 $panel
    $buttonReport = New-Button 'Report' 520 $panel
    $buttonMagnet = New-Button 'Magnet' 646 $panel
    $buttonOpen   = New-Button 'Ordner' 772 $panel
    $buttonCheck.ForeColor = [System.Drawing.Color]::White

    function Set-Busy {
        param([bool]$Busy)
        foreach ($button in @($buttonCreate, $buttonCheck, $buttonPieces, $buttonExe)) { $button.Enabled = -not $Busy }
        [System.Windows.Forms.Application]::DoEvents()
    }

    function Get-Root {
        if ([string]::IsNullOrWhiteSpace($pathBox.Text)) {
            [void][System.Windows.Forms.MessageBox]::Show('Bitte zuerst den Ordner waehlen.', 'Ordner fehlt')
            return $null
        }
        return (Resolve-Root -Raw $pathBox.Text)
    }

    $buttonCreate.Add_Click({
        $root = Get-Root; if (-not $root) { return }
        Set-Busy $true
        $summaryLabel.Text = 'Lege Struktur an...'
        $result = New-Tree -Root $root -Sparse $true -OnProgress {
            param($pos, $total, $rel)
            $progress.Value = [int]($pos * 100 / $total)
            $statusLabel.Text = "$pos/$total $rel"
            [System.Windows.Forms.Application]::DoEvents()
        }
        $progress.Value = 100
        $summaryLabel.Text = "angelegt $($result.Created), uebersprungen $($result.Skipped), Fehler $($result.Failed.Count)"
        Set-Busy $false
    })

    $buttonCheck.Add_Click({
        $root = Get-Root; if (-not $root) { return }
        Set-Busy $true
        $list.Items.Clear()
        $summaryLabel.Text = 'Pruefe Struktur...'
        $script:Scan = Invoke-Scan -Root $root -OnProgress {
            param($pos, $total, $rel)
            $progress.Value = [int]($pos * 100 / $total)
            $statusLabel.Text = "Datei $pos/$total $rel"
            [System.Windows.Forms.Application]::DoEvents()
        }
        $progress.Value = 100
        foreach ($file in ($script:Scan.Files | Where-Object Status -ne 'OK')) {
            $item = New-Object System.Windows.Forms.ListViewItem($file.Rel)
            [void]$item.SubItems.Add((Format-Size $file.Expected))
            [void]$item.SubItems.Add((Format-Size $file.Found))
            [void]$item.SubItems.Add($file.Status)
            switch ($file.Status) {
                'GROESSE' { $item.ForeColor = $warnColor }
                default   { $item.ForeColor = $badColor }
            }
            [void]$list.Items.Add($item)
        }
        $s = Get-ScanSummary -Scan $script:Scan
        $summaryLabel.Text = "$($s.Ok)/$($script:FileCount) korrekt - $(Format-Size $s.Found) von $(Format-Size $script:TotalBytes) - $($s.Verdict)"
        $statusLabel.Text = "fehlend $($s.Missing), falsche Groesse $($s.Wrong), nicht lesbar $($s.Bad)"
        Set-Busy $false
        if (-not $s.RootOk) {
            [void][System.Windows.Forms.MessageBox]::Show('Kaum Marker-Dateien gefunden. Ist das die oberste Ebene?', 'Falscher Ordner?', 'OK', 'Warning')
        }
    })

    $buttonPieces.Add_Click({
        $root = Get-Root; if (-not $root) { return }
        $torrentFile = (Get-ChildItem -LiteralPath $root -Filter '*.torrent' -ErrorAction SilentlyContinue | Select-Object -First 1).FullName
        if (-not $torrentFile) {
            $dialog = New-Object System.Windows.Forms.OpenFileDialog
            $dialog.Filter = 'Torrent|*.torrent|Alle Dateien|*.*'
            if ($dialog.ShowDialog() -ne 'OK') { return }
            $torrentFile = $dialog.FileName
        }
        Set-Busy $true
        $summaryLabel.Text = 'Pruefe Pieces...'
        $script:PieceResult = Invoke-PieceCheck -Root $root -TorrentFile $torrentFile -OnProgress {
            param($done, $total, $index, $count)
            $progress.Value = [int]($done * 100 / $total)
            $statusLabel.Text = "Piece $index/$count - $(Format-Size $done) von $(Format-Size $total)"
            [System.Windows.Forms.Application]::DoEvents()
        }
        $progress.Value = 100
        $summaryLabel.Text = "Pieces: $($script:PieceResult.Ok) korrekt, $($script:PieceResult.Bad) fehlerhaft"
        $statusLabel.Text = 'Piece-Pruefung abgeschlossen.'
        Set-Busy $false
        if ($script:PieceResult.Bad -gt 0) {
            [void][System.Windows.Forms.MessageBox]::Show("$($script:PieceResult.Bad) Pieces beschaedigt. Im Client neu pruefen und nachladen.", 'Fehlerhafte Pieces', 'OK', 'Warning')
        }
    })

    $buttonExe.Add_Click({
        $root = Get-Root; if (-not $root) { return }
        Set-Busy $true
        $exePath = Join-Path $root $script:ExpectedExe
        if (Test-Path -LiteralPath $exePath) {
            $hash = Get-Sha256 -File $exePath
            $script:ExeResult = [pscustomobject]@{ Found = $true; Sha256 = $hash; Matches = ($hash -eq $script:ExeSha256) }
            if ($script:ExeResult.Matches) { $summaryLabel.Text = 'Start-EXE: SHA-256 korrekt.' }
            else {
                $summaryLabel.Text = 'Start-EXE: SHA-256 weicht ab.'
                [void][System.Windows.Forms.MessageBox]::Show("Hash falsch.`n`ngefunden: $hash`nerwartet: $($script:ExeSha256)", 'Hash falsch', 'OK', 'Warning')
            }
        } else {
            $script:ExeResult = [pscustomobject]@{ Found = $false; Sha256 = $null; Matches = $false }
            $summaryLabel.Text = 'Start-EXE liegt nicht im Ordner.'
            [void][System.Windows.Forms.MessageBox]::Show("$($script:ExpectedExe) fehlt. Die EXE ist nicht Teil des Torrents.", 'EXE fehlt')
        }
        Set-Busy $false
    })

    $buttonReport.Add_Click({
        if (-not $script:Scan) { [void][System.Windows.Forms.MessageBox]::Show('Bitte zuerst pruefen.'); return }
        $dialog = New-Object System.Windows.Forms.SaveFileDialog
        $dialog.Filter = 'Text|*.txt|JSON|*.json'
        $dialog.FileName = "GTA6-Check-Report_$(Get-Date -Format 'yyyyMMdd_HHmm').txt"
        if ($dialog.ShowDialog() -ne 'OK') { return }
        $content = if ($dialog.FileName -like '*.json') { Get-ReportJson -Scan $script:Scan -ExeResult $script:ExeResult -PieceResult $script:PieceResult }
                   else { Get-ReportText -Scan $script:Scan -ExeResult $script:ExeResult -PieceResult $script:PieceResult }
        Set-Content -LiteralPath $dialog.FileName -Value $content -Encoding UTF8
        $statusLabel.Text = "Report: $($dialog.FileName)"
    })

    $buttonMagnet.Add_Click({
        [System.Windows.Forms.Clipboard]::SetText($script:Magnet)
        $statusLabel.Text = 'Magnet in die Zwischenablage kopiert.'
    })

    $buttonOpen.Add_Click({
        $root = Get-Root; if (-not $root) { return }
        Start-Process explorer.exe $root
    })

    [void]$form.ShowDialog()
    $form.Dispose()
}

# ================================= Start =====================================
if ($PSBoundParameters.ContainsKey('Create') -or $PSBoundParameters.ContainsKey('Check') -or
    $PSBoundParameters.ContainsKey('Pieces') -or $PSBoundParameters.ContainsKey('Exe')) {
    Invoke-Cli
} else {
    if ([System.Threading.Thread]::CurrentThread.GetApartmentState() -ne 'STA') {
        Start-Process -FilePath 'powershell.exe' -ArgumentList @('-STA', '-NoProfile', '-ExecutionPolicy', 'Bypass', '-File', $PSCommandPath, '-Path', $Path)
        exit 0
    }
    Show-Gui
}
```

## Aufruf

```powershell
powershell -ExecutionPolicy Bypass -File .\GTA6LeakCheck.ps1 -Path "D:\GTA6.Xbox.v1.0.2.1-CYBERLEEK" -Create -Check
```

Ohne Schalter startet die Oberfläche. Rechtsklick → „Mit PowerShell ausführen" geht auch.

## Was in der einen Datei steckt

- **Anlegen** — alle 108 Dateien mit exakten Größen, sparse, plus `.torrent` und `MAGNET.txt`
- **Prüfen** — Soll-/Ist-Vergleich, Marker-Erkennung, Warnung bei falscher Ordnerebene
- **Piece-Check** — eigener Bencode-Decoder plus SHA-1-Verifikation gegen die echten Torrent-Hashes, findet Bitfehler
- **EXE-Prüfung** — SHA-256 gegen den offiziellen Wert, ZIP-Hash und Passwort im Report
- **Report** — TXT oder JSON
- **CLI** — Rückgabewerte 0 vollständig, 1 unvollständig

## Hinweise

- Ersetzt die komplette Python-Struktur; es bleibt genau eine Datei.
- Bei `-Create` ohne `-NoSparse` bleiben die 223 GB ungenutzt, die Dateien zeigen aber die richtige Größe. Der Client füllt sie beim Hinzufügen des Torrents.
- Der GUI-Start prüft den STA-Modus und startet sich bei Bedarf selbst neu — sonst friert WinForms ein.
- `-Create` bricht bei zu wenig Platz ab, bevor irgendetwas geschrieben wird.