# Chocolatey Server on LXD (Windows VM) - Air-Gapped Deployment Guide

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        LXD Host (Ubuntu)                                 │
│                                                                          │
│  ┌────────────────────────────────────────────────────────────────┐     │
│  │              Windows Server VM (LXD)                            │     │
│  │                                                                 │     │
│  │  ┌─────────────────────┐    ┌────────────────────────────┐    │     │
│  │  │  Chocolatey.Server  │    │  Z:\ (Mapped NAS Share)    │    │     │
│  │  │  (IIS + NuGet)      │───▶│  └── ChocolateyPackages    │    │     │
│  │  │  Port: 80/443       │    │      ├── packages/         │    │     │
│  │  └─────────────────────┘    │      └── logs/             │    │     │
│  │                             └────────────────────────────────┘    │     │
│  └────────────────────────────────────────────────────────────────┘     │
│                                    │                                     │
│  /mnt/nas/chocolatey ◄─────────────┘ (passed through to VM)             │
│         │                                                                │
└─────────│────────────────────────────────────────────────────────────────┘
          │ NFS/SMB Mount
          ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                          Buffalo NAS                                     │
│  ┌────────────────────────────────────────────────────────────────┐     │
│  │  Share: /chocolatey-repo                                        │     │
│  │    ├── packages/         (.nupkg files)                         │     │
│  │    ├── logs/             (IIS & Chocolatey logs)                │     │
│  │    └── backup/           (configuration backups)                │     │
│  └────────────────────────────────────────────────────────────────┘     │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Pre-Deployment: Items to Stage (Air-Gap Preparation)

Download these on an internet-connected machine and transfer to your air-gapped network:

### Required Downloads

| Item | Source | Notes |
|------|--------|-------|
| Windows Server 2022 ISO | Microsoft VLSC / Eval Center | Standard or Datacenter |
| VirtIO Drivers ISO | `https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/stable-virtio/virtio-win.iso` | Required for LXD/KVM |
| Chocolatey Offline Install | `https://community.chocolatey.org/install.ps1` | Save as install.ps1 |
| Chocolatey nupkg | `https://community.chocolatey.org/api/v2/package/chocolatey` | Rename to chocolatey.nupkg |
| Chocolatey.Server nupkg | `https://community.chocolatey.org/api/v2/package/chocolatey.server` | Core server package |
| .NET Framework 4.8 Offline | Microsoft Download Center | If not in Windows image |
| Any packages you need | community.chocolatey.org | Download .nupkg files |

### Transfer Method
```bash
# Example: Create transfer archive
mkdir -p /transfer/chocolatey-staging
# Copy all downloaded files to this directory
tar -czvf chocolatey-airgap-bundle.tar.gz /transfer/chocolatey-staging

# Transfer via approved method (USB, DVD, secure file transfer, etc.)
```

---

## Part 1: Buffalo NAS Configuration

### 1.1 Create Share via NAS Web Interface

1. Access Buffalo NAS admin: `http://<NAS-IP>/`
2. Navigate to: **Shares** → **Folder Setup** → **Create Folder**
3. Create share: `chocolatey-repo`

### 1.2 Configure SMB Share (Recommended for Windows VM)

1. **Shares** → **chocolatey-repo** → **SMB Settings**
2. Enable SMB sharing
3. Create service account:
   - Username: `chocosvc`
   - Password: `<strong-password>`
   - Permissions: Read/Write

### 1.3 Configure NFS Export (Alternative - for LXD Host Mount)

1. **Network** → **NFS** → **Enable NFS**
2. Add export rule:
   - Path: `/chocolatey-repo`
   - Allowed Host: `<LXD-Host-IP>`
   - Access: Read/Write
   - Root Squash: No

### 1.4 Note Your NAS Details

```
NAS IP Address:      ___________________
Share Name:          chocolatey-repo
SMB Username:        chocosvc
SMB Password:        ___________________
```

---

## Part 2: LXD Host Preparation (Ubuntu)

### 2.1 Install LXD (If Not Already Installed)

