# WSUS Download Stuck – Advanced Troubleshooting

**Applies to:** Windows Server Update Services (WSUS)  
**Scenario:** Updates stuck downloading after `wsusutil reset`, PrivateMemory set to `0`, queue increased to `25000`

---

## 1. Check Current Download State

```powershell
# Check WSUS service and dependent services status
Get-Service -Name WsusService, W3SVC, MSSQL* | Select-Object Name, Status, StartType

# Check if downloads are actually progressing (run twice, 60s apart)
$wsus = [Microsoft.UpdateServices.Administration.AdminProxy]::GetUpdateServer()
$status = $wsus.GetStatus()
$status | Select-Object DownloadedUpdateCount, NotApprovedUpdateCount, UpdatesNeedingFilesCount
```

---

## 2. Identify Specifically Stuck Updates

```powershell
$wsus = [Microsoft.UpdateServices.Administration.AdminProxy]::GetUpdateServer()
$updateScope = New-Object Microsoft.UpdateServices.Administration.UpdateScope
$updateScope.IncludedInstallationStates = [Microsoft.UpdateServices.Administration.UpdateInstallationStates]::NotInstalled

# Find updates with missing files
$wsus.GetUpdates($updateScope) | Where-Object { $_.HasLicenseAgreement -eq $false -and $_.State -eq "Failed" } |
    Select-Object Title, CreationDate | Sort-Object CreationDate -Descending | Select-Object -First 20
```

---

## 3. Check IIS Application Pool Health

WSUS runs under IIS — a recycling or crashed app pool silently stalls downloads.

```powershell
# Check WsusPool state
Import-Module WebAdministration
Get-WebConfiguration system.applicationHost/applicationPools/add |
    Where-Object { $_.name -eq "WsusPool" } |
    Select-Object name, state, processModel

# Force recycle if needed
Restart-WebAppPool -Name "WsusPool"
```

---

## 4. Review WSUS/IIS Logs for Actual Errors

```powershell
# Check WSUS application event log for errors
Get-EventLog -LogName Application -Source "Windows Server Update Services" -EntryType Error,Warning -Newest 30 |
    Select-Object TimeGenerated, EntryType, Message | Format-List

# Check IIS logs for 500 errors hitting WSUS endpoints
# Adjust drive letter/path if needed
Get-Content "C:\inetpub\logs\LogFiles\W3SVC1\*.log" |
    Select-String " 500 " | Select-Object -Last 20
```

---

## 5. Check Disk Space and Content Directory

```powershell
# Free space on WSUS content drive
Get-PSDrive | Where-Object { $_.Used -gt 0 } |
    Select-Object Name,
        @{N="FreeGB"; E={[math]::Round($_.Free/1GB, 2)}},
        @{N="UsedGB"; E={[math]::Round($_.Used/1GB, 2)}}

# Count files in content store vs DB expectation
$wsus = [Microsoft.UpdateServices.Administration.AdminProxy]::GetUpdateServer()
$config = $wsus.GetConfiguration()
$contentDir = $config.LocalContentCachePath
Write-Host "Content directory: $contentDir"
(Get-ChildItem $contentDir -Recurse -File).Count
```

---

## 6. Check BITS (Background Intelligent Transfer Service)

WSUS uses BITS for downloading. Throttled or broken BITS jobs will silently stall downloads.

```powershell
# Check active BITS jobs
Get-BitsTransfer -AllUsers | Select-Object DisplayName, JobState, BytesTotal, BytesTransferred

# Cancel any stuck BITS jobs tied to WSUS
Get-BitsTransfer -AllUsers |
    Where-Object { $_.DisplayName -like "*WSUS*" -or $_.JobState -eq "Transient Error" } |
    Remove-BitsTransfer
```

---

## 7. Check Sync Status and Phase

```powershell
$wsus = [Microsoft.UpdateServices.Administration.AdminProxy]::GetUpdateServer()
$sub = $wsus.GetSubscription()
Write-Host "Last sync result: $($sub.LastSynchronizationResult)"
Write-Host "Last sync time:   $($sub.LastSynchronizationTime)"
Write-Host "Sync running:     $($sub.GetSynchronizationProgress().Phase)"
```

---

## Most Likely Culprits

| Issue | Indicator | Fix |
|---|---|---|
| **WsusPool crash loop** | IIS event errors, pool in stopped state | Recycle pool; increase or set PrivateMemory to `0` |
| **Stuck BITS job** | BITS job in `Transient Error` state | Cancel stale BITS transfers |
| **Corrupt update metadata** | Specific updates always fail | Decline → re-approve affected updates |
| **SSL/TLS cert issue** | IIS 403/500 on `/selfupdate` | Verify WSUS SSL cert binding in IIS |
| **DB transaction log full** | SQL errors in event log | Shrink SUSDB transaction log |

---

## Steps Already Completed

- [x] Ran `wsusutil reset`
- [x] Set `PrivateMemory` to `0` in WsusPool advanced settings
- [x] Increased queue length from `1000` → `25000`

---

*Last updated: 2026-03-09*
