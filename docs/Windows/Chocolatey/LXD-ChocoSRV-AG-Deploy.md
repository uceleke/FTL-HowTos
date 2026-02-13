# Chocolatey for Business (C4B) - Air-Gapped Deployment Guide with SMB Repository Storage

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────────┐
│                   Windows Server (CHOCOSERVER)                          │
│                                                                         │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────────┐  │
│  │  Nexus Repository │  │  Chocolatey CCM  │  │  Chocolatey Agent    │  │
│  │  (Port 8443)      │  │  (Port 8443/CCM) │  │  (Licensed)          │  │
│  │                   │  │                   │  │                      │  │
│  │  Blob Store ──────│──│───────────────────│──│──► Z:\ (SMB Share)   │  │
│  └──────────────────┘  └──────────────────┘  └──────────────────────┘  │
│                                                                         │
│  Service Accounts: svc.LDAP, svc.ChocoServ                             │
│  Certificate: Web Server Auth (FQDN in Subject)                        │
└───────────────────────────────────┬─────────────────────────────────────┘
                                    │ SMB (TCP 445)
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                          Buffalo NAS                                    │
│  ┌────────────────────────────────────────────────────────────────┐    │
│  │  SMB Share: \\<NAS-IP>\ChocoShare                              │    │
│  │    ├── nexus-repo/          (Nexus blob store - .nupkg files)  │    │
│  │    ├── staging/             (offline install files)             │    │
│  │    └── backup/              (configuration backups)             │    │
│  └────────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────────┘
                                    ▲
                                    │ SMB / Ansible
┌─────────────────────────────────────────────────────────────────────────┐
│                      Client Machines (Endpoints)                        │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐              │
│  │ Choco    │  │ Choco    │  │ Choco    │  │ Choco    │              │
│  │ Agent    │  │ Agent    │  │ Agent    │  │ Agent    │              │
│  │ Licensed │  │ Licensed │  │ Licensed │  │ Licensed │              │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘              │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Table of Contents