```bash
# Install LXD via snap
sudo snap install lxd

# Initialize LXD (use defaults or customize)
sudo lxd init
```

**Interactive prompts - recommended answers:**
```
Would you like to use LXD clustering? no
Do you want to configure a new storage pool? yes
Name of the new storage pool: default
Name of the storage backend to use: dir (or zfs if available)
Would you like to connect to a MAAS server? no
Would you like to create a new local network bridge? yes
What should the new bridge be called? lxdbr0
What IPv4 address should be used? auto
What IPv6 address should be used? none
Would you like LXD to be available over the network? no
Would you like stale cached images to be updated automatically? no (air-gapped)
Would you like a YAML "lxd init" preseed to be printed? no
```

### 2.2 Verify LXD Installation

```bash
# Check LXD status
lxc version

# List networks
lxc network list

# List storage pools
lxc storage list
```

### 2.3 Install NFS Client Utilities

```bash
sudo apt update
sudo apt install -y nfs-common cifs-utils
```

### 2.4 Create NAS Mount Point

```bash
# Create mount directory
sudo mkdir -p /mnt/nas/chocolatey-repo

# Set permissions
sudo chmod 755 /mnt/nas/chocolatey-repo
```

### 2.5 Mount Buffalo NAS (Choose One Method)

#### Option A: NFS Mount
```bash
# Test mount (temporary)
sudo mount -t nfs <NAS-IP>:/chocolatey-repo /mnt/nas/chocolatey-repo -o rw,sync,hard,intr

# Verify mount
df -h /mnt/nas/chocolatey-repo
ls -la /mnt/nas/chocolatey-repo

# Make permanent - add to /etc/fstab
echo "<NAS-IP>:/chocolatey-repo /mnt/nas/chocolatey-repo nfs rw,sync,hard,intr,_netdev 0 0" | sudo tee -a /etc/fstab
```

#### Option B: SMB/CIFS Mount
```bash
# Create credentials file
sudo tee /etc/nas-creds-choco << 'EOF'
username=chocosvc
password=<YOUR-PASSWORD>
domain=WORKGROUP
EOF

sudo chmod 600 /etc/nas-creds-choco

# Test mount (temporary)
sudo mount -t cifs //<NAS-IP>/chocolatey-repo /mnt/nas/chocolatey-repo \
    -o credentials=/etc/nas-creds-choco,uid=0,gid=0,file_mode=0777,dir_mode=0777

# Verify mount
df -h /mnt/nas/chocolatey-repo

# Make permanent - add to /etc/fstab
echo "//<NAS-IP>/chocolatey-repo /mnt/nas/chocolatey-repo cifs credentials=/etc/nas-creds-choco,uid=0,gid=0,file_mode=0777,dir_mode=0777,_netdev 0 0" | sudo tee -a /etc/fstab
```

### 2.6 Create Directory Structure on NAS

```bash
sudo mkdir -p /mnt/nas/chocolatey-repo/{packages,logs,backup,staging}
sudo chmod -R 777 /mnt/nas/chocolatey-repo
```

### 2.7 Verify Mount Persists After Reboot

```bash
# Remount from fstab
sudo mount -a

# Verify
mount | grep chocolatey
```

---

## Part 3: Stage Installation Files

### 3.1 Copy Staged Files to NAS

```bash
# Assuming your transfer bundle is at /tmp/chocolatey-airgap-bundle.tar.gz
cd /tmp
tar -xzvf chocolatey-airgap-bundle.tar.gz

# Copy to NAS staging area
cp -r /tmp/transfer/chocolatey-staging/* /mnt/nas/chocolatey-repo/staging/

# Verify files
ls -la /mnt/nas/chocolatey-repo/staging/
```

Expected contents:
```
/mnt/nas/chocolatey-repo/staging/
├── windows-server-2022.iso
├── virtio-win.iso
├── install.ps1                 (Chocolatey installer)
├── chocolatey.nupkg
├── chocolatey.server.nupkg
├── dotnet-framework-48.exe     (if needed)
└── packages/                   (any additional .nupkg files)
```

---

## Part 4: Create Windows Server VM in LXD

### 4.1 Create VM Instance

