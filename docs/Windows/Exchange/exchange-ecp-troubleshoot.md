# Exchange ECP Access Troubleshooting Runbook

## Airgapped Environment Edition

**Purpose:** Systematic troubleshooting guide for resolving Exchange Control Panel (ECP) access issues where Organization Management members cannot access ECP despite correct group membership.

**Environment Considerations:** This runbook is designed for airgapped networks where internet access is unavailable. All commands are self-contained and do not require external downloads.

---

## Table of Contents

1. [Pre-Troubleshooting Information Gathering](#1-pre-troubleshooting-information-gathering)
2. [Verify RBAC Group Membership](#2-verify-rbac-group-membership)
3. [Check RBAC Role Assignments](#3-check-rbac-role-assignments)
4. [Verify Active Directory Permissions](#4-verify-active-directory-permissions)
5. [IIS Authentication Configuration](#5-iis-authentication-configuration)
6. [Certificate and SSL/TLS Issues](#6-certificate-and-ssltls-issues)
7. [Virtual Directory Configuration](#7-virtual-directory-configuration)
8. [DNS and Name Resolution](#8-dns-and-name-resolution)
9. [Kerberos and Authentication](#9-kerberos-and-authentication)
10. [Event Log Analysis](#10-event-log-analysis)
11. [PrepareAD Re-application](#11-preparead-re-application)
12. [Additional Airgapped Considerations](#12-additional-airgapped-considerations)
13. [Resolution Verification](#13-resolution-verification)

---

## 1. Pre-Troubleshooting Information Gathering

Before making any changes, collect baseline information.

### 1.1 Document Current Environment

Run all commands from **Exchange Management Shell (EMS)** as the working admin:

```powershell
# Get Exchange Server version and build
Get-ExchangeServer | Format-List Name, Edition, AdminDisplayVersion, ServerRole

# Get server FQDN
[System.Net.Dns]::GetHostByName($env:COMPUTERNAME).HostName

# Document the installing admin account
Get-ExchangeServer | Select-Object Name, WhenCreated

# Get current logged-in user context
whoami /all
```

### 1.2 Document the Problem

Answer these questions before proceeding:

| Question | Your Answer |
|----------|-------------|
| Exchange Version (2016/2019)? | |
| Cumulative Update installed? | |
| Single server or DAG? | |
| How long since installation? | |
| Did ECP ever work for other admins? | |
| Any recent AD changes? | |
| Any recent certificate changes? | |
| Error message when accessing ECP? | |
| Browser being used? | |
| Accessing via FQDN, short name, or IP? | |

### 1.3 Capture the Exact Error

When you attempt to access ECP, document:

```
URL attempted: https://________________________________/ecp
Error displayed: ________________________________________________
HTTP status code (if shown): ____________________________________
```

Common errors and their sections:

| Error | Jump to Section |
|-------|-----------------|
| 403 Forbidden | Section 5 (IIS) or Section 3 (RBAC) |
| 401 Unauthorized | Section 9 (Kerberos) or Section 5 (IIS) |
| 500 Internal Server Error | Section 10 (Event Logs) |
| Certificate Error | Section 6 (Certificates) |
| Page cannot be displayed | Section 8 (DNS) or Section 7 (Virtual Directories) |
| Blank page after login | Section 3 (RBAC) or Section 4 (AD Permissions) |

---

## 2. Verify RBAC Group Membership

### 2.1 Check Organization Management Membership

```powershell
# List all members of Organization Management
Get-RoleGroupMember "Organization Management" | Format-Table Name, RecipientType

# Check if your specific account is a member
Get-RoleGroupMember "Organization Management" | Where-Object {$_.Name -like "*YourUsername*"}

# Check nested group membership (if you're in a group that's in Org Management)
Get-RoleGroupMember "Organization Management" | Where-Object {$_.RecipientType -eq "Group"}
```

### 2.2 Verify Your Account Has a Mailbox

ECP requires the admin account to be mail-enabled:

```powershell
# Check if your account has a mailbox
Get-Mailbox -Identity "DOMAIN\YourUsername" -ErrorAction SilentlyContinue

# If no mailbox, check if it's a mail user
Get-MailUser -Identity "DOMAIN\YourUsername" -ErrorAction SilentlyContinue

# If neither returns results, your account needs to be mail-enabled
```

**CRITICAL:** In Exchange 2013+, administrators accessing ECP must have a mailbox or be a mail user. If your account has neither:

```powershell
# Option A: Enable mailbox for existing AD user
Enable-Mailbox -Identity "DOMAIN\YourUsername" -Database "YourMailboxDatabase"

# Option B: Create as mail user (no mailbox, but mail-enabled)
# This requires an external email address
Enable-MailUser -Identity "DOMAIN\YourUsername" -ExternalEmailAddress "admin@external.com"
```

### 2.3 Check for Explicit Deny Entries

```powershell
# Check if there are any explicit denies on your account
Get-ManagementRoleAssignment -RoleAssignee "DOMAIN\YourUsername" | 
    Where-Object {$_.RoleAssigneeType -eq "User"} | 
    Format-Table Name, Role, RoleAssigneeType, Enabled
```

---

## 3. Check RBAC Role Assignments

### 3.1 Verify Role Group Has Correct Roles

```powershell
# List all roles assigned to Organization Management
Get-ManagementRoleAssignment -RoleAssignee "Organization Management" | 
    Format-Table Name, Role, RoleAssigneeType, Enabled | 
    Out-String -Width 200

# Count roles (should be 70+ for Organization Management)
(Get-ManagementRoleAssignment -RoleAssignee "Organization Management").Count
```

Expected critical roles for ECP access:

- View-Only Configuration
- View-Only Recipients  
- Organization Client Access
- User Options

### 3.2 Check for Missing Role Assignments

```powershell
# Compare against default roles - export current assignments
Get-ManagementRoleAssignment -RoleAssignee "Organization Management" | 
    Select-Object Name, Role | 
    Export-Csv C:\Temp\OrgMgmt_Roles.csv -NoTypeInformation

# Check if specific ECP-critical roles exist
$criticalRoles = @(
    "View-Only Configuration",
    "View-Only Recipients",
    "Mail Recipients",
    "Organization Client Access",
    "User Options"
)

foreach ($role in $criticalRoles) {
    $assignment = Get-ManagementRoleAssignment -Role $role -RoleAssignee "Organization Management" -ErrorAction SilentlyContinue
    if ($assignment) {
        Write-Host "FOUND: $role" -ForegroundColor Green
    } else {
        Write-Host "MISSING: $role" -ForegroundColor Red
    }
}
```

### 3.3 Verify RBAC Scopes

```powershell
# Check management scopes
Get-ManagementScope | Format-Table Name, ScopeRestrictionType, Exclusive

# Check if any exclusive scopes might be blocking access
Get-ManagementScope | Where-Object {$_.Exclusive -eq $true}
```

---

## 4. Verify Active Directory Permissions

### 4.1 Check Exchange Security Groups in AD

Run from a Domain Controller or machine with RSAT:

```powershell
# Import AD module
Import-Module ActiveDirectory

# Check Exchange security groups exist
$exchangeGroups = @(
    "Exchange Organization Administrators",
    "Exchange Servers",
    "Exchange Trusted Subsystem",
    "Exchange Windows Permissions",
    "ExchangeLegacyInterop"
)

foreach ($group in $exchangeGroups) {
    $grp = Get-ADGroup -Filter "Name -eq '$group'" -ErrorAction SilentlyContinue
    if ($grp) {
        Write-Host "EXISTS: $group" -ForegroundColor Green
    } else {
        Write-Host "MISSING: $group" -ForegroundColor Red
    }
}
```

### 4.2 Verify Permissions on Exchange Configuration Container

```powershell
# Get the configuration naming context
$configNC = (Get-ADRootDSE).configurationNamingContext

# Check Exchange organization container
$exchangeOrg = Get-ADObject -Filter 'objectClass -eq "msExchOrganizationContainer"' `
    -SearchBase $configNC -Properties ntSecurityDescriptor

# Export ACL for review
$acl = Get-ACL "AD:\$($exchangeOrg.DistinguishedName)"
$acl.Access | Where-Object {$_.IdentityReference -like "*Exchange*"} | 
    Format-Table IdentityReference, AccessControlType, ActiveDirectoryRights
```

### 4.3 Check Split Permissions Mode

```powershell
# Verify Exchange is not in AD split permissions mode (which restricts access)
Get-ExchangeServer | Select-Object Name, IsExchangeTrialEdition

# Check organization config
Get-OrganizationConfig | Select-Object Name, IsExchangeTrialEdition
```

---

## 5. IIS Authentication Configuration

### 5.1 Check ECP Virtual Directory Authentication

Run from the Exchange server:

```powershell
# Check ECP authentication settings via Exchange
Get-ECPVirtualDirectory | Format-List Identity, *Authentication*

# Expected output should show:
# WindowsAuthentication: True
# BasicAuthentication: True (optional)
# FormsAuthentication: True (for external access)
```

### 5.2 Verify IIS Authentication Manually

```powershell
# Import IIS module
Import-Module WebAdministration

# Check authentication on ECP
Get-WebConfigurationProperty -Filter /system.webServer/security/authentication/windowsAuthentication `
    -PSPath "IIS:\Sites\Default Web Site\ecp" -Name enabled

Get-WebConfigurationProperty -Filter /system.webServer/security/authentication/anonymousAuthentication `
    -PSPath "IIS:\Sites\Default Web Site\ecp" -Name enabled
```

**Expected Results:**
- Windows Authentication: **True**
- Anonymous Authentication: **False**

### 5.3 Fix IIS Authentication if Incorrect

```powershell
# Enable Windows Authentication
Set-WebConfigurationProperty -Filter /system.webServer/security/authentication/windowsAuthentication `
    -PSPath "IIS:\Sites\Default Web Site\ecp" -Name enabled -Value True

# Disable Anonymous Authentication
Set-WebConfigurationProperty -Filter /system.webServer/security/authentication/anonymousAuthentication `
    -PSPath "IIS:\Sites\Default Web Site\ecp" -Name enabled -Value False

# Restart IIS
iisreset /noforce
```

### 5.4 Check Application Pool Identity

```powershell
# Get ECP application pool
Get-WebApplication -Site "Default Web Site" -Name "ecp" | Select-Object applicationPool

# Check the app pool identity (should be MSExchangeECPAppPool)
Get-ItemProperty IIS:\AppPools\MSExchangeECPAppPool | Select-Object name, state, processModel

# Verify it's running
Get-WebAppPoolState -Name "MSExchangeECPAppPool"
```

### 5.5 Reset Application Pool if Needed

```powershell
# Recycle the ECP application pool
Restart-WebAppPool -Name "MSExchangeECPAppPool"

# If that fails, restart all Exchange app pools
Get-WebAppPoolState | Where-Object {$_.Value -eq "Started"} | 
    Where-Object {$_.ItemXPath -like "*Exchange*"} | 
    ForEach-Object { Restart-WebAppPool -Name ($_.ItemXPath -split "'")[1] }
```

---

## 6. Certificate and SSL/TLS Issues

### 6.1 Check Exchange Certificates

```powershell
# List all Exchange certificates
Get-ExchangeCertificate | Format-List Subject, Services, NotAfter, Thumbprint, Status

# Verify IIS certificate is assigned
Get-ExchangeCertificate | Where-Object {$_.Services -like "*IIS*"}
```

### 6.2 Verify Certificate is Valid

```powershell
# Get detailed certificate info
$cert = Get-ExchangeCertificate | Where-Object {$_.Services -like "*IIS*"}

# Check expiration
if ($cert.NotAfter -lt (Get-Date)) {
    Write-Host "CERTIFICATE EXPIRED!" -ForegroundColor Red
} else {
    Write-Host "Certificate valid until: $($cert.NotAfter)" -ForegroundColor Green
}

# Check if certificate is trusted (self-signed will show as not trusted)
$cert | Format-List Subject, Issuer, NotBefore, NotAfter, Status
```

### 6.3 Verify Certificate Binding in IIS

```powershell
# Check IIS HTTPS binding
Get-WebBinding -Name "Default Web Site" -Protocol https | Format-List

# Get certificate thumbprint bound to IIS
netsh http show sslcert ipport=0.0.0.0:443
```

### 6.4 Certificate Trust (Airgapped Consideration)

In airgapped environments, certificate trust chains can be problematic:

```powershell
# Check if the certificate's issuing CA is trusted
$cert = Get-ExchangeCertificate | Where-Object {$_.Services -like "*IIS*"}
$certPath = "Cert:\LocalMachine\My\$($cert.Thumbprint)"
$certObj = Get-Item $certPath

# Verify the chain
$chain = New-Object System.Security.Cryptography.X509Certificates.X509Chain
$chain.Build($certObj)
$chain.ChainStatus | Format-Table Status, StatusInformation
```

**If using internal CA:** Ensure the root CA certificate is in the Trusted Root Certification Authorities store on all client machines.

---

## 7. Virtual Directory Configuration

### 7.1 Check ECP Virtual Directory Settings

```powershell
# Get full ECP virtual directory configuration
Get-ECPVirtualDirectory | Format-List *

# Check internal and external URLs
Get-ECPVirtualDirectory | Select-Object Identity, InternalUrl, ExternalUrl
```

### 7.2 Verify Virtual Directory URLs Match Access Method

```powershell
# Compare how you're accessing ECP vs configured URLs
$ecp = Get-ECPVirtualDirectory
Write-Host "Internal URL: $($ecp.InternalUrl)"
Write-Host "External URL: $($ecp.ExternalUrl)"
Write-Host ""
Write-Host "You should access ECP using one of these URLs exactly"
```

### 7.3 Check OWA Virtual Directory (ECP Depends on OWA)

```powershell
# ECP requires OWA to function
Get-OWAVirtualDirectory | Format-List Identity, InternalUrl, ExternalUrl, *Authentication*
```

### 7.4 Reset Virtual Directories (If Corrupted)

**CAUTION:** Only do this if other troubleshooting fails.

```powershell
# Remove and recreate ECP virtual directory
# First, document current settings
Get-ECPVirtualDirectory | Export-Clixml C:\Temp\ECP_Backup.xml

# Remove (on specific server)
Remove-ECPVirtualDirectory -Identity "YOURSERVER\ecp (Default Web Site)" -Confirm:$false

# Recreate
New-ECPVirtualDirectory -Server YOURSERVER -InternalUrl "https://mail.domain.com/ecp" 

# Reset IIS
iisreset /noforce
```

---

## 8. DNS and Name Resolution

### 8.1 Verify DNS Resolution

From a client machine attempting to access ECP:

```powershell
# Resolve the Exchange server FQDN
Resolve-DnsName mail.yourdomain.com

# Resolve from the Exchange server itself
Resolve-DnsName $env:COMPUTERNAME
Resolve-DnsName "$env:COMPUTERNAME.yourdomain.com"

# Check reverse DNS
[System.Net.Dns]::GetHostEntry("EXCHANGE_SERVER_IP")
```

### 8.2 Check Hosts File (Airgapped Common Issue)

```powershell
# View hosts file
Get-Content C:\Windows\System32\drivers\etc\hosts

# Ensure no conflicting entries for Exchange server
```

### 8.3 Verify SPN Registration

```powershell
# Check HTTP SPNs for Exchange server
setspn -L YOURSERVER

# Expected SPNs should include:
# HTTP/mail.yourdomain.com
# HTTP/YOURSERVER
# HTTP/YOURSERVER.yourdomain.com
```

---

## 9. Kerberos and Authentication

### 9.1 Test Kerberos Authentication

```powershell
# Check current Kerberos tickets
klist

# Purge and re-acquire tickets
klist purge
```

### 9.2 Verify SPNs are Correct

```powershell
# Check for duplicate SPNs (common cause of Kerberos failures)
setspn -X

# List SPNs for the Exchange computer account
setspn -L YOUREXCHANGESERVER$

# Check for HTTP SPN
setspn -Q HTTP/mail.yourdomain.com
```

### 9.3 Fix Missing or Duplicate SPNs

```powershell
# Add missing HTTP SPN
setspn -S HTTP/mail.yourdomain.com YOUREXCHANGESERVER

# Remove duplicate (if found on wrong account)
setspn -D HTTP/mail.yourdomain.com WRONGACCOUNT
```

### 9.4 Check Kerberos Constrained Delegation

```powershell
# On DC, check if Exchange server has delegation configured
Get-ADComputer YOUREXCHANGESERVER -Properties TrustedForDelegation, msDS-AllowedToDelegateTo

# If using constrained delegation, verify HTTP service is listed
```

### 9.5 Test with NTLM (Bypass Kerberos)

To test if Kerberos is the issue:

1. Access ECP using IP address instead of hostname: `https://192.168.x.x/ecp`
2. If this works, the issue is Kerberos-related

---

## 10. Event Log Analysis

### 10.1 Check Exchange-Specific Logs

```powershell
# ECP-specific events
Get-WinEvent -LogName "MSExchange Management" -MaxEvents 50 | 
    Where-Object {$_.TimeCreated -gt (Get-Date).AddHours(-2)} |
    Format-Table TimeCreated, Id, Message -Wrap

# RBAC events
Get-WinEvent -FilterHashtable @{LogName='MSExchange Management'; ID=1} -MaxEvents 20 -ErrorAction SilentlyContinue
```

### 10.2 Check IIS Logs

```powershell
# Find IIS log location
$logPath = (Get-WebConfigurationProperty -Filter "system.applicationHost/sites/site[@name='Default Web Site']/logFile" -Name "directory").Value
$logPath = [System.Environment]::ExpandEnvironmentVariables($logPath)

# Get recent ECP entries (last modified log file)
$latestLog = Get-ChildItem "$logPath\W3SVC1" | Sort-Object LastWriteTime -Descending | Select-Object -First 1
Get-Content $latestLog.FullName -Tail 100 | Select-String "/ecp"
```

Look for HTTP status codes:
- **401:** Authentication failure
- **403:** Authorization failure (RBAC issue)
- **500:** Server error (check Application log)

### 10.3 Check Application Event Log

```powershell
# Application errors from Exchange
Get-WinEvent -LogName Application -MaxEvents 100 | 
    Where-Object {$_.ProviderName -like "*Exchange*" -and $_.LevelDisplayName -eq "Error"} |
    Format-Table TimeCreated, Id, Message -Wrap

# MSExchange Front End HTTP Proxy errors
Get-WinEvent -LogName Application -MaxEvents 50 | 
    Where-Object {$_.ProviderName -eq "MSExchange Front End HTTP Proxy"} |
    Format-Table TimeCreated, Message -Wrap
```

### 10.4 Check Security Event Log

```powershell
# Failed logon attempts
Get-WinEvent -FilterHashtable @{LogName='Security'; ID=4625} -MaxEvents 20 |
    Where-Object {$_.TimeCreated -gt (Get-Date).AddHours(-1)} |
    Format-Table TimeCreated, Message -Wrap
```

---

## 11. PrepareAD Re-application

If other troubleshooting hasn't resolved the issue, re-running PrepareAD can fix RBAC and AD permission issues.

### 11.1 Pre-PrepareAD Checklist

- [ ] Have Exchange installation media available locally
- [ ] Schema Admin and Enterprise Admin rights
- [ ] All Exchange services stopped on target server
- [ ] Backup of current configuration exported
- [ ] Document current CU level

### 11.2 Verify Current AD Preparation Level

```powershell
# Check schema version
$schema = [ADSI]"LDAP://CN=ms-Exch-Schema-Version-Pt,CN=Schema,CN=Configuration,DC=yourdomain,DC=com"
$schema.rangeUpper

# Check organization version  
$org = [ADSI]"LDAP://CN=YourOrgName,CN=Microsoft Exchange,CN=Services,CN=Configuration,DC=yourdomain,DC=com"
$org.objectVersion

# Check domain version
$domain = [ADSI]"LDAP://CN=Microsoft Exchange System Objects,DC=yourdomain,DC=com"
$domain.objectVersion
```

### 11.3 Run PrepareAD

From the Exchange installation media directory:

```powershell
# Navigate to Exchange setup
cd E:\  # Or wherever your media is mounted

# Run PrepareAD (choose appropriate license acceptance)
# For diagnostic data ON:
.\Setup.exe /PrepareAD /IAcceptExchangeServerLicenseTerms_DiagnosticDataON

# For diagnostic data OFF:
.\Setup.exe /PrepareAD /IAcceptExchangeServerLicenseTerms_DiagnosticDataOFF
```

### 11.4 Post-PrepareAD Steps

```powershell
# Wait for AD replication (or force it)
repadmin /syncall /AdeP

# Restart Exchange services
Get-Service *Exchange* | Restart-Service

# Reset IIS
iisreset /noforce

# Clear local admin's credential cache
klist purge

# Log off and log back on

# Test ECP access
```

---

## 12. Additional Airgapped Considerations

### 12.1 Time Synchronization

Kerberos is time-sensitive. In airgapped networks, time drift is common:

```powershell
# Check time offset from DC
w32tm /stripchart /computer:YOURDCNAME /samples:3

# If time is off by more than 5 minutes, Kerberos will fail
# Force time sync
w32tm /resync /force
```

### 12.2 Certificate Revocation List (CRL) Issues

Airgapped networks can't check CRLs online:

```powershell
# Check if CRL checking is causing issues
# In IE/Edge: Internet Options > Advanced > Security
# Uncheck "Check for server certificate revocation"

# Or via registry (per-machine):
reg query "HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Internet Settings" /v CertificateRevocation

# Disable CRL checking for testing (not recommended for production):
reg add "HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Internet Settings" /v CertificateRevocation /t REG_DWORD /d 0 /f
```

### 12.3 Windows Update and Patches

Even in airgapped environments, ensure Windows and Exchange are patched:

```powershell
# Check installed Exchange updates
Get-Command Exsetup.exe | ForEach-Object {$_.FileVersionInfo}

# Check Windows updates
Get-HotFix | Sort-Object InstalledOn -Descending | Select-Object -First 10
```

### 12.4 .NET Framework Version

```powershell
# Check .NET version
Get-ItemProperty "HKLM:\SOFTWARE\Microsoft\NET Framework Setup\NDP\v4\Full" | Select-Object Release, Version

# Exchange 2019 requires .NET 4.8 (release 528040+)
# Exchange 2016 CU23+ requires .NET 4.8
```

### 12.5 PowerShell Execution Policy

```powershell
# Check execution policy
Get-ExecutionPolicy -List

# If restricted, ECP may have issues with PowerShell remoting
Set-ExecutionPolicy RemoteSigned -Scope LocalMachine
```

### 12.6 WinRM Configuration

ECP uses PowerShell remoting:

```powershell
# Test WinRM
Test-WSMan -ComputerName localhost

# Check WinRM service
Get-Service WinRM

# Verify PowerShell remoting to Exchange
$session = New-PSSession -ConfigurationName Microsoft.Exchange -ConnectionUri http://YOURSERVER/PowerShell/ -Authentication Kerberos
$session
Remove-PSSession $session
```

---

## 13. Resolution Verification

After applying fixes, verify ECP access:

### 13.1 Clear All Caches

```powershell
# On the Exchange server
iisreset /noforce

# Clear browser cache on client (or use incognito/private mode)

# Clear Kerberos tickets on client
klist purge

# Log off and log back on to refresh group membership
```

### 13.2 Test Access

1. Open browser in private/incognito mode
2. Navigate to: `https://mail.yourdomain.com/ecp`
3. Enter credentials: `DOMAIN\YourUsername`
4. Verify you reach the ECP dashboard

### 13.3 Verify Functionality

Once in ECP, confirm you can:

- [ ] View recipients
- [ ] View server configuration
- [ ] Access mail flow settings
- [ ] View organization settings

### 13.4 Document Resolution

Record what fixed the issue:

| Item | Value |
|------|-------|
| Root Cause | |
| Fix Applied | |
| Date/Time | |
| Verified By | |

---

## Quick Reference: Command Summary

```powershell
# Group membership
Get-RoleGroupMember "Organization Management"

# RBAC roles
Get-ManagementRoleAssignment -RoleAssignee "Organization Management"

# ECP virtual directory
Get-ECPVirtualDirectory | Format-List *

# IIS authentication
Import-Module WebAdministration
Get-WebConfigurationProperty -Filter /system.webServer/security/authentication/windowsAuthentication -PSPath "IIS:\Sites\Default Web Site\ecp" -Name enabled

# Certificates
Get-ExchangeCertificate | Where-Object {$_.Services -like "*IIS*"}

# SPNs
setspn -L YOUREXCHANGESERVER

# PrepareAD
.\Setup.exe /PrepareAD /IAcceptExchangeServerLicenseTerms_DiagnosticDataON

# Restart services
Get-Service *Exchange* | Restart-Service; iisreset /noforce
```

---

## Appendix A: Common Error Messages and Solutions

| Error | Likely Cause | Solution |
|-------|--------------|----------|
| "You don't have permission to access this page" | RBAC misconfiguration | Section 3 |
| "403 - Forbidden" | IIS authentication or RBAC | Sections 3 and 5 |
| "401 - Unauthorized" | Kerberos or IIS auth | Sections 5 and 9 |
| Blank page after login | Missing mailbox on admin account | Section 2.2 |
| "Can't connect to server" | DNS or virtual directory | Sections 7 and 8 |
| Certificate warning | SSL certificate issue | Section 6 |
| Slow/timeout | WinRM or backend connectivity | Section 12.6 |

---

## Appendix B: Exchange 2016 vs 2019 Differences

| Item | Exchange 2016 | Exchange 2019 |
|------|---------------|---------------|
| Minimum .NET | 4.8 (CU23+) | 4.8 |
| Default ECP URL | /ecp | /ecp |
| RBAC behavior | Same | Same |
| PrepareAD switch | Same | Same |

---

*Runbook Version: 1.0*  
*Last Updated: January 2026*  
*For use in airgapped Exchange environments*
