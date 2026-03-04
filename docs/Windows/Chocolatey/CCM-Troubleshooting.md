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

## Step 6: Check Windows Event Logs

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

## Step 7: Known Version-Specific Issues

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

## Step 8: Restart All CCM Components

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
