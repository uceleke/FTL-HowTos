# WDS Image Update Runbook

**Purpose:** Update a Windows Deployment Services (WDS) instance with a new desktop image (e.g., after driver updates to a reference machine).

---

## Prerequisites

- New image already captured or ready to capture as a `.wim` file
- WDS console or PowerShell access to the WDS server
- Determine your deployment model before proceeding:
  - **Standalone WDS** → follow Sections 1–3
  - **WDS + MDT** → follow Sections 1–4
  - **MDT driver management** → Section 5

---

## 1. Capture the New Image (if not already done)

Boot the reference machine into WinPE and run:

```cmd
dism /Capture-Image /ImageFile:\\<WDS-SERVER>\reminst\Images\<filename>.wim /CaptureDir:C:\ /Name:"<Image Name>" /Description:"Updated drivers - <date>"
```

> **Note:** If using MDT, use the Capture Task Sequence instead of manual DISM capture.

---

## 2. Add the New Image to WDS

### Option A — Replace Existing Image (PowerShell)

```powershell
# Remove the old image
Remove-WdsInstallImage -ImageName "Windows 10 Enterprise" -ImageGroup "Desktop"

# Import the new .wim
Import-WdsInstallImage -ImageGroup "Desktop" `
    -Path "C:\Path\To\NewImage.wim" `
    -ImageName "Windows 10 Enterprise" `
    -NewImageName "Windows 10 Enterprise - Updated Drivers"
```

### Option B — Add as New Image Alongside Old One (Recommended for rollback)

```powershell
Import-WdsInstallImage -ImageGroup "Desktop" `
    -Path "C:\Path\To\NewImage.wim" `
    -ImageName "Windows 10 Enterprise" `
    -NewImageName "Win10 Ent - Driver Update $(Get-Date -Format 'yyyy-MM-dd')"
```

### Option C — Via WDS MMC Console (GUI)

1. Open **Windows Deployment Services** console
2. Expand your server → **Install Images**
3. Right-click the image group → **Add Install Image**
4. Browse to the `.wim` file and follow the wizard
5. Once imported, right-click the old image → **Disable** or **Delete**

---

## 3. Validate the New Image

```powershell
# Confirm the image is listed in WDS
Get-WdsInstallImage -ImageGroup "Desktop" | Select ImageName, CreationTime, FileSize
```

- Test PXE boot a spare machine and deploy the new image end-to-end
- Confirm drivers load correctly post-deployment (Device Manager — no yellow bangs)
- Review WDS event logs:
  **Applications and Services Logs → Microsoft → Windows → Deployment-Services-Diagnostics**

---

## 4. MDT — Update Deployment Share (if applicable)

If WDS is fronting an MDT deployment share, regenerate boot media and re-import it into WDS.

### 4a. Update the MDT Deployment Share

```powershell
Update-MDTDeploymentShare -Path "DS001:" -Force -Verbose
```

### 4b. Re-import LiteTouchPE Boot Image into WDS

1. In WDS console, right-click **Boot Images** → **Add Boot Image**
2. Browse to:
   ```
   \\<MDTShare>\Boot\LiteTouchPE_x64.wim
   ```
3. Replace the existing boot image if prompted

---

## 5. MDT Driver Management

In a WDS + MDT environment, MDT owns driver injection via the task sequence — not the WDS driver store. Drivers are managed through the **Out-of-Box Drivers** node in the Deployment Workbench and targeted to hardware using `CustomSettings.ini` variables or task sequence steps.

### 5a. Recommended Folder Structure

Organize drivers under `Out-of-Box Drivers` by make and model to allow precise targeting:

```
Out-of-Box Drivers\
  Windows 10 x64\
    Dell\
      OptiPlex 7090\
        Network\
        Video\
        Chipset\
    HP\
      EliteDesk 800 G6\
        Network\
        Video\
        Chipset\
```

> **Tip:** Keep drivers separated by model — avoid dumping all drivers into a single flat folder. MDT will inject everything in the targeted path, so a flat structure causes driver bloat and potential conflicts.

### 5b. Import Drivers via Deployment Workbench (GUI)