```bash
# Create empty Windows VM
lxc init win2022-choco --empty --vm

# Configure VM resources
lxc config set win2022-choco limits.cpu 4
lxc config set win2022-choco limits.memory 8GB

# Set secure boot off (required for Windows in LXD)
lxc config set win2022-choco security.secureboot false

# Enable TPM (Windows 11/Server 2022 may require)
lxc config set win2022-choco security.csm false
```

### 4.2 Add Storage Devices

```bash
# Expand root disk (Windows needs at least 40GB)
lxc config device override win2022-choco root size=80GB

# Attach Windows ISO for installation
lxc config device add win2022-choco iso disk \
    source=/mnt/nas/chocolatey-repo/staging/windows-server-2022.iso \
    boot.priority=10

# Attach VirtIO drivers ISO
lxc config device add win2022-choco virtio disk \
    source=/mnt/nas/chocolatey-repo/staging/virtio-win.iso
```

### 4.3 Configure Network

```bash
# Verify network device (should be auto-added)
lxc config show win2022-choco | grep -A5 "eth0"

# If needed, manually add network
lxc config device add win2022-choco eth0 nic \
    nictype=bridged \
    parent=lxdbr0
```

### 4.4 Add NAS Share as Disk Device

```bash
# Pass through NAS mount to VM
lxc config device add win2022-choco nas-share disk \
    source=/mnt/nas/chocolatey-repo \
    path=/nas-share
```

### 4.5 Start VM and Access Console

```bash
# Start the VM
lxc start win2022-choco

# Access console for Windows installation
lxc console win2022-choco --type=vga

# Alternative: Get VM IP and use RDP later
lxc list win2022-choco
```

**Note:** Press any key quickly when "Press any key to boot from CD/DVD" appears.

---

## Part 5: Windows Server Installation

### 5.1 Install Windows Server

1. Select language, time, keyboard → **Next**
2. Click **Install now**
3. Select: **Windows Server 2022 Standard (Desktop Experience)**
4. Accept license → **Next**
5. Select: **Custom: Install Windows only**
6. **Load driver** → Browse to VirtIO CD → `viostor\2k22\amd64` → Select driver
7. Select the disk → **Next**
8. Wait for installation to complete

### 5.2 Initial Windows Configuration

After Windows boots:

1. Set Administrator password
2. Log in as Administrator

### 5.3 Install VirtIO Guest Tools (From VirtIO ISO)

```powershell
# In Windows, open PowerShell as Administrator
# Navigate to VirtIO CD drive (usually D: or E:)
D:
.\virtio-win-guest-tools.exe /S
```

### 5.4 Configure Static IP (Recommended)

```powershell
# Get adapter name
Get-NetAdapter

# Set static IP (adjust values for your network)
New-NetIPAddress -InterfaceAlias "Ethernet" `
    -IPAddress "10.10.10.100" `
    -PrefixLength 24 `
    -DefaultGateway "10.10.10.1"

# Set DNS (use your internal DNS or leave empty for air-gap)
Set-DnsClientServerAddress -InterfaceAlias "Ethernet" `
    -ServerAddresses "10.10.10.1"

# Verify
Get-NetIPConfiguration
```

### 5.5 Rename Computer

```powershell
Rename-Computer -NewName "CHOCOSERVER" -Restart
```

---

## Part 6: Map NAS Share in Windows

### 6.1 Option A: Map Network Drive via SMB

```powershell
# Store credentials
cmdkey /add:<NAS-IP> /user:chocosvc /pass:<PASSWORD>

# Map drive persistently
New-PSDrive -Name "Z" -PSProvider FileSystem `
    -Root "\\<NAS-IP>\chocolatey-repo" `
    -Persist `
    -Credential (Get-Credential)

# Or via net use (simpler)
net use Z: \\<NAS-IP>\chocolatey-repo /user:chocosvc <PASSWORD> /persistent:yes

# Verify
Get-PSDrive Z
dir Z:\
```

### 6.2 Option B: Use LXD Passthrough (9P Filesystem)

If you added the disk device in LXD, it may appear as a local disk. Check:

```powershell
# List all drives
Get-PSDrive -PSProvider FileSystem

