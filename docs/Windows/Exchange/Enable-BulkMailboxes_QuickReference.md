# Bulk Mailbox Enablement - Quick Reference Guide

## Prerequisites

1. Run from Exchange Management Shell (as administrator)
2. Account must have Organization Management or Recipient Management permissions
3. Active Directory PowerShell module must be available
4. Target mailbox database must be mounted and healthy

## Pre-Flight Checks

Before running the script, verify your environment:

```powershell
# Check mailbox databases are healthy
Get-MailboxDatabase | Select-Object Name, Mounted, DatabaseSize

# Check available space
Get-MailboxDatabase | Get-MailboxDatabaseCopyStatus

# Verify your permissions
Get-RoleGroupMember "Organization Management" | Where-Object {$_.Name -like "*YourAccount*"}
```

## Usage Examples

### Test Run (WhatIf Mode)
Always start with a test run:

```powershell
.\Enable-BulkMailboxes.ps1 -TargetOU "OU=Users,OU=Corporate,DC=domain,DC=com" -WhatIf
```

### Basic Execution
Let Exchange auto-distribute across databases:

```powershell
.\Enable-BulkMailboxes.ps1 -TargetOU "OU=Users,OU=Corporate,DC=domain,DC=com"
```

### Specify Target Database
Direct all new mailboxes to a specific database:

```powershell
.\Enable-BulkMailboxes.ps1 -TargetOU "OU=Users,DC=domain,DC=com" -Database "MailboxDB01"
```

### Exclude Specific OUs
Skip service accounts and admin accounts:

```powershell
.\Enable-BulkMailboxes.ps1 -TargetOU "OU=Users,DC=domain,DC=com" -ExcludeOUs @("Service Accounts", "Admin Accounts", "Contractors")
```

### Custom Logging Location

```powershell
.\Enable-BulkMailboxes.ps1 -TargetOU "OU=Users,DC=domain,DC=com" -LogPath "D:\Logs\MailboxEnablement.log"
```

### Adjust Throttling for Large Environments

```powershell
.\Enable-BulkMailboxes.ps1 -TargetOU "OU=Users,DC=domain,DC=com" -BatchSize 100 -ThrottleSeconds 10
```

## Parameters Reference

| Parameter | Required | Default | Description |
|-----------|----------|---------|-------------|
| TargetOU | Yes | - | Distinguished name of OU to process |
| Database | No | Auto | Specific mailbox database to use |
| LogPath | No | C:\Temp\MailboxEnablement.log | Log file location |
| ExcludeOUs | No | Service Accounts, Admin Accounts, Disabled Users | OUs to skip |
| BatchSize | No | 50 | Users per batch before throttle pause |
| ThrottleSeconds | No | 5 | Pause duration between batches |
| WhatIf | No | False | Test mode - no changes made |

## Output Files

The script generates two files:

1. **Log file** (`.log`) - Detailed execution log
2. **Results CSV** (`.csv`) - Structured results for each user

## Post-Execution Verification

After running the script:

```powershell
# Verify mailboxes were created
Get-Mailbox -ResultSize Unlimited | Measure-Object

# Check for any users still without mailboxes
Get-ADUser -SearchBase "OU=Users,DC=domain,DC=com" -Filter {Enabled -eq $true} | ForEach-Object {
    $mb = Get-Mailbox -Identity $_.SamAccountName -ErrorAction SilentlyContinue
    if (-not $mb) {
        Write-Host "No mailbox: $($_.SamAccountName)"
    }
}

# Review the results CSV
Import-Csv "C:\Temp\MailboxEnablement_Results.csv" | Where-Object {$_.Status -eq "Error"}
```

## Troubleshooting

| Error | Cause | Solution |
|-------|-------|----------|
| "Database is mandatory" | Corrupted AD object | Check user's homeMDB attribute in AD |
| "Insufficient permissions" | RBAC issue | Verify account is in Recipient Management |
| "Database not mounted" | DB offline | Mount database or choose different DB |
| "Object not found" | User deleted during run | Verify user exists in AD |

## Best Practices

1. **Always run WhatIf first** - Verify scope before making changes
2. **Run during off-hours** - Reduces impact on production
3. **Monitor database size** - Ensure adequate storage
4. **Review logs** - Check for errors after completion
5. **Backup AD first** - Especially in new environments
6. **Start small** - Test on a small OU first

## Rollback

If you need to remove mailboxes that were incorrectly created:

```powershell
# Disable mailbox (keeps AD account, removes Exchange attributes)
Disable-Mailbox -Identity "username" -Confirm:$false

# Bulk disable from CSV of errors
Import-Csv "C:\Temp\MailboxEnablement_Results.csv" | 
    Where-Object {$_.Status -eq "Success"} | 
    ForEach-Object { Disable-Mailbox -Identity $_.SamAccountName -Confirm:$false }
```
