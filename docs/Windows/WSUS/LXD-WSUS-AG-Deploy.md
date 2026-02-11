# WSUS Server on LXD (Windows VM) - Air-Gapped Deployment Guide

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        LXD Host (Ubuntu)                                 │
│                                                                          │
│  ┌────────────────────────────────────────────────────────────────┐     │
│  │              Windows Server VM (LXD)                            │     │
│  │                                                                 │     │
│  │  ┌─────────────────────┐    ┌────────────────────────────┐    │     │
│  │  │       WSUS          │    │  W:\ (Mapped NAS Share)    │    │     │
│  │  │  (Windows Server    │───▶│  └── WSUSContent           │    │     │
│  │  │   Update Services)  │    │      ├── WsusContent/      │    │     │
│  │  │  Port: 8530/8531    │    │      ├── UpdateServicesDBFiles/│    │     │
│  │  └─────────────────────┘    │      └── logs/             │    │     │
│  │                             └────────────────────────────────┘    │     │
│  └────────────────────────────────────────────────────────────────┘     │
│                                    │                                     │
│  /mnt/nas/wsus ◄───────────────────┘ (passed through to VM)             │
│         │                                                                │
└─────────│────────────────────────────────────────────────────────────────┘
          │ NFS/SMB Mount
          ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                          Buffalo NAS                                     │
│  ┌────────────────────────────────────────────────────────────────┐     │
│  │  Share: /wsus-content                                           │     │
│  │    ├── WsusContent/      (Update files - LARGE)                 │     │
│  │    ├── UpdateServicesDBFiles/ (WSUS database - optional)        │     │
│  │    ├── logs/             (WSUS logs)                            │     │
│  │    └── backup/           (Configuration backups)                │     │
│  └────────────────────────────────────────────────────────────────┘     │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Storage Sizing Recommendations

### WSUS Storage Requirements

| Scenario | Products | Languages | Recommended Size | Notes |
|----------|----------|-----------|------------------|-------|
| **Minimal** | Windows 10/11, Server 2019/2022 only | 1 language | **150-250 GB** | Critical + Security updates only |
| **Standard** | Windows 10/11, Server 2016-2022, Office | 1 language | **350-500 GB** | All classifications |
| **Full** | All Windows, Office, SQL, Exchange | 1 language | **500-800 GB** | All classifications |
| **Enterprise** | All products | Multiple languages | **1-2+ TB** | Full catalog |

**Key factors affecting size:**
- Each additional language adds ~30-50% more storage
- Feature updates (Windows 10/11 upgrades) are 3-5 GB each
- Declining old updates can recover 20-40% space
- Express installation files double storage but reduce client bandwidth

### Chocolatey Storage Requirements

| Scenario | Recommended Size | Notes |
|----------|------------------|-------|
| **Small** (< 50 packages) | **10-20 GB** | Basic tools and utilities |
| **Medium** (50-200 packages) | **50-100 GB** | Department-level deployment |
| **Large** (200+ packages) | **150-300 GB** | Enterprise with custom packages |

### Combined NAS Allocation Recommendation

```
┌─────────────────────────────────────────────────────────────┐
│              RECOMMENDED NAS STORAGE ALLOCATION              │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  WSUS Content:                                               │
│    Minimum:     250 GB                                       │
│    Recommended: 500 GB                                       │
│    Comfortable: 750 GB - 1 TB                                │
│                                                              │
│  Chocolatey Packages:                                        │
│    Minimum:     20 GB                                        │
│    Recommended: 50 GB                                        │
│    Comfortable: 100 GB                                       │
│                                                              │
│  ─────────────────────────────────────────────────────────── │
│  TOTAL RECOMMENDED: 500 GB - 1 TB                            │
│  COMFORTABLE TOTAL: 1 - 1.5 TB                               │
│                                                              │
│  + 20% headroom for growth = Final recommendation            │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Pre-Deployment: Items to Stage (Air-Gap Preparation)

Download these on an internet-connected machine and transfer to your air-gapped network:

### Required Downloads

| Item | Source | Notes |
|------|--------|-------|
| Windows Server 2022 ISO | Microsoft VLSC / Eval Center | Standard or Datacenter |
| VirtIO Drivers ISO | `https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/stable-virtio/virtio-win.iso` | Required for LXD/KVM |
| WSUS Offline Update | `https://github.com/aker-gateway/wsusoffline` or `https://download.wsusoffline.net/` | For importing updates |
| Windows Updates (CAB/MSU) | Microsoft Update Catalog | Updates to import |