# The NAS share may appear as a new drive letter
# Or check Device Manager for new storage devices
```

### 6.3 Create Chocolatey Directories on NAS

```powershell
# Create directory structure
New-Item -ItemType Directory -Path "Z:\http-packages" -Force
New-Item -ItemType Directory -Path "Z:\logs" -Force
New-Item -ItemType Directory -Path "Z:\backup" -Force

# Verify
dir Z:\
```

---

## Part 7: Install IIS and ASP.NET

### 7.1 Install IIS with Required Features

```powershell
# Install IIS with ASP.NET 4.8
Install-WindowsFeature -Name Web-Server -IncludeManagementTools
Install-WindowsFeature -Name Web-Asp-Net45
Install-WindowsFeature -Name Web-AppInit
Install-WindowsFeature -Name Web-Windows-Auth

# Verify installation
Get-WindowsFeature -Name Web-* | Where-Object Installed -eq $true
```

### 7.2 Verify IIS Is Running

```powershell
# Check IIS service
Get-Service W3SVC

# Test default site
Invoke-WebRequest -Uri "http://localhost" -UseBasicParsing
```

---

## Part 8: Install Chocolatey (Offline/Air-Gapped)

### 8.1 Prepare Offline Installation

```powershell
# Create local Chocolatey directory
New-Item -ItemType Directory -Path "C:\choco-install" -Force

# Copy installation files from NAS staging
Copy-Item "Z:\staging\install.ps1" "C:\choco-install\"
Copy-Item "Z:\staging\chocolatey.nupkg" "C:\choco-install\"
```

### 8.2 Install Chocolatey Offline

```powershell
# Set execution policy
Set-ExecutionPolicy Bypass -Scope Process -Force

# Set environment variable for offline install
$env:chocolateyDownloadUrl = "C:\choco-install\chocolatey.nupkg"

# Run installer
& C:\choco-install\install.ps1

# Refresh environment
$env:Path = [System.Environment]::GetEnvironmentVariable("Path","Machine") + ";" + [System.Environment]::GetEnvironmentVariable("Path","User")

# Verify installation
choco --version
```

### 8.3 Configure Chocolatey for Air-Gapped Environment

```powershell
# Disable default public source
choco source remove --name="chocolatey"

# Disable telemetry/phone home features
choco feature disable --name="usePackageExitCodes"
choco config set --name="commandExecutionTimeoutSeconds" --value="14400"

# Verify sources (should be empty or only local)
choco source list
```

---

## Part 9: Install Chocolatey.Server

### 9.1 Install from Offline Package

```powershell
# Install Chocolatey.Server from local nupkg
choco install chocolatey.server -y --source="Z:\staging" --ignore-dependencies

# If dependencies are missing, install them first from your staged packages
# choco install <dependency> -y --source="Z:\staging"
```

### 9.2 Alternative: Manual Installation

If the choco install fails, extract and configure manually:

```powershell
# Create web application directory
New-Item -ItemType Directory -Path "C:\tools\chocolatey.server" -Force

# Extract nupkg (it's a zip file)
Expand-Archive -Path "Z:\staging\chocolatey.server.nupkg" -DestinationPath "C:\tools\chocolatey.server" -Force

# The web app content is in the tools folder
Move-Item "C:\tools\chocolatey.server\tools\*" "C:\tools\chocolatey.server\" -Force
```

---

## Part 10: Configure Chocolatey.Server

### 10.1 Configure Package Path to NAS

```powershell
# Edit web.config
$webConfig = "C:\tools\chocolatey.server\web.config"

# Backup original
Copy-Item $webConfig "$webConfig.backup"

# Load and modify
[xml]$config = Get-Content $webConfig

# Find and update packagesPath setting
$appSettings = $config.configuration.appSettings
$packagePath = $appSettings.add | Where-Object { $_.key -eq "packagesPath" }

