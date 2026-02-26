# Hyper-V NIC Separation: Management vs VM Traffic
**Windows Server 2022**

---

## Overview

This guide covers separating Hyper-V host management traffic from VM traffic across two physical NICs:

| Adapter | Role | IP |
|---|---|---|
| Ethernet 1 | Host Management | Static IP (assigned in Windows) |
| Ethernet 2 | VM Traffic | No host IP — owned by Hyper-V virtual switch |

---

## Configuration Steps

### Step 1 — Verify Ethernet 1 (Management)

Ensure your static IP is set on Ethernet 1 and leave it completely alone. Do not bind it to any Hyper-V virtual switch.

You can confirm via PowerShell:
```powershell
Get-NetIPAddress -InterfaceAlias "Ethernet 1"
```

---

### Step 2 — Create the External Virtual Switch on Ethernet 2

**Via Hyper-V Manager (GUI):**

1. Open **Hyper-V Manager**
2. In the right-hand Actions pane, click **Virtual Switch Manager**
3. Select **External** → click **Create Virtual Switch**
4. Set a name (e.g. `vSwitch-VM-Traffic`)
5. Under **Connection type**, select **External network** and choose **Ethernet 2**
6. **Uncheck** `Allow management operating system to share this network adapter`
7. Click **Apply** → accept the warning about network disruption

**Via PowerShell:**
```powershell
New-VMSwitch -Name "vSwitch-VM-Traffic" `
             -NetAdapterName "Ethernet 2" `
             -AllowManagementOS $false
```

> ⚠️ After this step, Ethernet 2 will no longer have an IP on the host. This is expected and correct.

---

### Step 3 — Attach VMs to the Virtual Switch

For each VM, set its virtual NIC to use the new switch:

**Via Hyper-V Manager:**
1. Open VM Settings → Network Adapter
2. Set Virtual Switch to `vSwitch-VM-Traffic`

**Via PowerShell:**
```powershell
# List VMs
Get-VM

# Attach VM NIC to switch
Connect-VMNetworkAdapter -VMName "YourVMName" -SwitchName "vSwitch-VM-Traffic"
```

---

### Step 4 — Assign IPs Inside VMs

VMs will now communicate through Ethernet 2 to your physical network. Assign IPs inside each VM via:
- DHCP (from your network's DHCP server), or
- Static IP configured within the VM's OS

---

## Verification

Run these on the **host** to confirm correct setup:

```powershell
# Confirm switch exists and is bound to Ethernet 2
Get-VMSwitch | Select Name, SwitchType, NetAdapterInterfaceDescription, AllowManagementOS

# Confirm Ethernet 1 has your management IP, Ethernet 2 has no IP
Get-NetIPAddress | Select InterfaceAlias, IPAddress, AddressFamily

# Confirm Ethernet 2 adapter state (should show Up, no IP)
Get-NetAdapter | Select Name, Status, LinkSpeed
```

**Expected output:**
- `Ethernet 1` → has your static management IP
- `Ethernet 2` → no IP address assigned at host level
- `vSwitch-VM-Traffic` → `AllowManagementOS: False`

---

## Troubleshooting

### ❌ Lost management access to the host after creating the switch

**Cause:** The virtual switch was accidentally created on Ethernet 1, or management sharing was left enabled on Ethernet 2 and took the IP.

**Fix:**
1. Connect via console/iDRAC/IPMI (out-of-band)
2. Check which adapter the switch is bound to:
   ```powershell
   Get-VMSwitch | Select Name, NetAdapterInterfaceDescription
   ```
3. If bound to the wrong adapter, remove and recreate:
   ```powershell
   Remove-VMSwitch -Name "vSwitch-VM-Traffic"
   New-VMSwitch -Name "vSwitch-VM-Traffic" -NetAdapterName "Ethernet 2" -AllowManagementOS $false
   ```

---

### ❌ VMs have no network connectivity

**Cause:** VM NIC not connected to the switch, or Ethernet 2 is not physically connected/linked.

**Fix:**
1. Confirm the physical cable on Ethernet 2 is connected and the port is active:
   ```powershell
   Get-NetAdapter -Name "Ethernet 2"
   # Status should show "Up"
   ```
2. Confirm the VM's NIC is attached to the correct switch:
   ```powershell
   Get-VMNetworkAdapter -VMName "YourVMName" | Select VMName, SwitchName
   ```
3. If not connected, attach it:
   ```powershell
   Connect-VMNetworkAdapter -VMName "YourVMName" -SwitchName "vSwitch-VM-Traffic"
   ```

---

### ❌ Ethernet 2 shows "Unidentified Network" or no IP on host

**This is normal and expected behavior.** When `AllowManagementOS` is set to `$false`, the host OS intentionally has no access to that adapter. You may see it listed in `ncpa.cpl` as unidentified — ignore it.

If you need to revert and give the host an IP on Ethernet 2 (e.g. for a VLAN management scenario), you can enable management OS sharing:
```powershell
Set-VMSwitch -Name "vSwitch-VM-Traffic" -AllowManagementOS $true
```
Then assign an IP to the new `vEthernet (vSwitch-VM-Traffic)` virtual adapter that appears.

---

### ❌ VM traffic is still going through Ethernet 1

**Cause:** VM is attached to an internal/private switch or the wrong external switch.

**Fix:**
```powershell
# Check all VM network adapters and their switch assignments
Get-VMNetworkAdapter * | Select VMName, SwitchName, MacAddress

# Reconnect to correct switch
Connect-VMNetworkAdapter -VMName "YourVMName" -SwitchName "vSwitch-VM-Traffic"
```

---

### ❌ Virtual Switch Manager shows no adapters available for Ethernet 2

**Cause:** Ethernet 2 may be part of a NIC team (LBFO team) or already assigned to another switch.

**Fix:**
1. Check for existing teams:
   ```powershell
   Get-NetLbfoTeam
   Get-NetLbfoTeamMember
   ```
2. Remove from team if needed, or check for duplicate switch bindings:
   ```powershell
   Get-VMSwitch
   ```

---

### ❌ After reboot, VMs lose connectivity

**Cause:** Rare — usually a switch or adapter ordering issue, or driver problem.

**Fix:**
1. Confirm the switch survived the reboot:
   ```powershell
   Get-VMSwitch
   ```
2. Check Ethernet 2 link state post-boot:
   ```powershell
   Get-NetAdapter -Name "Ethernet 2"
   ```
3. If the switch lost its adapter binding, recreate it (see above).

---

## Quick Reference — Key PowerShell Commands

```powershell
# List all virtual switches
Get-VMSwitch

# List all VM network adapters
Get-VMNetworkAdapter *

# List host network adapters and IPs
Get-NetAdapter
Get-NetIPAddress

# Create switch (dedicated, no mgmt sharing)
New-VMSwitch -Name "vSwitch-VM-Traffic" -NetAdapterName "Ethernet 2" -AllowManagementOS $false

# Remove a switch
Remove-VMSwitch -Name "vSwitch-VM-Traffic"

# Connect VM to switch
Connect-VMNetworkAdapter -VMName "VMName" -SwitchName "vSwitch-VM-Traffic"

# Toggle management OS sharing on existing switch
Set-VMSwitch -Name "vSwitch-VM-Traffic" -AllowManagementOS $true/$false
```