### WSUS Offline Update Tool

For air-gapped WSUS, you'll need to download updates externally and import them. Options:

1. **WSUS Offline Update** - Downloads updates to folder for offline import
2. **Microsoft Update Catalog** - Manual download of specific updates
3. **Export/Import from connected WSUS** - If you have one elsewhere

### Transfer Method
```bash
# Example: Create transfer archive
mkdir -p /transfer/wsus-staging
# Copy all downloaded files to this directory
# This will be LARGE - use appropriate media (external HDD, etc.)

# Verify space before transfer
du -sh /transfer/wsus-staging/
```

---

## Part 1: Buffalo NAS Configuration

### 1.1 Create WSUS Share via NAS Web Interface

1. Access Buffalo NAS admin: `http://<NAS-IP>/`
2. Navigate to: **Shares** → **Folder Setup** → **Create Folder**
3. Create share: `wsus-content`
4. **IMPORTANT**: Ensure share has enough space allocated (see sizing above)

### 1.2 Configure SMB Share

1. **Shares** → **wsus-content** → **SMB Settings**
2. Enable SMB sharing
3. Create/use service account:
   - Username: `wsussvc`
   - Password: `<strong-password>`
   - Permissions: **Full Control** (WSUS needs full permissions)

### 1.3 Performance Optimization for NAS

For WSUS, I/O performance matters. On Buffalo NAS:

1. **Storage** → **RAID** - Verify RAID configuration (RAID 5/6 recommended)
2. **Network** → Enable Jumbo Frames if supported (MTU 9000)
3. **Services** → Disable unnecessary services to free resources

### 1.4 Note Your NAS Details

```
NAS IP Address:      ___________________
WSUS Share Name:     wsus-content
SMB Username:        wsussvc  
SMB Password:        ___________________
Allocated Space:     ___________________ GB
```

---

## Part 2: LXD Host Preparation (Ubuntu)

### 2.1 Verify LXD Installation

```bash
# Check LXD status
lxc version

# If not installed
sudo snap install lxd
sudo lxd init
```

### 2.2 Create NAS Mount Point for WSUS

```bash
# Create mount directory
sudo mkdir -p /mnt/nas/wsus-content

# Set permissions
sudo chmod 755 /mnt/nas/wsus-content
```

### 2.3 Mount Buffalo NAS for WSUS

#### SMB/CIFS Mount (Recommended for Windows VM)
```bash
# Create credentials file
sudo tee /etc/nas-creds-wsus << 'EOF'
username=wsussvc
password=<YOUR-PASSWORD>
domain=WORKGROUP
EOF

sudo chmod 600 /etc/nas-creds-wsus

# Test mount (temporary)
sudo mount -t cifs //<NAS-IP>/wsus-content /mnt/nas/wsus-content \
    -o credentials=/etc/nas-creds-wsus,uid=0,gid=0,file_mode=0777,dir_mode=0777

# Verify mount
df -h /mnt/nas/wsus-content

# Make permanent - add to /etc/fstab
echo "//<NAS-IP>/wsus-content /mnt/nas/wsus-content cifs credentials=/etc/nas-creds-wsus,uid=0,gid=0,file_mode=0777,dir_mode=0777,_netdev,nofail 0 0" | sudo tee -a /etc/fstab
```

### 2.4 Create Directory Structure on NAS

```bash
sudo mkdir -p /mnt/nas/wsus-content/{WsusContent,UpdateServicesDBFiles,logs,backup,staging}
sudo chmod -R 777 /mnt/nas/wsus-content

# Verify structure
ls -la /mnt/nas/wsus-content/
```

### 2.5 Verify Mount Persists

```bash
# Test fstab
sudo mount -a

# Verify
mount | grep wsus
df -h /mnt/nas/wsus-content
```

---

## Part 3: Create Windows Server VM for WSUS

### 3.1 Create VM Instance

```bash
# Create empty Windows VM for WSUS
lxc init win2022-wsus --empty --vm

# Configure VM resources (WSUS needs more resources than Chocolatey)
lxc config set win2022-wsus limits.cpu 4
lxc config set win2022-wsus limits.memory 16GB

# Set secure boot off
lxc config set win2022-wsus security.secureboot false
```