if ($packagePath) {
    $packagePath.value = "Z:\http-packages"
} else {
    $newSetting = $config.CreateElement("add")
    $newSetting.SetAttribute("key", "packagesPath")
    $newSetting.SetAttribute("value", "Z:\http-packages")
    $appSettings.AppendChild($newSetting)
}

# Save
$config.Save($webConfig)
```

### 10.2 Generate and Set API Key

```powershell
# Generate a GUID for API key
$apiKey = [guid]::NewGuid().ToString()
Write-Host "Your API Key: $apiKey"
Write-Host "SAVE THIS KEY - you will need it for clients"

# Add to web.config
[xml]$config = Get-Content $webConfig

$apiKeySetting = $config.configuration.appSettings.add | Where-Object { $_.key -eq "apiKey" }
if ($apiKeySetting) {
    $apiKeySetting.value = $apiKey
} else {
    $newSetting = $config.CreateElement("add")
    $newSetting.SetAttribute("key", "apiKey")
    $newSetting.SetAttribute("value", $apiKey)
    $config.configuration.appSettings.AppendChild($newSetting)
}

$config.Save($webConfig)

# Save API key to NAS for reference
$apiKey | Out-File "Z:\backup\api-key.txt"
```

### 10.3 Full web.config appSettings Reference

Your `appSettings` section should look like this:

```xml
<appSettings>
    <!-- Package storage location on NAS -->
    <add key="packagesPath" value="Z:\http-packages" />
    
    <!-- API key for pushing packages -->
    <add key="apiKey" value="YOUR-GUID-API-KEY-HERE" />
    
    <!-- Allow package overwrites (set to true if needed) -->
    <add key="allowOverrideExistingPackageOnPush" value="false" />
    
    <!-- Cache duration in seconds -->
    <add key="cacheExpirationInSeconds" value="30" />
</appSettings>
```

---

## Part 11: Configure IIS Site

### 11.1 Create Application Pool

```powershell
Import-Module WebAdministration

# Create application pool
New-WebAppPool -Name "ChocolateyServer"

# Configure pool
Set-ItemProperty "IIS:\AppPools\ChocolateyServer" -Name "managedRuntimeVersion" -Value "v4.0"
Set-ItemProperty "IIS:\AppPools\ChocolateyServer" -Name "managedPipelineMode" -Value "Integrated"
Set-ItemProperty "IIS:\AppPools\ChocolateyServer" -Name "startMode" -Value "AlwaysRunning"

# Set identity (use LocalSystem to access network shares)
Set-ItemProperty "IIS:\AppPools\ChocolateyServer" -Name "processModel.identityType" -Value "LocalSystem"
```

### 11.2 Create Website

```powershell
# Remove default site (optional)
Remove-Website -Name "Default Web Site"

# Create Chocolatey website
New-Website -Name "ChocolateyServer" `
    -PhysicalPath "C:\tools\chocolatey.server" `
    -ApplicationPool "ChocolateyServer" `
    -Port 80 `
    -Force

# Start the site
Start-Website -Name "ChocolateyServer"
```

### 11.3 Set Permissions

```powershell
# Grant IIS access to web directory
$acl = Get-Acl "C:\tools\chocolatey.server"
$rule = New-Object System.Security.AccessControl.FileSystemAccessRule(
    "IIS_IUSRS", "ReadAndExecute", "ContainerInherit,ObjectInherit", "None", "Allow")
$acl.SetAccessRule($rule)
Set-Acl "C:\tools\chocolatey.server" $acl

# Grant full control to App_Data
$appData = "C:\tools\chocolatey.server\App_Data"
if (!(Test-Path $appData)) { New-Item -ItemType Directory -Path $appData }
$acl = Get-Acl $appData
$rule = New-Object System.Security.AccessControl.FileSystemAccessRule(
    "IIS_IUSRS", "FullControl", "ContainerInherit,ObjectInherit", "None", "Allow")
$acl.SetAccessRule($rule)
Set-Acl $appData $acl
```

### 11.4 Verify IIS Configuration

```powershell
# List sites
Get-Website

# Check application pool status
Get-WebAppPoolState -Name "ChocolateyServer"

