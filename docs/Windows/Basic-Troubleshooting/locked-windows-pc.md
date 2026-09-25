# Windows 11 Offline Account Recovery — utilman.exe Method
| | |
|---|---|
|**Author** | Jesse Esposo "Big Dawg"|
|**Version** | 1.0 |
|**Last updated** | 2026-09-25 | 

**Purpose:** Regain administrative access to a Windows 11 machine you administer when the password is lost or undocumented, by enabling the built-in Administrator account and setting its password from a SYSTEM shell at the login screen.

> **Authorization note:** This procedure requires physical possession of the machine and (if encrypted) the BitLocker recovery key. Use it only on systems you own or are authorized to recover.

---

## How it works

`utilman.exe` (the Accessibility/Ease-of-Access handler) and `sethc.exe` (Sticky Keys) both launch as **SYSTEM** at the logon screen *before* any user authenticates. By replacing one of them with `cmd.exe` from an offline environment, clicking the Accessibility button at login yields a SYSTEM command prompt — enough to enable the Administrator account and reset passwords.

**The catch:** the file swap is an *offline write* to the OS volume. If that volume is BitLocker-encrypted, you cannot write to it until it is unlocked with the recovery key. This is why the BitLocker check comes first.

---

## Step 0 — Boot to a command prompt

Boot the target from **Windows 11 installation media** (or any WinPE / recovery USB).

At the first setup screen (language/keyboard), press:

```
Shift + F10
```

This opens a command prompt in the WinPE environment.

---

## Step 1 — Identify the OS volume

In WinPE, drive letters shift — the Windows install is usually **not** `C:`.

```
diskpart
list volume
exit
```

Find the volume holding Windows (largest NTFS volume). Verify:

```
dir D:\Windows\System32\utilman.exe
```

Use whatever letter is correct in every command below (examples assume `D:`).

---

## Step 2 — Check BitLocker status

Check the OS volume before attempting any writes:

```
manage-bde -status D:
```

Interpret the output:

| Field | What to look for |
|-------|------------------|
| **Conversion Status** | `Fully Decrypted` = no encryption, proceed freely. `Fully Encrypted` / `Used Space Only Encrypted` = must unlock first. |
| **Protection Status** | `Protection On` = keys are sealed, volume is locked to offline access. `Protection Off` = suspended, may be writable. |
| **Lock Status** | `Locked` = you cannot read/write until unlocked. `Unlocked` = accessible. |
| **Percentage Encrypted** | Confirms extent of encryption. |

If **Lock Status = Unlocked** and you can `dir` into `D:\Windows\System32`, skip to Step 4.

If **Lock Status = Locked**, continue to Step 3.

> If `manage-bde` reports the volume as not found or not encrypted, it is plaintext — proceed to Step 4.

---

## Step 3 — Unlock or decrypt BitLocker

You need the **48-digit recovery key** (or a numeric recovery password). Sources:

- Microsoft account: `account.microsoft.com/devices/recoverykey`
- Entra ID / Azure AD: **Devices → the device → BitLocker keys**
- On-prem AD: the computer object's `msFVE-RecoveryInformation` child object (via ADUC with the BitLocker Recovery tab, or `Get-ADObject`)
- MDM (Intune/other): device encryption / recovery key blade
- A saved `.BEK` file or printed key

### Option A — Unlock for this session only (recommended)

Unlocks the volume so you can write, without decrypting the whole disk. Fast, and leaves encryption intact.

```
manage-bde -unlock D: -RecoveryPassword 123456-123456-123456-123456-123456-123456-123456-123456
```

Then confirm:

```
manage-bde -status D:
```

`Lock Status` should now read `Unlocked`. You can now write to the volume. Proceed to Step 4.

> **Optional — prevent re-locking on reboot:** after unlocking, suspend protection so the next boot doesn't demand the key again:
> ```
> manage-bde -protectors -disable D:
> ```
> This suspends (does not remove) protectors. Re-enable later with `manage-bde -protectors -enable D:`.

### Option B — Fully decrypt the volume

Only if you actually want the disk decrypted (e.g., decommissioning, or persistent troubleshooting). This is **slow** — potentially hours on a large drive — and reduces the machine's security posture.

```
manage-bde -off D:
```

Monitor progress:

```
manage-bde -status D:
```

Wait until `Conversion Status` reads `Fully Decrypted` before continuing.

> For most recovery jobs, **Option A (unlock) is correct** — you get write access in seconds and the machine stays encrypted. Only decrypt if there's a real reason to.

---

## Step 4 — Swap utilman.exe for cmd.exe

Back up the original, then replace it:

```
copy D:\Windows\System32\utilman.exe D:\Windows\System32\utilman.exe.bak
copy /y D:\Windows\System32\cmd.exe D:\Windows\System32\utilman.exe
```

> **Alternate target:** `sethc.exe` (Sticky Keys, triggered by pressing **Shift 5×**) works identically if the Accessibility button is unavailable.

Reboot into normal Windows and remove the USB:

```
wpeutil reboot
```

---

## Step 5 — Open the SYSTEM shell at login

At the Windows 11 lock/login screen, click the **Accessibility** button (bottom-right, the person icon). A command prompt opens running as **SYSTEM**.

---

## Step 6 — Enable the Administrator account and set a password

The built-in Administrator is disabled by default on Windows 11, so activate it first:

```
net user Administrator /active:yes
net user Administrator "YourNewP@ssw0rd"
```

**Reset an existing local account instead:**

```
net user "AccountName" "YourNewP@ssw0rd"
```

**Or create a fresh local admin:**

```
net user recoveryadmin "YourNewP@ssw0rd" /add
net localgroup Administrators recoveryadmin /add
```

Close the prompt. The Administrator account now appears at the login screen — sign in.

> **Domain-joined machines:** this only touches the **local SAM**. It recovers local accounts, not domain accounts. For domain admin recovery you need a domain controller / DSRM path instead.

---

## Step 7 — Clean up (do not skip)

A replaced `utilman.exe` is a permanent SYSTEM-shell backdoor at the login screen. Restore it.

The file is locked while Windows runs, so boot back to the USB, press **Shift + F10**, unlock BitLocker again if needed (Step 3, Option A), then:

```
copy /y D:\Windows\System32\utilman.exe.bak D:\Windows\System32\utilman.exe
del D:\Windows\System32\utilman.exe.bak
```

If you suspended BitLocker protectors, re-enable them:

```
manage-bde -protectors -enable D:
```

Reboot and verify the Accessibility button behaves normally again.

---

## Post-recovery caveats

- **Windows Hello / Microsoft account logins:** resetting a local password can orphan EFS-encrypted files and DPAPI-protected credentials (saved passwords, certs). For MSA-backed accounts, an online reset via `account.microsoft.com` is cleaner where possible.
- **Credential loss:** after any offline password reset, expect stored browser/app credentials tied to DPAPI to be unrecoverable for that account.

---

## Hardening — how to block this method

This chain is the standard argument for pre-boot defenses. To prevent offline `utilman.exe` swaps:

- **BitLocker with pre-boot authentication** (TPM+PIN, not TPM-only) — the volume can't be modified offline without the recovery key.
- **UEFI/firmware password** and **locked boot order** — prevents booting external media.
- **Secure Boot enabled.**
- Disable or restrict **WinRE** access where policy requires.

These map directly to DISA STIG controls covering data-at-rest encryption and boot integrity — the mitigations that turn "physical access = SYSTEM in five minutes" into "physical access = a brick without the recovery key."