### 3.2 Add Storage Devices

```bash
# Expand root disk (Windows + WSUS application needs space)
lxc config device override win2022-wsus root size=100GB

# Attach Windows ISO for installation
lxc config device add win2022-wsus iso disk \
    source=/mnt/nas/chocolatey-repo/staging/windows-server-2022.iso \
    boot.priority=10

# Attach VirtIO drivers ISO
lxc config device add win2022-wsus virtio disk \
    source=/mnt/nas/chocolatey-repo/staging/virtio-win.iso
```

### 3.3 Add NAS Share as Disk Device

```bash
# Pass through WSUS NAS mount to VM
lxc config device add win2022-wsus wsus-nas disk \
    source=/mnt/nas/wsus-content \
    path=/wsus-nas
```

### 3.4 Configure Network

```bash
# Verify network device exists
lxc config show win2022-wsus | grep -A5 "eth0"

# If needed, add network
lxc config device add win2022-wsus eth0 nic \
    nictype=bridged \
    parent=lxdbr0
```

### 3.5 Start VM

```bash
# Start the VM
lxc start win2022-wsus

# Access console for Windows installation
lxc console win2022-wsus --type=vga
```

---

## Part 4: Windows Server Installation

### 4.1 Install Windows Server

1. Boot from ISO, press any key
2. Select language → **Next** → **Install now**
3. Select: **Windows Server 2022 Standard (Desktop Experience)**
4. Accept license → **Next**
5. Select: **Custom: Install Windows only**
6. **Load driver** → Browse VirtIO CD → `viostor\2k22\amd64`
7. Select disk → **Next**
8. Wait for installation

### 4.2 Initial Configuration

After first boot:

```powershell
# Set Administrator password when prompted

# Install VirtIO Guest Tools (from VirtIO ISO, usually D: or E:)
D:\virtio-win-guest-tools.exe /S

# Rename computer
Rename-Computer -NewName "WSUSSERVER" -Restart
```

### 4.3 Configure Static IP

```powershell
# Get adapter name
Get-NetAdapter

# Set static IP (adjust for your network)
New-NetIPAddress -InterfaceAlias "Ethernet" `
    -IPAddress "10.10.10.101" `
    -PrefixLength 24 `
    -DefaultGateway "10.10.10.1"

# Set DNS (your internal DNS or domain controller)
Set-DnsClientServerAddress -InterfaceAlias "Ethernet" `
    -ServerAddresses "10.10.10.10","10.10.10.11"

# Verify
Get-NetIPConfiguration
```

---

## Part 5: Map NAS Share in Windows

### 5.1 Map WSUS Network Drive

```powershell
# Store credentials permanently
cmdkey /add:<NAS-IP> /user:wsussvc /pass:<PASSWORD>

# Map drive W: for WSUS content
net use W: \\<NAS-IP>\wsus-content /user:wsussvc <PASSWORD> /persistent:yes

# Verify
Get-PSDrive W
dir W:\
```

### 5.2 Alternative: Use Disk Device from LXD

If the LXD disk device appears:
```powershell
# Check for new drives
Get-PSDrive -PSProvider FileSystem

# May appear as different drive letter - note it
```

### 5.3 Create WSUS Directories

```powershell
# Create WSUS content directory structure
New-Item -ItemType Directory -Path "W:\WsusContent" -Force
New-Item -ItemType Directory -Path "W:\UpdateServicesDBFiles" -Force
New-Item -ItemType Directory -Path "W:\logs" -Force
New-Item -ItemType Directory -Path "W:\backup" -Force

# Set permissions (NETWORK SERVICE needs access)
$acl = Get-Acl "W:\WsusContent"
$rule = New-Object System.Security.AccessControl.FileSystemAccessRule(
    "NETWORK SERVICE", "FullControl", "ContainerInherit,ObjectInherit", "None", "Allow")
$acl.SetAccessRule($rule)
Set-Acl "W:\WsusContent" $acl

# Also for Administrators
$rule = New-Object System.Security.AccessControl.FileSystemAccessRule(
    "Administrators", "FullControl", "ContainerInherit,ObjectInherit", "None", "Allow")
$acl.SetAccessRule($rule)
Set-Acl "W:\WsusContent" $acl
```

