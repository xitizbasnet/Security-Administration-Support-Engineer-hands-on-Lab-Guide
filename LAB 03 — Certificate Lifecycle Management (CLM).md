# LAB 03 — Certificate Lifecycle Management (CLM)

> **Track, renew, revoke, and manage certificates from Private CA to OCSP validation**

## Objective

Manage the full certificate lifecycle — from issuance through renewal, revocation, and validation — using OpenSSL and automation scripts.

---

## Task 1: Track Certificate Expiry

### Step 1: Check a Single Certificate Expiry

```bash
openssl x509 -in server.crt -noout -enddate
```

### Step 2: Check Expiry on a Live Server

```bash
echo | openssl s_client -servername mysite.example.com \
-connect mysite.example.com:443 2>/dev/null \
| openssl x509 -noout -dates
```

### Step 3: Calculate Days Until Expiry (Bash Script)

```bash
#!/bin/bash
DOMAIN=$1
EXPIRY=$(echo | openssl s_client -servername $DOMAIN -connect $DOMAIN:443 2>/dev/null \
| openssl x509 -noout -enddate | cut -d= -f2)
EXPIRY_EPOCH=$(date -d "$EXPIRY" +%s)
NOW_EPOCH=$(date +%s)
DAYS=$(( ($EXPIRY_EPOCH - $NOW_EPOCH) / 86400 ))
echo "$DOMAIN expires in $DAYS days ($EXPIRY)"
```

> **❗ Best Practice**
>
> Set up monitoring alerts at **60 days, 30 days, and 7 days** before expiry. Most CLM platforms like Venafi and AppViewX automate this.

---

## Task 2: Renew a Certificate

### Step 1: Generate a New CSR

Generate a new CSR using an existing key or a new key.

**Reuse Existing Key:**

```bash
# Reuse existing key
openssl req -new -key server.key -out renewal.csr \
-subj "/C=IN/ST=Maharashtra/O=MyCompany/CN=mysite.example.com"
```

**Generate a New Key + CSR:**

```bash
# Or generate a new key + CSR
openssl req -newkey rsa:2048 -keyout newserver.key -out renewal.csr \
-subj "/C=IN/ST=Maharashtra/O=MyCompany/CN=mysite.example.com"
```

### Step 2: Submit CSR to the CA

Submit to CA and receive new certificate, then install following **Lab 2 procedures**.

### Step 3: Let's Encrypt Automated Renewal

Install Certbot and the NGINX plugin:

```bash
sudo apt install certbot python3-certbot-nginx -y
```

Request a certificate:

```bash
sudo certbot --nginx -d mysite.example.com
```

Test auto-renewal:

```bash
# Auto-renewal (runs twice daily)
sudo certbot renew --dry-run
```

> **❗ Best Practice**
>
> Let's Encrypt certificates expire in 90 days. Certbot installs a cron/systemd timer for auto-renewal. Verify it runs with:
>
> ```bash
> systemctl status certbot.timer
> ```

---

## Task 3: Revoke a Certificate

> **📄 Source Reference:** Security Administration & Support Engineer Lab Guide | Page 8 of 27

### Step 1: Revoke Using OpenSSL (Private CA)

```bash
openssl ca -revoke /etc/ssl/CA/certs/server.crt -config /etc/ssl/CA/openssl.cnf
```

**Reason codes:** `unspecified`, `keyCompromise`, `CACompromise`, `affiliationChanged`

### Step 2: Update the CRL (Certificate Revocation List)

Generate the CRL:

```bash
openssl ca -gencrl -out crl.pem -config /etc/ssl/CA/openssl.cnf
```

Verify the CRL contents:

```bash
openssl crl -in crl.pem -text -noout # Verify contents
```

> **❗ Best Practice**
>
> OCSP (Online Certificate Status Protocol) is preferred over CRL. CRLs can be huge and are downloaded entirely; OCSP gives real-time, per-cert status.

---

## Task 4: Build a Private CA with OpenSSL

### Step 1: Create CA Directory Structure

```bash
mkdir -p ~/myCA/{certs,crl,newcerts,private}
chmod 700 ~/myCA/private
touch ~/myCA/index.txt
echo 1000 > ~/myCA/serial
```

### Step 2: Create CA Config (`openssl-ca.cnf`) and Generate Root CA

Generate the Root CA private key:

```bash
openssl genrsa -aes256 -out ~/myCA/private/ca.key 4096
```

Generate the Root CA certificate:

```bash
openssl req -new -x509 -days 3650 -key ~/myCA/private/ca.key \
-out ~/myCA/certs/ca.crt -subj "/CN=MyPrivateCA/O=MyOrg/C=IN"
```

### Step 3: Sign a Server CSR with Your Private CA

```bash
openssl ca -in server.csr -out server-signed.crt \
-config ~/myCA/openssl-ca.cnf -days 365 -batch
```

> **❗ Best Practice**
>
> Distribute your Root CA certificate to all clients/browsers that need to trust it. On Windows: via **GPO > Computer Config > Policies > Windows Settings > Security Settings > Public Key Policies**.

---

## Task 5: OCSP Stapling & CRL Validation

### Step 1: Enable OCSP Stapling in NGINX

Add the following configuration:

```nginx
ssl_stapling on;
ssl_stapling_verify on;
resolver 8.8.8.8 1.1.1.1 valid=300s;
resolver_timeout 5s;
```

### Step 2: Query OCSP Manually

```bash
openssl ocsp -issuer ca.crt -cert server.crt \
-url http://ocsp.yourca.com -resp_text
```

### Step 3: Verify CRL

```bash
openssl verify -crl_check -CAfile ca.crt \
-CRLfile crl.pem server.crt
```
