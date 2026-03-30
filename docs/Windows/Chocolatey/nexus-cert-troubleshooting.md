# Nexus Repository Certificate Troubleshooting Runbook

## Overview

Nexus Repository Manager maintains its own Java-based certificate infrastructure, separate from the Windows certificate store. This means certificates imported into Windows (via MMC or PowerShell) are **not** automatically trusted by Nexus. This runbook covers the most common certificate issues encountered with Nexus, including HTTPS configuration, LDAPS integration, and truststore management.

---

## Table of Contents

1. [Key Concepts](#key-concepts)
2. [Certificate Renewal (HTTPS / Keystore)](#certificate-renewal-https--keystore)
3. [LDAPS / Truststore Issues](#ldaps--truststore-issues)
4. [Diagnosing What Cert Nexus Is Actually Seeing](#diagnosing-what-cert-nexus-is-actually-seeing)
5. [Keytool Quick Reference](#keytool-quick-reference)
6. [Common Errors and Fixes](#common-errors-and-fixes)

---

## Key Concepts

| Term | Description |
|------|-------------|
| `keystore.jks` | Holds Nexus's own TLS certificate (used for HTTPS). Located at `C:\ProgramData\nexus\etc\ssl\`. |
| `cacerts` | Java's default truststore. Nexus uses this to validate outbound connections (e.g., LDAPS). Located at `C:\ProgramData\nexus\jre\lib\security\cacerts`. Default password: `changeit`. |
| `keytool.exe` | Java utility for managing keystores and truststores. Located at `C:\ProgramData\nexus\jre\bin\`. |
| Windows Cert Store | **Separate from Nexus.** Importing a cert into Windows does not make it available to Nexus. |
| Thumbprint | SHA-1 fingerprint of a certificate. Must be exactly 40 characters. Copying from MMC often adds a hidden leading character. |

---

## Certificate Renewal (HTTPS / Keystore)

### Prerequisites

- Certificate must be present in the Windows cert store (typically `Cert:\LocalMachine\TrustedPeople` or `Cert:\LocalMachine\My`)
- PowerShell running as Administrator
- Nexus service stopped during keystore replacement (optional but recommended)

### Step 1 — Verify the Certificate Exists and Thumbprint Is Clean

```powershell
# Check cert store (adjust store path as needed)
Get-ChildItem Cert:\LocalMachine\TrustedPeople | Select-Object Subject, Thumbprint, NotAfter

# Verify thumbprint length — MUST be exactly 40 characters
$Thumbprint = 'YOUR_THUMBPRINT_HERE'
$Thumbprint.Length

# Clean hidden characters (common when copying from MMC)
$Thumbprint = ($Thumbprint -replace '[^\x20-\x7E]', '').Trim().ToUpper()
$Thumbprint.Length  # Confirm it's now 40
```

> **Note:** A length of 41 indicates a hidden Unicode character (U+200E, left-to-right mark) was copied from the MMC console. The clean step above strips it.

### Step 2 — Confirm the Lookup Returns the Certificate

```powershell
Get-ChildItem Cert:\LocalMachine\TrustedPeople\ |
    Where-Object { $_.Thumbprint -eq $Thumbprint }
```

If this returns nothing, either the cert is in a different store or the thumbprint is still dirty. Check `Cert:\LocalMachine\My` as an alternative.

### Step 3 — Export to PFX and Import to Keystore

```powershell
$KeyTool  = "C:\ProgramData\nexus\jre\bin\keytool.exe"
$password = "chocolatey" | ConvertTo-SecureString -AsPlainText -Force

# Export from Windows cert store
$certificate | Export-PfxCertificate -FilePath C:\cert.pfx -Password $password

# Import into LocalMachine\My (required for keytool to read it)
Import-PfxCertificate -FilePath C:\cert.pfx `
    -CertStoreLocation Cert:\LocalMachine\My `
    -Exportable `
    -Password $password

# Get the alias from the PFX
$string      = ("chocolatey" | & $KeyTool -list -v -keystore C:\cert.pfx) -match '^Alias.*'
$currentAlias = ($string -split ':')[1].Trim()

# Remove old keystore if it exists
Remove-Item C:\ProgramData\nexus\etc\ssl\keystore.jks -Force -ErrorAction SilentlyContinue

# Import into Nexus keystore
$passkey = '9hPRGDmfYE3bGyBZCer6AUsh4RTZXbkw'
& $KeyTool -importkeystore `
    -srckeystore C:\cert.pfx `
    -srcstoretype PKCS12 `
    -srcstorepass chocolatey `
    -destkeystore C:\ProgramData\nexus\etc\ssl\keystore.jks `
    -deststoretype JKS `
    -alias $currentAlias `
    -destalias jetty `
    -deststorepass $passkey

& $KeyTool -keypasswd `
    -keystore C:\ProgramData\nexus\etc\ssl\keystore.jks `
    -alias jetty `
    -storepass $passkey `
    -keypass chocolatey `
    -new $passkey

# Clean up
Remove-Item C:\cert.pfx
```

### Step 4 — Verify Keystore Contents

```powershell
$KeyTool  = "C:\ProgramData\nexus\jre\bin\keytool.exe"
$passkey  = '9hPRGDmfYE3bGyBZCer6AUsh4RTZXbkw'

& $KeyTool -list -v `
    -keystore C:\ProgramData\nexus\etc\ssl\keystore.jks `
    -storepass $passkey
```

Confirm the alias is `jetty` and the expiry date matches your new cert.

---

## LDAPS / Truststore Issues

### Why the Wrong Fingerprint Appears in the Nexus UI

When configuring LDAPS in Nexus and checking "Use certificates stored in Nexus truststore," Nexus shows the fingerprint of the cert it finds for that LDAP host. If this fingerprint looks wrong, it usually means:

- Nexus is pulling a **stale/cached cert** from a previous connection attempt
- The correct CA cert was **never imported** into the Nexus truststore (`cacerts`)
- Nexus is presenting the fingerprint of the **CA/issuer**, not the leaf cert (expected behavior for chained certs)

### Step 1 — Check What Nexus Is Actually Trusting

```powershell
$KeyTool    = "C:\ProgramData\nexus\jre\bin\keytool.exe"
$TrustStore = "C:\ProgramData\nexus\jre\lib\security\cacerts"

# List all certs in the truststore
& $KeyTool -list -keystore $TrustStore -storepass changeit

# Search for a specific alias
& $KeyTool -list -keystore $TrustStore -storepass changeit | Select-String 'ldap'
```

### Step 2 — Pull the Cert Your LDAP Server Is Actually Presenting

Run this from the Nexus server to see exactly what cert is being served on port 636:

```powershell
$ldapServer = 'your-ldap-server.domain.com'
$tcp = [System.Net.Sockets.TcpClient]::new($ldapServer, 636)
$ssl = [System.Net.Security.SslStream]::new($tcp.GetStream(), $false, { $true })
$ssl.AuthenticateAsClient($ldapServer)
$cert = $ssl.RemoteCertificate

Write-Host "Subject    : $($cert.Subject)"
Write-Host "Issuer     : $($cert.Issuer)"
Write-Host "Thumbprint : $($cert.GetCertHashString())"
Write-Host "Expires    : $($cert.GetExpirationDateString())"
$ssl.Close(); $tcp.Close()
```

Compare this thumbprint against what Nexus shows in the UI. They should match. If they don't, a different cert is being served than expected (check for load balancers or network devices performing TLS inspection).

### Step 3 — Export the LDAP CA Cert

If the LDAP cert is issued by an internal CA, you need the **CA cert** (not the leaf/server cert) in the Nexus truststore.

```powershell
# Export from Windows cert store (adjust store and thumbprint as needed)
$caCert = Get-ChildItem Cert:\LocalMachine\CA |
    Where-Object { $_.Subject -match 'YourCAName' } |
    Select-Object -First 1

Export-Certificate -Cert $caCert -FilePath C:\ldap-ca.cer -Type CERT
```

Alternatively, pull it directly from AD:

```powershell
certutil -ca.cert C:\ldap-ca.cer
```

### Step 4 — Import CA Cert into Nexus Truststore

```powershell
$KeyTool    = "C:\ProgramData\nexus\jre\bin\keytool.exe"
$TrustStore = "C:\ProgramData\nexus\jre\lib\security\cacerts"

& $KeyTool -importcert `
    -keystore $TrustStore `
    -storepass changeit `
    -alias "internal-ldap-ca" `
    -file "C:\ldap-ca.cer" `
    -noprompt

# Verify it landed
& $KeyTool -list -keystore $TrustStore -storepass changeit | Select-String 'internal-ldap-ca'
```

### Step 5 — Restart Nexus and Retest

```powershell
Restart-Service nexus
```

Then in the Nexus UI: **Security → LDAP → Test Connection**. The fingerprint shown should now match your LDAP server's cert or CA cert.

---

## Diagnosing What Cert Nexus Is Actually Seeing

### Check Nexus JVM Truststore Configuration

Nexus may be configured to use a custom truststore instead of the default `cacerts`:

```powershell
# Check nexus.properties for custom truststore settings
Get-Content 'C:\ProgramData\sonatype-work\nexus3\etc\nexus.properties' |
    Select-String 'truststore|ssl|javax'

# Check JVM args
Get-Content 'C:\ProgramData\nexus\etc\nexus-default.properties' |
    Select-String 'trust|javax'
```

If you see `-Djavax.net.ssl.trustStore=C:\some\custom\path`, that file is the truststore you need to import into — **not** the default `cacerts`.

### Check Nexus Log for SSL Errors

```powershell
Get-Content 'C:\ProgramData\sonatype-work\nexus3\log\nexus.log' -Tail 100 |
    Select-String 'ssl|ldap|certificate|handshake|trust' -CaseSensitive:$false
```

Common log entries to look for:

| Log Message | Meaning |
|---|---|
| `PKIX path building failed` | CA cert not in truststore |
| `unable to find valid certification path` | Same as above |
| `Connection refused` | LDAP server not reachable on 636 |
| `SSLHandshakeException` | TLS negotiation failure — cert mismatch or protocol issue |
| `CertPathValidatorException` | Cert chain validation failure |

---

## Keytool Quick Reference

All commands assume `$KeyTool = "C:\ProgramData\nexus\jre\bin\keytool.exe"`.

```powershell
# List all entries in a keystore (verbose)
& $KeyTool -list -v -keystore <path> -storepass <password>

# List entries (summary only)
& $KeyTool -list -keystore <path> -storepass <password>

# Import a certificate into a truststore
& $KeyTool -importcert -keystore <path> -storepass <password> -alias <alias> -file <cert.cer> -noprompt

# Delete a certificate entry
& $KeyTool -delete -keystore <path> -storepass <password> -alias <alias>

# Export a certificate from a keystore
& $KeyTool -exportcert -keystore <path> -storepass <password> -alias <alias> -file <output.cer>

# Import a PKCS12/PFX into a JKS keystore
& $KeyTool -importkeystore `
    -srckeystore <src.pfx> -srcstoretype PKCS12 -srcstorepass <srcpass> `
    -destkeystore <dest.jks> -deststoretype JKS -deststorepass <destpass> `
    -alias <srcalias> -destalias <destalias>

# Change key password inside keystore
& $KeyTool -keypasswd -keystore <path> -alias <alias> -storepass <storepass> -keypass <old> -new <new>
```

---

## Common Errors and Fixes

### `Object reference not set to an instance of an object` on `Export-PfxCertificate`

**Cause:** `$certificate` is null — the thumbprint lookup returned nothing.

**Fix:**
1. Verify the cert is in the expected store: `Get-ChildItem Cert:\LocalMachine\TrustedPeople`
2. Check thumbprint length — should be exactly 40:
   ```powershell
   $Thumbprint.Length
   ```
3. Strip hidden characters:
   ```powershell
   $Thumbprint = ($Thumbprint -replace '[^\x20-\x7E]', '').Trim().ToUpper()
   ```

---

### Wrong fingerprint shown in Nexus LDAPS UI

**Cause:** Nexus is either showing a cached fingerprint or the CA cert instead of the leaf cert.

**Fix:**
1. Pull the cert directly from the LDAP server (see [Step 2 above](#step-2--pull-the-cert-your-ldap-server-is-actually-presenting))
2. Import the correct CA cert into `cacerts`
3. Restart Nexus

---

### `PKIX path building failed` in Nexus logs

**Cause:** The CA that signed the LDAP cert is not in the Nexus truststore.

**Fix:** Export the internal CA cert and import into `cacerts` (see [Step 3–4 above](#step-3--export-the-ldap-ca-cert)).

---

### Cert visible in Windows but Nexus doesn't use it

**Cause:** Nexus uses Java's truststore, not the Windows certificate store.

**Fix:** Always import certs directly into the Nexus JRE truststore using `keytool`. Windows store imports have no effect on Nexus.

---

### `keytool error: java.io.FileNotFoundException` on keystore

**Cause:** The keystore path doesn't exist yet, or the path is wrong.

**Fix:** Confirm the path exists. For a new keystore, `keytool` will create it on first import — ensure the directory exists:
```powershell
New-Item -ItemType Directory -Path C:\ProgramData\nexus\etc\ssl -Force
```

---

*Last updated: March 2026*