---

## Part 6: Install WSUS Role

### 6.1 Install WSUS with PowerShell

```powershell
# Install WSUS role with SQL Server Express (WID - Windows Internal Database)
Install-WindowsFeature -Name UpdateServices -IncludeManagementTools

# Alternatively, install with specific features
Install-WindowsFeature -Name UpdateServices-Services,UpdateServices-WidDB -IncludeManagementTools
```

### 6.2 Run Post-Installation Configuration

```powershell
# Navigate to WSUS tools directory
Set-Location "C:\Program Files\Update Services\Tools"

# Run post-install with content path on NAS
.\wsusutil.exe postinstall CONTENT_DIR=W:\WsusContent

# This may take several minutes
# Watch for "Post install has successfully completed" message
```

### 6.3 Verify WSUS Installation

```powershell
# Check WSUS service status
Get-Service WsusService, W3SVC

# Should show both as "Running"
```

---

## Part 7: Configure WSUS for Air-Gapped Operation

### 7.1 Launch WSUS Console

```powershell
# Open WSUS Management Console
wsus.msc

# Or via Server Manager → Tools → Windows Server Update Services
```

### 7.2 Initial Configuration Wizard

The WSUS Configuration Wizard will launch. For air-gapped:

1. **Before You Begin** → Next
2. **Join Microsoft Update Improvement Program** → Uncheck (air-gapped) → Next
3. **Choose Upstream Server**:
   - Select: **Synchronize from another WSUS server** 
   - OR if truly standalone: Select Microsoft Update but DON'T sync
   - For air-gapped: You'll import updates manually (see Part 9)
   - → Next
4. **Specify Proxy Server** → Leave blank (air-gapped) → Next
5. **Connect to Upstream Server** → Skip/Cancel (air-gapped, will fail)
6. **Choose Languages** → Select only needed languages → Next
7. **Choose Products** → Select only products you support:
   - Windows 10/11
   - Windows Server 2019/2022
   - Microsoft Defender Antivirus
   - (Uncheck everything else to save space)
   - → Next
8. **Choose Classifications**:
   - ✅ Critical Updates
   - ✅ Definition Updates
   - ✅ Security Updates
   - ✅ Update Rollups
   - ⬜ Feature Packs (optional, large)
   - ⬜ Service Packs (if needed)
   - → Next
9. **Configure Sync Schedule** → **Synchronize manually** (air-gapped) → Next
10. **Finished** → Do NOT check "Begin initial synchronization"

### 7.3 Configure WSUS Options via PowerShell

```powershell
# Get WSUS server object
$wsus = Get-WsusServer

# View current configuration
$wsusConfig = $wsus.GetConfiguration()
$wsusConfig

# Set to NOT sync from Microsoft (air-gapped)
$wsusConfig.SyncFromMicrosoftUpdate = $false
$wsusConfig.Save()

# Disable automatic sync
$subscription = $wsus.GetSubscription()
$subscription.SynchronizeAutomatically = $false
$subscription.Save()
```

---

## Part 8: Configure WSUS Database Location (Optional)

By default, WSUS uses Windows Internal Database (WID). For NAS storage:

### 8.1 Option A: Keep WID Local (Recommended)

Keep the database on local SSD for performance, only content on NAS.
- Database location: `C:\Windows\WID\Data\`
- Content location: `W:\WsusContent` (NAS)

### 8.2 Option B: Move WID to NAS (Not Recommended)

Moving WID to NAS can cause performance issues. Only do this if necessary:

```powershell
# Stop WSUS services
Stop-Service WsusService, W3SVC

# Detach WID database
# This requires sqlcmd or SSMS - complex process
# Generally NOT recommended for NAS storage

# Start services
Start-Service W3SVC, WsusService
```

---

## Part 9: Import Updates for Air-Gapped Environment

### 9.1 Method A: WSUS Offline Update Tool

On an **internet-connected machine**:

1. Download WSUS Offline Update from `https://download.wsusoffline.net/`
2. Run `UpdateGenerator.exe`
3. Select products and languages
4. Download updates (will take hours/days depending on selections)
5. Transfer downloaded folder to air-gapped network

On **WSUS server**:

```powershell
# Copy downloaded updates to NAS staging
Copy-Item -Path "E:\wsusoffline\client\*" -Destination "W:\staging\" -Recurse

# Updates need to be imported - see Method C for import
```

