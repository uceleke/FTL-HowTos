# WSUS Download Stuck – Advanced Troubleshooting

**Applies to:** Windows Server Update Services (WSUS)  
**Scenario:** Updates stuck downloading after `wsusutil reset`, PrivateMemory set to `0`, queue increased to `25000`

---

## 1. Check Current Download State

> **Note:** Always connect explicitly using server name, SSL flag, and port. Using `GetUpdateServer()` with no arguments can return null and cause downstream errors.

```powershell
# Load assembly first
[void][reflection.assembly]::LoadWithPartialName("Microsoft.UpdateServices.Administration")

# Connect explicitly - use 8530 (HTTP) or 8531 (HTTPS)
$wsus = [Microsoft.UpdateServices.Administration.AdminProxy]::GetUpdateServer("wsus01.domain.com", $false, 8530)

# Verify connection
$wsus | Select-Object Name, Version, ServerProtocolVersion

# Check download state (run twice, 60s apart to confirm progress)
$status = $wsus.GetStatus()
$status | Select-Object DownloadedUpdateCount, NotApprovedUpdateCount, UpdatesNeedingFilesCount
```

**If connection fails with HTTP 404 — find the correct port:**
```powershell
# Run ON the WSUS server to confirm actual IIS port binding
Import-Module WebAdministration
Get-WebBinding -Name "WSUS Administration" | Select-Object protocol, bindingInformation
```

---

## 2. Identify Specifically Stuck Updates

```powershell
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

# Confirm content directory path and file count
$config = $wsus.GetConfiguration()
$contentDir = $config.LocalContentCachePath
Write-Host "Content directory: $contentDir"
(Get-ChildItem $contentDir -Recurse -File).Count
```

---

## 6. Check BITS (Background Intelligent Transfer Service)

WSUS uses BITS for downloading. Transient errors with no description typically mean BITS is failing before it gets an HTTP response — common in airgapped environments.

### 6a. Inspect stuck jobs in detail BEFORE clearing

```powershell
# Get error codes and file URLs from stuck jobs
Get-BitsTransfer -AllUsers | Where-Object { $_.JobState -eq "Transient Error" } |
    ForEach-Object {
        Write-Host "=== Job: $($_.DisplayName) ===" -ForegroundColor Cyan
        Write-Host "Error code:    $($_.ErrorCode)"
        Write-Host "Error context: $($_.ErrorContext)"
        $_.FileList | Select-Object LocalName, RemoteName, BytesTotal, BytesTransferred
    }
```

> **Key check:** If `RemoteName` shows `http://download.microsoft.com/...` URLs, BITS is trying to reach the internet. This means update content was not properly imported into the airgapped content store.

### 6b. Check the BITS event log for silent errors

```powershell
Get-WinEvent -LogName "Microsoft-Windows-Bits-Client/Operational" -MaxEvents 50 |
    Where-Object { $_.LevelDisplayName -eq "Error" -or $_.LevelDisplayName -eq "Warning" } |
    Select-Object TimeCreated, Message | Format-List
```

### 6c. Test that WSUS IIS is actually serving content

```powershell
# Test WSUS content endpoint is reachable locally
Invoke-WebRequest -Uri "http://localhost:8530/selfupdate/iuident.cab" -UseBasicParsing

# Verify IIS Content virtual directory points to the right physical path
Import-Module WebAdministration
Get-WebVirtualDirectory -Site "WSUS Administration" | Select-Object path, physicalPath

# Confirm content folder actually has files
$contentPath = (Get-WebVirtualDirectory -Site "WSUS Administration" -Name "Content").physicalPath
Get-ChildItem $contentPath | Select-Object -First 10
```

### 6d. Clear stuck BITS jobs and re-trigger download

```powershell
# Remove all transient error BITS jobs
Get-BitsTransfer -AllUsers | Where-Object { $_.JobState -eq "Transient Error" } | Remove-BitsTransfer

# Force WSUS to re-queue missing files
& "C:\Program Files\Update Services\Tools\WsusUtil.exe" reset
```

