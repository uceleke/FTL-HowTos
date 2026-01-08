# Active Directory Backup with NTDSUTIL - Detailed Walkthrough

**Purpose:** Create recoverable snapshots of Active Directory before making destructive Exchange Server schema changes  
**Target Audience:** IT Infrastructure Engineers  
**Prerequisites:** Domain Admin rights, access to Domain Controller  
**Estimated Time:** 30-45 minutes

---

## Table of Contents

1. [Understanding NTDSUTIL vs Other Backup Methods](#understanding-ntdsutil-vs-other-backup-methods)
2. [Pre-Backup Preparation](#pre-backup-preparation)
3. [Method 1: NTDSUTIL Snapshot (Fastest)](#method-1-ntdsutil-snapshot-fastest)
4. [Method 2: Windows Server Backup (Most Complete)](#method-2-windows-server-backup-most-complete)
5. [Method 3: Export AD Objects (Surgical Rollback)](#method-3-export-ad-objects-surgical-rollback)
6. [Verification Procedures](#verification-procedures)
7. [Restoration Procedures](#restoration-procedures)
8. [Complete Pre-Exchange-Removal Checklist](#complete-pre-exchange-removal-checklist)

---

## Understanding NTDSUTIL vs Other Backup Methods

### What is NTDSUTIL?

NTDSUTIL (NT Directory Services Utility) is a command-line tool that performs maintenance and management of Active Directory. For backups, it creates **shadow copies** (VSS snapshots) of the AD database.

### Backup Method Comparison

| Method | Speed | Recovery Type | Best For | Limitations |
|--------|-------|---------------|----------|-------------|
| **NTDSUTIL Snapshot** | Fast (5-10 min) | Object-level or full DC | Quick rollback, specific object recovery | Requires disk space, 48hr limit |
| **Windows Server Backup** | Slow (30-60 min) | Full system state, bare metal | Complete DC recovery | Requires separate backup target |
| **AD Object Export** | Fast (2-5 min) | Specific objects only | Surgical fixes, documentation | Limited scope, manual restore |
| **AD Recycle Bin** | Instant | Deleted objects only | Accidental deletions | Must be enabled beforehand |

### Why NTDSUTIL for Exchange Removal?

1. **Fast** - Creates snapshot in minutes
2. **Point-in-time recovery** - Can mount and extract specific objects
3. **No downtime** - Uses VSS (Volume Shadow Copy)
4. **Surgical restoration** - Can restore just Exchange config without affecting other AD changes
5. **Validation** - Can mount snapshot to verify backup before making changes

---

## Pre-Backup Preparation

### Step 1: Check Disk Space

NTDSUTIL snapshots require free space on the volume hosting NTDS.dit.

```powershell
# Check available space on DC
Get-WmiObject Win32_LogicalDisk | 
    Where-Object {$_.DriveType -eq 3} | 
    Select-Object DeviceID, 
                  @{Name="Size(GB)";Expression={[math]::Round($_.Size/1GB,2)}},
                  @{Name="FreeSpace(GB)";Expression={[math]::Round($_.FreeSpace/1GB,2)}},
                  @{Name="PercentFree";Expression={[math]::Round(($_.FreeSpace/$_.Size)*100,2)}}
```

**Requirements:**
- Minimum: 10% free space on NTDS.dit volume
- Recommended: 20%+ free space
- NTDS.dit location: Usually `C:\Windows\NTDS\ntds.dit`

```powershell
# Verify NTDS.dit location
Get-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Services\NTDS\Parameters" -Name "DSA Database file"
```

### Step 2: Check AD Replication Health

Ensure AD is healthy before creating snapshot.

```powershell
# Test AD replication
repadmin /replsummary

# Check for replication errors
repadmin /showrepl

# Verify all DCs are replicating
Get-ADReplicationPartnerMetadata -Target * -Scope Domain | 
    Select-Object Server, LastReplicationSuccess, LastReplicationResult
```

**Expected Results:**
- All DCs show "successful replication"
- No errors in last 24 hours
- LastReplicationResult = 0 (success)

**If Errors Found:**
```powershell
# Force replication from all partners
repadmin /syncall /AdeP

# Wait 5 minutes, then recheck
Start-Sleep -Seconds 300
repadmin /replsummary
```

### Step 3: Document Current State

```powershell
# Create documentation directory
$BackupDate = Get-Date -Format "yyyyMMdd_HHmmss"
$BackupPath = "C:\AD_Backups\$BackupDate"
New-Item -Path $BackupPath -ItemType Directory -Force

# Export Exchange server configuration
Get-ExchangeServer | Export-Csv "$BackupPath\ExchangeServers.csv" -NoTypeInformation
Get-ExchangeServer | Format-List * | Out-File "$BackupPath\ExchangeServers_Full.txt"

# Export AD replication status
repadmin /showrepl > "$BackupPath\AD_Replication_PreBackup.txt"

# Export DC information
Get-ADDomainController -Filter * | Export-Csv "$BackupPath\DomainControllers.csv" -NoTypeInformation

# Document FSMO roles
netdom query fsmo > "$BackupPath\FSMO_Roles.txt"

Write-Host "Documentation saved to: $BackupPath" -ForegroundColor Green
```

### Step 4: Notify Stakeholders

Before taking snapshots:
- [ ] Maintenance window scheduled
- [ ] Change control approved
- [ ] Team members notified
- [ ] Service desk aware (in case of issues)
- [ ] Rollback plan communicated

---

## Method 1: NTDSUTIL Snapshot (Fastest)

This creates a VSS snapshot of AD that can be mounted for recovery.

### Understanding the Process

```
1. Create VSS snapshot of volume containing NTDS.dit
2. Mount snapshot to inspect/verify
3. Unmount when done
4. Keep snapshot for recovery (auto-expires after 48 hours)
```

### Step-by-Step Procedure

#### Phase 1: Create the Snapshot

**Option A: Interactive Mode (Recommended for First Time)**

```cmd
# Launch NTDSUTIL as Administrator
ntdsutil

# You'll see: ntdsutil:
```

Now execute these commands one at a time:

```
ntdsutil: activate instance ntds
```
**Output:** `Active instance set to "ntds".`

```
ntdsutil: snapshot
```
**Output:** You're now in snapshot mode: `snapshot:`

```
snapshot: create
```
**Output:** Something like:
```
Snapshot set {GUID1} generated successfully.
Snapshot {GUID2} created successfully.
```

**IMPORTANT:** Copy both GUIDs immediately to your documentation:
- Snapshot Set GUID: `{GUID1}` 
- Snapshot GUID: `{GUID2}`

```
snapshot: list all
```
**Output:** Shows all snapshots with their GUIDs and creation times

```
snapshot: quit
ntdsutil: quit
```

**Option B: Scripted Mode (For Experienced Users)**

```cmd
@echo off
echo Creating Active Directory Snapshot...
echo.

ntdsutil "activate instance ntds" "snapshot" "create" quit quit

echo.
echo Snapshot created successfully.
echo Listing all snapshots:
echo.

ntdsutil "activate instance ntds" "snapshot" "list all" quit quit

pause
```

Save as `Create_AD_Snapshot.cmd` and run as Administrator.

#### Phase 2: Document the Snapshot

```powershell
# Create snapshot log
$BackupPath = "C:\AD_Backups\$(Get-Date -Format 'yyyyMMdd_HHmmss')"
New-Item -Path $BackupPath -ItemType Directory -Force

# Document snapshot creation
$SnapshotInfo = @"
Active Directory Snapshot Created
==================================
Date/Time: $(Get-Date)
Server: $env:COMPUTERNAME
Domain: $env:USERDNSDOMAIN
Created By: $env:USERNAME

Snapshot Details:
-----------------
Snapshot Set GUID: {PASTE_GUID1_HERE}
Snapshot GUID: {PASTE_GUID2_HERE}

Purpose: Pre-Exchange Server Removal Backup
Server to Remove: EXCHANGE-TEST-01
Ticket Number: [Your Change Number]

Snapshot Location: VSS Shadow Copy on System Volume
Expiration: Automatic (48 hours from creation)

Notes:
------
- This snapshot is READ-ONLY
- Will auto-delete after 48 hours
- Can be mounted to extract AD objects if needed
- For full restoration, use Windows Server Backup

"@

$SnapshotInfo | Out-File "$BackupPath\Snapshot_Info.txt"

Write-Host "Snapshot documentation saved to: $BackupPath\Snapshot_Info.txt" -ForegroundColor Green
```

#### Phase 3: Verify the Snapshot

**Critical:** Always verify your snapshot before proceeding with changes.

```cmd
# Mount the snapshot to verify it
ntdsutil
activate instance ntds
snapshot
list all
```

You'll see output like:
```
C:\$Snap_datetime\Volume{GUID}\
  1 : 2026/01/08:14:30 {snapshot-guid}
```

Now mount it:
```
mount {snapshot-guid}
```

**Output:** `Snapshot {guid} mounted as C:\$SNAP_datetime_VOLUMEC$\`

Note the mount path, then verify:

```
quit
quit

# Exit ntdsutil and verify the mount
dir C:\$SNAP_*
```

**You should see:**
```
Directory of C:\$SNAP_2026_01_08_T14_30_00_000Z_VOLUMEC$\

Windows/
  NTDS/
    ntds.dit  <- Your AD database snapshot
    *.log     <- Transaction logs
```

**Verification Checklist:**
- [ ] ntds.dit file exists in snapshot
- [ ] File size matches current NTDS.dit (within 10%)
- [ ] Timestamp matches snapshot creation time
- [ ] Can browse directory structure

```powershell
# Compare snapshot to live database
$SnapshotPath = (Get-ChildItem "C:\`$SNAP*" -Directory | Select-Object -First 1).FullName
$SnapshotDB = Get-ChildItem "$SnapshotPath\Windows\NTDS\ntds.dit"
$LiveDB = Get-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Services\NTDS\Parameters" -Name "DSA Database file" | Select-Object -ExpandProperty "DSA Database file"
$LiveDBInfo = Get-Item $LiveDB

Write-Host "Snapshot Database Size: $([math]::Round($SnapshotDB.Length/1GB,2)) GB" -ForegroundColor Cyan
Write-Host "Live Database Size: $([math]::Round($LiveDBInfo.Length/1GB,2)) GB" -ForegroundColor Cyan

if ([math]::Abs($SnapshotDB.Length - $LiveDBInfo.Length) -lt ($LiveDBInfo.Length * 0.1)) {
    Write-Host "Snapshot verification PASSED - Sizes match within 10%" -ForegroundColor Green
} else {
    Write-Host "WARNING: Snapshot size differs significantly from live database" -ForegroundColor Yellow
}
```

#### Phase 4: Unmount the Snapshot

**Important:** Unmount after verification to free resources.

```cmd
ntdsutil
activate instance ntds
snapshot
list all

# Unmount specific snapshot
unmount {snapshot-guid}

# Verify it's unmounted
list all

quit
quit
```

**What NOT to Do:**
- ❌ Don't delete the snapshot yet (you still need it!)
- ❌ Don't mount multiple snapshots simultaneously (performance impact)
- ❌ Don't modify any files in the mounted snapshot (it's read-only anyway)

---

## Method 2: Windows Server Backup (Most Complete)

This creates a full system state backup that can restore the entire DC.

### When to Use This Method

Use Windows Server Backup when:
- You want complete DC recovery capability
- You're making high-risk schema changes
- You have available backup storage
- You want to keep backup >48 hours

### Prerequisites

```powershell
# Install Windows Server Backup feature
Install-WindowsFeature -Name Windows-Server-Backup -IncludeManagementTools

# Verify installation
Get-WindowsFeature -Name Windows-Server-Backup
```

### Step-by-Step Procedure

#### Step 1: Prepare Backup Target

**Option A: Local Disk (Fastest)**

```powershell
# Check available disks
Get-Disk | Where-Object {$_.PartitionStyle -ne 'RAW'} | 
    Format-Table Number, FriendlyName, Size, PartitionStyle

# Verify target has enough space
# Rule: Need at least 1.5x size of C:\ drive
$CDriveSize = (Get-Volume -DriveLetter C).Size
$RequiredSpace = $CDriveSize * 1.5

Write-Host "C:\ Drive Size: $([math]::Round($CDriveSize/1GB,2)) GB" -ForegroundColor Cyan
Write-Host "Required Backup Space: $([math]::Round($RequiredSpace/1GB,2)) GB" -ForegroundColor Yellow
```

**Option B: Network Share (Recommended for Production)**

```powershell
# Test network share access
$BackupShare = "\\FileServer\DC_Backups"
$Credential = Get-Credential -Message "Enter credentials for $BackupShare"

Test-Path $BackupShare -Credential $Credential
```

#### Step 2: Create System State Backup

**Method 2A: Using PowerShell (Recommended)**

```powershell
# Set variables
$BackupLocation = "E:\"  # Or network share: "\\FileServer\DC_Backups"
$BackupDate = Get-Date -Format "yyyyMMdd_HHmmss"

# Create backup policy
$Policy = New-WBPolicy

# Add System State to backup
Add-WBSystemState -Policy $Policy

# Set backup target (local disk)
$BackupTarget = New-WBBackupTarget -VolumePath $BackupLocation
Add-WBBackupTarget -Policy $Policy -Target $BackupTarget

# Start backup
Write-Host "Starting System State Backup..." -ForegroundColor Yellow
Write-Host "This may take 30-60 minutes depending on DC size..." -ForegroundColor Cyan

Start-WBBackup -Policy $Policy -Force

# Monitor progress
Get-WBJob -Previous 1
```

**Method 2B: Using Command Line**

```cmd
# Backup to local disk
wbadmin start systemstatebackup -backupTarget:E: -quiet

# Backup to network share
wbadmin start systemstatebackup -backupTarget:\\FileServer\DC_Backups -quiet
```

**Method 2C: Using GUI**

```
1. Open Windows Server Backup (from Server Manager → Tools)
2. Click "Backup Once..." in right panel
3. Select "Different options"
4. Choose "Custom" backup
5. Click "Add Items"
6. Check "System State"
7. Click "Next"
8. Select "Local drives" or "Remote shared folder"
9. Choose destination
10. Click "Backup"
11. Monitor progress
```

#### Step 3: Monitor Backup Progress

```powershell
# Real-time monitoring
while ($true) {
    Clear-Host
    Write-Host "System State Backup Progress" -ForegroundColor Cyan
    Write-Host "=============================" -ForegroundColor Cyan
    Write-Host ""
    
    $Job = Get-WBJob -Previous 1
    
    Write-Host "Status: $($Job.JobState)" -ForegroundColor Yellow
    Write-Host "Progress: $($Job.PercentComplete)%" -ForegroundColor Green
    Write-Host "Start Time: $($Job.StartTime)"
    Write-Host "Current Operation: $($Job.CurrentOperation)"
    
    if ($Job.JobState -eq "Completed" -or $Job.JobState -eq "Failed") {
        break
    }
    
    Start-Sleep -Seconds 30
}

Write-Host ""
Write-Host "Backup $($Job.JobState)!" -ForegroundColor $(if($Job.JobState -eq "Completed"){"Green"}else{"Red"})
```

#### Step 4: Verify Backup

```powershell
# List all backups
Get-WBBackupSet

# Get details of most recent backup
$LatestBackup = Get-WBBackupSet | Sort-Object BackupTime -Descending | Select-Object -First 1

Write-Host "Latest Backup Details:" -ForegroundColor Cyan
Write-Host "======================" -ForegroundColor Cyan
Write-Host "Backup Time: $($LatestBackup.BackupTime)"
Write-Host "Version: $($LatestBackup.VersionId)"
Write-Host "Backup Target: $($LatestBackup.BackupTarget)"
Write-Host "Volume Count: $($LatestBackup.Volume.Count)"

# Check if System State is included
if ($LatestBackup.Application.Name -contains "System State") {
    Write-Host "System State: INCLUDED" -ForegroundColor Green
} else {
    Write-Host "System State: NOT FOUND - BACKUP FAILED!" -ForegroundColor Red
}
```

**Verification Checklist:**
- [ ] Backup completed without errors
- [ ] System State is included in backup
- [ ] Backup size is reasonable (typically 5-20 GB)
- [ ] Backup files are accessible
- [ ] Backup metadata is readable

---

## Method 3: Export AD Objects (Surgical Rollback)

This method exports specific Exchange-related AD objects for granular recovery.

### When to Use This Method

- You want to restore specific objects (not full DC)
- You need to document current state
- You want faster, targeted recovery
- Combine with other backup methods

### Step-by-Step Procedure

#### Step 1: Export Exchange Configuration Container

```powershell
# Set variables
$BackupPath = "C:\AD_Backups\$(Get-Date -Format 'yyyyMMdd_HHmmss')"
New-Item -Path $BackupPath -ItemType Directory -Force

# Get domain info
$Domain = Get-ADDomain
$ConfigNC = "CN=Configuration,$($Domain.DistinguishedName)"

# Export entire Exchange organization
$ExchangeOrgPath = "CN=Microsoft Exchange,CN=Services,$ConfigNC"

Write-Host "Exporting Exchange Organization configuration..." -ForegroundColor Yellow

# Export to LDIF (LDAP Data Interchange Format)
ldifde -f "$BackupPath\Exchange_Org_Export.ldf" -d "$ExchangeOrgPath" -p SubTree

Write-Host "Exported to: $BackupPath\Exchange_Org_Export.ldf" -ForegroundColor Green
```

#### Step 2: Export Specific Exchange Server Objects

```powershell
# Export Exchange servers
$ServerToRemove = "EXCHANGE-TEST-01"
$ExchangeServersPath = "CN=Servers,CN=Exchange Administrative Group (FYDIBOHF23SPDLT),CN=Administrative Groups,$ExchangeOrgPath"

Write-Host "Exporting Exchange server: $ServerToRemove" -ForegroundColor Yellow

ldifde -f "$BackupPath\Exchange_Server_$ServerToRemove.ldf" -d "CN=$ServerToRemove,$ExchangeServersPath" -p SubTree

# Export Exchange databases
$DatabasesPath = "CN=Databases,CN=Exchange Administrative Group (FYDIBOHF23SPDLT),CN=Administrative Groups,$ExchangeOrgPath"

ldifde -f "$BackupPath\Exchange_Databases.ldf" -d "$DatabasesPath" -p SubTree

Write-Host "Exports completed successfully" -ForegroundColor Green
```

#### Step 3: Export AD Computer Account

```powershell
# Export computer account
$Computer = Get-ADComputer -Identity $ServerToRemove -Properties *

# Export to XML (PowerShell native format)
$Computer | Export-Clixml "$BackupPath\Computer_$ServerToRemove.xml"

# Export to CSV (human readable)
$Computer | Select-Object Name, DistinguishedName, DNSHostName, ObjectGUID, whenCreated, whenChanged | 
    Export-Csv "$BackupPath\Computer_$ServerToRemove.csv" -NoTypeInformation

# Export to LDIF (AD native format)
ldifde -f "$BackupPath\Computer_$ServerToRemove.ldf" -d "$($Computer.DistinguishedName)"

Write-Host "Computer account exported in multiple formats" -ForegroundColor Green
```

#### Step 4: Export Security Group Memberships

```powershell
# Get all Exchange security groups
$ExchangeGroups = Get-ADGroup -Filter {Name -like "*Exchange*"} -SearchBase "CN=Microsoft Exchange Security Groups,$($Domain.DistinguishedName)"

# Export each group with members
foreach ($Group in $ExchangeGroups) {
    $Members = Get-ADGroupMember -Identity $Group -Recursive | 
        Select-Object Name, SamAccountName, ObjectClass, DistinguishedName
    
    $Members | Export-Csv "$BackupPath\Group_$($Group.Name)_Members.csv" -NoTypeInformation
    
    Write-Host "Exported: $($Group.Name) ($($Members.Count) members)" -ForegroundColor Cyan
}

# Create master list of server's group memberships
$ServerGroups = Get-ADPrincipalGroupMembership -Identity "$ServerToRemove$" | 
    Select-Object Name, GroupCategory, GroupScope, DistinguishedName

$ServerGroups | Export-Csv "$BackupPath\Server_$ServerToRemove`_GroupMemberships.csv" -NoTypeInformation

Write-Host "Security group memberships exported" -ForegroundColor Green
```

#### Step 5: Create Restoration Script

```powershell
# Generate automated restoration script
$RestoreScript = @"
# Exchange Server AD Object Restoration Script
# Generated: $(Get-Date)
# Server: $ServerToRemove
# Purpose: Restore deleted objects from LDIF exports

`$BackupPath = "$BackupPath"

Write-Host "Exchange Server AD Restoration" -ForegroundColor Cyan
Write-Host "===============================" -ForegroundColor Cyan
Write-Host ""
Write-Host "WARNING: This will restore AD objects from backup!" -ForegroundColor Yellow
Write-Host "Backup Date: $(Get-Date)" -ForegroundColor Yellow
Write-Host ""

`$Confirm = Read-Host "Type 'YES' to proceed with restoration"

if (`$Confirm -ne "YES") {
    Write-Host "Restoration cancelled" -ForegroundColor Red
    exit
}

# Restore Exchange server object
Write-Host "Restoring Exchange server configuration..." -ForegroundColor Yellow
ldifde -i -f "`$BackupPath\Exchange_Server_$ServerToRemove.ldf"

# Restore databases
Write-Host "Restoring Exchange database configuration..." -ForegroundColor Yellow
ldifde -i -f "`$BackupPath\Exchange_Databases.ldf"

# Restore computer account
Write-Host "Restoring computer account..." -ForegroundColor Yellow
ldifde -i -f "`$BackupPath\Computer_$ServerToRemove.ldf"

# Restore group memberships
Write-Host "Restoring security group memberships..." -ForegroundColor Yellow
`$GroupMemberships = Import-Csv "`$BackupPath\Server_$ServerToRemove`_GroupMemberships.csv"

foreach (`$Group in `$GroupMemberships) {
    try {
        Add-ADGroupMember -Identity `$Group.Name -Members "$ServerToRemove$" -ErrorAction Stop
        Write-Host "  Added to: `$(`$Group.Name)" -ForegroundColor Green
    } catch {
        Write-Host "  Failed to add to: `$(`$Group.Name) - `$_" -ForegroundColor Red
    }
}

Write-Host ""
Write-Host "Restoration completed!" -ForegroundColor Green
Write-Host "Verify AD replication: repadmin /syncall /AdeP" -ForegroundColor Cyan
"@

$RestoreScript | Out-File "$BackupPath\Restore_$ServerToRemove.ps1" -Encoding UTF8

Write-Host ""
Write-Host "Restoration script created: $BackupPath\Restore_$ServerToRemove.ps1" -ForegroundColor Green
```

#### Step 6: Create Backup Summary

```powershell
# Generate comprehensive backup summary
$Summary = @"
Active Directory Backup Summary
================================
Backup Date: $(Get-Date)
Backup Location: $BackupPath
Server: $env:COMPUTERNAME
Domain: $($Domain.DNSRoot)
Created By: $env:USERNAME

Files Exported:
---------------
$(Get-ChildItem $BackupPath | Format-Table Name, Length, LastWriteTime | Out-String)

Export Statistics:
------------------
Total Files: $(Get-ChildItem $BackupPath | Measure-Object | Select-Object -ExpandProperty Count)
Total Size: $([math]::Round((Get-ChildItem $BackupPath | Measure-Object -Property Length -Sum).Sum / 1MB, 2)) MB

Included Objects:
-----------------
- Exchange Organization Configuration
- Exchange Server: $ServerToRemove
- Exchange Databases
- Computer Account: $ServerToRemove
- Security Group Memberships
- Restoration Script

Next Steps:
-----------
1. Review exported files to ensure completeness
2. Copy backup to secure location
3. Proceed with Exchange server removal
4. Use Restore_$ServerToRemove.ps1 if rollback needed

Notes:
------
- LDIF exports can be imported with: ldifde -i -f filename.ldf
- Computer account can be restored from XML: Import-Clixml | New-ADComputer
- CSV files are for reference/documentation only

Important:
----------
Keep this backup for at least 30 days after successful removal.
"@

$Summary | Out-File "$BackupPath\Backup_Summary.txt"

Write-Host ""
Write-Host "============================================" -ForegroundColor Green
Write-Host "Backup Summary" -ForegroundColor Green
Write-Host "============================================" -ForegroundColor Green
Write-Host $Summary
Write-Host ""
Write-Host "Full summary saved to: $BackupPath\Backup_Summary.txt" -ForegroundColor Cyan
```

---

## Verification Procedures

After creating any backup, always verify before proceeding.

### Verification Checklist

```powershell
# Comprehensive backup verification script
$BackupPath = "C:\AD_Backups\20260108_143000"  # Update with your path

Write-Host "Active Directory Backup Verification" -ForegroundColor Cyan
Write-Host "=====================================" -ForegroundColor Cyan
Write-Host ""

$Results = @()

# Test 1: Snapshot exists (if using NTDSUTIL)
Write-Host "[1/8] Checking NTDSUTIL snapshots..." -ForegroundColor Yellow
$SnapshotCheck = & ntdsutil "activate instance ntds" "snapshot" "list all" quit quit 2>&1
if ($SnapshotCheck -match "snapshot") {
    Write-Host "  ✓ Snapshot found" -ForegroundColor Green
    $Results += "Snapshot: PASS"
} else {
    Write-Host "  ✗ No snapshot found" -ForegroundColor Red
    $Results += "Snapshot: FAIL"
}

# Test 2: Windows Server Backup exists
Write-Host "[2/8] Checking Windows Server Backup..." -ForegroundColor Yellow
try {
    $WSB = Get-WBBackupSet -ErrorAction Stop | Sort-Object BackupTime -Descending | Select-Object -First 1
    if ($WSB -and ($WSB.BackupTime -gt (Get-Date).AddHours(-2))) {
        Write-Host "  ✓ Recent backup found: $($WSB.BackupTime)" -ForegroundColor Green
        $Results += "WSB: PASS"
    } else {
        Write-Host "  ⚠ Backup is older than 2 hours" -ForegroundColor Yellow
        $Results += "WSB: WARNING"
    }
} catch {
    Write-Host "  ⚠ No Windows Server Backup found" -ForegroundColor Yellow
    $Results += "WSB: N/A"
}

# Test 3: Export files exist
Write-Host "[3/8] Checking export files..." -ForegroundColor Yellow
$RequiredFiles = @(
    "Exchange_Org_Export.ldf",
    "Exchange_Databases.ldf",
    "Backup_Summary.txt"
)

$FilesFound = 0
foreach ($File in $RequiredFiles) {
    if (Test-Path "$BackupPath\$File") {
        $FilesFound++
        Write-Host "  ✓ Found: $File" -ForegroundColor Green
    } else {
        Write-Host "  ✗ Missing: $File" -ForegroundColor Red
    }
}

if ($FilesFound -eq $RequiredFiles.Count) {
    $Results += "Exports: PASS"
} else {
    $Results += "Exports: FAIL"
}

# Test 4: LDIF file integrity
Write-Host "[4/8] Validating LDIF files..." -ForegroundColor Yellow
$LDIFFiles = Get-ChildItem "$BackupPath\*.ldf"
$ValidLDIF = 0

foreach ($File in $LDIFFiles) {
    $Content = Get-Content $File.FullName -TotalCount 5
    if ($Content -match "dn:" -and $Content -match "changetype:") {
        $ValidLDIF++
    }
}

if ($ValidLDIF -eq $LDIFFiles.Count) {
    Write-Host "  ✓ All LDIF files are valid" -ForegroundColor Green
    $Results += "LDIF Integrity: PASS"
} else {
    Write-Host "  ✗ Some LDIF files may be corrupt" -ForegroundColor Red
    $Results += "LDIF Integrity: FAIL"
}

# Test 5: Backup size is reasonable
Write-Host "[5/8] Checking backup size..." -ForegroundColor Yellow
$BackupSize = (Get-ChildItem $BackupPath -Recurse | Measure-Object -Property Length -Sum).Sum / 1MB

if ($BackupSize -gt 1 -and $BackupSize -lt 50000) {
    Write-Host "  ✓ Backup size: $([math]::Round($BackupSize,2)) MB" -ForegroundColor Green
    $Results += "Size: PASS"
} else {
    Write-Host "  ⚠ Backup size seems unusual: $([math]::Round($BackupSize,2)) MB" -ForegroundColor Yellow
    $Results += "Size: WARNING"
}

# Test 6: Files are not empty
Write-Host "[6/8] Checking for empty files..." -ForegroundColor Yellow
$EmptyFiles = Get-ChildItem $BackupPath -File | Where-Object {$_.Length -eq 0}

if ($EmptyFiles.Count -eq 0) {
    Write-Host "  ✓ No empty files found" -ForegroundColor Green
    $Results += "Empty Files: PASS"
} else {
    Write-Host "  ✗ Found $($EmptyFiles.Count) empty files" -ForegroundColor Red
    $Results += "Empty Files: FAIL"
}

# Test 7: Restoration script exists and is valid
Write-Host "[7/8] Checking restoration script..." -ForegroundColor Yellow
$RestoreScript = Get-ChildItem "$BackupPath\Restore_*.ps1"

if ($RestoreScript -and (Test-Path $RestoreScript.FullName)) {
    $ScriptContent = Get-Content $RestoreScript.FullName -Raw
    if ($ScriptContent -match "ldifde" -and $ScriptContent -match "Add-ADGroupMember") {
        Write-Host "  ✓ Restoration script is valid" -ForegroundColor Green
        $Results += "Restore Script: PASS"
    } else {
        Write-Host "  ⚠ Restoration script may be incomplete" -ForegroundColor Yellow
        $Results += "Restore Script: WARNING"
    }
} else {
    Write-Host "  ✗ Restoration script not found" -ForegroundColor Red
    $Results += "Restore Script: FAIL"
}

# Test 8: Documentation complete
Write-Host "[8/8] Checking documentation..." -ForegroundColor Yellow
$SummaryFile = "$BackupPath\Backup_Summary.txt"

if (Test-Path $SummaryFile) {
    $Summary = Get-Content $SummaryFile -Raw
    if ($Summary -match "Backup Date:" -and $Summary -match "Total Files:") {
        Write-Host "  ✓ Documentation is complete" -ForegroundColor Green
        $Results += "Documentation: PASS"
    } else {
        Write-Host "  ⚠ Documentation may be incomplete" -ForegroundColor Yellow
        $Results += "Documentation: WARNING"
    }
} else {
    Write-Host "  ✗ Documentation file not found" -ForegroundColor Red
    $Results += "Documentation: FAIL"
}

# Final Summary
Write-Host ""
Write-Host "=====================================" -ForegroundColor Cyan
Write-Host "Verification Summary" -ForegroundColor Cyan
Write-Host "=====================================" -ForegroundColor Cyan

$PassCount = ($Results | Where-Object {$_ -match "PASS"}).Count
$FailCount = ($Results | Where-Object {$_ -match "FAIL"}).Count
$WarnCount = ($Results | Where-Object {$_ -match "WARNING"}).Count

Write-Host "Passed: $PassCount" -ForegroundColor Green
Write-Host "Failed: $FailCount" -ForegroundColor Red
Write-Host "Warnings: $WarnCount" -ForegroundColor Yellow

Write-Host ""

if ($FailCount -eq 0) {
    Write-Host "✓ VERIFICATION PASSED - Backup is ready" -ForegroundColor Green
    Write-Host "You may proceed with Exchange server removal" -ForegroundColor Cyan
} else {
    Write-Host "✗ VERIFICATION FAILED - DO NOT PROCEED" -ForegroundColor Red
    Write-Host "Fix the issues above before making any AD changes" -ForegroundColor Yellow
}

Write-Host ""
Write-Host "Detailed Results:" -ForegroundColor Cyan
$Results | ForEach-Object { Write-Host "  $_" }
```

---

## Restoration Procedures

### Scenario 1: Restore from NTDSUTIL Snapshot

**When to Use:** You need to recover specific AD objects without full DC restore.

#### Step 1: Mount the Snapshot

```cmd
ntdsutil
activate instance ntds
snapshot
list all
```

Identify your snapshot GUID, then:

```
mount {snapshot-guid}
```

Note the mount path (e.g., `C:\$SNAP_datetime_VOLUMEC$\`)

#### Step 2: Copy NTDS.dit from Snapshot

```cmd
# Create working directory
mkdir C:\ADRestore

# Copy database from snapshot
copy "C:\$SNAP_datetime_VOLUMEC$\Windows\NTDS\ntds.dit" C:\ADRestore\

# Copy log files (optional but recommended)
copy "C:\$SNAP_datetime_VOLUMEC$\Windows\NTDS\*.log" C:\ADRestore\
```

#### Step 3: Extract Objects Using NTDSUTIL

```cmd
ntdsutil
activate instance ntds
authoritative restore
```

Now you have two options:

**Option A: Restore Entire Subtree (e.g., Exchange Organization)**

```
restore subtree "CN=Microsoft Exchange,CN=Services,CN=Configuration,DC=yourdomain,DC=com"
```

**Option B: Restore Single Object (e.g., Specific Exchange Server)**

```
restore object "CN=EXCHANGE-TEST-01,CN=Servers,CN=Exchange Administrative Group (FYDIBOHF23SPDLT),CN=Administrative Groups,CN=Microsoft Exchange,CN=Services,CN=Configuration,DC=yourdomain,DC=com"
```

Type `quit` twice to exit, then:

```cmd
# Unmount snapshot
ntdsutil
activate instance ntds
snapshot
unmount {snapshot-guid}
quit
quit
```

#### Step 4: Force AD Replication

```powershell
# Force replication to all DCs
repadmin /syncall /AdeP

# Monitor replication
repadmin /replsummary

# Verify object restored
Get-ADObject -Filter {Name -like "*EXCHANGE-TEST-01*"} -IncludeDeletedObjects
```

### Scenario 2: Restore from Windows Server Backup

**When to Use:** Complete DC failure or corruption requiring full system state restore.

#### Step 1: Identify Backup Version

```powershell
# List all backups
Get-WBBackupSet | Format-Table BackupTime, VersionId

# Get specific backup details
$Backup = Get-WBBackupSet | Where-Object {$_.BackupTime -eq (Get-Date "2026-01-08 14:30")}
$Backup | Format-List *
```

#### Step 2: Start System State Recovery

**Method A: PowerShell**

```powershell
# Restore system state (NON-AUTHORITATIVE - default)
Start-WBSystemStateRecovery -BackupSet $Backup -Force

# For AUTHORITATIVE restore (overwrites newer data):
# Use ntdsutil after system state restore completes
```

**Method B: Command Line**

```cmd
# Non-authoritative restore
wbadmin start systemstaterecovery -version:01/08/2026-14:30 -backupTarget:E:

# Authoritative restore requires additional steps (see below)
```

#### Step 3: Perform Authoritative Restore (If Needed)

After system state restore completes, reboot into Directory Services Restore Mode (DSRM):

```
1. Restart DC
2. Press F8 during boot
3. Select "Directory Services Restore Mode"
4. Login with DSRM password
5. Open Command Prompt
6. Run: ntdsutil
```

```cmd
ntdsutil
activate instance ntds
authoritative restore
restore database
# Or for specific objects:
restore subtree "CN=Microsoft Exchange,CN=Services,CN=Configuration,DC=domain,DC=com"
quit
quit
```

#### Step 4: Restart and Verify

```powershell
# After reboot, verify AD health
dcdiag /v
repadmin /replsummary
repadmin /showrepl

# Verify Exchange objects
Get-ExchangeServer
Get-MailboxDatabase
```

### Scenario 3: Restore from LDIF Exports

**When to Use:** Surgical restoration of specific objects without affecting rest of AD.

#### Step 1: Review LDIF File

```powershell
# Check what's in the LDIF export
Get-Content "C:\AD_Backups\20260108_143000\Exchange_Server_EXCHANGE-TEST-01.ldf" | Select-Object -First 50
```

#### Step 2: Import LDIF File

```cmd
# Basic import
ldifde -i -f "C:\AD_Backups\20260108_143000\Exchange_Server_EXCHANGE-TEST-01.ldf"

# Import with logging
ldifde -i -f "C:\AD_Backups\20260108_143000\Exchange_Server_EXCHANGE-TEST-01.ldf" -j C:\ADRestore\Logs

# Import specific attributes only
ldifde -i -f "C:\AD_Backups\20260108_143000\Exchange_Server_EXCHANGE-TEST-01.ldf" -k
```

The `-k` switch ignores errors (useful if some objects already exist).

#### Step 3: Restore Group Memberships

```powershell
# Use the restoration script created during backup
& "C:\AD_Backups\20260108_143000\Restore_EXCHANGE-TEST-01.ps1"

# Or manually restore groups:
$GroupMemberships = Import-Csv "C:\AD_Backups\20260108_143000\Server_EXCHANGE-TEST-01_GroupMemberships.csv"

foreach ($Group in $GroupMemberships) {
    try {
        Add-ADGroupMember -Identity $Group.Name -Members "EXCHANGE-TEST-01$" -ErrorAction Stop
        Write-Host "Added to: $($Group.Name)" -ForegroundColor Green
    } catch {
        Write-Host "Failed: $($Group.Name) - $_" -ForegroundColor Red
    }
}
```

#### Step 4: Verify Restoration

```powershell
# Verify Exchange server object exists
Get-ADObject -Filter {Name -eq "EXCHANGE-TEST-01"} -SearchBase "CN=Configuration,$((Get-ADDomain).DistinguishedName)" -Properties *

# Verify computer account
Get-ADComputer -Identity EXCHANGE-TEST-01

# Verify group memberships
Get-ADPrincipalGroupMembership -Identity "EXCHANGE-TEST-01$" | Select-Object Name

# Verify in Exchange
Get-ExchangeServer -Identity EXCHANGE-TEST-01
```

---

## Complete Pre-Exchange-Removal Checklist

Use this comprehensive checklist before removing any Exchange server.

### Phase 1: Documentation (15 minutes)

- [ ] Document current Exchange environment
  ```powershell
  Get-ExchangeServer | Export-Csv "ExchangeServers_$(Get-Date -Format 'yyyyMMdd').csv"
  ```
- [ ] Export all Exchange databases
  ```powershell
  Get-MailboxDatabase | Export-Csv "MailboxDatabases_$(Get-Date -Format 'yyyyMMdd').csv"
  ```
- [ ] Document AD replication health
  ```powershell
  repadmin /replsummary > "AD_Replication_$(Get-Date -Format 'yyyyMMdd').txt"
  ```
- [ ] Note FSMO role holders
  ```cmd
  netdom query fsmo > "FSMO_Roles_$(Get-Date -Format 'yyyyMMdd').txt"
  ```
- [ ] Screenshot ADSIEdit structure (before changes)

### Phase 2: AD Health Validation (10 minutes)

- [ ] Check AD replication: `repadmin /replsummary`
- [ ] Verify no replication errors: `repadmin /showrepl`
- [ ] Run DCDIAG: `dcdiag /v /c /e`
- [ ] Check disk space: `Get-Volume`
- [ ] Verify all DCs are online: `Get-ADDomainController -Filter *`

### Phase 3: Create NTDSUTIL Snapshot (5-10 minutes)

- [ ] Create snapshot: `ntdsutil "activate instance ntds" "snapshot" "create" quit quit`
- [ ] Document snapshot GUID
- [ ] Verify snapshot: `ntdsutil "activate instance ntds" "snapshot" "list all" quit quit`
- [ ] Mount and verify snapshot accessibility
- [ ] Unmount snapshot: `ntdsutil "activate instance ntds" "snapshot" "unmount {guid}" quit quit`

### Phase 4: Create Windows Server Backup (30-60 minutes)

- [ ] Install Windows Server Backup feature if needed
- [ ] Verify backup target has sufficient space
- [ ] Start system state backup: `wbadmin start systemstatebackup -backupTarget:E:`
- [ ] Monitor backup progress: `Get-WBJob -Previous 1`
- [ ] Verify backup completed: `Get-WBBackupSet | Sort-Object BackupTime -Descending | Select-Object -First 1`

### Phase 5: Export AD Objects (10-15 minutes)

- [ ] Create backup directory structure
- [ ] Export Exchange organization configuration
  ```powershell
  ldifde -f "Exchange_Org_Export.ldf" -d "CN=Microsoft Exchange,CN=Services,CN=Configuration,DC=domain,DC=com" -p SubTree
  ```
- [ ] Export specific Exchange server object
- [ ] Export Exchange databases configuration
- [ ] Export computer account
- [ ] Export security group memberships
- [ ] Generate restoration script

### Phase 6: Backup Verification (10 minutes)

- [ ] Run verification script (provided above)
- [ ] All critical files exist and are not empty
- [ ] LDIF files are valid (contain `dn:` and `changetype:`)
- [ ] Snapshot can be mounted
- [ ] Windows Server Backup includes System State
- [ ] Backup size is reasonable
- [ ] Restoration script is present and valid

### Phase 7: Secure Backup Storage (5 minutes)

- [ ] Copy backups to network share
  ```powershell
  Copy-Item "C:\AD_Backups\*" -Destination "\\FileServer\DC_Backups\$(Get-Date -Format 'yyyyMMdd')" -Recurse
  ```
- [ ] Verify network copy completed
- [ ] Document backup locations
- [ ] Set backup retention (recommended: 30 days minimum)
- [ ] Add backup location to documentation

### Phase 8: Final Pre-Change Checks (5 minutes)

- [ ] Change control ticket approved
- [ ] Maintenance window scheduled
- [ ] Team notified of planned change
- [ ] Rollback plan documented and understood
- [ ] Emergency contacts available
- [ ] All backups verified and accessible

### Total Time Required

| Phase | Minimum | Maximum |
|-------|---------|---------|
| Documentation | 15 min | 20 min |
| AD Health Check | 10 min | 15 min |
| NTDSUTIL Snapshot | 5 min | 10 min |
| Windows Server Backup | 30 min | 90 min |
| AD Object Exports | 10 min | 20 min |
| Verification | 10 min | 15 min |
| Secure Storage | 5 min | 10 min |
| Final Checks | 5 min | 10 min |
| **TOTAL** | **90 min** | **190 min** |

**Recommended Window:** 3-4 hours (includes buffer for issues)

---

## Quick Reference Card

### Emergency Snapshot Creation

```cmd
ntdsutil "activate instance ntds" "snapshot" "create" "list all" quit quit
```

### Emergency Snapshot Restoration

```cmd
ntdsutil
activate instance ntds
authoritative restore
restore database
quit
quit
```

### Verify All Backups Exist

```powershell
# NTDSUTIL Snapshot
ntdsutil "activate instance ntds" "snapshot" "list all" quit quit

# Windows Server Backup
Get-WBBackupSet | Sort-Object BackupTime -Descending | Select-Object -First 1

# LDIF Exports
Get-ChildItem "C:\AD_Backups\$(Get-Date -Format 'yyyyMMdd')\*.ldf"
```

### Force AD Replication

```powershell
repadmin /syncall /AdeP
```

---

## Troubleshooting

### Issue: "Insufficient Disk Space" for Snapshot

**Solution:**
```powershell
# Find large files to clean up
Get-ChildItem C:\ -Recurse -ErrorAction SilentlyContinue | 
    Sort-Object Length -Descending | 
    Select-Object -First 20 |
    Format-Table Name, @{Name="Size(MB)";Expression={[math]::Round($_.Length/1MB,2)}}, Directory

# Clean up old snapshots
ntdsutil "activate instance ntds" "snapshot" "list all" "delete {old-guid}" quit quit

# Run Disk Cleanup
cleanmgr /verylowdisk
```

### Issue: Snapshot Creation Fails

**Solution:**
```powershell
# Check VSS service
Get-Service VSS | Restart-Service -Force

# Check disk errors
chkdsk C: /scan

# Verify NTDS service
Get-Service NTDS | Format-List *

# Try creating snapshot again
ntdsutil "activate instance ntds" "snapshot" "create" quit quit
```

### Issue: LDIF Import Fails

**Solution:**
```cmd
# Use -k switch to continue on errors
ldifde -i -f backup.ldf -k

# Use -j to create log file
ldifde -i -f backup.ldf -j C:\Logs

# Review specific error in log
notepad C:\Logs\ldif.log
```

### Issue: "Access Denied" During Restoration

**Solution:**
```powershell
# Verify you're running as Domain Admin
whoami /groups | findstr "Domain Admins"

# Check if you're on a DC
Get-ADDomainController -Identity $env:COMPUTERNAME

# If not on DC, connect to one
Enter-PSSession -ComputerName DC01
```

---

## Conclusion

You now have three complementary backup methods:

1. **NTDSUTIL Snapshot** - Fast, point-in-time, object-level recovery
2. **Windows Server Backup** - Complete system state, full DC recovery
3. **LDIF Exports** - Surgical, documented, scriptable recovery

**Best Practice:** Use all three methods for critical changes like Exchange server removal.

**Remember:**
- Always verify backups before proceeding
- Document everything
- Test restoration procedures in lab if possible
- Keep backups for 30+ days after successful change
- AD replication takes time - be patient after restoration

**Next Steps:**
After completing all backup procedures and verification, you're ready to proceed with the Exchange server removal using the main runbook.