### 9.2 Method B: Microsoft Update Catalog (Manual)

1. On internet-connected machine, go to: `https://www.catalog.update.microsoft.com/`
2. Search for needed updates (e.g., "Windows Server 2022 Cumulative")
3. Download .msu or .cab files
4. Transfer to WSUS server

### 9.3 Method C: Import Updates to WSUS

```powershell
# Navigate to WSUS tools
Set-Location "C:\Program Files\Update Services\Tools"

# Import a single update file
.\wsusutil.exe import <path-to-update.cab> <path-to-metadata.xml>

# For bulk import, you may need additional tools or scripts
```

### 9.4 Method D: Export/Import from Connected WSUS

If you have WSUS on a connected network:

**On SOURCE (connected) WSUS:**
```powershell
Set-Location "C:\Program Files\Update Services\Tools"

# Export updates and metadata
.\wsusutil.exe export export.xml.gz export.log
```

**Transfer files to air-gapped network, then on TARGET (air-gapped) WSUS:**
```powershell
Set-Location "C:\Program Files\Update Services\Tools"

# Import updates
.\wsusutil.exe import export.xml.gz import.log

# This imports metadata; content must be copied separately
```

**Copy WsusContent folder:**
```powershell
# On target, copy content from source export
# Source WsusContent → Target W:\WsusContent
robocopy "E:\SourceWsusContent" "W:\WsusContent" /E /Z /MT:8
```

---

## Part 10: Configure WSUS IIS Settings

### 10.1 Optimize IIS for WSUS

```powershell
Import-Module WebAdministration

# Set WSUS Application Pool settings
Set-ItemProperty "IIS:\AppPools\WsusPool" -Name "queueLength" -Value 25000
Set-ItemProperty "IIS:\AppPools\WsusPool" -Name "processModel.maxProcesses" -Value 0
Set-ItemProperty "IIS:\AppPools\WsusPool" -Name "recycling.periodicRestart.privateMemory" -Value 0

# Increase request limits
Set-WebConfigurationProperty -pspath 'MACHINE/WEBROOT/APPHOST' `
    -location 'WSUS Administration' `
    -filter "system.webServer/security/requestFiltering/requestLimits" `
    -name "maxAllowedContentLength" -value 2147483647
```

### 10.2 Configure WSUS Ports

Default WSUS ports:
- HTTP: 8530
- HTTPS: 8531

```powershell
# Verify WSUS website bindings
Get-WebBinding -Name "WSUS Administration"

# If you need to change ports (optional):
# Set-WebBinding -Name "WSUS Administration" -BindingInformation "*:8530:" -PropertyName Port -Value 80
```

---

## Part 11: Configure Windows Firewall

```powershell
# Allow WSUS ports
New-NetFirewallRule -DisplayName "WSUS HTTP 8530" `
    -Direction Inbound -Protocol TCP -LocalPort 8530 -Action Allow

New-NetFirewallRule -DisplayName "WSUS HTTPS 8531" `
    -Direction Inbound -Protocol TCP -LocalPort 8531 -Action Allow

# Allow SMB for NAS (if not already allowed)
New-NetFirewallRule -DisplayName "SMB for NAS" `
    -Direction Outbound -Protocol TCP -RemotePort 445 -Action Allow

# Verify
Get-NetFirewallRule -DisplayName "WSUS*" | Format-Table Name, Enabled
```

---

## Part 12: Create Computer Groups

### 12.1 Via WSUS Console

1. Open WSUS Console (`wsus.msc`)
2. Expand server → **Computers** → **All Computers**
3. Right-click **All Computers** → **Add Computer Group**
4. Create groups such as:
   - `Servers-Production`
   - `Servers-Test`
   - `Workstations-Production`
   - `Workstations-Test`

### 12.2 Via PowerShell

```powershell
# Get WSUS server
$wsus = Get-WsusServer

# Create computer groups
$wsus.CreateComputerTargetGroup("Servers-Production")
$wsus.CreateComputerTargetGroup("Servers-Test")
$wsus.CreateComputerTargetGroup("Workstations-Production")
$wsus.CreateComputerTargetGroup("Workstations-Test")

# List groups
$wsus.GetComputerTargetGroups() | Select-Object Name
```

