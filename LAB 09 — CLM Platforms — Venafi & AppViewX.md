# LAB 09 — CLM Platforms — Venafi & AppViewX

> **Manage enterprise certificate lifecycles using industry-leading CLM tools**

## Objective

Navigate Venafi Trust Protection Platform and AppViewX to manage certificate discovery, policies, requests, and automation at enterprise scale.

---

## Task 1: Venafi Dashboard Overview

**Venafi Trust Protection Platform (TPP)** is the industry-leading CLM for large enterprises.

### Key Modules

* **Policy Folders:** Organize certificates by environment, team, or project with enforced policies.
* **Certificate Authorities:** Pre-configured connections to DigiCert, Entrust, Sectigo, or private CAs.
* **Discovery:** Scans network for certificates on all ports (not just 443).
* **Workflow:** Approval chains for certificate requests.
* **Dashboard:** Expiry heatmaps, risk scores, compliance views.

### Step 1: Review All Certificates

Navigate to:

```text
Certificates > All Certificates
```

Review the expiry columns.

### Step 2: Find At-Risk Certificates

Filter by:

```text
Expiration < 30 days
```

This identifies certificates that are approaching expiration.

### Step 3: Configure Email Alerts

Navigate to:

```text
Policy > Notifications > Certificate Expiration
```

Configure email alerts for certificate expiration.

---

## Task 2: Create Policy & Request a Certificate

### Step 1: Create a Policy Folder

Navigate to:

```text
Platform > Policies > New Policy
```

Configure the following:

* **CA Template**
* **Key Algorithm:** RSA 2048+
* **Validity**
* **Subject DN rules**
* **Auto-renew settings**

### Step 2: Request a Certificate

**Venafi API — Request certificate:**

```bash id="4r8h2e"
curl -X POST https://venafi.example.com/vedsdk/certificates/request \
-H 'Content-Type: application/json' \
-H 'X-Venafi-API-Key: YOUR_API_KEY' \
-d '{
"PolicyDN": "\\VED\\Policy\\MyOrg\\WebServers",
"Subject": "mysite.example.com",
"SubjectAltNames": [{"TypeName": "DNS", "Name": "www.mysite.example.com"}]
}'
```

### Step 3: Check Certificate Status and Retrieve

```bash id="0o9kq8"
curl -X POST https://venafi.example.com/vedsdk/certificates/retrieve \
-H 'X-Venafi-API-Key: YOUR_API_KEY' \
-d '{"CertificateDN": "<DN>", "Format": "PEM", "IncludePrivateKey": true}'
```

> **❗ Best Practice**
>
> Always use Venafi policies to enforce certificate standards. Ad-hoc requests without policy lead to certificate sprawl and compliance failures.

---

## Task 3: Venafi Adaptable CA & API Basics

### Step 1: List All Certificates via API

```bash id="n6p0yx"
curl -X GET 'https://venafi.example.com/vedsdk/certificates?Limit=50&Offset=0' \
-H 'X-Venafi-API-Key: YOUR_API_KEY' | python3 -m json.tool
```

### Step 2: Python SDK Example

> **📄 Source Reference:** Security Administration & Support Engineer Lab Guide | Page 22 of 27

```python id="j9qz4h"
from vcert import Connection, TPPTokenConnection
conn = TPPTokenConnection(url='https://venafi.example.com',
user='svc_venafi', password='Secret123')
request = conn.request_cert(common_name='mysite.example.com',
zone='MyOrg\\WebServers')
cert = conn.retrieve_cert(request)
print(cert.cert, cert.key)
```

---

## Task 4: AppViewX Workflow Setup

**AppViewX** specializes in network and F5/NetScaler certificate automation.

### Core Concepts

* **Workflows:** Visual drag-and-drop certificate lifecycle automation.
* **Connectors:** Pre-built integrations with F5, Citrix, AWS, Azure, Kubernetes.
* **Certificate Groups:** Logical grouping for bulk operations.
* **Audit Trail:** Full history of all cert operations for compliance.

### Step 1: Create Renewal Workflow

Navigate to:

```text
Workflows > Create > Certificate Renewal
```

### Step 2: Add Workflow Steps

Configure the workflow with the following steps:

```text
Discover > Generate CSR > Submit to CA > Install on Device > Verify
```

### Step 3: Schedule Automatic Renewal

Set the trigger at **60 days before expiry** for auto-renewal.

> **❗ Best Practice**
>
> AppViewX is particularly powerful for F5 BIG-IP environments. Its connector handles cert install + partition binding automatically.

---

## Task 5: Certificate Discovery Scan

### Step 1: Run Network Discovery with OpenSSL

The following simple Bash script scans a subnet for HTTPS certificates:

```bash id="6n8q0w"
#!/bin/bash
# Scan a subnet for HTTPS certs
for ip in 192.168.1.{1..254}; do
echo | timeout 3 openssl s_client -connect $ip:443 2>/dev/null \
| openssl x509 -noout -subject -dates 2>/dev/null \
&& echo "Host: $ip"
done
```

### Step 2: Use Nmap for Broader Discovery

```bash id="h5k2qy"
nmap -p 443,8443,8080 --script ssl-cert 192.168.1.0/24 \
| grep -E 'ssl-cert|subject|validity'
```

### Step 3: Export Results to CSV for Reporting

Generate XML output:

```bash id="w3z6pf"
nmap -p 443 --script ssl-cert 192.168.1.0/24 -oX scan.xml
```

Parse the results with Python/xsltproc for CSV output.
