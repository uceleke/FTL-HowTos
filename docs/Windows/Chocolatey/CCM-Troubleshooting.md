# Chocolatey Central Management (CCM) — HTTP 500 Troubleshooting Guide

## Overview

An HTTP 500 error on the CCM web interface with no application logs typically means the failure is occurring too early in the startup process for the app logger to capture it. Work through the steps below in order.

---

## Step 1: Enable ASP.NET Core stdout Logging

These errors happen very early in application execution and are not logged to the standard log location. Enable stdout logging in IIS to capture the actual error.

Edit `C:\tools\chocolatey-management-web\web.config` and set `stdoutLogEnabled` to `true`:

```xml
<aspNetCore stdoutLogEnabled="true" stdoutLogFile=".\logs\stdout" ... />
```

After saving, reproduce the error by loading the CCM page, then check:

```
C:\tools\chocolatey-management-web\logs\
```

Look for a `stdout_*.log` file — this will contain the real error message.

> **Remember to set `stdoutLogEnabled` back to `false` after troubleshooting.**

---

## Step 2: Verify the Correct Log File Location

Make sure you are checking the right log path for your CCM version.

| CCM Version | Log Path |
|---|---|
| v0.2.0 and later | `C:\tools\chocolatey-management-web\App_Data\Logs\ccm-website.log` |
| Prior to v0.2.0 | `C:\tools\chocolatey-management-web\App_Data\Logs\Logs.txt` |

---

## Step 3: Check for ASP.NET Core Module Mismatch (Error 500.21)

CCM's website is locked to specific versions of the ASP.NET Core runtime and hosting module. A version mismatch will produce a 500.21 and nothing in the app logs.

**Verify the AspNetCoreModuleV2 is registered in IIS:**

```powershell
Get-WebConfiguration "system.webServer/globalModules" | Where-Object { $_.name -like "*AspNetCore*" }
```

**Verify required packages are installed:**

```powershell
choco list --local-only dotnet-aspnetcoremodule-v2
choco list --local-only dotnet-8.0-runtime
choco list --local-only dotnet-8.0-aspnetruntime
```

Ensure versions are consistent. CCM requires the following IIS features as well:

```powershell
# Check IIS-ApplicationInit (required by CCM)
Get-WindowsOptionalFeature -Online -FeatureName IIS-ApplicationInit

# Check IIS-WebServer
Get-WindowsOptionalFeature -Online -FeatureName IIS-WebServer
```

If `IIS-ApplicationInit` is not enabled, install it:

```powershell
choco install IIS-ApplicationInit --source windowsfeatures --no-progress -y
```

---

## Step 4: Check IIS Application Pool

A stopped or misconfigured app pool will cause a 500 with no app-level logging.

```powershell
Import-Module WebAdministration

# Check CCM app pool state and identity
Get-WebConfiguration "system.applicationHost/applicationPools/add" |
  Where-Object { $_.name -like "*chocolatey*" } |
  Select-Object name, state, @{n="Identity";e={$_.processModel.userName}}
```

**Things to verify:**
- App pool is in **Started** state
- Identity account has read access to `C:\tools\chocolatey-management-web\`
- App pool is set to **No Managed Code** (CCM is ASP.NET Core, not classic .NET)

**Restart the app pool if needed:**

```powershell
Restart-WebAppPool -Name "ChocolateyCentralManagement"
```

---

## Step 5: Test SQL Server Connectivity

A database connection failure at startup will produce a 500 with nothing in the app logs, since the app cannot initialize at all.

**Check the connection string in appsettings.json:**

```powershell
Get-Content "C:\tools\chocolatey-management-web\appsettings.json" |
  Select-String "ConnectionString"
```

**Test connectivity manually:**

```powershell
$conn = New-Object System.Data.SqlClient.SqlConnection
$conn.ConnectionString = "Server=<YOUR_SQL_SERVER>;Database=ChocolateyManagement;Integrated Security=True"
try {
    $conn.Open()
    Write-Host "Connection successful. State: $($conn.State)" -ForegroundColor Green
    $conn.Close()
} catch {
    Write-Host "Connection failed: $_" -ForegroundColor Red
}
```

**Things to verify:**
- SQL Server service is running
- The CCM app pool identity has `db_owner` or appropriate rights on the `ChocolateyManagement` database
- Firewall is not blocking the SQL port (default: 1433)
- If using a named instance, ensure SQL Server Browser service is running

---

## Step 6: Fix SQL Login for IIS App Pool Identity

If the SQL connectivity test fails, the most likely cause is that the SQL login for the app pool identity was never created or was removed. CCM uses Windows/integrated authentication by default — switching app pool identities does **not** automatically create SQL logins.

### 6a: Find the SQL Instance Name

Before running sqlcmd, confirm what instance is running:

```powershell
# Check SQL service names to identify the instance
Get-Service | Where-Object { $_.Name -like "MSSQL*" } | Select-Object Name, DisplayName, Status