---

## Part 13: Configure Update Approvals

### 13.1 Manual Approval (Recommended for Air-Gapped)

After importing updates:

1. WSUS Console → **Updates** → **All Updates**
2. Change view: **Approval: Any** and **Status: Any**
3. Select updates to approve
4. Right-click → **Approve**
5. Select computer groups → **Approved for Install**

### 13.2 Automatic Approval Rules (Optional)

```powershell
# Get WSUS server
$wsus = Get-WsusServer

# Create automatic approval rule
$rule = $wsus.CreateInstallApprovalRule("Auto-Approve Critical Updates")

# Set rule to apply to specific group
$group = $wsus.GetComputerTargetGroups() | Where-Object { $_.Name -eq "Workstations-Test" }
$rule.SetComputerTargetGroups($group)

# Set classifications (Critical and Security)
$criticalClass = $wsus.GetUpdateClassifications() | Where-Object { $_.Title -eq "Critical Updates" }
$securityClass = $wsus.GetUpdateClassifications() | Where-Object { $_.Title -eq "Security Updates" }
$rule.SetUpdateClassifications($criticalClass + $securityClass)

# Enable rule
$rule.Enabled = $true
$rule.Save()
```

---

## Part 14: Client Configuration (GPO)

### 14.1 Configure via Group Policy

On your Domain Controller or using local GPO:

**Computer Configuration → Administrative Templates → Windows Components → Windows Update**

| Setting | Value |
|---------|-------|
| **Specify intranet Microsoft update service location** | Enabled |
| - Set the intranet update service | `http://WSUSSERVER:8530` |
| - Set the intranet statistics server | `http://WSUSSERVER:8530` |
| **Configure Automatic Updates** | Enabled |
| - Configure automatic updating | 4 - Auto download and schedule install |
| **Enable client-side targeting** | Enabled (if using computer groups) |
| - Target group name | `Workstations-Production` |

### 14.2 Configure via Registry (Non-Domain)

```powershell
# Run on each client machine
$wsusServer = "http://WSUSSERVER:8530"

# Set WSUS server
Set-ItemProperty -Path "HKLM:\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate" `
    -Name "WUServer" -Value $wsusServer -Type String
Set-ItemProperty -Path "HKLM:\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate" `
    -Name "WUStatusServer" -Value $wsusServer -Type String

# Enable Windows Update to use WSUS
Set-ItemProperty -Path "HKLM:\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate\AU" `
    -Name "UseWUServer" -Value 1 -Type DWord

# Set automatic updates (4 = auto download and schedule)
Set-ItemProperty -Path "HKLM:\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate\AU" `
    -Name "AUOptions" -Value 4 -Type DWord

# Restart Windows Update service
Restart-Service wuauserv
```

### 14.3 Client Detection Script

```powershell
# Force client to detect updates and report to WSUS
wuauclt /detectnow /reportnow

# On newer Windows (Windows 10+):
Start-ScheduledTask -TaskPath "\Microsoft\Windows\WindowsUpdate\" -TaskName "Scheduled Start"
usoclient StartScan
```

---

## Part 15: Maintenance Commands

### 15.1 WSUS Service Management

```powershell
# Check WSUS services
Get-Service WsusService, W3SVC, MSSQL`$MICROSOFT##WID

# Restart WSUS
Restart-Service WsusService

# Full restart (including IIS)
Stop-Service WsusService
iisreset /stop
iisreset /start
Start-Service WsusService
```

### 15.2 WSUS Cleanup Wizard

Run monthly to reclaim space:

```powershell
# Get WSUS server
$wsus = Get-WsusServer

# Run cleanup (all options)
$cleanup = $wsus.GetCleanupManager()
$cleanupScope = New-Object Microsoft.UpdateServices.Administration.CleanupScope

$cleanupScope.DeclineSupersededUpdates = $true
$cleanupScope.DeclineExpiredUpdates = $true
$cleanupScope.CleanupObsoleteUpdates = $true
$cleanupScope.CompressUpdates = $true
$cleanupScope.CleanupObsoleteComputers = $true
$cleanupScope.CleanupUnneededContentFiles = $true

$results = $cleanup.PerformCleanup($cleanupScope)

