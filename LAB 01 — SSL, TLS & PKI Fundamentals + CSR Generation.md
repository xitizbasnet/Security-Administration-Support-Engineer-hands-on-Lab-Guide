# LAB 01 — SSL, TLS & PKI Fundamentals + CSR Generation

> **Understand certificates, keys, and trust — the foundation of all security work**

## Objective

Build a solid understanding of PKI concepts and generate your first CSR and certificate using OpenSSL.

---

## Task 1: Understand PKI Core Concepts

**Public Key Infrastructure (PKI)** is the backbone of digital trust. Before any practical work, understand these pillars:

* **CA (Certificate Authority)** — Issues and signs digital certificates (e.g., DigiCert, Let's Encrypt, internal ADCS).
* **SSL/TLS Certificate** — Binds a public key to an identity (domain, organization).
* **Private Key** — Never leaves the server; used to decrypt data encrypted with the matching public key.
* **CSR (Certificate Signing Request)** — Contains your public key + identity info; sent to a CA for signing.
* **Trust Chain** — Root CA → Intermediate CA → End-Entity Certificate. All must be trusted.
* **SAN (Subject Alternative Name)** — Allows one cert to cover multiple domains/IPs.
* **Validity Period** — Certificates expire; lifecycle management tracks and renews them in time.

> **❗ Best Practice**
>
> Always distinguish between Root CA, Intermediate CA, and Leaf certificates in any chain. Chain ordering matters during installation.

---

## Task 2: Install OpenSSL & Verify Environment

### Step 1: Install OpenSSL (Linux/Ubuntu)

```bash
sudo apt update && sudo apt install openssl -y
```

### Step 2: Verify Installation

```bash
openssl version -a
```

### Step 3: Check Available Commands

```bash
openssl help
```

> **❗ Best Practice**
>
> On Windows, use Git Bash or WSL for OpenSSL commands. For production, prefer HSM-backed key generation.

---

## Task 3: Generate Private Key & CSR

### Step 1: Create a Working Directory

```bash
mkdir ~/pki-lab && cd ~/pki-lab
```

### Step 2: Generate a 2048-bit RSA Private Key

```bash
openssl genrsa -out server.key 2048
```

### Step 3: Generate CSR

You will be prompted for details:

```bash
openssl req -new -key server.key -out server.csr \
-subj "/C=IN/ST=Maharashtra/L=Mumbai/O=MyCompany/CN=mysite.example.com"
```

### Step 4: View the CSR Contents

```bash
openssl req -text -noout -verify -in server.csr
```

### Step 5: Generate CSR with SAN (Recommended)

> **📄 Source Reference:** Security Administration & Support Engineer Lab Guide | Page 3 of 27

Create the SAN configuration file:

```bash
cat > san.cnf <<EOF
[req]
req_extensions = v3_req
distinguished_name = req_distinguished_name
[req_distinguished_name]
[v3_req]
subjectAltName = @alt_names
[alt_names]
DNS.1 = mysite.example.com
DNS.2 = www.mysite.example.com
IP.1 = 192.168.1.10
EOF
```

Generate the CSR using the SAN configuration:

```bash
openssl req -new -key server.key -out server-san.csr -config san.cnf
```

> **❗ Best Practice**
>
> Always use SANs. Browsers no longer accept certificates with only a Common Name (CN). Include www and non-www variants.

---

## Task 4: Create a Self-Signed Certificate

### Step 1: Generate a Self-Signed Certificate

For testing/dev:

```bash
openssl x509 -req -days 365 -in server.csr \
-signkey server.key -out server.crt
```

### Step 2: Combine Key + Certificate into PEM Bundle

```bash
cat server.crt server.key > server-bundle.pem
```

### Step 3: Create PKCS#12 (PFX)

**PKCS#12 (PFX)** is needed for IIS/Windows:

```bash
openssl pkcs12 -export -out server.pfx \
-inkey server.key -in server.crt -certfile ca.crt \
-passout pass:YourPassword123
```

---

## Task 5: Decode & Verify a Certificate

### Step 1: View Full Certificate Details

```bash
openssl x509 -in server.crt -text -noout
```

### Step 2: Check Expiry Date Only

```bash
openssl x509 -in server.crt -noout -dates
```

### Step 3: Verify Certificate Against CA

```bash
openssl verify -CAfile ca.crt server.crt
```

### Step 4: Check if Certificate Matches Private Key

Fingerprints must match:

```bash
openssl x509 -noout -modulus -in server.crt | openssl md5
openssl rsa -noout -modulus -in server.key | openssl md5
```

> **❗ Best Practice**
>
> Always verify key-cert pair match before deployment. A mismatch causes TLS handshake failure immediately.