# Check site status
Get-WebsiteState -Name "ChocolateyServer"
```

---

## Part 12: Configure Windows Firewall

```powershell
# Allow HTTP (port 80)
New-NetFirewallRule -DisplayName "Chocolatey Server HTTP" `
    -Direction Inbound `
    -Protocol TCP `
    -LocalPort 80 `
    -Action Allow

# Allow HTTPS (port 443) - for future use
New-NetFirewallRule -DisplayName "Chocolatey Server HTTPS" `
    -Direction Inbound `
    -Protocol TCP `
    -LocalPort 443 `
    -Action Allow

# Verify rules
Get-NetFirewallRule -DisplayName "Chocolatey*" | Format-Table Name, Enabled, Direction, Action
```

---

## Part 13: Test the Server

### 13.1 Test Locally

```powershell
# Test the feed endpoint
Invoke-WebRequest -Uri "http://localhost/nuget/Packages" -UseBasicParsing

# Expected: XML response with empty feed (no packages yet)
```

### 13.2 Test from LXD Host

```bash
# Get Windows VM IP
lxc list win2022-choco

# Test from Ubuntu host
curl -v http://<VM-IP>/nuget/Packages
```

### 13.3 Test Package Push

```powershell
# Get server IP
$serverIP = (Get-NetIPAddress -AddressFamily IPv4 | Where-Object { $_.InterfaceAlias -eq "Ethernet" }).IPAddress
$apiKey = Get-Content "Z:\backup\api-key.txt"

# Push a test package
choco push Z:\staging\packages\some-package.nupkg `
    --source="http://$serverIP/chocolatey" `
    --api-key="$apiKey"

# Verify package is in NAS storage
dir Z:\http-packages
```

---

## Part 14: Client Configuration

### 14.1 Configure Client Machines (Air-Gapped)

On each Windows client that needs to use the Chocolatey server:

```powershell
# Variables - update these
$serverIP = "10.10.10.100"  # Your Chocolatey server IP
$apiKey = "your-api-key-here"  # From Z:\backup\api-key.txt

# Remove default source (air-gapped, won't work anyway)
choco source remove --name="chocolatey"

# Add internal server
choco source add --name="internal" `
    --source="http://$serverIP/chocolatey" `
    --priority=1

# Save API key for pushing (optional - only for package maintainers)
choco apikey add --source="http://$serverIP/chocolatey" --key="$apiKey"

# Verify
choco source list
```

### 14.2 Test Client Access

```powershell
# Search for packages
choco search * --source="internal"

# Install a package
choco install <package-name> --source="internal" -y
```

---

## Part 15: Maintenance Commands

### 15.1 Service Management

```powershell
# Restart IIS
iisreset

# Restart just the app pool
Restart-WebAppPool -Name "ChocolateyServer"

# Check IIS logs
Get-Content "C:\inetpub\logs\LogFiles\W3SVC1\*.log" -Tail 50
```

### 15.2 Package Management

```powershell
# List packages on server (via NAS)
Get-ChildItem "Z:\http-packages" -Filter "*.nupkg"

# Delete a package
Remove-Item "Z:\http-packages\package-name.1.0.0.nupkg"

# After adding/removing packages, may need to clear cache
iisreset
```

### 15.3 Backup Configuration

```powershell
# Backup web.config
Copy-Item "C:\tools\chocolatey.server\web.config" "Z:\backup\web.config.$(Get-Date -Format 'yyyyMMdd')"

# Backup IIS configuration
& "$env:windir\system32\inetsrv\appcmd.exe" add backup "ChocolateyBackup-$(Get-Date -Format 'yyyyMMdd')"
```

### 15.4 Monitor Disk Space

```powershell
# Check NAS space
Get-PSDrive Z | Select-Object Used, Free

# Check local space
Get-PSDrive C | Select-Object Used, Free
```

---

## Part 16: LXD VM Management Commands

Run these on the Ubuntu LXD host:

### 16.1 VM Lifecycle

```bash
# Start VM
lxc start win2022-choco

# Stop VM (graceful)
lxc stop win2022-choco

# Force stop
lxc stop win2022-choco --force

# Restart
lxc restart win2022-choco