# Display results
Write-Host "Superseded updates declined: $($results.SupersededUpdatesDeclined)"
Write-Host "Expired updates declined: $($results.ExpiredUpdatesDeclined)"
Write-Host "Obsolete updates deleted: $($results.ObsoleteUpdatesDeleted)"
Write-Host "Disk space freed: $($results.DiskSpaceFreed / 1GB) GB"
```

### 15.3 Check Content Directory Size

```powershell
# Check NAS WSUS content size
$size = (Get-ChildItem -Path "W:\WsusContent" -Recurse | Measure-Object -Property Length -Sum).Sum
Write-Host "WSUS Content Size: $([math]::Round($size / 1GB, 2)) GB"

# Check drive space
Get-PSDrive W | Select-Object Used, Free | ForEach-Object {
    Write-Host "Used: $([math]::Round($_.Used / 1GB, 2)) GB"
    Write-Host "Free: $([math]::Round($_.Free / 1GB, 2)) GB"
}
```

### 15.4 Database Maintenance

```powershell
# Reindex WSUS database (improves performance)
# This uses the Windows Internal Database

# Method 1: Using WSUS built-in maintenance
Set-Location "C:\Program Files\Update Services\Tools"
.\wsusutil.exe checkhealth

# Method 2: SQL script for deeper maintenance (requires sqlcmd)
# Download and run Microsoft WSUS maintenance script
```

### 15.5 Backup WSUS Configuration

```powershell
# Export WSUS metadata
Set-Location "C:\Program Files\Update Services\Tools"
.\wsusutil.exe export W:\backup\wsus-export-$(Get-Date -Format 'yyyyMMdd').xml.gz W:\backup\wsus-export.log

# Backup IIS config
& "$env:windir\system32\inetsrv\appcmd.exe" add backup "WSUS-$(Get-Date -Format 'yyyyMMdd')"

# Backup registry settings
reg export "HKLM\SOFTWARE\Microsoft\Update Services" "W:\backup\wsus-registry-$(Get-Date -Format 'yyyyMMdd').reg"
```

---

## Part 16: LXD VM Management Commands

Run these on the Ubuntu LXD host:

### 16.1 VM Lifecycle

```bash
# Start VM
lxc start win2022-wsus

# Stop VM (graceful)
lxc stop win2022-wsus

# Force stop
lxc stop win2022-wsus --force

# Restart
lxc restart win2022-wsus

# Check status
lxc list win2022-wsus
lxc info win2022-wsus
```

### 16.2 Console Access

```bash
# VGA console (graphical)
lxc console win2022-wsus --type=vga

# Exit: Ctrl+A then Q
```

### 16.3 Snapshots

```bash
# Create snapshot before changes
lxc snapshot win2022-wsus pre-update

# List snapshots
lxc info win2022-wsus | grep -A20 Snapshots

# Restore if needed
lxc restore win2022-wsus pre-update

# Delete snapshot
lxc delete win2022-wsus/pre-update
```

### 16.4 Full Backup

```bash
# Export VM (takes time, creates large file)
lxc export win2022-wsus /mnt/nas/wsus-content/backup/win2022-wsus-$(date +%Y%m%d).tar.gz

# Import/restore
lxc import /mnt/nas/wsus-content/backup/win2022-wsus-20250201.tar.gz win2022-wsus-restored
```

---

## Troubleshooting

### Issue: WSUS Console Won't Connect

```powershell
# Check services
Get-Service WsusService, W3SVC, MSSQL`$MICROSOFT##WID

# Reset WSUS
Stop-Service WsusService
iisreset
Start-Service WsusService

# Check event logs
Get-EventLog -LogName Application -Source "Windows Server Update Services" -Newest 20
```

### Issue: Clients Not Reporting

```powershell
# On client, force detection
wuauclt /detectnow /reportnow

# Check client registry
Get-ItemProperty "HKLM:\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate"
Get-ItemProperty "HKLM:\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate\AU"

# Check client can reach WSUS
Test-NetConnection -ComputerName WSUSSERVER -Port 8530
Invoke-WebRequest -Uri "http://WSUSSERVER:8530/selfupdate/wuident.cab" -UseBasicParsing
```

### Issue: NAS Share Disconnected

```powershell
# Check connection
Test-NetConnection -ComputerName <NAS-IP> -Port 445

# Remap drive
net use W: /delete
net use W: \\<NAS-IP>\wsus-content /user:wsussvc <PASSWORD> /persistent:yes

# Restart WSUS after remapping
Restart-Service WsusService
```

