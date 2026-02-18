# MDT Task Sequence – Fix Administrator Password Step
**Runbook: Move & Set a Known Admin Password**

---

## Overview

This runbook covers two things:
1. Moving the **Set Local Admin Password** step to the correct position at the end of the task sequence
2. Changing from a random password to a **known, controlled password**

**Environment:** Pure MDT / WDS — Airgapped Network

---

## Prerequisites

- Access to the MDT server with **Deployment Workbench** installed
- Administrative rights on the MDT server
- Your deployment share path (e.g. `D:\DeploymentShare`)
- Decided on your new Administrator password (strong, documented in your vault)

---

## Part 1 — Reorder the Task Sequence Step

### Step 1: Open Deployment Workbench

1. On the MDT server, open **Deployment Workbench**
   - Start Menu → Microsoft Deployment Toolkit → Deployment Workbench
2. Expand **Deployment Shares** → expand your deployment share
3. Click **Task Sequences**
4. Right-click your target task sequence → click **Properties**
5. Click the **Task Sequence** tab

---

### Step 2: Locate the Password Step

1. In the task sequence editor, expand the **State Restore** phase
2. Look for a step named **"Set Local Admin Password"**
3. Note its current position — it may be near the top of State Restore

---

### Step 3: Move the Step to the End of State Restore

1. Click the **"Set Local Admin Password"** step to select it
2. Use the **down arrow** button (right side of the editor) to move it down
3. Keep moving it until it is the **last step** in the State Restore phase

> **Target order inside State Restore should look like this:**
>
> ```
> State Restore
> ├── Install Applications
> ├── Windows Update (Pre-Application Installation)
> ├── Windows Update (Post-Application Installation)
> ├── Join Domain or Workgroup
> ├── [Any post-domain scripts]
> └── Set Local Admin Password    ← HERE (last)
> ```

4. Click **OK** to close the task sequence editor

---

## Part 2 — Set a Known Password (Two Options)

Choose **Option A** for a quick fix or **Option B** for a cleaner, centralised approach.

---

### Option A: Set Password Directly in the Task Sequence Step

1. In the task sequence editor, click the **"Set Local Admin Password"** step
2. In the properties pane on the right, locate the **Password** field
3. Clear any existing value
4. Type your known password and confirm it in the **Confirm Password** field
5. Ensure **"Random"** or any randomisation checkbox is **unchecked**
6. Click **Apply** → **OK**

> **Use this if:** You only have one or two task sequences and want a fast fix.

---

### Option B: Set Password via CustomSettings.ini (Recommended)

This method manages the password centrally — all task sequences on this deployment share will use it automatically.

1. Navigate to your deployment share `Control` folder:
   ```
   D:\DeploymentShare\Control\
   ```
2. Open **CustomSettings.ini** in Notepad or your preferred text editor
3. Under the `[Default]` section, add the following line:
   ```ini
   AdminPassword=YourPasswordHere
   ```

   Example of what your `[Default]` section might look like:
   ```ini
   [Settings]
   Priority=Default

   [Default]
   OSInstall=Y
   SkipCapture=YES
   SkipAdminPassword=YES
   AdminPassword=YourStrongPasswordHere
   ```

4. Save and close the file

> **Note:** `SkipAdminPassword=YES` suppresses the password prompt during the wizard so it uses the value you set here automatically.

> **Use this if:** You have multiple task sequences or want one central place to manage the password.

---

## Part 3 — Update the Deployment Share

This step is **mandatory** — without it your boot image won't reflect the changes.

1. In Deployment Workbench, right-click your **Deployment Share**
2. Click **Update Deployment Share**
3. Choose **Optimize the boot image updating process** (faster) or **Completely regenerate the boot images** (thorough — use this if unsure)
4. Click **Next** → **Next** → wait for the process to complete
5. Click **Finish**

---

## Part 4 — Verify the Change

After your next imaging run, verify the Administrator password was set correctly:

1. Log into a freshly imaged machine with `.\Administrator` and your known password
2. Confirm login succeeds
3. Check **Event Viewer → Windows Logs → Application** for any MDT task sequence errors if the password step fails

---

## Troubleshooting

| Issue | Likely Cause | Fix |
|---|---|---|
| Password step fails mid-sequence | Step still running too early | Recheck order — ensure it's last in State Restore |
| Known password not applying | `CustomSettings.ini` typo or missing `SkipAdminPassword=YES` | Review ini file and re-update deployment share |
| Boot image not reflecting changes | Deployment share not updated | Re-run Update Deployment Share |
| Domain join fails after reorder | Domain join step is now after password step | Ensure domain join is **before** the password step |

---

## Security Notes

- Store the Administrator password in your team's **secure password vault**
- Use a **complex password** — minimum 15 characters, mixed case, numbers, symbols
- Consider implementing **LAPS (Local Administrator Password Solution)** post-domain-join for ongoing management — this lets LAPS rotate and manage the local admin password automatically after imaging is complete
- In your airgapped environment, LAPS stores passwords in Active Directory, keeping them accessible to admins without needing a shared known password long-term

---

*Runbook prepared for airgapped MDT/WDS environment — tested against MDT 8456*