# Check via registry
Get-ItemProperty "HKLM:\SOFTWARE\Microsoft\Microsoft SQL Server" -Name InstalledInstances
```

| Instance Type | sqlcmd Connection |
|---|---|
| Default instance (`MSSQLSERVER`) | `sqlcmd -S localhost` or `sqlcmd -S .` |
| Named instance (e.g. `SQLEXPRESS`) | `sqlcmd -S .\SQLEXPRESS` |

> **Verify you can see databases before proceeding:**
> ```cmd
> sqlcmd -Q "SELECT name FROM sys.databases;"
> ```

### 6b: Use the Default App Pool Identity (Recommended)

The CCM installer sets up SQL permissions for `IIS APPPOOL\ChocolateyCentralManagement` by default. Reverting to this identity is the easiest fix.

**Switch the app pool back to ApplicationPoolIdentity:**

```powershell
Import-Module WebAdministration
$appPool = Get-Item "IIS:\AppPools\ChocolateyCentralManagement"
$appPool.processModel.identityType = "ApplicationPoolIdentity"
$appPool | Set-Item

iisreset /restart
```

**Verify the SQL login exists:**

```cmd
sqlcmd -Q "SELECT name, type_desc, is_disabled FROM sys.server_principals WHERE name LIKE '%ChocolateyC%';"
```

**If the login is missing, create it:**

```cmd
sqlcmd -Q "USE [master]; CREATE LOGIN [IIS APPPOOL\ChocolateyCentralManagement] FROM WINDOWS;"

sqlcmd -d ChocolateyManagement -Q "CREATE USER [IIS APPPOOL\ChocolateyCentralManagement] FOR LOGIN [IIS APPPOOL\ChocolateyCentralManagement]; ALTER ROLE [db_owner] ADD MEMBER [IIS APPPOOL\ChocolateyCentralManagement];"
```

> The square brackets around `IIS APPPOOL\ChocolateyCentralManagement` are required due to the space in the name.

**Verify the login and role assignment:**

```cmd
sqlcmd -d ChocolateyManagement -Q "SELECT dp.name, rp.name AS role FROM sys.database_principals dp JOIN sys.database_role_members drm ON dp.principal_id = drm.member_principal_id JOIN sys.database_principals rp ON drm.role_principal_id = rp.principal_id WHERE dp.name LIKE '%Chocolatey%';"
```

### 6c: Using a Domain Service Account Instead (Optional)

If you prefer to run the app pool under a domain service account rather than the virtual app pool identity:

**1. Update the app pool identity:**

```powershell
Import-Module WebAdministration
$appPool = Get-Item "IIS:\AppPools\ChocolateyCentralManagement"
$appPool.processModel.identityType = "SpecificUser"
$appPool.processModel.userName = "DOMAIN\svc.account"
$appPool.processModel.password = "YourPasswordHere"
$appPool | Set-Item
```

**2. Create the SQL login for the service account:**

```cmd
sqlcmd -Q "USE [master]; CREATE LOGIN [DOMAIN\svc.account] FROM WINDOWS;"

sqlcmd -d ChocolateyManagement -Q "CREATE USER [DOMAIN\svc.account] FOR LOGIN [DOMAIN\svc.account]; ALTER ROLE [db_owner] ADD MEMBER [DOMAIN\svc.account];"
```

**3. Restart IIS and verify:**

```cmd
iisreset /restart
```

```cmd
sqlcmd -d ChocolateyManagement -Q "SELECT dp.name, rp.name AS role FROM sys.database_principals dp JOIN sys.database_role_members drm ON dp.principal_id = drm.member_principal_id JOIN sys.database_principals rp ON drm.role_principal_id = rp.principal_id WHERE dp.name LIKE '%svc%';"
```

---

## Step 7: Check Windows Event Logs

IIS and .NET errors that don't make it into app logs often surface in the Windows Event Log.

```powershell
# IIS and ASP.NET errors
Get-EventLog -LogName Application -Source "IIS*","W3SVC*","ASP.NET*" -Newest 20 |
  Format-List TimeGenerated, Source, EntryType, Message