# Check status
lxc list win2022-choco
lxc info win2022-choco
```

### 16.2 Console Access

```bash
# VGA console (graphical)
lxc console win2022-choco --type=vga

# Exit console: Ctrl+A then Q
```

### 16.3 Snapshots and Backups

```bash
# Create snapshot
lxc snapshot win2022-choco pre-update

# List snapshots
lxc info win2022-choco

# Restore snapshot
lxc restore win2022-choco pre-update

# Delete snapshot
lxc delete win2022-choco/pre-update

# Full export (backup)
lxc export win2022-choco /mnt/nas/chocolatey-repo/backup/win2022-choco-$(date +%Y%m%d).tar.gz
```

### 16.4 Resource Adjustment

```bash
# Change memory (requires restart)
lxc config set win2022-choco limits.memory 16GB

# Change CPU
lxc config set win2022-choco limits.cpu 8

# View current config
lxc config show win2022-choco
```

---

## Troubleshooting

### Issue: NAS Share Not Accessible from Windows

```powershell
# Test connectivity
Test-NetConnection -ComputerName <NAS-IP> -Port 445

# Check stored credentials
cmdkey /list

# Re-add credentials
cmdkey /delete:<NAS-IP>
cmdkey /add:<NAS-IP> /user:chocosvc /pass:<PASSWORD>

# Remap drive
net use Z: /delete
net use Z: \\<NAS-IP>\chocolatey-repo /user:chocosvc <PASSWORD> /persistent:yes
```

### Issue: IIS Returns 500 Error

```powershell
# Check application pool
Get-WebAppPoolState -Name "ChocolateyServer"

# Check Event Viewer
Get-EventLog -LogName Application -Newest 20 -Source "ASP.NET*"

# Enable detailed errors (temporarily)
# In web.config, set: <customErrors mode="Off" />
```

### Issue: Package Push Fails

```powershell
# Verify API key
Get-Content "Z:\backup\api-key.txt"

# Check write permissions on NAS
New-Item -ItemType File -Path "Z:\http-packages\test.txt"
Remove-Item "Z:\http-packages\test.txt"

# Check IIS identity
(Get-ItemProperty "IIS:\AppPools\ChocolateyServer").processModel.identityType
```

### Issue: VM Cannot Reach Network

```bash
# On LXD host, check network
lxc network list
lxc network show lxdbr0

# Check VM network config
lxc config device show win2022-choco

# Restart VM networking
lxc restart win2022-choco
```

---

## Quick Reference Card

```
┌─────────────────────────────────────────────────────────────────┐
│                    QUICK REFERENCE                              │
├─────────────────────────────────────────────────────────────────┤
│ Server IP:        _____________________                         │
│ Server URL:       http://<IP>/chocolatey                        │
│ API Key:          _____________________                         │
│ NAS IP:           _____________________                         │
│ NAS Share:        \\<NAS-IP>\chocolatey-repo                    │
├─────────────────────────────────────────────────────────────────┤
│ LXD Commands (Ubuntu Host):                                     │
│   lxc start win2022-choco                                       │
│   lxc stop win2022-choco                                        │
│   lxc console win2022-choco --type=vga                          │
│   lxc list                                                      │
├─────────────────────────────────────────────────────────────────┤
│ IIS Commands (Windows):                                         │
│   iisreset                                                      │
│   Restart-WebAppPool -Name "ChocolateyServer"                   │
│   Get-Website                                                   │
├─────────────────────────────────────────────────────────────────┤
│ Client Setup:                                                   │
│   choco source add --name="internal" `                          │
│     --source="http://<IP>/chocolatey"                           │
│   choco apikey add --source="http://<IP>/chocolatey" `          │
│     --key="<API-KEY>"                                           │
├─────────────────────────────────────────────────────────────────┤
│ Test Commands:                                                  │
│   curl http://<IP>/nuget/Packages                               │
│   choco search * --source="internal"                            │
│   choco push package.nupkg --source="internal"                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Document Version

- Version: 1.0
- Last Updated: February 2025
- Environment: Air-Gapped / Offline Deployment
