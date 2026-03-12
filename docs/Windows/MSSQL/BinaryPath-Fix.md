# MSSQLSERVER Service – Binary Path Fix Runbook

## Problem Summary

The MSSQLSERVER service fails to start because the `ImagePath` registry value points to an incorrect drive letter. This commonly occurs when a mounted virtual disk (e.g., from OpenShift) takes the drive letter that SQL Server's binary path references.

> **Important:** Unmounting the virtual disk alone does **not** fix the issue. The registry `ImagePath` value is static and must be manually corrected — it will not self-update after a reboot.

---

## Fix Options

### Option 1: Registry Editor (Most Direct)

The service binary path is stored at:

```
HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\MSSQLSERVER
```

1. Open `regedit.exe` as Administrator
2. Navigate to the key above
3. Find the **`ImagePath`** value
4. Edit it to reflect the correct drive letter:

```
"C:\Program Files\Microsoft SQL Server\MSSQL16.MSSQLSERVER\MSSQL\Binn\sqlservr.exe" -sMSSQLSERVER
```

> Adjust the version folder (`MSSQL16`) to match your SQL Server version — see version reference below.

---

### Option 2: SC Command (Elevated Command Prompt)

```cmd
sc config MSSQLSERVER binPath= "\"C:\Program Files\Microsoft SQL Server\MSSQL16.MSSQLSERVER\MSSQL\Binn\sqlservr.exe\" -sMSSQLSERVER"
```

> **Note:** The space after `binPath=` is required syntax for `sc config`.

Verify the change:

```cmd
sc qc MSSQLSERVER
```

---

### Option 3: PowerShell

```powershell
# Check current (broken) path first
$svc = Get-WmiObject Win32_Service -Filter "Name='MSSQLSERVER'"
$svc.PathName

# Fix via registry (more reliable for SQL Server's quoted path)
$regPath = "HKLM:\SYSTEM\CurrentControlSet\Services\MSSQLSERVER"
Set-ItemProperty -Path $regPath -Name "ImagePath" -Value '"C:\Program Files\Microsoft SQL Server\MSSQL16.MSSQLSERVER\MSSQL\Binn\sqlservr.exe" -sMSSQLSERVER'
```

---

## After Fixing – Start the Service

```cmd
net start MSSQLSERVER
```

or

```cmd
sc start MSSQLSERVER
```

---

## Recommended Order of Operations

When the drive letter conflict is caused by a mounted virtual disk (e.g., OpenShift):

1. **Correct the `ImagePath`** registry value to the right drive letter first
2. **Unmount the virtual disk** in OpenShift
3. **Reboot** the server
4. **Confirm** MSSQLSERVER starts cleanly
5. **Check the SQL Server Error Log** to validate a clean startup

> Do not reboot before correcting the registry — the service will fail again even after the disk is unmounted.

---

## Preventing Future Drive Letter Conflicts

- After unmounting, verify the freed drive letter isn't being reclaimed by another volume in **Disk Management**
- If needed, explicitly reserve the SQL Server drive letter using Disk Management or `diskpart`
- Configure any OpenShift-mounted virtual disks to use a **specific non-conflicting drive letter** to avoid recurrence

---

## Reference

### Verify the Binary Exists at the Target Path

```cmd
dir "C:\Program Files\Microsoft SQL Server\MSSQL16.MSSQLSERVER\MSSQL\Binn\sqlservr.exe"
```

### SQL Server Version Folder Reference

| SQL Server Version | Folder Name  |
|--------------------|--------------|
| SQL Server 2022    | MSSQL16      |
| SQL Server 2019    | MSSQL15      |
| SQL Server 2017    | MSSQL14      |
| SQL Server 2016    | MSSQL13      |

### Named Instances

If using a named instance (e.g., `SERVER\PROD`), the service name will be `MSSQL$PROD` instead of `MSSQLSERVER`. Update all commands and registry paths accordingly.

### SQL Server Error Log Location

```
C:\Program Files\Microsoft SQL Server\MSSQL1x.MSSQLSERVER\MSSQL\Log\ERRORLOG
```