# System-level errors
Get-EventLog -LogName System -EntryType Error -Newest 10 |
  Format-List TimeGenerated, Source, Message
```

Also check **Event Viewer** under:
- `Windows Logs > Application`
- `Windows Logs > System`

Filter by **Error** and **Warning** around the time you accessed the CCM site.

---

## Step 8: Known Version-Specific Issues

### CCM v0.6.0 and v0.6.1 — In-Process Hosting Bug

The IIS hosting model changed in v0.6.0 to use the in-process model, which can cause startup failures. This is a known bug resolved in **v0.6.2**.

**Check your installed version:**

```powershell
choco list --local-only chocolatey-management-web
```

If you are on v0.6.0 or v0.6.1, upgrade:

```powershell
choco upgrade chocolatey-management-web -y --source="'<YOUR_INTERNAL_REPO>'"
```

### All Versions — Ensure All CCM Components Are the Same Version

CCM's three packages must always be on the same version:

```powershell
choco list --local-only chocolatey-management-database
choco list --local-only chocolatey-management-service
choco list --local-only chocolatey-management-web
```

Mixed versions are unsupported and will cause unpredictable errors.

---

## Step 9: Restart All CCM Components

After making any changes, do a clean restart of all CCM components:

```powershell
# Stop all Chocolatey services
Get-Service chocolatey-* | Stop-Service -Force

# Kill the web process if still running
Get-Process -Name "ChocolateySoftware.ChocolateyManagement.Web.Mvc" `
  -ErrorAction SilentlyContinue | Stop-Process -Force

# Restart IIS
iisreset /restart

# Start Chocolatey services
Get-Service chocolatey-* | Start-Service

# Verify services are running
Get-Service chocolatey-* | Select-Object Name, Status
```

---

## Agent Registration HTTP 500 (register-c4bendpoint.ps1)

If you get a 500 error running `register-c4bendpoint.ps1` on an endpoint, this hits the CCM **service** endpoint (port 24020), not the website. The troubleshooting path is different from the IIS 500 above.

### Check the CCM Service Log First

Run this on the CCM server immediately after a failed registration attempt:

```powershell
Get-Content "$env:ChocolateyInstall\logs\ccm-service.log" -Tail 50
```

This will usually point directly at the cause.

---

### Cause 1: CCM Service Not Running

```powershell
Get-Service chocolatey-management-service
```

If stopped:

```powershell
Start-Service chocolatey-management-service
```

Retry the registration script after confirming the service is running.

---

### Cause 2: Certificate Thumbprint Mismatch

The agent and CCM service communicate over a certificate. If the thumbprint configured on the endpoint doesn't match what CCM is bound to, registration will fail with a 500.

**Check the thumbprint on the endpoint:**

```powershell
choco config get centralManagementCertificateThumbprint
```

**Check what cert CCM service is actually bound to on the server:**

```cmd
netsh http show sslcert | findstr /A 5 "4430"
```

The thumbprints must match. If not, correct it on the endpoint:

```powershell
choco config set centralManagementCertificateThumbprint "<CORRECT_THUMBPRINT>"
```

---

### Cause 3: Agent Pointing at Wrong CCM URL

```powershell
choco config get centralManagementServiceUrl
```

Should resolve to:

```
https://<ccm-server-fqdn>:24020/ChocolateyManagementService
```

If wrong, correct it:

```powershell
choco config set centralManagementServiceUrl "https://chocoserver.domain.com:24020/ChocolateyManagementService"
```

---

### Cause 4: Communication Salt Mismatch

If your CCM install has a communication salt configured, every endpoint must have the matching value set.

**Check if a salt is configured on the CCM server:**

```powershell
Get-Content "$env:ChocolateyInstall\config\chocolatey.config" | Select-String "Salt"
```

**Set the matching salt on the endpoint:**

```powershell
choco config set centralManagementClientCommunicationSaltAdditivePassword "<SALT_VALUE>"
```

---

### Cause 5: Version Incompatibility

The Chocolatey Agent and CCM Service must be on compatible versions. Mixed versions will cause communication failures.

**Check versions:**

```powershell
# On the CCM server
choco list --local-only chocolatey-management-service

# On the endpoint
choco list --local-only chocolatey-agent
```

