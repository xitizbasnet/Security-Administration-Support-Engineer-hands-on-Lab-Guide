# LAB 07 — SSL, TLS Troubleshooting

> **Diagnose and resolve trust chain, expiry, handshake, and configuration failures**

## Objective

Systematically troubleshoot the most common SSL/TLS errors encountered in enterprise support — trust chain issues, certificate expiry, handshake failures, and configuration errors.

---

## Task 1: Diagnose Trust Chain Issues

### Error

`SSL certificate problem: unable to get local issuer certificate` or `ERR_CERT_AUTHORITY_INVALID`

### Step 1: Check the Full Chain Being Served

```bash id="v7w5xe"
openssl s_client -connect mysite.example.com:443 -showcerts 2>/dev/null \
| awk '/BEGIN CERTIFICATE/,/END CERTIFICATE/' > chain.pem
cat chain.pem
```

### Step 2: Verify Each Certificate in the Chain

```bash id="8qv2dj"
openssl verify -CAfile root-ca.crt -untrusted intermediate.crt server.crt
```

### Step 3: Use Online Tools

* `whatsmychaincert.com`
* `sslshopper.com/ssl-checker`

### Step 4: Fix the Trust Chain

Ensure the intermediate CA is bundled with the certificate on the server.

> **❗ Best Practice**
>
> Trust chain issues are the #1 SSL support ticket. The server must serve:
>
> **Leaf cert → Intermediate CA cert** (in order).
>
> Root CA is in the browser trust store, not served.

---

## Task 2: Certificate Expiry Diagnosis & Response

### Error

`NET::ERR_CERT_DATE_INVALID` or `certificate has expired`

### Step 1: Confirm Expiry

```bash id="dj8h7u"
echo | openssl s_client -connect mysite.example.com:443 2>/dev/null \
| openssl x509 -noout -dates
```

### Step 2: Batch Check Multiple Domains

```bash id="5r3cx0"
#!/bin/bash
for domain in site1.com site2.com site3.com; do
DAYS=$(( ($(date -d "$(echo | openssl s_client -connect $domain:443 2>/dev/null \
| openssl x509 -noout -enddate | cut -d= -f2)" +%s) - $(date +%s)) / 86400 ))
echo "$domain: $DAYS days remaining"
done
```

### Step 3: Immediate Remediation

Obtain and install a renewed certificate. Refer to **Lab 3, Task 2**.

> **❗ Best Practice**
>
> For emergencies, install a self-signed cert temporarily to restore service, then replace with a CA-signed cert. Communicate to users that trust warnings are temporary.

---

## Task 3: Debug TLS Handshake Failures

### Error

`SSL_ERROR_HANDSHAKE_FAILURE_ALERT` or `ssl handshake failure`

### Step 1: Full `s_client` Diagnostic

```bash id="fz9x4e"
openssl s_client -connect mysite.example.com:443 \
-servername mysite.example.com -tls1_2 -debug 2>&1 | head -100
```

### Step 2: Test Specific TLS Versions

> **📄 Source Reference:** Security Administration & Support Engineer Lab Guide | Page 17 of 27

Test TLS 1.0:

```bash id="3qk1b7"
openssl s_client -connect mysite.example.com:443 -tls1 # Should fail if disabled
```

Test TLS 1.2:

```bash id="0f56j1"
openssl s_client -connect mysite.example.com:443 -tls1_2 # Should succeed
```

Test TLS 1.3:

```bash id="d6pj5v"
openssl s_client -connect mysite.example.com:443 -tls1_3 # Test TLS 1.3
```

### Step 3: Common Causes and Fixes

* **Protocol mismatch:** Client supports TLS1.2, server configured TLS1.3 only — align protocol versions.
* **Cipher mismatch:** No common cipher suite — add broader cipher list in server config.
* **SNI issue:** Old clients don't send SNI — use a dedicated IP for those legacy clients.
* **Client cert required:** Server expects mTLS — provide client certificate.

> **❗ Best Practice**
>
> Always test with multiple tools: `openssl s_client`, `curl -v`, `nmap --script ssl-enum-ciphers`, and Qualys SSL Labs. Each reveals different details.

---

## Task 4: Configuration Errors & Mixed Content

### Error

`Mixed Content: The page was loaded over HTTPS, but requested an insecure resource`

### Step 1: Find Mixed Content in Browser

Open the browser developer tools:

```text id="wvlz0h"
F12 > Console
```

Look for **Mixed Content** warnings.

### Step 2: Fix in Application/HTML

Change all `http://` links to `https://` or use protocol-relative `//` URLs.

### Step 3: Force HTTPS at Web Server Level (HSTS)

**Apache:**

```apache id="0udtgj"
Header always set Strict-Transport-Security 'max-age=31536000; includeSubDomains'
```

**NGINX:**

```nginx id="ebpsr5"
add_header Strict-Transport-Security 'max-age=31536000; includeSubDomains' always;
```

### Step 4: Test Configuration with `testssl.sh`

Clone the repository:

```bash id="8i0xnl"
git clone https://github.com/drwetter/testssl.sh.git
```

Navigate to the directory:

```bash id="j7g0v5"
cd testssl.sh
```

Run the test:

```bash id="v9sl2m"
bash testssl.sh mysite.example.com
```

---

## Task 5: Full `s_client` Diagnostic Workflow

### Step 1: Full Chain + Certificate Information

```bash id="7c4z2j"
# 1. Full chain + certificate info
openssl s_client -connect mysite.example.com:443 -showcerts
```

### Step 2: Check Server Certificate SANs & CN

```bash id="d0pryq"
# 2. Check server certificate sans & CN
openssl s_client -connect mysite.example.com:443 \
| openssl x509 -noout -text | grep -A1 'Subject Alternative'
```

### Step 3: Check if Certificate Is Revoked via OCSP

```bash id="u3m7nq"
# 3. Check if cert is revoked via OCSP
openssl s_client -connect mysite.example.com:443 -status 2>/dev/null \
| grep 'OCSP response'
```

### Step 4: Check Cipher Negotiated

```bash id="6z8n3h"
# 4. Check cipher negotiated
openssl s_client -connect mysite.example.com:443 2>/dev/null \
| grep 'Cipher is'
```

### Step 5: Test STARTTLS for SMTP/IMAP

```bash id="o7y2fk"
# 5. Test STARTTLS (for SMTP/IMAP)
openssl s_client -connect mail.example.com:587 -starttls smtp
```