### 6e. Monitor BITS job states after reset

```powershell
# Watch BITS job states every 30 seconds
while ($true) {
    $jobs = Get-BitsTransfer -AllUsers
    Write-Host "$(Get-Date -Format 'HH:mm:ss') - Total jobs: $($jobs.Count)"
    $jobs | Group-Object JobState | Select-Object Name, Count | Format-Table -AutoSize
    Start-Sleep 30
}
```

### Silent Transient Error Root Causes

| Scenario | What's Happening | Fix |
|---|---|---|
| **RemoteName = microsoft.com URLs** | Content not imported into airgapped store | Re-import update content; verify WSUS content source |
| **IIS Content vdir broken** | Physical path moved or wrong — IIS returns nothing | Re-map vdir in IIS to correct content folder |
| **Loopback connection blocked** | BITS can't connect to `localhost:8530` | Check `DisableLoopbackCheck` registry key or loopback GPO |
| **WSUS App Pool stopped** | IIS serves no response; BITS retries indefinitely | Recycle WsusPool; check for crash loop in event log |
| **BITS quota GPO** | Policy limiting concurrent BITS jobs | Check `Computer Config > Admin Templates > BITS` |

---

## 7. Check Sync Status and Phase

```powershell
$sub = $wsus.GetSubscription()
Write-Host "Last sync result: $($sub.LastSynchronizationResult)"
Write-Host "Last sync time:   $($sub.LastSynchronizationTime)"
Write-Host "Sync running:     $($sub.GetSynchronizationProgress().Phase)"
```

---

## 8. SUSDB Database Cleanup

Run periodically to remove obsolete update data, reduce DB bloat, and improve WSUS performance. Large unoptimised databases can contribute to slow syncs and stuck downloads.

> **Prerequisites:** Run on the WSUS server. Requires SQL Server Management Studio or `sqlcmd`. For Windows Internal Database (WID), use named pipe `\\.\pipe\MICROSOFT##WID\tsql\query`.