Refer to the [CCM Component Compatibility Matrix](https://docs.chocolatey.org/en-us/central-management/#ccm-component-compatibility-matrix) to confirm your versions are compatible. If not, upgrade the agent on the endpoint:

```powershell
choco upgrade chocolatey-agent -y --source="'<YOUR_INTERNAL_REPO>'"
```

---

### Agent Registration Quick Checklist

| Check | Command |
|---|---|
| CCM service running | `Get-Service chocolatey-management-service` |
| Service log errors | `Get-Content "$env:ChocolateyInstall\logs\ccm-service.log" -Tail 50` |
| Agent CCM URL | `choco config get centralManagementServiceUrl` |
| Certificate thumbprint | `choco config get centralManagementCertificateThumbprint` |
| Agent version | `choco list --local-only chocolatey-agent` |
| CCM service version | `choco list --local-only chocolatey-management-service` |

---

## register-c4bendpoint.ps1 — 'AdditionalFeatures' Parameter Error

This error occurs when running the bootstrap/registration script on an endpoint where Chocolatey is not yet installed. The script is responsible for installing Chocolatey, the agent, and registering with CCM in one shot.

```
Parameter does not match 'AdditionalFeatures'
&([scriptblock]::Create($script)) @params
```

This is almost always a version mismatch between the `register-c4bendpoint.ps1` script you are running and the packages currently in your Nexus repo — commonly triggered after moving the repo to a new location.

---

### Step 1: Inspect the Script's Parameter Block

Check what parameters the script you have actually accepts:

```powershell
Get-Content "<path-to-register-c4bendpoint.ps1>" | Select-String "param" -Context 0,20
```

Then find exactly where `AdditionalFeatures` appears:

```powershell
Get-Content "<path-to-register-c4bendpoint.ps1>" | Select-String "AdditionalFeatures"
```

---

### Step 2: Dump the Params Hashtable at Runtime

Add this line immediately before the failing `&([scriptblock]` line to see exactly what is being passed:

```powershell
$params | Format-Table -AutoSize
```

This confirms which keys are in the hashtable and which one is causing the mismatch.

---

### Step 3: Check the Script Version

Look for a version comment or date at the top of the script:

```powershell
Get-Content "<path-to-register-c4bendpoint.ps1>" -TotalCount 20
```

---

### Step 4: Get a Fresh Copy of the Script (Most Likely Fix)

The `register-c4bendpoint.ps1` script ships with `chocolatey.extension` and is version-specific to your CCM install. A stale script from before the repo move will have a mismatched parameter signature.

**Option A — Download fresh from Nexus:**

```
https://chocoserver.domain.com:8443/repository/choco-install/register-c4bendpoint.ps1
```

**Option B — Generate from the CCM Web UI (guaranteed to match your CCM version):**

1. Log into CCM at `https://chocoserver.domain.com:8443`
2. Go to **Administration** → **Deploy Chocolatey Agent**
3. Copy or download the generated script — it will have the correct parameters for your exact CCM version

---

### Step 5: Verify CCM Version on the Server

```powershell
choco list --local-only chocolatey-management-web
choco list --local-only chocolatey.extension
```

The script must match the `chocolatey.extension` version. If the extension was upgraded after the repo move, any pre-existing copy of `register-c4bendpoint.ps1` is potentially stale.

---

## Quick Reference: Key File & Log Paths

| Item | Path |
|---|---|
| CCM Website root | `C:\tools\chocolatey-management-web\` |
| CCM Website log | `C:\tools\chocolatey-management-web\App_Data\Logs\ccm-website.log` |
| CCM Service log | `$env:ChocolateyInstall\logs\ccm-service.log` |
| appsettings.json | `C:\tools\chocolatey-management-web\appsettings.json` |
| web.config | `C:\tools\chocolatey-management-web\web.config` |
| stdout logs (temp) | `C:\tools\chocolatey-management-web\logs\stdout_*.log` |

---

## Information to Collect Before Contacting Support

If the above steps do not resolve the issue, gather the following before escalating:

```powershell
# CCM component versions
choco list --local-only | Select-String "chocolatey-management"

# .NET runtime versions
choco list --local-only | Select-String "dotnet"

# IIS modules
Get-WebConfiguration "system.webServer/globalModules" | Select-Object name

# App pool config
Import-Module WebAdministration
Get-WebConfiguration "system.applicationHost/applicationPools/add" |
  Where-Object { $_.name -like "*chocolatey*" } |
  Select-Object name, state, managedRuntimeVersion,
    @{n="Identity";e={$_.processModel.userName}}
```

Provide the contents of:
- `ccm-website.log`
- `stdout_*.log` (if captured)
- Relevant Windows Event Log entries
