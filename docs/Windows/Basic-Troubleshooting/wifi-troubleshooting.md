
# Windows WiFi / Internet Troubleshooting Runbook

Author: Home IT Support
Last Updated: 2026-03-04
Applies To: Windows 10 / Windows 11 laptops

---

# Overview

This runbook provides a structured process for diagnosing and fixing WiFi or internet connectivity issues on a Windows laptop.

Issues may occur due to:

- Windows updates
- Network driver changes
- Corrupted DNS cache
- DHCP lease problems
- Router issues
- WiFi profile corruption

The troubleshooting process follows Tier 1 → Tier 2 escalation steps.

---

# Tier 1 – Basic Connectivity Checks

## Step 1 – Verify WiFi is Enabled

1. Click the WiFi icon in the bottom-right system tray.
2. Confirm WiFi is turned ON.

---

## Step 2 – Check Airplane Mode

Press:

Windows + A

Verify:

Airplane Mode = OFF

---

## Step 3 – Disconnect and Reconnect to WiFi

1. Click the WiFi icon.
2. Select the home network.
3. Click Disconnect.
4. Wait 10 seconds.
5. Reconnect.

---

## Step 4 – Restart the Laptop

Restarting clears temporary networking issues.

1. Click Start
2. Select Power
3. Click Restart

Reconnect to WiFi after reboot.

---

# Tier 2 – Refresh Network Configuration

## Step 5 – Open Command Prompt as Administrator

1. Press Windows Key
2. Search: cmd
3. Right-click Command Prompt
4. Select Run as Administrator

---

## Step 6 – Clear DNS Cache

Run:

ipconfig /flushdns

Purpose:
Clears cached DNS records and fixes domain resolution problems.

---

## Step 7 – Release Current IP Address

Run:

ipconfig /release

Purpose:
Drops the current DHCP lease.

---

## Step 8 – Request New IP Address

Run:

ipconfig /renew

Purpose:
Requests a new IP address from the router.

---

# Tier 3 – Network Connectivity Testing

## Step 9 – Check Network Configuration

Run:

ipconfig /all

Verify:

IPv4 Address is NOT 169.254.x.x  
Default Gateway shows router IP  
DNS server is populated

If you see a 169.254 address DHCP failed.

---

## Step 10 – Test Local Network Connectivity

Ping the router:

ping 192.168.1.1

Expected result:

Reply from 192.168.1.1

If this fails the laptop cannot reach the router.

---

## Step 11 – Test Internet Connectivity

Ping Google DNS:

ping 8.8.8.8

If successful:

Reply from 8.8.8.8

This confirms internet connectivity exists.

---

## Step 12 – Test DNS Resolution

Run:

nslookup google.com

Expected:

Name: google.com  
Address returned

If this fails DNS configuration may be broken.

---

## Step 13 – Trace Internet Path

Run:

tracert google.com

This shows the route to the internet.

---

# Tier 4 – WiFi Profile Repair

## Step 14 – Forget the WiFi Network

1. Open Settings
2. Go to Network & Internet
3. Select WiFi
4. Click Manage Known Networks
5. Select your WiFi network
6. Click Forget

Reconnect to WiFi again.

---

# Tier 5 – Advanced Network Repair

## Step 15 – Reset Winsock

Run:

netsh winsock reset

Restart the computer after running.

---

## Step 16 – Reset TCP/IP Stack

Run:

netsh int ip reset

Restart the system.

---

## Step 17 – Restart Network Adapter

1. Open Control Panel
2. Go to Network and Sharing Center
3. Click Change Adapter Settings
4. Right-click WiFi adapter
5. Select Disable
6. Wait 10 seconds
7. Select Enable

---

# Tier 6 – Driver and Adapter Checks

## Step 18 – Check Device Manager

1. Right-click Start
2. Select Device Manager
3. Expand Network Adapters

Look for warning icons or disabled adapters.

---

## Step 19 – Update WiFi Driver

Right-click the WiFi adapter and select:

Update Driver → Search Automatically

---

# Tier 7 – Router Troubleshooting

If multiple devices have issues:

1. Unplug router
2. Wait 30 seconds
3. Plug router back in
4. Wait 2 minutes for full startup

Reconnect WiFi.

---

# Tier 8 – Full Network Reset (Last Resort)

1. Open Settings
2. Go to Network & Internet
3. Select Advanced Network Settings
4. Click Network Reset
5. Restart computer

Note: All saved WiFi networks will be removed.

---

# Command Reference

ipconfig /all
ipconfig /flushdns
ipconfig /release
ipconfig /renew
ping 8.8.8.8
nslookup google.com
tracert google.com
netsh winsock reset
netsh int ip reset

---

# Escalation to IT

Provide:

Laptop hostname  
Screenshot of error  
Output of ipconfig /all  
Time issue started  
Confirmation troubleshooting steps were completed

---
