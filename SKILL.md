---
name: SKILL
description: Diagnose and fix repeated "Reconnecting 1/5" through "Reconnecting 5/5" errors in the Codex Windows desktop app when a local proxy such as v2rayN, Clash, Mihomo, or a Windows system proxy is in use. Use when Codex streaming disconnects or Codex child processes may not inherit the proxy.
---

# Fix Codex Reconnecting on Windows

Execute this workflow directly. Do not require any files besides this one.

## Safety

- Do not copy a port from another computer.
- Detect that computer's enabled Windows user proxy.
- Confirm the detected local port is listening before changing anything.
- Modify only the current Windows user's environment variables.
- Back up the four existing proxy variables before applying changes.
- Do not edit Codex conversations, providers, credentials, or `config.toml`.
- Do not terminate Codex automatically. Ask the user to exit it fully.
- Treat HTTP `401` or `403` from an unauthenticated probe as proof that the
  server was reached, not as a network failure.

## 1. Diagnose

Run this in PowerShell:

```powershell
$settings = Get-ItemProperty 'HKCU:\Software\Microsoft\Windows\CurrentVersion\Internet Settings'
$server = [string]$settings.ProxyServer

if ($settings.ProxyEnable -ne 1 -or [string]::IsNullOrWhiteSpace($server)) {
    throw 'Windows system proxy is not enabled. Ask the user for the HTTP or mixed proxy address.'
}

if ($server -match '=') {
    $map = @{}
    foreach ($part in ($server -split ';')) {
        if ($part -match '^\s*([^=]+)=(.+)\s*$') {
            $map[$matches[1].ToLowerInvariant()] = $matches[2]
        }
    }
    if ($map.ContainsKey('https')) { $server = $map['https'] }
    elseif ($map.ContainsKey('http')) { $server = $map['http'] }
    else { throw 'No HTTP-compatible proxy was found in ProxyServer.' }
}

if ($server -notmatch '^[a-z]+://') { $server = "http://$server" }
$proxyUrl = $server
$uri = [Uri]$proxyUrl
$test = Test-NetConnection -ComputerName $uri.Host -Port $uri.Port -WarningAction SilentlyContinue

[PSCustomObject]@{
    ProxyUrl = $proxyUrl
    Host = $uri.Host
    Port = $uri.Port
    Listening = $test.TcpTestSucceeded
} | Format-List

Get-ItemProperty 'HKCU:\Environment' |
    Select-Object HTTP_PROXY, HTTPS_PROXY, ALL_PROXY, NO_PROXY |
    Format-List

Get-Process -ErrorAction SilentlyContinue |
    Where-Object { $_.ProcessName -match 'codex|clash|mihomo|v2ray|sing|surge' } |
    Select-Object ProcessName, Id, StartTime |
    Format-Table -AutoSize
```

Stop if `Listening` is not `True`. Ask the user to start the proxy application
or provide the correct HTTP/mixed proxy URL. Do not guess the port.

## 2. Apply

Continue in the same PowerShell session so `$proxyUrl` retains the diagnosed
value:

```powershell
$environmentPath = 'HKCU:\Environment'
$names = @('HTTP_PROXY', 'HTTPS_PROXY', 'ALL_PROXY', 'NO_PROXY')
$backupDirectory = Join-Path $env:USERPROFILE '.codex\proxy-backups'
New-Item -ItemType Directory -Path $backupDirectory -Force | Out-Null
$backupPath = Join-Path $backupDirectory ("proxy-env-{0}.json" -f (Get-Date -Format 'yyyyMMdd-HHmmss'))

$backup = [ordered]@{}
foreach ($name in $names) {
    $backup[$name] = [Environment]::GetEnvironmentVariable($name, 'User')
}
$backup | ConvertTo-Json | Set-Content -LiteralPath $backupPath -Encoding UTF8

Set-ItemProperty -LiteralPath $environmentPath -Name HTTP_PROXY -Value $proxyUrl -Type String
Set-ItemProperty -LiteralPath $environmentPath -Name HTTPS_PROXY -Value $proxyUrl -Type String
Set-ItemProperty -LiteralPath $environmentPath -Name ALL_PROXY -Value $proxyUrl -Type String
Set-ItemProperty -LiteralPath $environmentPath -Name NO_PROXY -Value 'localhost,127.0.0.1,::1' -Type String

Write-Host "Backup: $backupPath"
Get-ItemProperty -LiteralPath $environmentPath |
    Select-Object HTTP_PROXY, HTTPS_PROXY, ALL_PROXY, NO_PROXY |
    Format-List
```

Preserve the printed backup path for rollback.

## 3. Probe

Run five short requests through the detected proxy:

```powershell
1..5 | ForEach-Object {
    $attempt = $_
    $watch = [Diagnostics.Stopwatch]::StartNew()
    try {
        $response = Invoke-WebRequest -Uri 'https://chatgpt.com/' -Method Head `
            -Proxy $proxyUrl -TimeoutSec 15 -UseBasicParsing
        $watch.Stop()
        [PSCustomObject]@{
            Attempt = $attempt
            ReachedServer = $true
            Status = $response.StatusCode
            Milliseconds = $watch.ElapsedMilliseconds
        }
    } catch {
        $watch.Stop()
        $status = if ($_.Exception.Response) {
            [int]$_.Exception.Response.StatusCode
        } else {
            $null
        }
        [PSCustomObject]@{
            Attempt = $attempt
            ReachedServer = ($status -in 401, 403)
            Status = $status
            Milliseconds = $watch.ElapsedMilliseconds
        }
    }
} | Format-Table -AutoSize
```

Accept `200`, `401`, or `403` as reaching the server. Investigate the proxy if
requests time out, reset, or cannot connect.

## 4. Restart and verify

Tell the user:

1. Fully exit every Codex window.
2. Confirm Codex is no longer present in Task Manager if necessary.
3. Reopen Codex.
4. Start a new conversation and send several messages or run a longer task.
5. Confirm that no `Reconnecting 1/5` banner appears.

After reopening, verify the saved values:

```powershell
Get-ItemProperty 'HKCU:\Environment' |
    Select-Object HTTP_PROXY, HTTPS_PROXY, ALL_PROXY, NO_PROXY |
    Format-List
```

If short probes succeed but Codex still reconnects, investigate proxy node
quality, packet loss, HTTP CONNECT/WebSocket support, or multiple competing
proxy applications. Do not modify conversation files.

## 5. Roll back when needed

Replace the path with the backup printed during Apply:

```powershell
$backupPath = 'C:\Users\USERNAME\.codex\proxy-backups\proxy-env-YYYYMMDD-HHMMSS.json'
$backup = Get-Content -LiteralPath $backupPath -Raw | ConvertFrom-Json
$environmentPath = 'HKCU:\Environment'

foreach ($name in @('HTTP_PROXY', 'HTTPS_PROXY', 'ALL_PROXY', 'NO_PROXY')) {
    $value = $backup.$name
    if ([string]::IsNullOrEmpty($value)) {
        Remove-ItemProperty -LiteralPath $environmentPath -Name $name -ErrorAction SilentlyContinue
    } else {
        Set-ItemProperty -LiteralPath $environmentPath -Name $name -Value $value -Type String
    }
}

Get-ItemProperty -LiteralPath $environmentPath |
    Select-Object HTTP_PROXY, HTTPS_PROXY, ALL_PROXY, NO_PROXY |
    Format-List
```

Fully exit and reopen Codex after rollback.
