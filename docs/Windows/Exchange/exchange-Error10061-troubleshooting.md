# Exchange MRS Move Request Troubleshooting

## Error: TCP 10061 - Connection Actively Refused

**Symptom:**
```
The call to 'net.tcp://server.domain.com/Microsoft.Exchange.MailboxReplicationService.ProxyService' failed because no service was listening on the specified endpoint.
TCP error 10061: No connection could be made because the target machine actively refused it
```

**Root Cause:** MRS Proxy not enabled or MSExchangeMailboxReplication service not running.

---

## Quick Fix (On-Prem to On-Prem, Same Org)

For same-org moves, MRS Proxy isn't required. Run the move request from the target server specifying the destination database:

```powershell
# Get available databases on target
Get-MailboxDatabase -Server exch02

# Move all mailboxes from source server
Get-Mailbox -Server testexch01 | New-MoveRequest -TargetDatabase "DB01-Exch02"

# Monitor progress
Get-MoveRequest | Get-MoveRequestStatistics

# Cleanup after completion
Get-MoveRequest -MoveStatus Completed | Remove-MoveRequest
```

---

## Service Checks

```powershell
# Check MRS service status
Get-Service MSExchangeMailboxReplication -ComputerName testexch01, exch02

# Restart MRS on both servers
Get-Service MSExchangeMailboxReplication -ComputerName testexch01, exch02 | Restart-Service

# Test MRS health
Test-MRSHealth -Identity testexch01
```

---

## If MRS Proxy Is Actually Needed (Cross-Forest/Hybrid)

```powershell
# Check if MRS Proxy is enabled
Get-WebServicesVirtualDirectory -Server testexch01 | fl MRSProxyEnabled

# Enable MRS Proxy
Set-WebServicesVirtualDirectory -Identity "testexch01\EWS (Default Web Site)" -MRSProxyEnabled $true

# Restart services
Restart-Service MSExchangeMailboxReplication
iisreset
```

---

## Connectivity Verification

```powershell
# Test port 808 (WCF internal port)
Test-NetConnection -ComputerName testexch01 -Port 808

# Verify FQDN resolution
Resolve-DnsName testexch01.domain.com
```

---

## Checklist

- [ ] Both servers in same AD site
- [ ] MSExchangeMailboxReplication service running on both servers
- [ ] FQDN resolution working between servers
- [ ] No Windows Firewall blocking port 808
- [ ] Target database specified explicitly in move request