1. Open **Deployment Workbench**
2. Expand your deployment share → **Out-of-Box Drivers**
3. Create subfolders matching your folder structure above (right-click → **New Folder**)
4. Right-click the target model folder → **Import Drivers**
5. Browse to the driver source folder — MDT will recurse and import all `.inf` files found
6. After import, run **Update Deployment Share** (Section 4a) to regenerate boot media

### 5c. Import Drivers via PowerShell

```powershell
# Load MDT PowerShell module
Import-Module "C:\Program Files\Microsoft Deployment Toolkit\bin\MicrosoftDeploymentToolkit.psd1"

# Connect to deployment share
New-PSDrive -Name "DS001" -PSProvider MDTProvider -Root "\\<MDTServer>\DeploymentShare$"

# Import drivers into a specific model folder
Import-MDTDriver -Path "DS001:\Out-of-Box Drivers\Windows 10 x64\Dell\OptiPlex 7090" `
    -SourcePath "C:\Drivers\Dell\OptiPlex7090" `
    -Verbose
```

### 5d. Target Drivers by Model in CustomSettings.ini

Use MDT's built-in `%Make%` and `%Model%` variables to automatically select the correct driver group at deployment time. Edit `CustomSettings.ini` on the deployment share:

```ini
[Settings]
Priority=Model, Default

[Default]
DriverSelectionProfile=Nothing

[Dell OptiPlex 7090]
DriverGroup001=Windows 10 x64\Dell\OptiPlex 7090

[HP EliteDesk 800 G6]
DriverGroup001=Windows 10 x64\HP\EliteDesk 800 G6
```

> **How it works:** MDT queries WMI for `Win32_ComputerSystem.Model` at boot. The value must match the section name exactly. To find the exact model string for a machine, run:
> ```powershell
> (Get-WmiObject Win32_ComputerSystem).Model
> ```

### 5e. Configure the Task Sequence — Inject Drivers Step

Verify the task sequence has an `Inject Drivers` step configured to use the selection profile driven by `DriverGroup001`:

1. Open **Deployment Workbench** → Task Sequences → right-click your TS → **Properties**
2. Go to the **Task Sequence** tab
3. Under **State Restore** → find or add **Inject Drivers**
4. Set **Selection Profile** to `Nothing` (let `DriverGroup001` from CustomSettings.ini control selection)

> **Alternative:** If you prefer task sequence-level control, set the Selection Profile directly to a named profile instead of using `DriverGroup001`. Use one approach consistently — mixing both can cause unpredictable injection behavior.

### 5f. Validate Driver Injection During Deployment

During a test deployment, MDT logs all driver activity to `X:\MININT\SMSOSD\OSDLOGS\BDD.log` (WinPE phase) and `C:\MININT\SMSOSD\OSDLOGS\BDD.log` (post-reboot). Check for:

```
Inject Drivers: DriverGroup001 = Windows 10 x64\Dell\OptiPlex 7090
Searching for driver INF files in: \\<MDTShare>\Out-of-Box Drivers\Windows 10 x64\Dell\OptiPlex 7090
```

After deployment completes, confirm in Device Manager that no devices show yellow bangs. If drivers are missing or wrong:
- Verify the `%Model%` value matched a `CustomSettings.ini` section exactly
- Check `BDD.log` for `DriverGroup001` assignment and which INF files were staged
- Confirm the driver folder path in Deployment Workbench matches what's in `DriverGroup001`

---

## Notes

- **Driver-only updates:** If the change is limited to driver additions (no OS changes), consider offline servicing with DISM instead of a full recapture:
  ```cmd
  dism /Mount-Image /ImageFile:"C:\Images\image.wim" /Index:1 /MountDir:"C:\Mount"
  dism /Image:"C:\Mount" /Add-Driver /Driver:"C:\Drivers" /Recurse
  dism /Unmount-Image /MountDir:"C:\Mount" /Commit
  ```
- **Airgapped environments:** Ensure the `.wim` is transferred to the WDS server via an approved method (USB, internal file share) — no external network access required for any step above.
- **Rollback:** Using Option B (adding as a new image) preserves the previous image as a fallback until the new one is validated.
