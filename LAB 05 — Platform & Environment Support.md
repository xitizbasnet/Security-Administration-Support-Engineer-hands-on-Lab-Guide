# LAB 05 — Platform & Environment Support

> **Configure TLS on cPanel, Plesk, Firewalls, AWS ALB, and Azure App Gateway**

## Objective

Deploy and troubleshoot certificates across diverse hosting platforms, network devices, and cloud load balancers.

---

## Task 1: SSL Installation on cPanel

### Step 1: Access the SSL/TLS Manager

Log in to:

```text
cPanel > Security > SSL/TLS Manager
```

### Step 2: Upload the Certificate

Select:

```text
Manage SSL Sites
```

Paste the following certificate components:

* **CRT**
* **KEY**
* **CA Bundle**

### Step 3: Use AutoSSL

Alternatively, use AutoSSL:

```text
cPanel > Security > SSL/TLS Status > Run AutoSSL
```

### Step 4: Verify via CLI (WHM)

Run:

```bash id="w0hzdb"
 /usr/local/cpanel/bin/checkallsslcerts --verbose
```

**Check a specific domain:**

```bash id="d2c0qv"
# Check specific domain:
openssl s_client -connect yourdomain.com:443 -servername yourdomain.com
```

> **❗ Best Practice**
>
> cPanel's AutoSSL uses Let's Encrypt or Sectigo. Ensure the domain's DNS resolves correctly before running AutoSSL, or it will fail DV validation.

---

## Task 2: Plesk SSL Configuration

### Step 1: Open SSL/TLS Certificates

Navigate to:

```text
Plesk Admin > Domains > mysite.example.com > SSL/TLS Certificates
```

### Step 2: Upload or Request a Certificate

Upload a certificate or request a Let's Encrypt certificate directly in the UI.

### Step 3: Configure via CLI (Plesk)

Add the certificate:

```bash id="i5p7fv"
plesk bin certificate --add -name mysite-cert \
-key-file server.key -cert-file server.crt -chain-file ca-chain.crt \
-domain mysite.example.com
```

Update the domain:

```bash id="0gwhod"
plesk bin domain --update mysite.example.com -ssl-certificate mysite-cert
```

> **❗ Best Practice**
>
> Plesk stores SSL configs separately from Apache/NGINX. Changes via Plesk UI override manual edits to vhost config files.

---

## Task 3: TLS Termination on Firewall / WAF / Load Balancer

In enterprise setups, TLS is often terminated at the perimeter (WAF or LB), not the web server. This is called **TLS offloading**.

### Step 1: Concept — TLS Termination Flow

```text
Client → [HTTPS 443] → WAF/LB [TLS Terminate] → [HTTP 80] → Backend Servers
```

### Step 2: Import Certificate on F5 BIG-IP

```bash id="8c7z0f"
tmsh install sys crypto cert mysite.crt from-local-file /tmp/server.crt
tmsh install sys crypto key mysite.key from-local-file /tmp/server.key
tmsh create ltm profile client-ssl mysite-ssl cert mysite.crt key mysite.key chain ca-chain.crt
```

### Step 3: HAProxy TLS Termination

```haproxy id="0u5m6t"
frontend https_front
bind *:443 ssl crt /etc/haproxy/certs/mysite.pem
default_backend webservers
```

> **📄 Source Reference:** Security Administration & Support Engineer Lab Guide | Page 12 of 27

Configure the backend servers:

```haproxy id="9y6qwv"
backend webservers
server web1 192.168.1.10:80 check
```

> **❗ Best Practice**
>
> After TLS offloading, use `X-Forwarded-For` and `X-Forwarded-Proto` headers to preserve client IP and original scheme info for backend apps.

---

## Task 4: AWS ALB HTTPS Listener

### Step 1: Upload Certificate to ACM (AWS Certificate Manager)

Import the certificate:

```bash id="8y4t9j"
aws acm import-certificate \
--certificate fileb://server.crt \
--private-key fileb://server.key \
--certificate-chain fileb://ca-chain.crt \
--region ap-south-1
```

### Step 2: Request a Certificate Directly in ACM

Use DNS or email validation:

```bash id="xw50jz"
aws acm request-certificate \
--domain-name mysite.example.com \
--validation-method DNS \
--subject-alternative-names www.mysite.example.com
```

### Step 3: Add HTTPS Listener to ALB

```bash id="6bthbz"
aws elbv2 create-listener \
--load-balancer-arn <ALB-ARN> \
--protocol HTTPS --port 443 \
--certificates CertificateArn=<ACM-CERT-ARN> \
--default-actions Type=forward,TargetGroupArn=<TG-ARN>
```

> **❗ Best Practice**
>
> ACM certificates auto-renew and cannot be exported (no private key access). For EC2-based servers needing PEM files, use Let's Encrypt or import your own cert.

---

## Task 5: Azure App Gateway TLS

### Step 1: Convert Certificate to PFX Format

Convert the certificate to PFX format first (required by Azure):

```bash id="e0yq6m"
openssl pkcs12 -export -out mysite.pfx \
-inkey server.key -in server.crt -certfile ca-chain.crt \
-passout pass:AzurePassword123
```

### Step 2: Upload to App Gateway via CLI

```bash id="owv2qf"
az network application-gateway ssl-cert create \
--gateway-name myAppGW --resource-group myRG \
--name mysite-ssl --cert-file mysite.pfx --cert-password AzurePassword123
```

### Step 3: Attach to HTTPS Listener

```bash id="4o7lq9"
az network application-gateway http-listener update \
--gateway-name myAppGW --resource-group myRG \
--name httpsListener --ssl-cert mysite-ssl
```