```sql
-- ============================================================
-- WSUS SUSDB Cleanup Script
-- Run against: SUSDB database
-- Recommended: Monthly, or before/after large sync operations
-- ============================================================

USE SUSDB;
GO

-- -------------------------------------------------------
-- Step 1: Decline and delete superseded updates
-- Removes updates that have been replaced by newer versions
-- -------------------------------------------------------
EXEC spDeclineExpiredUpdates;
EXEC spDeclineSupersededUpdates;
GO

-- -------------------------------------------------------
-- Step 2: Delete obsolete updates from the database
-- -------------------------------------------------------
EXEC spDeleteUpdate;
GO

-- -------------------------------------------------------
-- Step 3: Clean up obsolete computers and targeting data
-- -------------------------------------------------------
EXEC spCleanupObsoleteComputers;
GO

-- -------------------------------------------------------
-- Step 4: Clean up unused update files metadata
-- (Does not delete files from disk — use wsusutil for that)
-- -------------------------------------------------------
EXEC spCleanupUnneededContentFiles;
GO

-- -------------------------------------------------------
-- Step 5: Compress update revisions
-- Reduces row count in revision tables significantly
-- -------------------------------------------------------
EXEC spCompressUpdate;
GO

-- -------------------------------------------------------
-- Step 6: Re-index SUSDB tables
-- Dramatically improves query performance after cleanup
-- -------------------------------------------------------
SET NOCOUNT ON;

DECLARE @TableName NVARCHAR(255)
DECLARE @IndexName NVARCHAR(255)
DECLARE @Fragmentation FLOAT
DECLARE @SQL NVARCHAR(MAX)

DECLARE IndexCursor CURSOR FOR
    SELECT
        t.name AS TableName,
        i.name AS IndexName,
        s.avg_fragmentation_in_percent
    FROM sys.dm_db_index_physical_stats(DB_ID(), NULL, NULL, NULL, 'LIMITED') s
    JOIN sys.tables t ON s.object_id = t.object_id
    JOIN sys.indexes i ON s.object_id = i.object_id AND s.index_id = i.index_id
    WHERE s.avg_fragmentation_in_percent > 10
      AND i.name IS NOT NULL
    ORDER BY s.avg_fragmentation_in_percent DESC

OPEN IndexCursor
FETCH NEXT FROM IndexCursor INTO @TableName, @IndexName, @Fragmentation

WHILE @@FETCH_STATUS = 0
BEGIN
    IF @Fragmentation > 30
        SET @SQL = 'ALTER INDEX [' + @IndexName + '] ON [' + @TableName + '] REBUILD'
    ELSE
        SET @SQL = 'ALTER INDEX [' + @IndexName + '] ON [' + @TableName + '] REORGANIZE'

    PRINT 'Processing: ' + @TableName + '.' + @IndexName + 
          ' (' + CAST(CAST(@Fragmentation AS INT) AS NVARCHAR) + '% fragmented)'
    EXEC sp_executesql @SQL

    FETCH NEXT FROM IndexCursor INTO @TableName, @IndexName, @Fragmentation
END

CLOSE IndexCursor
DEALLOCATE IndexCursor
GO

-- -------------------------------------------------------
-- Step 7: Update statistics on all tables
-- -------------------------------------------------------
EXEC sp_updatestats;
GO

-- -------------------------------------------------------
-- Step 8: Shrink transaction log (use cautiously)
-- Only run if log file has grown unexpectedly large
-- Verify recovery model first before running
-- -------------------------------------------------------
-- SELECT name, log_reuse_wait_desc FROM sys.databases WHERE name = 'SUSDB'
-- BACKUP LOG SUSDB TO DISK = 'NUL';  -- Required if in FULL recovery model
-- DBCC SHRINKFILE (SUSDB_log, 100);  -- Shrink log to ~100MB
GO

PRINT 'SUSDB cleanup complete.';
GO
```

**Run via sqlcmd:**
```powershell
# Windows Internal Database (WID)
sqlcmd -S "\\.\pipe\MICROSOFT##WID\tsql\query" -i "C:\Scripts\SUSDB_Cleanup.sql" -o "C:\Scripts\SUSDB_Cleanup_Log.txt"

# SQL Server Express (adjust instance name as needed)
sqlcmd -S "localhost\SQLEXPRESS" -i "C:\Scripts\SUSDB_Cleanup.sql" -o "C:\Scripts\SUSDB_Cleanup_Log.txt"
```

**After DB cleanup, sync file state with DB:**
```powershell
& "C:\Program Files\Update Services\Tools\WsusUtil.exe" reset
```

---

## Most Likely Culprits

| Issue | Indicator | Fix |
|---|---|---|
| **WsusPool crash loop** | IIS event errors, pool in stopped state | Recycle pool; set PrivateMemory to `0` |
| **Stuck BITS jobs** | `Transient Error` state, no description | Inspect `RemoteName` URLs; clear jobs; reset WSUS |
| **BITS pointing to internet** | `RemoteName` = `download.microsoft.com` | Re-import content into airgapped store |
| **IIS Content vdir broken** | `Invoke-WebRequest` to localhost fails | Re-map Content vdir to correct physical path |
| **Corrupt update metadata** | Specific updates always fail | Decline → re-approve affected updates |
| **SSL/TLS cert issue** | IIS 403/500 on `/selfupdate` | Verify WSUS SSL cert binding in IIS |
| **DB bloat / fragmentation** | Slow syncs, high SQL CPU | Run SUSDB cleanup script (Section 8) |

---

## Steps Already Completed

- [x] Ran `wsusutil reset`
- [x] Set `PrivateMemory` to `0` in WsusPool advanced settings
- [x] Increased queue length from `1000` → `25000`
- [x] Cleared 10 transient BITS errors

---

*Last updated: 2026-03-09*