1. [Needed Files and Equipment](#1-needed-files-and-equipment)
2. [Buffalo NAS Configuration](#2-buffalo-nas-configuration)
3. [Preparation Before Install](#3-preparation-before-install)
4. [Install Chocolatey (C4B)](#4-install-chocolatey-c4b)
5. [Configure Nexus to Use SMB Share](#5-configure-nexus-to-use-smb-share)
6. [Nexus Setup](#6-nexus-setup)
7. [Chocolatey Central Management (CCM) Setup](#7-chocolatey-central-management-ccm-setup)
8. [Import Packages to Repository](#8-import-packages-to-repository)
9. [Client Setup](#9-client-setup)
10. [Auditing Chocolatey](#10-auditing-chocolatey)
11. [Maintenance](#11-maintenance)
12. [Troubleshooting](#12-troubleshooting)
13. [Quick Reference](#13-quick-reference)

---

## 1. Needed Files and Equipment

**Note:** File locations without a drive letter refer to the media that contains the install files (USB, DVD, or transfer share).

### Required Downloads (Stage on Internet-Connected Machine)

| Item | Source | Notes |
|------|--------|-------|
| Windows Server 2022 ISO | Microsoft VLSC / Eval Center | Standard or Datacenter |
| Choco-Setup Files | Chocolatey Licensed Portal | C4B quick-start environment bundle |
| JJJ.Chocolatey.Licensed | Internal package | Import to `C:\choco-setup\files\files` |
| Latest Choco Repos | `\\xxx.xxx.2.62\ChocoShare` (or staging media) | All current .nupkg files |
| .NET Framework 4.8 Offline | Microsoft Download Center | If not included in Windows image |

### Required Infrastructure

| Item | Purpose |
|------|---------|
| Buffalo NAS | SMB share for Nexus blob store (package repository storage) |
| Active Directory | Service accounts and LDAP authentication |
| HashiCorp Vault | Secrets management (`/ui/vault/secrets/systemcreds/Chocolatey`) |
| Web Server Certificate | Server Authentication cert with exportable key (FQDN in Subject) |
| Chocolatey GPOs | Firewall GPOs, User Rights GPOs |

### Transfer Method (Air-Gapped)

```bash
# On internet-connected machine: create transfer bundle
mkdir -p /transfer/chocolatey-staging
# Copy all downloaded files to this directory
# Transfer via approved method (USB, DVD, secure file transfer, etc.)
```

---

## 2. Buffalo NAS Configuration

### 2.1 Create SMB Share via NAS Web Interface

1. Access Buffalo NAS admin: `http://<NAS-IP>/`
2. Navigate to: **Shares** > **Folder Setup** > **Create Folder**
3. Create share: `ChocoShare`

### 2.2 Configure SMB Share Permissions

1. **Shares** > **ChocoShare** > **SMB Settings**
2. Enable SMB sharing
3. Create or assign a service account:
   - Username: `chocosvc` (or use AD account `svc.ChocoServ`)
   - Password: `<strong-password>` (store in Vault)
   - Permissions: **Read/Write**

### 2.3 Create Directory Structure on Share

From any Windows machine that can reach the NAS:

```powershell
# Map temporary drive to configure
net use T: \\<NAS-IP>\ChocoShare /user:chocosvc <PASSWORD>

# Create directory structure
New-Item -ItemType Directory -Path "T:\nexus-repo" -Force
New-Item -ItemType Directory -Path "T:\staging" -Force
New-Item -ItemType Directory -Path "T:\backup" -Force

# Verify
dir T:\

# Disconnect temporary drive
net use T: /delete
```

### 2.4 Record NAS Details

```
NAS IP Address:      ___________________
Share Name:          ChocoShare
SMB Path:            \\<NAS-IP>\ChocoShare
Service Account:     chocosvc (or svc.ChocoServ)
Password:            (stored in Vault at /ui/vault/secrets/systemcreds/Chocolatey)
```

---

## 3. Preparation Before Install

Complete all of these steps before running the C4B installer.

### 3.1 Windows Server Base Setup

1. Install Windows Server 2022 (Standard, Desktop Experience)
2. Set static IP, DNS, and join to domain
3. Rename computer:

```powershell
Rename-Computer -NewName "CHOCOSERVER" -Restart
```

### 3.2 Create Web Server Certificate

1. Open `certlm.msc` (Certificate Manager - Local Machine)
2. Create a **Web Server Authentication** certificate with an **exportable key**
3. Place it in the **Personal** store

   - A template policy configuration file can be found within the choco-setup folder
   - **The certificate must contain the machine FQDN in the Subject or the installation will error out**

4. Copy the **thumbprint** for the certificate you just created:

```powershell
# List certificates and find your new one
Get-ChildItem Cert:\LocalMachine\My | Format-Table Subject, Thumbprint, NotAfter
```

Save the thumbprint - you will need it for the install script.

### 3.3 Create Active Directory Service Accounts

1. **svc.LDAP** - Create in AD with **Domain Admin** privileges
   - Used for LDAP authentication in Nexus and CCM

2. **svc.ChocoServ** - Create in AD, add to **Domain Users** and **Local Admins** groups
   - Needs admin privileges on each machine where Chocolatey is installed
   - Will run the Chocolatey Central Management and Chocolatey Agent services

### 3.4 Configure Vault Secrets

Access Vault and create a secret named **Chocolatey** at:
`/ui/vault/secrets/systemcreds/Chocolatey`

Keys to store (populated during setup):

| Key | Value | When to set |
|-----|-------|-------------|
| `nexusadmin` | Nexus admin password | After Step 6.a |
| `nexusadminapi` | Nexus API key | After Step 6.b |
| `ccmadmin` | CCM admin password | After Step 7 |
| `sqlaccess` | CCM SQL encryption password | After Step 7.c |
| `chocosmb` | NAS SMB account password | During Step 2 |
| `salts` | ClientCommunicationSalt, ServiceCommunicationSalt | After Step 4.b |

### 3.5 Map SMB Share on Chocolatey Server

Map the Buffalo NAS share persistently as the service account:

```powershell
# Store credentials for the NAS
cmdkey /add:<NAS-IP> /user:chocosvc /pass:<PASSWORD>

# Map drive persistently
net use Z: \\<NAS-IP>\ChocoShare /user:chocosvc <PASSWORD> /persistent:yes

# Verify access
dir Z:\
dir Z:\nexus-repo
dir Z:\staging
```

**Important:** The mapped drive must be accessible by the account running Nexus. If Nexus runs as `LocalSystem`, the drive mapping must exist for that account. See [Section 5](#5-configure-nexus-to-use-smb-share) for configuring this properly.

### 3.6 Stage Installation Files

```powershell
# Copy choco-setup files to server
Copy-Item -Path "<media>:\choco-setup" -Destination "C:\choco-setup" -Recurse

# Copy licensed package into the correct location
Copy-Item -Path "<media>:\JJJ.Chocolatey.Licensed.nupkg" `
    -Destination "C:\choco-setup\files\files\"

# Copy latest repo packages to NAS staging area
Copy-Item -Path "<media>:\packages\*" -Destination "Z:\staging\" -Recurse
```

### 3.7 Configure Firewall

Ensure the following ports are open (apply via GPO or manually):

```powershell
# Nexus HTTPS (NuGet repository)
New-NetFirewallRule -DisplayName "Nexus HTTPS" `
    -Direction Inbound -Protocol TCP -LocalPort 8443 -Action Allow

# CCM HTTPS
New-NetFirewallRule -DisplayName "CCM HTTPS" `
    -Direction Inbound -Protocol TCP -LocalPort 443 -Action Allow

# SMB to NAS (outbound - usually allowed by default)
New-NetFirewallRule -DisplayName "SMB to NAS" `
    -Direction Outbound -Protocol TCP -LocalPort 445 -Action Allow

# Verify
Get-NetFirewallRule -DisplayName "Nexus*","CCM*","SMB*" |
    Format-Table Name, Enabled, Direction, Action
```

---

## 4. Install Chocolatey (C4B)

### 4.1 Run C4B Setup Script

```powershell
# Open an elevated PowerShell window (NOT ISE!)
# Navigate to the setup directory
cd C:\choco-setup\files

# Set execution policy for this session
Set-ExecutionPolicy Bypass -Scope Process -Force

# Run the C4B setup script
# Replace <THUMBPRINT> with the certificate thumbprint from Step 3.2
& .\Start-C4bSetup.ps1 -Thumbprint <THUMBPRINT> -Unattend -Verbose
```

### 4.2 Capture Output and Salt Values

The setup script produces two important files in `C:\choco-setup\files\scripts`:
- `Register-Endpoint.ps1`
- `ClientSetup.ps1`

**Copy the salt values:**

1. Open `Register-Endpoint.ps1` and copy the values for:
   - **FQDN**
   - **ClientCommunicationSalt**
   - **ServiceCommunicationSalt**

2. Paste these values into the existing `Register-C4bEndpoint.ps1` script at:
   `C:\choco-setup\files\scripts\JJJUtilityScripts\Register-C4bEndpoint.ps1`

3. Store the salt passwords in Vault under the `salts` key

> **Note:** The salt passwords can also be found in the ReadMe file produced by the script.

### 4.3 Verify Chocolatey Installation

```powershell
# Refresh environment
$env:Path = [System.Environment]::GetEnvironmentVariable("Path","Machine") + ";" + [System.Environment]::GetEnvironmentVariable("Path","User")

# Verify
choco --version

# Should show no public sources
choco source list
```

---

## 5. Configure Nexus to Use SMB Share

This is the key section that redirects Nexus package storage from local disk to the Buffalo NAS SMB share.

### 5.1 Ensure SMB Access for the Nexus Service Account

The Nexus service needs to access the mapped SMB share. By default, C4B installs Nexus as a Windows service. You need to configure the service to run under an account that can access the share.

**Option A: Run Nexus service as svc.ChocoServ (Recommended)**

```powershell
# Stop the Nexus service
Stop-Service nexus

# Change the service logon account
$cred = Get-Credential -UserName "DOMAIN\svc.ChocoServ" -Message "Enter svc.ChocoServ password"
$service = Get-WmiObject Win32_Service -Filter "Name='nexus'"
$service.Change($null, $null, $null, $null, $null, $null, $cred.UserName, $cred.GetNetworkCredential().Password)

# Grant svc.ChocoServ full control of the Nexus data directory
$nexusData = "C:\ProgramData\sonatype-work\nexus3"
icacls $nexusData /grant "DOMAIN\svc.ChocoServ:(OI)(CI)F" /T

# Map the SMB share for svc.ChocoServ
# Log in as svc.ChocoServ or use PsExec to run as that account:
# Note: Persistent mapped drives are per-user. For a service, use UNC paths
# or map the drive in a startup script.
```

**Option B: Use UNC path directly (Simpler, no drive mapping needed)**

Instead of mapping `Z:\`, configure Nexus blob store with a UNC path:
`\\<NAS-IP>\ChocoShare\nexus-repo`

This avoids reliance on mapped drive letters, which can be unreliable for services.

### 5.2 Create a Nexus Blob Store on the SMB Share

1. Open Nexus in a browser: `https://<ChocoServerFQDN>:8443/`
2. Log in as admin (see [Section 6.a](#6-nexus-setup) for credentials)
3. Navigate to: **Server Administration** (gear icon) > **Blob Stores**
4. Click **Create Blob Store**
   - **Type:** File
   - **Name:** `smb-packages`
   - **Path:** `\\<NAS-IP>\ChocoShare\nexus-repo`
     (or `Z:\nexus-repo` if using a mapped drive with proper service account)
5. Click **Create**

### 5.3 Update Repositories to Use SMB Blob Store

For each hosted repository that should store packages on the NAS:

1. Go to **Server Administration** > **Repositories**
2. Select the repository (e.g., `choco-install`, `ChocolateyInternal`)
3. Under **Storage**, change **Blob Store** to `smb-packages`
4. Click **Save**

> **Note:** Existing packages in the old blob store will NOT be automatically migrated. You will need to re-upload them or manually move the blob data. It is easier to do this before importing packages.

### 5.4 Verify SMB Blob Store is Working

```powershell
# Check that Nexus created its blob structure on the NAS
dir \\<NAS-IP>\ChocoShare\nexus-repo

# You should see Nexus blob store metadata files/directories
# After pushing a package, .nupkg data will be stored here
```

---

## 6. Nexus Setup

After the C4B setup script finishes, you will be presented with a webpage for Nexus and CCM credentials.

### 6.a Sign In to Nexus

1. Open a browser to `https://<ChocoServerFQDN>:8443/`
2. Click **Sign In** with default credentials:

   ```
   Username: admin
   Password: <found at C:\ProgramData\sonatype-work\nexus3\admin.password>
   ```

3. Set a new admin password. **Store the new password in Vault** under key `nexusadmin`.

### 6.b Copy NuGet API Key

1. Click **admin** (top right) > **NuGet API Key** > **Access API Key**
2. Copy the API key
3. **Upload to Vault** under key `nexusadminapi`

### 6.c Enable Anonymous Access

1. Go to **Server Administration** (gear icon) > **Anonymous Access**
2. Ensure anonymous access is **enabled**

### 6.d Upload Utility Scripts to Nexus

1. Go to the Nexus main page, click the **Upload** tab
2. Select the **choco-install** repository
3. Upload:
   - `ChocolateyInstall.ps1` from `C:\choco-setup\files\scripts`
   - `ClientSetup.ps1` from `C:\choco-setup\files\scripts\JJJUtilityScripts`

### 6.e Configure LDAP Authentication

1. Go to **Server Administration** > **Security** > **LDAP**
2. Configure LDAP connection:
   - **Name:** AD LDAP
   - **Protocol:** LDAPS (or LDAP)
   - **Hostname:** `<Domain Controller FQDN>`
   - **Port:** 636 (LDAPS) or 389 (LDAP)
   - **Search Base:** Your domain's base DN (e.g., `DC=domain,DC=local`)
   - **Authentication:** Simple Authentication
   - **Username:** `svc.LDAP@domain.local`
   - **Password:** svc.LDAP password
3. Configure user/group mapping per your AD structure
4. **Verify Connection** before saving

### 6.f Configure LDAP Role

1. Go to **Security** > **Roles**
2. Create an LDAP role named **Domain Admins**
3. Connect it to the AD **Domain Admins** group
4. Assign the **NX Admin** privilege to this role

### 6.g Configure Anonymous User Role

1. Go to **Security** > **Users**
2. Select the **anonymous** user
3. Assign the **JJJ-anonymous** role
4. Remove all other roles
5. Save

---

## 7. Chocolatey Central Management (CCM) Setup

### 7.a Sign In to CCM

1. Open a browser to `https://<ChocoServerFQDN>:8443/` (CCM port)
2. Log in with default credentials:

   ```
   Username: ccmadmin
   Password: 123qwe
   ```

3. Set a new admin password. **Store in Vault** under key `ccmadmin`.

### 7.b SQL Encryption

On the Security page, set a strong encryption password for SQL Server computer record management.

**Store in Vault** under key `sqlaccess`.

### 7.c Enable LDAP Authentication in CCM

1. Click **Administration** > **User Management**
2. Enable **LDAP Authentication**
3. Enter:
   - **Domain name:** `<Domain FQDN>` (e.g., `domain.local`)
   - **User name:** svc.LDAP account from Step 3.3

4. Click **Update LDAP Password** and enter the svc.LDAP password

> **Note:** For admins to use LDAP authentication, the `mail` attribute must be configured in Active Directory for each admin account. The address does not need to be valid, but it must be unique per user.

### 7.d Configure CCM Settings

| Section | Setting | Value |
|---------|---------|-------|
| Dashboard | Notified of Computers not reporting in | 5 |
| Security | Maximum number of failed login | 3 |
| Security | Account locking duration | 0 |
| Security | Disable concurrent login for a user | Checked |
| Retention Policies | Enable Audit Retention | Checked |
| Retention Policies | Days to keep audit logs | 31 |

Click **Save All**.

### 7.e Configure Service Accounts for CCM and Agent

1. Open `services.msc` on the Chocolatey Server
2. Find **Chocolatey Central Management** service
3. Change **Log On** to `svc.ChocoServ`
4. Find **Chocolatey Agent** service
5. Change **Log On** to `svc.ChocoServ`
6. Restart both services

---

## 8. Import Packages to Repository

### 8.1 Configure the Import Script

1. Open `Import-ChocoPackages.ps1` (located in choco-setup or JJJUtilityScripts)
2. Set the `$APIKey` variable to the API key copied in Step 6.b
3. Verify all file paths within the script are correct
4. Save and exit

### 8.2 Run the Import

```powershell
# Run from an elevated PowerShell on the Chocolatey Server only
# (to preserve security around the API key)
& .\Import-ChocoPackages.ps1
```

### 8.3 Verify Package Upload

```powershell
# Search for all packages
choco search -a

# Or verify via Nexus web UI: browse the repository

# Verify packages are stored on the NAS
dir \\<NAS-IP>\ChocoShare\nexus-repo
```

> **Note:** Only run package imports from the Chocolatey/Nexus server to preserve security around the API key.

---

## 9. Client Setup

### 9.1 Manual Client Registration

On each client machine, execute:

```powershell
C:\choco-setup\files\JJJUtilityScripts\Register-C4BEndpoints.ps1 `
    -fqdn <ChocoServerFQDN> `
    -RepositoryCredential (Get-Credential) `
    -AgentCredential (Get-Credential -UserName "svc.ChocoServ@$($env:USERDNSDOMAIN)")
```

### 9.2 Ansible Mass Deployment (Recommended)

An Ansible playbook is included for mass deployments. It uses Vault to pass credentials for the Chocolatey agents and Nexus repositories.

**Prerequisites:**

1. Ensure credentials are stored in Vault at:
   `/ui/vault/secrets/systemcreds/Chocolatey`

2. Copy the following files to the Ansible server:
   - `win-choco-install.yml`
   - `Register-C4BEndpoints.ps1`

3. Place the registration script at:
   `/etc/ansible/playbooks/windows/psScripts/Register-C4bEndpoint.ps1`

4. Run the playbook:

```bash
ansible-playbook win-choco-install.yml -i <inventory>
```

### 9.3 Verify Client Registration

On the client machine:

```powershell
# Verify sources
choco source list

# Search for packages
choco search * --source="<internal-source-name>"

# Install a test package
choco install <package-name> -y
```

In CCM, verify the client appears under **Computers**.

---

## 10. Auditing Chocolatey

These commands supplement standard auditing practices. Chocolatey does not circumvent event logs for traditional packages (.exe, .msi, etc.). However, Chocolatey-specific actions (e.g., `choco config`, `choco sources`) can only be viewed from the local Chocolatey log. CCM does not collect these logs.

| Requirement | Command | Scope |
|-------------|---------|-------|
| Software Installation History | `choco list -lo --audit` | Client Machine |
| Configuration Changes | `Get-Content C:\ProgramData\chocolatey\logs\chocolatey.log -Tail 100 -Wait` | Client Machine |
| CCM Computer Reports | CCM Web UI > Computers | Server (CCM) |
| Nexus Repository Activity | Nexus Web UI > System > Tasks/Logs | Server (Nexus) |
| NAS Storage Usage | `Get-ChildItem \\<NAS-IP>\ChocoShare\nexus-repo -Recurse | Measure-Object -Property Length -Sum` | Any machine with share access |

---

## 11. Maintenance

### 11.1 Service Management

```powershell
# Restart Nexus
Restart-Service nexus

# Restart CCM
Restart-Service chocolatey-central-management

# Restart Chocolatey Agent
Restart-Service chocolatey-agent

# Check all Chocolatey-related services
Get-Service nexus, chocolatey-central-management, chocolatey-agent |
    Format-Table Name, Status, StartType
```

### 11.2 Package Management

```powershell
# List packages stored on NAS (blob store level)
Get-ChildItem "\\<NAS-IP>\ChocoShare\nexus-repo" -Recurse -Filter "*.nupkg" |
    Select-Object Name, Length, LastWriteTime

# Push a new package
choco push <package>.nupkg `
    --source="https://<ChocoServerFQDN>:8443/repository/ChocolateyInternal/" `
    --api-key="<API-KEY>"

# Verify via choco search
choco search -a
```

### 11.3 Backup

```powershell
# Backup Nexus configuration
$backupDate = Get-Date -Format 'yyyyMMdd'

# Nexus config directory
Copy-Item "C:\ProgramData\sonatype-work\nexus3\etc" `
    "\\<NAS-IP>\ChocoShare\backup\nexus-etc-$backupDate" -Recurse

# Chocolatey configuration
Copy-Item "C:\ProgramData\chocolatey\config" `
    "\\<NAS-IP>\ChocoShare\backup\choco-config-$backupDate" -Recurse

# C4B setup scripts (preserve salt values and configs)
Copy-Item "C:\choco-setup" `
    "\\<NAS-IP>\ChocoShare\backup\choco-setup-$backupDate" -Recurse

# CCM database backup (SQL Express)
# Use SQL Server Management Studio or:
sqlcmd -S localhost\YOURINSTANCE -Q "BACKUP DATABASE ChocolateyManagement TO DISK='\\<NAS-IP>\ChocoShare\backup\ccm-db-$backupDate.bak'"
```

### 11.4 Monitor Storage

```powershell
# Check NAS share space
Get-ChildItem "\\<NAS-IP>\ChocoShare" -Recurse |
    Measure-Object -Property Length -Sum |
    Select-Object @{N='TotalSizeMB';E={[math]::Round($_.Sum/1MB,2)}}

# Check local disk
Get-PSDrive C | Select-Object Used, Free

# Check mapped drive (if using Z:)
Get-PSDrive Z | Select-Object Used, Free
```

### 11.5 SMB Share Health Check

```powershell
# Verify SMB connectivity to NAS
Test-NetConnection -ComputerName <NAS-IP> -Port 445

# Verify share is accessible
Test-Path "\\<NAS-IP>\ChocoShare\nexus-repo"

# Check stored credentials
cmdkey /list | Select-String "<NAS-IP>"

# Re-map drive if lost after reboot
net use Z: \\<NAS-IP>\ChocoShare /user:chocosvc <PASSWORD> /persistent:yes
```

---

## 12. Troubleshooting

### Issue: SMB Share Not Accessible from Chocolatey Server

```powershell
# Test basic connectivity
Test-NetConnection -ComputerName <NAS-IP> -Port 445

# Check stored credentials
cmdkey /list

# Re-add credentials
cmdkey /delete:<NAS-IP>
cmdkey /add:<NAS-IP> /user:chocosvc /pass:<PASSWORD>

# Re-map drive
net use Z: /delete
net use Z: \\<NAS-IP>\ChocoShare /user:chocosvc <PASSWORD> /persistent:yes

# Test write access
New-Item -ItemType File -Path "\\<NAS-IP>\ChocoShare\nexus-repo\test.txt"
Remove-Item "\\<NAS-IP>\ChocoShare\nexus-repo\test.txt"
```

### Issue: Nexus Cannot Write to SMB Blob Store

```powershell
# Check which account the Nexus service runs as
Get-WmiObject Win32_Service -Filter "Name='nexus'" |
    Select-Object Name, StartName, State

# If running as LocalSystem, mapped drives won't work - use UNC paths instead

# Verify the service account has write access to the share
# Run as the Nexus service account:
runas /user:DOMAIN\svc.ChocoServ "cmd /c dir \\<NAS-IP>\ChocoShare\nexus-repo"

# Check Nexus logs for blob store errors
Get-Content "C:\ProgramData\sonatype-work\nexus3\log\nexus.log" -Tail 100
```

### Issue: Nexus Returns 500 Error

```powershell
# Check Nexus service status
Get-Service nexus

# Review Nexus logs
Get-Content "C:\ProgramData\sonatype-work\nexus3\log\nexus.log" -Tail 50

# Check Windows Event Viewer
Get-EventLog -LogName Application -Newest 20 -Source "*nexus*","*sonatype*"

# Restart Nexus
Restart-Service nexus
```

### Issue: Package Push Fails

```powershell
# Verify API key
# Compare with what's stored in Vault under nexusadminapi

# Test Nexus endpoint directly
Invoke-WebRequest -Uri "https://<ChocoServerFQDN>:8443/service/rest/v1/status" `
    -UseBasicParsing -SkipCertificateCheck

# Check write permissions on NAS for the Nexus service account
Test-Path "\\<NAS-IP>\ChocoShare\nexus-repo"

# Check Nexus blob store status in web UI:
# Server Administration > Blob Stores > smb-packages
# Verify it shows "Started" state
```

### Issue: CCM Not Showing Clients

```powershell
# On the client, check agent service
Get-Service chocolatey-agent

# Check agent log
Get-Content "C:\ProgramData\chocolatey\logs\chocolatey-agent.log" -Tail 50

# Verify client can reach CCM
Test-NetConnection -ComputerName <ChocoServerFQDN> -Port 24020

# Verify salt values match between server and client
# Compare values in Register-C4bEndpoint.ps1 with server config
```

### Issue: Mapped Drive Lost After Reboot

Services running under `LocalSystem` cannot access per-user mapped drives. Use one of these approaches:

**Option A: Use UNC paths (Recommended)**
Configure Nexus blob store with `\\<NAS-IP>\ChocoShare\nexus-repo` instead of `Z:\nexus-repo`.

**Option B: Startup script to re-map drive**
```powershell
# Create a scheduled task to map the drive at system startup
$action = New-ScheduledTaskAction -Execute "net.exe" `
    -Argument "use Z: \\<NAS-IP>\ChocoShare /user:chocosvc <PASSWORD> /persistent:yes"
$trigger = New-ScheduledTaskTrigger -AtStartup
$principal = New-ScheduledTaskPrincipal -UserId "SYSTEM" -LogonType ServiceAccount
Register-ScheduledTask -TaskName "Map Choco NAS Drive" `
    -Action $action -Trigger $trigger -Principal $principal
```

**Option C: Run Nexus under a domain service account**
Change the Nexus service logon to `svc.ChocoServ` and log in as that account to create the persistent mapping (see [Section 5.1](#51-ensure-smb-access-for-the-nexus-service-account)).

---

## 13. Quick Reference

```
┌─────────────────────────────────────────────────────────────────┐
│                    QUICK REFERENCE                               │
├─────────────────────────────────────────────────────────────────┤
│ Server FQDN:      _____________________                         │
│ Nexus URL:        https://<FQDN>:8443/                          │
│ CCM URL:          https://<FQDN>:8443/ (CCM)                    │
│ API Key:          (see Vault: nexusadminapi)                     │
│ NAS IP:           _____________________                         │
│ NAS Share:        \\<NAS-IP>\ChocoShare                          │
│ Blob Store Path:  \\<NAS-IP>\ChocoShare\nexus-repo               │
├─────────────────────────────────────────────────────────────────┤
│ Service Accounts:                                                │
│   svc.LDAP         - LDAP auth for Nexus and CCM                │
│   svc.ChocoServ    - Nexus, CCM, Agent services                 │
│   chocosvc         - NAS SMB access (if separate from above)    │
├─────────────────────────────────────────────────────────────────┤
│ Vault Path:  /ui/vault/secrets/systemcreds/Chocolatey            │
│   nexusadmin       - Nexus admin password                        │
│   nexusadminapi    - Nexus NuGet API key                         │
│   ccmadmin         - CCM admin password                          │
│   sqlaccess        - CCM SQL encryption password                 │
│   salts            - Communication salt values                   │
│   chocosmb         - NAS SMB password                            │
├─────────────────────────────────────────────────────────────────┤
│ Key Services (Windows):                                          │
│   Restart-Service nexus                                          │
│   Restart-Service chocolatey-central-management                  │
│   Restart-Service chocolatey-agent                               │
├─────────────────────────────────────────────────────────────────┤
│ Client Deployment:                                               │
│   Manual:   Register-C4BEndpoints.ps1 -fqdn <FQDN>              │
│   Ansible:  ansible-playbook win-choco-install.yml               │
│   Script:   /etc/ansible/playbooks/windows/psScripts/            │
│             Register-C4bEndpoint.ps1                             │
├─────────────────────────────────────────────────────────────────┤
│ Test Commands:                                                   │
│   choco search -a                                                │
│   choco push pkg.nupkg --source="https://<FQDN>:8443/..." `     │
│     --api-key="<KEY>"                                            │
│   Test-NetConnection <NAS-IP> -Port 445                          │
│   Test-Path "\\<NAS-IP>\ChocoShare\nexus-repo"                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Document Info

| | |
|---|---|
| Version | 2.0 |
| Last Updated | February 2026 |
| Environment | Air-Gapped / Offline Deployment |
| Stack | Chocolatey for Business (C4B), Nexus, CCM |
| Storage | Buffalo NAS via SMB |
| Based On | ChocoDeployment.md (infrastructure), Choco-Deploy-JJJ.md (operational procedures) |
