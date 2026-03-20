# Nexus SSL Certificate Update Runbook

**Applies to:** Sonatype Nexus Repository Manager 3.x (Windows)  
**Environment:** Internal ADCS / Airgapped  
**Last Updated:** 2026-03-20

---

## Overview

Nexus uses a Java Keystore (JKS) to store its SSL certificate. When a cert expires or needs to be replaced, you must:

1. Request a new cert from the internal CA (ADCS)
2. Export it as a PFX
3. Convert the PFX to JKS format
4. Replace the keystore in Nexus
5. Restart the service and verify
6. Update Chocolatey client trust if needed

---

## Prerequisites

- Access to the Nexus server (local or via jump server)
- Certificate enrollment rights against the internal CA
- JDK `keytool` — available under `<nexus-install>\jre\bin\`
- PowerShell (run as Administrator)

---

## Step 1 — Request a Certificate from ADCS

Request the certificate from the internal CA. Ensure it meets these requirements:

| Requirement | Value |
|---|---|
| EKU | Server Authentication (1.3.6.1.5.5.7.3.1) |
| Subject / CN | Nexus server FQDN |
| SAN | DNS name matching the FQDN used by clients |
| Private Key | Exportable |
| Key Length | 2048-bit minimum |

Enroll via MMC (`certlm.msc`) or `certreq`, targeting the appropriate web server template.

---

## Step 2 — Export as PFX

From the machine where the cert was enrolled, export it with the private key:

```powershell
# Find the cert by subject
$cert = Get-ChildItem Cert:\LocalMachine\My |
    Where-Object { $_.Subject -like "*nexus*" }

# Export as PFX
$pfxPassword = Read-Host -AsSecureString "Enter PFX password"
Export-PfxCertificate -Cert $cert `
    -FilePath C:\temp\nexus.pfx `
    -Password $pfxPassword
```

Transfer `nexus.pfx` to the Nexus server if the cert was enrolled on a different machine.

---

## Step 3 — Verify the PFX Alias

Before converting, check the alias name inside the PFX. You will need it in the next step.

```cmd
cd "<nexus-install>\jre\bin"

keytool -list -keystore C:\temp\nexus.pfx -storetype PKCS12 -storepass <pfx-password>
```

Note the alias shown (commonly `1` when exported from Windows).

---

## Step 4 — Convert PFX to JKS

```cmd
keytool -importkeystore ^
  -srckeystore C:\temp\nexus.pfx ^
  -srcstoretype PKCS12 ^
  -srcstorepass <pfx-password> ^
  -destkeystore C:\temp\nexus.jks ^
  -deststoretype JKS ^
  -deststorepass changeit ^
  -destkeypass changeit ^
  -alias 1
```

> **Note:** The destination store password and key password must match. `changeit` is the Nexus default — keep it consistent with what is configured in `nexus.properties`.

---

## Step 5 — Replace the Keystore

### Backup the existing keystore

```powershell
$sslDir = "C:\ProgramData\sonatype-work\nexus3\etc\ssl"

Copy-Item "$sslDir\keystore.jks" "$sslDir\keystore.jks.bak"
```

### Copy in the new keystore

```powershell
Copy-Item C:\temp\nexus.jks "$sslDir\keystore.jks" -Force
```

---

## Step 6 — Verify Nexus Configuration

### nexus.properties

Located at: `C:\ProgramData\sonatype-work\nexus3\etc\nexus.properties`

Confirm the keystore password matches what was set in Step 4:

```properties
nexus.ssl.keystore.password=changeit
nexus.ssl.key.password=changeit
```

### Jetty HTTPS config

Located at: `<nexus-install>\etc\jetty\jetty-https.xml`

Confirm the keystore path is correct and the HTTPS connector is not commented out:

```xml
<Set name="KeyStorePath"><Property name="ssl.etc"/>/keystore.jks</Set>
<Set name="KeyStorePassword">changeit</Set>
<Set name="KeyManagerPassword">changeit</Set>
```

---

## Step 7 — Restart the Nexus Service

```powershell
# Restart
Restart-Service nexus

# Confirm it came back up
Get-Service nexus

# Tail the log for errors
Get-Content "C:\ProgramData\sonatype-work\nexus3\log\nexus.log" -Wait -Tail 50
```

Allow 30–60 seconds for Nexus to fully initialize before testing.

---

## Step 8 — Verify the New Certificate

### From the Nexus server

```powershell
# Quick TLS check using .NET
[System.Net.ServicePointManager]::SecurityProtocol = 'Tls12'
$req = [System.Net.WebRequest]::Create("https://<nexus-fqdn>:8443/")
$req.GetResponse() | Out-Null
```

### From a client machine

```powershell
# Check cert details
$cert = (Invoke-WebRequest -Uri "https://<nexus-fqdn>:8443/" -UseBasicParsing).BaseResponse
# Or use openssl if available:
# openssl s_client -connect <nexus-fqdn>:8443
```

Confirm:
- Certificate CN/SAN matches the FQDN
- Issuer is the internal CA
- Expiry date is correct

---

## Step 9 — Update Client Trust (If CA Changed)

If the signing CA changed, push the new root cert to all client machines and update Chocolatey sources if needed.

### Import new root cert via GPO or PowerShell

```powershell
# Run on clients (or deploy via GPO / SCCM)
Import-Certificate `
    -FilePath "\\<share>\certs\internal-root-ca.cer" `
    -CertStoreLocation Cert:\LocalMachine\Root
```

### Verify Chocolatey source still works

```powershell
choco source list
choco search <package-name> --source="https://<nexus-fqdn>:8443/repository/choco-hosted/"
```

---

## Troubleshooting

| Symptom | Likely Cause | Fix |
|---|---|---|
| Nexus fails to start | Wrong alias in JKS | Re-run `keytool -list` to verify alias and re-import |
| `keytool error: java.io.IOException` | PFX password mismatch | Confirm password used matches export |
| Store/key password mismatch error | `deststorepass` ≠ `destkeypass` | Re-run conversion with matching passwords |
| Clients get cert warning | Root CA not trusted | Import root cert to `Cert:\LocalMachine\Root` |
| Choco push/install fails after cert change | Source URL or cert trust issue | Re-verify source and client trust store |
| CCM agent not connecting | CCM still references old cert trust | Restart CCM service; verify CCM config URL |

**Log location:**
```
C:\ProgramData\sonatype-work\nexus3\log\nexus.log
```

---

## Quick Reference — Key Paths

| Item | Path |
|---|---|
| Nexus keystore | `C:\ProgramData\sonatype-work\nexus3\etc\ssl\keystore.jks` |
| nexus.properties | `C:\ProgramData\sonatype-work\nexus3\etc\nexus.properties` |
| Jetty HTTPS config | `<nexus-install>\etc\jetty\jetty-https.xml` |
| Nexus log | `C:\ProgramData\sonatype-work\nexus3\log\nexus.log` |
| keytool binary | `<nexus-install>\jre\bin\keytool.exe` |
