# Exchange Server Removal Runbook

**Document Type:** Operational Runbook  
**Last Updated:** January 7, 2026  
**Situation:** Removing first Exchange server (approval/test) - Production server remains active

---

## Table of Contents

1. [Overview](#overview)
2. [Prerequisites & Pre-Flight Checks](#prerequisites--pre-flight-checks)
3. [Critical Warnings](#critical-warnings)
4. [Backup & Rollback Strategy](#backup--rollback-strategy)
5. [Removal Procedure](#removal-procedure)
6. [Post-Removal Validation](#post-removal-validation)
7. [Common Issues & Troubleshooting](#common-issues--troubleshooting)
8. [Additional Recommendations](#additional-recommendations)

---

## Overview

### Purpose
Remove a non-production Exchange server from Active Directory after it has served its initial approval/testing purpose.

### Scope
- **Server to Remove:** [EXCHANGE-TEST-01] (approval/test server)
- **Remaining Server:** [EXCHANGE-PROD-01] (production server)
- **Environment:** On-premises Exchange Server

### Key Differences from Article
The referenced article focuses on "failed" servers that cannot be gracefully uninstalled. This runbook covers **both scenarios**:
- **Preferred Path:** Graceful uninstall (if server is still accessible)
- **Emergency Path:** Forced removal via ADSIEdit (if server failed/inaccessible)

---

## Prerequisites & Pre-Flight Checks

### 1. Verify Current State

```powershell
# Run from Exchange Management Shell on production server
Get-ExchangeServer | Format-Table Name, ServerRole, AdminDisplayVersion

# Check for any active databases on the server to be removed
Get-MailboxDatabase -Server EXCHANGE-TEST-01 | Format-List Name, MountedOnServer, DatabaseSize

# Verify no active mailboxes
Get-Mailbox -Server EXCHANGE-TEST-01 | Measure-Object
```

**Expected Results:**
- Server to remove shows in list
- No active databases on test server
- Zero mailboxes on test server

### 2. Verify Production Server Health

```powershell
# Test mail flow
Test-ServiceHealth -Server EXCHANGE-PROD-01

# Verify all databases are mounted
Get-MailboxDatabase -Status | Where-Object {$_.Mounted -eq $false}

# Check replication status (if using DAG)
Get-MailboxDatabaseCopyStatus
```

### 3. Document Current Configuration

```powershell
# Export current Exchange configuration
Get-ExchangeServer | Export-Csv "C:\Temp\ExchangeServers_Baseline.csv" -NoTypeInformation

# Export Receive Connectors
Get-ReceiveConnector | Export-Csv "C:\Temp\ReceiveConnectors_Baseline.csv" -NoTypeInformation

# Export Send Connectors
Get-SendConnector | Export-Csv "C:\Temp\SendConnectors_Baseline.csv" -NoTypeInformation
```

### 4. Check for Remaining Dependencies

```powershell
# Check for public folder databases
Get-PublicFolderDatabase -Server EXCHANGE-TEST-01

# Check for address lists pointing to server
Get-AddressList | Where-Object {$_.RecipientContainer -like "*EXCHANGE-TEST-01*"}

# Check OAB generation server
Get-OfflineAddressBook | Select-Object Name, Server

# Check Autodiscover SCP
Get-ClientAccessServer EXCHANGE-TEST-01 | Select-Object AutoDiscoverServiceInternalUri
```

---

## Critical Warnings

### ⚠️ STOP - Read This Before Proceeding

1. **ADSIEdit is Destructive**: No undo button exists. Wrong deletions can break entire Active Directory.

2. **Article Error - Wrong Order**: The article has steps out of sequence. Database removal should happen BEFORE server removal in ADSIEdit.

3. **Missing Critical Steps**: Article omits:
   - Checking for arbitration mailboxes
   - Removing from CAS arrays/load balancers
   - Cleaning DNS records
   - Removing certificates
   - Checking Edge Transport subscriptions

4. **Graceful Uninstall Preferred**: If the server is accessible, use `Setup.exe /mode:Uninstall` instead of ADSIEdit method.

---

## Backup & Rollback Strategy

### Pre-Change Backups (MANDATORY)

```bash
# 1. System State Backup of Domain Controller
wbadmin start systemstatebackup -backupTarget:E: -quiet

# 2. Export Active Directory Partitions
ntdsutil "activate instance ntds" "snapshot" "create" "mount {GUID}" quit quit

# 3. Document what you'll change
# Take screenshots of ADSIEdit paths BEFORE deletion
# Export registry keys related to Exchange on DC
```

### Backup Verification Checklist
- [ ] System state backup completed successfully
- [ ] Backup verified and tested
- [ ] Screenshots taken of ADSIEdit structure
- [ ] Current AD schema documented
- [ ] Change control ticket created
- [ ] Rollback window scheduled

---

## Removal Procedure

### Path A: Graceful Uninstall (PREFERRED)

Use this if the test server is still accessible and functioning.

#### Step 1: Move or Remove Mailboxes

```powershell
# Check for any system mailboxes
Get-Mailbox -Server EXCHANGE-TEST-01 -Arbitration

# Move arbitration mailboxes to production server
Get-Mailbox -Server EXCHANGE-TEST-01 -Arbitration | New-MoveRequest -TargetDatabase "DB01-PROD"

# Monitor move progress
Get-MoveRequest | Get-MoveRequestStatistics
```

#### Step 2: Remove Databases

```powershell
# Dismount databases
Get-MailboxDatabase -Server EXCHANGE-TEST-01 | Dismount-Database -Confirm:$false

# Remove database objects (keep files for now)
Get-MailboxDatabase -Server EXCHANGE-TEST-01 | Remove-MailboxDatabase -Confirm:$false
```

#### Step 3: Clean Up Connectors

```powershell
# Remove receive connectors specific to this server
Get-ReceiveConnector -Server EXCHANGE-TEST-01 | Remove-ReceiveConnector -Confirm:$false

# Update send connectors (remove server from source list)
Get-SendConnector | Where-Object {$_.SourceTransportServers -contains "EXCHANGE-TEST-01"} | 
    Set-SendConnector -SourceTransportServers @{Remove="EXCHANGE-TEST-01"}
```

#### Step 4: Run Exchange Uninstaller

```cmd
# From the Exchange installation media directory
Setup.exe /mode:Uninstall /IAcceptExchangeServerLicenseTerms

# Monitor setup logs
notepad "C:\ExchangeSetupLogs\ExchangeSetup.log"
```

**If uninstall fails, proceed to Path B.**

---

### Path B: Forced Removal (EMERGENCY ONLY)

Use only if server is dead/inaccessible or graceful uninstall failed.

#### Critical Correction to Article Sequence

The article presents steps in a dangerous order. **Correct sequence:**

1. Remove databases from configuration FIRST
2. THEN remove server object
3. THEN clean up AD groups
4. THEN remove computer account

#### Step 1: Backup Active Directory

```powershell
# Create AD snapshot
ntdsutil "activate instance ntds" "snapshot" "create" quit quit

# Export current configuration
repadmin /showrepl > C:\Temp\AD_Replication_Before.txt
```

#### Step 2: Launch ADSIEdit

```
1. Open Run (Win + R)
2. Type: adsiedit.msc
3. Click OK
```

#### Step 3: Connect to Configuration Naming Context

```
1. Right-click "ADSI Edit" → Connect to...
2. Select "Configuration" from dropdown
3. Click OK
```

#### Step 4: Navigate to Exchange Configuration

```
Path: CN=Configuration,DC=yourdomain,DC=com
      └─ CN=Services
         └─ CN=Microsoft Exchange
            └─ CN=[Your-Org-Name]
               └─ CN=Administrative Groups
                  └─ CN=Exchange Administrative Group (FYDIBOHF23SPDLT)
```

**IMPORTANT:** Your administrative group name may be different. The default is shown above.

#### Step 5: Remove Database Objects FIRST

**Article Error:** Article shows database removal as optional/later step. This is wrong.

```
Navigate to:
CN=Administrative Groups
  └─ CN=Exchange Administrative Group (FYDIBOHF23SPDLT)
     └─ CN=Databases

Steps:
1. Expand CN=Databases
2. Find databases associated with EXCHANGE-TEST-01
   (Look for "CN=[DatabaseName]" with server property pointing to your test server)
3. Right-click each database → Delete
4. Confirm deletion
5. Repeat for ALL databases from this server
```

**Validation:**
```powershell
# Run from Exchange Management Shell
Get-MailboxDatabase -Server EXCHANGE-TEST-01
# Should return error or empty
```

#### Step 6: Remove Server Object

```
Navigate to:
CN=Administrative Groups
  └─ CN=Exchange Administrative Group (FYDIBOHF23SPDLT)
     └─ CN=Servers

Steps:
1. Expand CN=Servers
2. Locate "CN=EXCHANGE-TEST-01" (or your server name)
3. Right-click → Delete
4. Confirm deletion
```

**Screenshot Before Deletion:** Always screenshot the object properties before deleting.

#### Step 7: Clean Additional Exchange Configuration

**Missing from Article - Check these locations:**

##### A. Remove from Availability Service

```
Path: CN=Administrative Groups
      └─ CN=Exchange Administrative Group (FYDIBOHF23SPDLT)
         └─ CN=Servers
            └─ CN=EXCHANGE-PROD-01
               └─ CN=InformationStore

Check: Ensure no references to old server remain
```

##### B. Remove Routing Group Connectors (if present)

```
Path: CN=Administrative Groups
      └─ CN=Exchange Administrative Group (FYDIBOHF23SPDLT)
         └─ CN=Routing Groups

Check and remove any connectors referencing the old server
```

##### C. Remove from Address Lists

```
Path: CN=Administrative Groups
      └─ CN=Exchange Administrative Group (FYDIBOHF23SPDLT)
         └─ CN=Address Lists Container

Check each address list for server references
```

#### Step 8: Remove from Active Directory Security Groups

```
1. Open "Active Directory Users and Computers"
2. Enable "Advanced Features" (View menu)
3. Navigate to: [Domain] → Microsoft Exchange Security Groups
4. Open each relevant group:
   - Exchange Servers
   - Exchange Trusted Subsystem
   - ExchangeLegacyInterop
5. Go to Members tab
6. Find and Remove EXCHANGE-TEST-01$
7. Click Apply
```

**Missing from Article:** Also check these groups:
- Exchange Windows Permissions
- Managed Availability Servers
- Exchange Install Domain Servers

#### Step 9: Remove Computer Account

```
1. In "Active Directory Users and Computers"
2. Click Find (binoculars icon) or Ctrl+F
3. Find: "Computers"
4. Search for: EXCHANGE-TEST-01
5. Right-click server → Delete
6. Confirm deletion
```

#### Step 10: Clean DNS Records (MISSING FROM ARTICLE)

```powershell
# From DNS Manager or PowerShell

# Remove A records
Remove-DnsServerResourceRecord -ZoneName "yourdomain.com" -Name "EXCHANGE-TEST-01" -RRType "A" -Force

# Remove autodiscover references if they exist
Get-DnsServerResourceRecord -ZoneName "yourdomain.com" -Name "autodiscover" | 
    Where-Object {$_.RecordData -like "*EXCHANGE-TEST-01*"} | 
    Remove-DnsServerResourceRecord -Force

# Remove any SRV records
Get-DnsServerResourceRecord -ZoneName "_tcp.yourdomain.com" -RRType "SRV" | 
    Where-Object {$_.RecordData -like "*EXCHANGE-TEST-01*"}
```

#### Step 11: Update Autodiscover SCP (MISSING FROM ARTICLE)

```powershell
# Verify production server Autodiscover
Get-ClientAccessServer EXCHANGE-PROD-01 | Format-List AutoDiscoverServiceInternalUri

# Check for orphaned SCPs
Get-ADObject -Filter {objectClass -eq "serviceConnectionPoint" -and Name -like "*Autodiscover*"} -Properties *

# Remove SCPs pointing to old server if found
# This requires careful review - don't delete production SCPs
```

---

## Post-Removal Validation

### Step 1: Verify Exchange Configuration

```powershell
# Should NOT show removed server
Get-ExchangeServer

# Should show only production server
Get-ExchangeServer -Identity EXCHANGE-PROD-01 | Format-List

# Verify databases
Get-MailboxDatabase | Format-Table Name, Server, Mounted

# Test mail flow
Test-ServiceHealth

# Test Outlook connectivity
Test-OutlookConnectivity -Protocol:HTTP
```

### Step 2: Verify Active Directory

```powershell
# Check computer accounts
Get-ADComputer -Filter {Name -like "*EXCHANGE-TEST*"}
# Should return nothing

# Check Exchange security groups
Get-ADGroupMember "Exchange Servers"
# Should NOT include old server

# Verify AD replication
repadmin /replsummary
repadmin /showrepl
```

### Step 3: Test Client Connectivity

```powershell
# Test Autodiscover
Test-OutlookWebServices -Identity testuser@domain.com -MailboxCredential (Get-Credential)

# Test ActiveSync
Test-ActiveSyncConnectivity -MailboxCredential (Get-Credential)

# Test OWA
Test-OwaConnectivity -URL "https://mail.yourdomain.com/owa"
```

### Step 4: Monitor Event Logs

```powershell
# Check for Exchange-related errors
Get-EventLog -LogName Application -Source "MSExchange*" -EntryType Error -Newest 50

# Check AD replication
Get-EventLog -LogName "Directory Service" -EntryType Error -Newest 20
```

### Step 5: Verify External Services

- [ ] Outlook clients connect successfully
- [ ] OWA/ECP accessible
- [ ] ActiveSync working
- [ ] SMTP mail flow functioning
- [ ] Autodiscover returns correct server
- [ ] No certificate warnings

---

## Common Issues & Troubleshooting

### Issue 1: "Object Cannot Be Deleted" in ADSIEdit

**Cause:** Object has child objects or protected status

**Resolution:**
```
1. Expand the object in ADSIEdit
2. Check for child objects
3. Delete child objects first (databases, protocols, etc.)
4. Then delete parent object
5. If still protected, right-click → Properties → Object tab
6. Uncheck "Protect object from accidental deletion"
```

### Issue 2: Server Still Appears in Get-ExchangeServer

**Cause:** AD replication delay or cached results

**Resolution:**
```powershell
# Force AD replication
repadmin /syncall /AdeP

# Wait 15 minutes for AD replication
Start-Sleep -Seconds 900

# Clear Exchange server cache
Get-ExchangeServer | Update-ExchangeServer

# Restart Exchange services on production server
Restart-Service MSExchangeServiceHost -Force
```

### Issue 3: Autodiscover Returns Removed Server

**Cause:** Stale SCP records in Active Directory

**Resolution:**
```powershell
# Find all Autodiscover SCPs
$domain = (Get-ADDomain).DistinguishedName
Get-ADObject -SearchBase "CN=Configuration,$domain" -Filter {objectClass -eq "serviceConnectionPoint" -and Name -like "*Autodiscover*"} -Properties serviceBindingInformation

# Remove SCPs pointing to old server
# CAREFUL: Verify it's the old server before removing
Remove-ADObject -Identity [DistinguishedName of old SCP] -Confirm:$false
```

### Issue 4: Mailbox Moves Fail

**Cause:** Arbitration mailboxes not moved first

**Resolution:**
```powershell
# Move arbitration mailboxes first
Get-Mailbox -Arbitration -Server EXCHANGE-TEST-01 | New-MoveRequest -TargetDatabase "DB01-PROD"

# Monitor until complete
Get-MoveRequest | Get-MoveRequestStatistics | Where-Object {$_.Status -ne "Completed"}

# Then move user mailboxes
Get-Mailbox -Server EXCHANGE-TEST-01 | New-MoveRequest -TargetDatabase "DB01-PROD"
```

### Issue 5: Certificate Errors After Removal

**Cause:** Certificates still reference old server

**Resolution:**
```powershell
# Review certificates
Get-ExchangeCertificate

# Remove certificates referencing old server
Get-ExchangeCertificate -Server EXCHANGE-PROD-01 | Where-Object {$_.Services -like "*"} | 
    Remove-ExchangeCertificate -Confirm:$false
```

---

## Additional Recommendations

### What the Article Got Wrong

1. **Sequence Error:** Databases should be removed BEFORE server object, not after
2. **Missing Validation:** No health checks before or after removal
3. **No Backup Strategy:** Doesn't mention AD backups despite "grave repercussions" warning
4. **Incomplete Cleanup:** Misses DNS, SCPs, certificates, load balancers
5. **No Graceful Path:** Jumps straight to ADSIEdit without trying normal uninstall
6. **No Testing:** Doesn't validate functionality after removal

### Additional Best Practices

#### 1. Use Exchange Setup for Preparation (If Server Accessible)

Even if you plan forced removal, try preparation mode first:

```cmd
Setup.exe /PrepareAD /OrganizationName:[YourOrg]
Setup.exe /PrepareDomain
```

#### 2. Export Before Removal

```powershell
# Export complete configuration
Get-ExchangeServer EXCHANGE-TEST-01 | Export-Clixml C:\Temp\OldServer_Config.xml

# Document all settings
Get-ExchangeServer EXCHANGE-TEST-01 | Format-List * | Out-File C:\Temp\OldServer_Full.txt
```

#### 3. Check Third-Party Integration

Before removal, verify:
- [ ] Backup software (Veeam, Commvault, etc.) not targeting this server
- [ ] Monitoring tools updated to ignore server
- [ ] Load balancers updated
- [ ] Firewall rules cleaned up
- [ ] External mail flow systems updated

#### 4. Plan Maintenance Window

Required downtime:
- **Graceful uninstall:** 1-2 hours (minimal user impact)
- **Forced removal:** 30 minutes AD work + 2 hours testing
- **Recommended window:** Off-hours, 4-hour window with rollback plan

#### 5. Document Everything

Create a detailed change log:
```
Date: [Date]
Engineer: [Name]
Action: Removed EXCHANGE-TEST-01 from environment
Method: [Graceful/Forced]
Issues Encountered: [List]
Resolution: [Details]
Post-validation Results: [Pass/Fail with details]
Rollback Performed: [Yes/No]
```

#### 6. Modern Alternative: Use PowerShell

For modern Exchange (2016+), you can script portions of the cleanup:

```powershell
# Script-based cleanup (requires customization)
# Remove server object
Set-ADObject -Identity "CN=EXCHANGE-TEST-01,CN=Servers,CN=Exchange Administrative Group..." -ProtectedFromAccidentalDeletion:$false
Remove-ADObject -Identity "CN=EXCHANGE-TEST-01,CN=Servers,CN=Exchange Administrative Group..." -Confirm:$false

# Remove from security groups
Remove-ADGroupMember -Identity "Exchange Servers" -Members "EXCHANGE-TEST-01$" -Confirm:$false
```

### Recovery Options

If something goes wrong during removal:

#### Option 1: Restore from System State Backup

```cmd
wbadmin start systemstaterecovery -backupTarget:E: -recoveryTarget:C:
```

#### Option 2: Authoritative AD Restore

```cmd
ntdsutil
activate instance ntds
authoritative restore
restore subtree "CN=EXCHANGE-TEST-01,CN=Servers..."
quit
quit
```

#### Option 3: Use Stellar Repair for Exchange (Article Recommendation)

If databases become inaccessible:
- Mount EDB files externally
- Export to PST or Office 365
- Last-resort recovery option

**Better Alternatives:**
- Native Exchange recovery database
- Windows Server Backup
- Third-party Exchange-aware backup (Veeam Backup for Microsoft 365)

---

## Checklist: Complete Removal

### Pre-Removal
- [ ] All mailboxes moved or deleted
- [ ] All databases removed
- [ ] Backups completed and verified
- [ ] Change control approved
- [ ] Rollback plan documented
- [ ] Maintenance window scheduled
- [ ] Team notified

### During Removal
- [ ] Graceful uninstall attempted (if possible)
- [ ] Databases removed from ADSIEdit
- [ ] Server object removed from ADSIEdit
- [ ] AD security groups cleaned
- [ ] Computer account deleted
- [ ] DNS records removed
- [ ] SCP records cleaned
- [ ] Certificates removed

### Post-Removal
- [ ] Get-ExchangeServer shows no old server
- [ ] No database references remain
- [ ] Mail flow tested successfully
- [ ] Client connectivity verified
- [ ] Autodiscover returns correct server
- [ ] No errors in event logs
- [ ] AD replication healthy
- [ ] Documentation updated

### 24-48 Hours After
- [ ] Monitor event logs
- [ ] Verify user reports
- [ ] Check backup jobs
- [ ] Confirm no performance issues
- [ ] Update disaster recovery docs

---

## References

- **Original Article:** https://www.stellarinfo.com/article/remove-failed-exchange-server-from-active-directory.php
- **Microsoft Docs:** [Remove a Server from Exchange](https://docs.microsoft.com/exchange)
- **ADSIEdit Documentation:** [Microsoft Active Directory Service Interfaces Editor](https://docs.microsoft.com/windows-server/administration)

---

## Change Log

| Date | Version | Author | Changes |
|------|---------|--------|---------|
| 2026-01-07 | 1.0 | [Your Name] | Initial runbook creation |

---

**Notes:**
- This runbook assumes on-premises Exchange Server (not Exchange Online)
- Test in lab environment if possible before production
- Always have backups before making schema changes
- Consider engaging Microsoft support for complex scenarios