### Issue: Content Download Errors

```powershell
# Check content directory permissions
Get-Acl "W:\WsusContent" | Format-List

# Test write access
New-Item -ItemType File -Path "W:\WsusContent\test.txt"
Remove-Item "W:\WsusContent\test.txt"

# Verify BITS service
Get-Service BITS
Start-Service BITS
```

### Issue: Database Errors

```powershell
# Check WID service
Get-Service MSSQL`$MICROSOFT##WID

# Restart WID and WSUS
Restart-Service MSSQL`$MICROSOFT##WID
Restart-Service WsusService

# Check WID logs
Get-EventLog -LogName Application -Source "MSSQL*" -Newest 20
```

---

## Quick Reference Card

```
┌─────────────────────────────────────────────────────────────────┐
│                    WSUS QUICK REFERENCE                          │
├─────────────────────────────────────────────────────────────────┤
│ Server Name:      WSUSSERVER                                     │
│ Server IP:        _____________________                          │
│ WSUS URL:         http://<IP>:8530                               │
│ NAS IP:           _____________________                          │
│ NAS Share:        \\<NAS-IP>\wsus-content                        │
│ Content Path:     W:\WsusContent                                 │
├─────────────────────────────────────────────────────────────────┤
│ LXD Commands (Ubuntu Host):                                      │
│   lxc start win2022-wsus                                         │
│   lxc stop win2022-wsus                                          │
│   lxc console win2022-wsus --type=vga                            │
│   lxc snapshot win2022-wsus <name>                               │
├─────────────────────────────────────────────────────────────────┤
│ WSUS Commands (Windows):                                         │
│   wsus.msc                    (Open WSUS Console)                │
│   Restart-Service WsusService (Restart WSUS)                     │
│   iisreset                    (Restart IIS)                      │
│   wsusutil.exe postinstall    (Post-installation config)         │
│   wsusutil.exe export/import  (Air-gap sync)                     │
├─────────────────────────────────────────────────────────────────┤
│ Client Commands:                                                 │
│   wuauclt /detectnow /reportnow                                  │
│   usoclient StartScan                                            │
│   gpupdate /force             (Refresh GPO)                      │
├─────────────────────────────────────────────────────────────────┤
│ Ports:                                                           │
│   8530 - WSUS HTTP                                               │
│   8531 - WSUS HTTPS                                              │
│   445  - SMB (NAS access)                                        │
├─────────────────────────────────────────────────────────────────┤
│ Storage Used:     ______________ GB                              │
│ Storage Free:     ______________ GB                              │
└─────────────────────────────────────────────────────────────────┘
```

---

## Combined Infrastructure Summary

If running both Chocolatey and WSUS:

```
┌───────────────────────────────────────────────────────────────────────┐
│                     COMPLETE INFRASTRUCTURE                            │
├───────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  LXD Host (Ubuntu)                                                     │
│  ├── win2022-choco (VM)                                                │
│  │   ├── IP: 10.10.10.100                                              │
│  │   ├── RAM: 8GB, CPU: 4                                              │
│  │   ├── Service: Chocolatey.Server (IIS)                              │
│  │   └── NAS Mount: Z:\ → \\NAS\chocolatey-repo                        │
│  │                                                                     │
│  └── win2022-wsus (VM)                                                 │
│      ├── IP: 10.10.10.101                                              │
│      ├── RAM: 16GB, CPU: 4                                             │
│      ├── Service: WSUS                                                 │
│      └── NAS Mount: W:\ → \\NAS\wsus-content                           │
│                                                                        │
│  Buffalo NAS                                                           │
│  ├── Share: chocolatey-repo (50-100 GB)                                │
│  │   └── packages/, logs/, backup/                                     │
│  │                                                                     │
│  └── Share: wsus-content (500 GB - 1 TB)                               │
│      └── WsusContent/, logs/, backup/                                  │
│                                                                        │
│  TOTAL NAS: 600 GB - 1.2 TB recommended                                │
│                                                                        │
└───────────────────────────────────────────────────────────────────────┘
```

---

## Document Version

- Version: 1.0
- Last Updated: February 2025
- Environment: Air-Gapped / Offline Deployment
- Related: Chocolatey Server Guide
