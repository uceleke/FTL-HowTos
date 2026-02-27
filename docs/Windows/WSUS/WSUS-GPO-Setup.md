# WSUS GPO Configuration Guide
## Pushing Updates with Controlled Reboots — Day-Staggered for Servers, Scheduled for Workstations

---

## Table of Contents

1. [Prerequisites](#1-prerequisites)
2. [WSUS Server Setup Overview](#2-wsus-server-setup-overview)
3. [GPO Architecture Planning](#3-gpo-architecture-planning)
4. [Creating the Base WSUS GPO](#4-creating-the-base-wsus-gpo)
5. [Configuring the Workstation Update GPO](#5-configuring-the-workstation-update-gpo)
6. [Configuring the Server Update GPO (Staggered Reboots)](#6-configuring-the-server-update-gpo-staggered-reboots)
7. [Linking GPOs to OUs](#7-linking-gpos-to-ous)
8. [Staggered Reboot Strategy for Servers](#8-staggered-reboot-strategy-for-servers)
9. [Verification and Testing](#9-verification-and-testing)
10. [Monitoring and Troubleshooting](#10-monitoring-and-troubleshooting)
11. [Reference: Key Policy Settings Cheat Sheet](#11-reference-key-policy-settings-cheat-sheet)

---

## 1. Prerequisites

Before configuring GPOs, ensure the following are in place:

- **Windows Server Update Services (WSUS)** is installed and configured on a server (Windows Server 2016/2019/2022 recommended)
- **Active Directory Domain Services (AD DS)** is operational with appropriate OUs created for Servers and Workstations
- **Group Policy Management Console (GPMC)** is installed on your management workstation
- **WSUS synchronization** has been configured and at least one sync has completed successfully
- **WSUS computer groups** have been created to match your intended targeting:
  - `All Workstations`
  - `Servers - Wave 1` (non-critical / test servers)
  - `Servers - Wave 2` (secondary/application servers)
  - `Servers - Tier0` (critical/production servers — manual reboot only)
- You have **Domain Admin** or equivalent Group Policy delegation rights
- WSUS server is reachable from all target machines (HTTP, default port **8530** or HTTPS **8531**)

---

## 2. WSUS Server Setup Overview

> If WSUS is already configured, you may skip to Section 3.

### 2.1 Install the WSUS Role

On your WSUS server, open PowerShell as Administrator:

```powershell
Install-WindowsFeature -Name UpdateServices, UpdateServices-UI -IncludeManagementTools
```

After installation, run the post-installation configuration:

```powershell
& 'C:\Program Files\Update Services\Tools\WsusUtil.exe' postinstall CONTENT_DIR="D:\WSUS"
```

> Replace `D:\WSUS` with your preferred content storage path. Ensure adequate disk space (minimum 100 GB recommended).

### 2.2 Configure WSUS via GUI

1. Open **Windows Server Update Services** from Server Manager → Tools
2. Complete the **WSUS Configuration Wizard**:
   - Choose upstream server (Microsoft Update or upstream WSUS)
   - Select languages to support
   - Select **Products** (e.g., Windows 10, Windows 11, Windows Server 2019, Windows Server 2022, Microsoft 365, etc.)
   - Select **Classifications** — recommended minimum:
     - Critical Updates
     - Security Updates
     - Update Rollups
     - Service Packs (optional)
3. Set a **synchronization schedule** (e.g., daily at 3:00 AM)
4. Run an initial manual synchronization

### 2.3 Create WSUS Computer Groups

In the WSUS console:

1. Expand **Computers → All Computers**
2. Right-click and choose **Add Computer Group** for each of the following:
   - `Workstations`
   - `Servers - Wave 1`
   - `Servers - Wave 2`
   - `Servers - Tier0`

> Computers will be assigned to these groups via GPO using **client-side targeting** (configured in Section 4).

### 2.4 Configure Auto-Approval Rules (Optional but Recommended)

1. In WSUS, go to **Options → Automatic Approvals**
2. Create a rule to auto-approve **Critical** and **Security** updates for the `Workstations` group after a 7-day delay
3. Leave Server groups for **manual approval** for change-control compliance

---

## 3. GPO Architecture Planning

Using multiple GPOs keeps settings modular, auditable, and easier to troubleshoot.

### Recommended GPO Structure

| GPO Name | Scope | Purpose |
|---|---|---|
| `WSUS - Base Settings` | Domain or top-level OU | Points all machines to the WSUS server |
| `WSUS - Workstation Update Policy` | Workstations OU | Scheduled install + fixed maintenance window reboot |
| `WSUS - Server Wave 1 Policy` | Servers Wave 1 OU | Tuesday 01:00 — install and auto-reboot |
| `WSUS - Server Wave 2 Policy` | Servers Wave 2 OU | Thursday 01:00 — install and auto-reboot |
| `WSUS - Server Tier0 Policy` | Servers Tier0 OU | Download and install only — **no automatic reboot** |

### OU Structure (Example)

```
domain.local
├── Workstations
│   ├── Desktops
│   └── Laptops
└── Servers
    ├── Wave1-NonCritical
    ├── Wave2-Application
    └── Tier0-Critical
```

---

## 4. Creating the Base WSUS GPO

This GPO points all machines to the WSUS server and enables client-side targeting. It should be linked at the **domain level** or a high-level OU so all managed machines inherit it.

### 4.1 Create the GPO

1. Open **Group Policy Management Console (GPMC)**: `gpmc.msc`
2. Right-click your domain or top-level OU → **Create a GPO in this domain, and Link it here...**
3. Name it: `WSUS - Base Settings`
4. Right-click the GPO → **Edit**

### 4.2 Navigate to Windows Update Policies

In the Group Policy Management Editor:

```
Computer Configuration
  └── Policies
        └── Administrative Templates
              └── Windows Components
                    └── Windows Update
                          └── Manage updates offered from Windows Server Update Service
```

> **Note:** In Windows 11 / Server 2022 environments, some settings have moved to:
> `Windows Update for Business` — verify which node applies to your OS targets.

### 4.3 Configure Base Settings

#### Specify Intranet Microsoft Update Service Location

- **Setting:** `Specify intranet Microsoft update service location`
- **State:** Enabled
- **Set the intranet update service for detecting updates:** `http://wsus-server.domain.local:8530`
- **Set the intranet statistics server:** `http://wsus-server.domain.local:8530`

> Replace `wsus-server.domain.local` with your actual WSUS server FQDN. Use `8531` if using HTTPS.

#### Enable Client-Side Targeting

- **Setting:** `Enable client-side targeting`
- **State:** Enabled
- **Target group name for this computer:** `Workstations`

> This is a **base default**. The Server GPOs will override this with the appropriate Wave group name.

#### Do Not Connect to Windows Update Internet Locations

- **Setting:** `Do not connect to any Windows Update Internet locations`
- **State:** Enabled

> This prevents managed machines from bypassing WSUS and going directly to Microsoft.

#### Allow Signed Updates From an Intranet Service (Optional)

- **Setting:** `Allow signed updates from an intranet Microsoft update service location`
- **State:** Enabled

> Required if you use third-party patch management tools that sign their own updates.

---

## 5. Configuring the Workstation Update GPO

Workstations use a **fixed maintenance window** — they download and install updates automatically, and reboot only during the designated window (e.g., **02:00 – 04:00**).

### 5.1 Create the GPO

1. In GPMC, right-click the **Workstations OU** → **Create a GPO in this domain, and Link it here...**
2. Name it: `WSUS - Workstation Update Policy`
3. Right-click → **Edit**

### 5.2 Navigate to Policies

```
Computer Configuration
  └── Policies
        └── Administrative Templates
              └── Windows Components
                    └── Windows Update
```

### 5.3 Configure Update Settings

#### Configure Automatic Updates

- **Setting:** `Configure Automatic Updates`
- **State:** Enabled
- **Configure automatic updating:** `4 - Auto download and schedule the install`
- **Scheduled install day:** `0 - Every day` *(or choose a specific day, e.g., `3 - Every Wednesday`)*
- **Scheduled install time:** `02:00`

> The install time defines when updates are applied. Reboots must be further constrained (see below).

#### No Auto-Restart with Logged-On Users (for desktop scenarios)

- **Setting:** `No auto-restart with logged on users for scheduled automatic updates installations`
- **State:** Disabled

> Set to **Disabled** to allow reboots to occur during the maintenance window even if a user is somehow logged in on a workstation outside of hours. Adjust to **Enabled** if you have 24/7 shift workstations.

#### Specify Active Hours (Maintenance Window — Workstations)

Navigate to:

```
Windows Update
  └── Legacy Policies (or "Manage end user experience")
```

- **Setting:** `Turn off auto-restart for updates during active hours`
- **State:** Enabled
- **Start:** `08:00` (beginning of business hours — no reboots allowed from here)
- **End:** `22:00` (end of evening — reboots allowed between 22:00–08:00)

> This defines a **no-reboot window** during business hours. Windows will only reboot between **22:00 and 08:00**, aligning with your 02:00 install schedule.

#### Always Automatically Restart at the Scheduled Time

- **Setting:** `Always automatically restart at the scheduled time`
- **State:** Enabled
- **The restart will execute automatically within the following number of minutes:** `15`

> This ensures machines reboot promptly after the install completes rather than waiting indefinitely for a user-triggered restart.

#### Re-prompt for Restart with Scheduled Installations

- **Setting:** `Re-prompt for restart with scheduled installations`
- **State:** Enabled
- **Wait the following period before prompting again for a scheduled restart (hours):** `1`

#### Delay Restart for Scheduled Installations

- **Setting:** `Delay Restart for scheduled installations`
- **State:** Enabled
- **Wait the following period (minutes) before proceeding with a scheduled restart:** `5`

### 5.4 Client-Side Targeting for Workstations

If not already set in the base GPO, ensure:

- **Setting:** `Enable client-side targeting`
- **State:** Enabled
- **Target group:** `Workstations`

---

## 6. Configuring the Server Update GPO (Staggered Reboots)

Servers use **day-staggered maintenance windows** — Wave 1 patches and reboots on Tuesday, Wave 2 on Thursday, and Tier 0 (critical) servers install updates automatically but **never reboot without manual intervention**. This gives you full change-control over your most critical systems while still keeping them patched.

You will create **three separate GPOs**, one per wave/tier.

---

### 6.1 Server Wave 1 GPO (Non-Critical / Test Servers)

**Target reboot window:** Tuesday 01:00 AM

#### Create and Edit GPO

1. Right-click **Servers\Wave1-NonCritical OU** → **Create a GPO and link here**
2. Name: `WSUS - Server Wave 1 Policy`
3. Edit the GPO

#### Settings

Navigate to:
```
Computer Configuration → Policies → Administrative Templates → Windows Components → Windows Update
```

| Setting | Value |
|---|---|
| Configure Automatic Updates | **Enabled** — Option 4 (Auto download and schedule install) |
| Scheduled install day | `3 - Every Tuesday` |
| Scheduled install time | `01:00` |
| No auto-restart with logged-on users | **Disabled** (allow forced reboot) |
| Always automatically restart at scheduled time | **Enabled**, 15 minutes |
| Turn off auto-restart during active hours | **Enabled** — Start: 06:00, End: 22:00 |
| Enable client-side targeting | **Enabled** — Group: `Servers - Wave 1` |
| Do not connect to Windows Update Internet locations | **Enabled** |

---

### 6.2 Server Wave 2 GPO (Application / Secondary Servers)

**Target reboot window:** Thursday 01:00 AM

> Wave 1 servers have been up and verified for two days by the time Wave 2 reboots, giving your team time to catch any issues before they affect application or secondary servers.

#### Create and Edit GPO

1. Right-click **Servers\Wave2-Application OU** → **Create a GPO and link here**
2. Name: `WSUS - Server Wave 2 Policy`
3. Edit the GPO

#### Settings

| Setting | Value |
|---|---|
| Configure Automatic Updates | **Enabled** — Option 4 |
| Scheduled install day | `5 - Every Thursday` |
| Scheduled install time | `01:00` |
| No auto-restart with logged-on users | **Disabled** |
| Always automatically restart at scheduled time | **Enabled**, 15 minutes |
| Turn off auto-restart during active hours | **Enabled** — Start: 06:00, End: 22:00 |
| Enable client-side targeting | **Enabled** — Group: `Servers - Wave 2` |
| Do not connect to Windows Update Internet locations | **Enabled** |

---

### 6.3 Server Tier 0 GPO (Critical / Production Servers — Manual Reboot)

**Install behavior:** Download and install updates automatically — **reboot on administrator schedule only**

> Tier 0 servers (domain controllers, production databases, critical infrastructure) receive updates automatically but the reboot is **never triggered by Windows Update**. An administrator must manually initiate the reboot during a planned change window after verifying service health and completing any required change management steps.

#### Create and Edit GPO

1. Right-click **Servers\Tier0-Critical OU** → **Create a GPO and link here**
2. Name: `WSUS - Server Tier0 Policy`
3. Edit the GPO

#### Settings

| Setting | Value |
|---|---|
| Configure Automatic Updates | **Enabled** — Option **4** (Auto download and schedule install) |
| Scheduled install day | `5 - Every Thursday` |
| Scheduled install time | `01:00` |
| No auto-restart with logged-on users | **Enabled** ← key setting, prevents auto-reboot if any session exists |
| Always automatically restart at scheduled time | **Disabled** |
| Turn off auto-restart during active hours | **Enabled** — Start: 00:00, End: 23:59 (blocks all automatic reboots 24/7) |
| Re-prompt for restart with scheduled installations | **Disabled** |
| Delay restart for scheduled installations | **Disabled** |
| Enable client-side targeting | **Enabled** — Group: `Servers - Tier0` |
| Do not connect to Windows Update Internet locations | **Enabled** |

> **How "No auto-restart with logged-on users" works for servers:** On a server, the System account and background services count as sessions. Setting this to **Enabled** effectively prevents Windows Update from ever initiating an automatic reboot since there is almost always an active session on a server. Combined with blocking active hours across the full 24-hour day, this creates a belt-and-suspenders approach ensuring no unplanned reboots occur.

#### Manual Reboot Procedure for Tier 0 Servers

After updates have been installed (verify via WSUS console or Event ID 19 in Event Viewer), follow your change management process and then reboot manually:

```powershell
# Check if a reboot is pending before scheduling
(New-Object -ComObject Microsoft.Update.SystemInfo).RebootRequired

# Graceful restart with a 5-minute warning to any connected sessions
shutdown /r /t 300 /c "Planned maintenance reboot — patch cycle complete"

# Or immediately if during a confirmed change window
Restart-Computer -Force
```

For Domain Controllers specifically, verify replication is healthy before and after:

```powershell
# Before reboot — confirm replication is clean
repadmin /replsummary
repadmin /showrepl

# After reboot — confirm DC is advertising correctly
nltest /dsgetdc:domain.local
dcdiag /test:replications
```

---

## 7. Linking GPOs to OUs

### 7.1 Verify OU Links

In GPMC, confirm each GPO is linked correctly:

| GPO | Linked To |
|---|---|
| `WSUS - Base Settings` | Domain root or top-level OU |
| `WSUS - Workstation Update Policy` | `domain.local/Workstations` |
| `WSUS - Server Wave 1 Policy` | `domain.local/Servers/Wave1-NonCritical` |
| `WSUS - Server Wave 2 Policy` | `domain.local/Servers/Wave2-Application` |
| `WSUS - Server Tier0 Policy` | `domain.local/Servers/Tier0-Critical` |

### 7.2 Set GPO Link Order and Enforcement

- Ensure the **Wave-specific GPO** has a **higher priority (lower link order number)** than the Base GPO so its `client-side targeting` and `install schedule` settings take precedence
- If needed, right-click a Wave GPO → **Enforced** to prevent child OUs from overriding it

### 7.3 Configure GPO Security Filtering

By default, GPOs apply to **Authenticated Users**. For server GPOs, you may want to scope more tightly:

1. Select the GPO in GPMC
2. In the **Scope** tab → **Security Filtering**
3. Remove `Authenticated Users`
4. Add the relevant **computer group** or **AD security group** containing those servers

> This prevents the GPO from applying to admin workstations that might sit in the same OU by accident.

---

## 8. Staggered Reboot Strategy for Servers

### 8.1 Why Stagger?

Simultaneous reboots of all servers can cause:

- **Service outages** — backend services going down before front-end servers finish rebooting
- **Authentication failures** — domain controllers or LDAP servers briefly unavailable
- **Database connectivity drops** — app servers rebooting while DB is still applying patches
- **Monitoring alert storms** — overwhelming your on-call team

### 8.2 Recommended Wave Timing

| Wave | Server Type | Patch Install Day/Time | Expected Reboot |
|---|---|---|---|
| Wave 1 | Test / Dev / Non-critical | Tuesday 01:00 | ~01:05–01:20 Tuesday |
| Wave 2 | Application / Secondary | Thursday 01:00 | ~01:05–01:20 Thursday |
| Tier 0 | Critical / Production / Domain Controllers | Thursday 01:00 (install only) | **Manual — change window** |

> Staggering by **days rather than hours** gives your team meaningful time to validate Wave 1 health before Wave 2 reboots, and ensures Tier 0 servers are always patched but never rebooted without explicit sign-off. Any patch-related issues that surface on Tuesday non-critical servers can be assessed and addressed before Thursday's wave begins.

### 8.3 Domain Controller Special Considerations

Domain Controllers require extra care and should all be placed in the **Tier 0** group so they never auto-reboot:

- **Never reboot all DCs simultaneously.** Stagger DC reboots manually during the change window — reboot secondaries first, verify replication, then reboot the PDC Emulator last.
- Confirm AD replication health **before rebooting any DC**: `repadmin /replsummary`
- Allow at least 15 minutes between each DC reboot to confirm the domain is healthy
- The PDC Emulator should always be the **last** DC to reboot

Example DC manual reboot order during a Tier 0 change window:

| DC | Reboot Order | Notes |
|---|---|---|
| DC01 (secondary) | 1st | Verify replication before proceeding |
| DC02 (secondary) | 2nd | Verify replication before proceeding |
| DC03 (PDC Emulator) | Last | Confirm full AD health after reboot |

### 8.4 Adding an Extra Randomization Delay (Optional)

For large environments with many servers in a single wave, Windows Update can be configured to add a random delay to prevent all machines in a wave from rebooting at the exact same second. This is achieved via:

```
Computer Configuration → Policies → Administrative Templates → Windows Components → Windows Update
```

- **Setting:** `Update Power Policy for Cart Restarts` *(if using Modern Policy)*

Or via a startup/shutdown script using PowerShell that introduces a random sleep:

```powershell
# Random delay of 0-10 minutes before initiating a triggered restart
$delay = Get-Random -Minimum 0 -Maximum 600
Start-Sleep -Seconds $delay
Restart-Computer -Force
```

> Deploy this as a **Computer Startup Script** via GPO only on very large server waves where you have 20+ machines that could reboot simultaneously.

---

## 9. Verification and Testing

### 9.1 Force GPO Refresh on Target Machines

After linking GPOs, refresh policy on test machines:

```powershell
# Run on the target machine or remotely
gpupdate /force

# Remote GPO refresh (requires WinRM)
Invoke-GPUpdate -Computer "server01.domain.local" -Force
```

### 9.2 Confirm GPO Application

On a target machine, run:

```powershell
gpresult /r /scope computer
```

Look for your WSUS GPOs listed under **Applied Group Policy Objects**.

For a detailed HTML report:

```powershell
gpresult /H C:\Temp\GPReport.html
Start-Process C:\Temp\GPReport.html
```

### 9.3 Verify Registry Keys Are Set

The Windows Update GPO settings write to the registry. Confirm the values on a target machine:

```powershell
Get-ItemProperty -Path "HKLM:\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate" |
  Select-Object WUServer, WUStatusServer, TargetGroup, TargetGroupEnabled

Get-ItemProperty -Path "HKLM:\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate\AU" |
  Select-Object AUOptions, ScheduledInstallDay, ScheduledInstallTime, NoAutoRebootWithLoggedOnUsers
```

**Expected registry values for a Wave 2 server:**

| Registry Value | Key | Expected |
|---|---|---|
| `WUServer` | `WindowsUpdate` | `http://wsus-server:8530` |
| `TargetGroup` | `WindowsUpdate` | `Servers - Wave 2` |
| `AUOptions` | `AU` | `4` |
| `ScheduledInstallDay` | `AU` | `5` (Thursday) |
| `ScheduledInstallTime` | `AU` | `60` (01:00 in minutes) |
| `NoAutoRebootWithLoggedOnUsers` | `AU` | `0` |

**Expected registry values for a Tier 0 server:**

| Registry Value | Key | Expected |
|---|---|---|
| `WUServer` | `WindowsUpdate` | `http://wsus-server:8530` |
| `TargetGroup` | `WindowsUpdate` | `Servers - Tier0` |
| `AUOptions` | `AU` | `4` |
| `ScheduledInstallDay` | `AU` | `5` (Thursday) |
| `ScheduledInstallTime` | `AU` | `60` (01:00 in minutes) |
| `NoAutoRebootWithLoggedOnUsers` | `AU` | `1` ← must be 1 |

### 9.4 Verify WSUS Reports the Machine

In the WSUS console:

1. Go to **Reports → Computer Status**
2. Run a report filtered to the appropriate computer group
3. Confirm machines appear and show a **Last Reported** timestamp within the last 24 hours

> If machines don't appear, check network connectivity to the WSUS server on port 8530, and confirm the `wuauserv` service is running.

### 9.5 Test on a Non-Production Machine First

Before rolling out broadly:

1. Place one test workstation and one test server in Wave 1
2. Approve a non-critical update in WSUS for those groups
3. Wait for the scheduled window or trigger manually:
   ```powershell
   wuauclt /detectnow
   wuauclt /reportnow
   ```
   Or on newer Windows versions:
   ```powershell
   UsoClient StartScan
   UsoClient StartDownload
   UsoClient StartInstall
   ```
4. Observe that the reboot occurs **only** within the configured maintenance window

---

## 10. Monitoring and Troubleshooting

### 10.1 Windows Update Log

```powershell
# Generate a readable WindowsUpdate.log (Windows 10/11/Server 2016+)
Get-WindowsUpdateLog -LogPath C:\Temp\WindowsUpdate.log
notepad C:\Temp\WindowsUpdate.log
```

### 10.2 Event Viewer — Key Event IDs

Check **Event Viewer → Windows Logs → System** and filter for source `Windows Update Client` or `Microsoft-Windows-WindowsUpdateClient`:

| Event ID | Meaning |
|---|---|
| 19 | Update successfully installed |
| 20 | Installation failure |
| 21 | Restart required to complete installation |
| 22 | Reboot is scheduled |
| 43 | Installation started |
| 44 | Update download started |

### 10.3 Check WSUS-Side Logs

On the WSUS server, logs are located at:

```
C:\Program Files\Update Services\LogFiles\Change.log
C:\Program Files\Update Services\LogFiles\SoftwareDistribution.log
```

### 10.4 Common Issues and Fixes

| Issue | Likely Cause | Fix |
|---|---|---|
| Machine not reporting to WSUS | Wrong WUServer URL or port blocked | Verify registry, check firewall on port 8530 |
| GPO not applying | Machine not in correct OU | Move computer object to correct OU; run `gpupdate /force` |
| Machine still targeting Microsoft Update | Base GPO not linked / processing | Check GPO link, run `gpresult /r` |
| Reboots happening during business hours | Active hours not configured | Set `Turn off auto-restart during active hours` |
| Wrong WSUS group | Client-side targeting group name mismatch | Verify exact group name in WSUS matches GPO value |
| Updates not installing on schedule | `AUOptions` not set to 4 | Verify registry value and GPO precedence |

### 10.5 Useful PowerShell Commands

```powershell
# Check current Windows Update settings
Get-Service wuauserv | Select-Object Status, StartType

# Force update detection
UsoClient StartScan

# Check pending reboots
(New-Object -ComObject Microsoft.Update.SystemInfo).RebootRequired

# View update history
Get-HotFix | Sort-Object -Property InstalledOn -Descending | Select-Object -First 20
```

---

## 11. Reference: Key Policy Settings Cheat Sheet

### Path: Computer Configuration → Policies → Administrative Templates → Windows Components → Windows Update

| Policy Setting | Workstations | Wave 1 Servers | Wave 2 Servers | Tier 0 Servers |
|---|---|---|---|---|
| Configure Automatic Updates | Enabled (4) | Enabled (4) | Enabled (4) | Enabled (4) |
| Scheduled install day | Every Wednesday (4) | Every Tuesday (3) | Every Thursday (5) | Every Thursday (5) |
| Scheduled install time | 02:00 | 01:00 | 01:00 | 01:00 |
| Specify intranet WSUS location | `https://wsus:8531` | `https://wsus:8531` | `https://wsus:8531` | `https://wsus:8531` |
| Enable client-side targeting | Workstations | Servers - Wave 1 | Servers - Wave 2 | Servers - Tier0 |
| No auto-restart with logged-on users | Disabled | Disabled | Disabled | **Enabled** |
| Always auto-restart at scheduled time | Enabled (15 min) | Enabled (15 min) | Enabled (15 min) | **Disabled** |
| Active hours (no-reboot window) | 08:00–22:00 | 06:00–22:00 | 06:00–22:00 | **00:00–23:59 (full day)** |
| Do not connect to Windows Update | Enabled | Enabled | Enabled | Enabled |

---

## Summary: Full Weekly Patch Cycle Timeline

```
Weekly Patch Cycle — Tuesday through Friday
─────────────────────────────────────────────────────────────────────────
TUESDAY
01:00      │ Wave 1 (non-critical servers) install and auto-reboot
           │   → Expected back online by ~01:20
06:00      │ Active hours begin — no further auto-reboots
           │   → Team verifies Wave 1 server health during business day
─────────────────────────────────────────────────────────────────────────
WEDNESDAY
22:00      │ Workstation reboot window opens
02:00 Thu  │ Workstations install and auto-reboot (~02:05–02:20)
─────────────────────────────────────────────────────────────────────────
THURSDAY
01:00      │ Wave 2 (application servers) install and auto-reboot
           │   → Expected back online by ~01:20
01:00      │ Tier 0 servers install updates — NO automatic reboot
           │   → Pending reboot flag set; awaiting admin action
06:00      │ Active hours begin — team verifies Wave 2 health
─────────────────────────────────────────────────────────────────────────
FRIDAY (or planned change window)
           │ Admin confirms Tier 0 patch status in WSUS console
           │ Change ticket raised and approved
           │ DCs rebooted manually in order: secondary → secondary → PDC
           │ Other Tier 0 servers rebooted per service dependency order
           │ Full health check completed
─────────────────────────────────────────────────────────────────────────
```

---

*Guide written for environments running Windows Server 2016/2019/2022 (WSUS) with Windows 10/11 workstations joined to Active Directory. Always test in a non-production environment before rolling out changes domain-wide.*
