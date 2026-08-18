# LAB 08 — Automation & CI, CD Integration

> **Automate certificate workflows with Bash, Ansible, GitHub Actions, and Python**

## Objective

Build automated certificate management pipelines using industry-standard DevOps tools and scripting languages.

---

## Task 1: Bash Script — Automated CSR Generation

The following Bash script automates private key and CSR generation:

```bash
#!/bin/bash
# auto-csr.sh — Automated CSR + Key Generation
set -euo pipefail
DOMAIN=${1:-mysite.example.com}
KEY_SIZE=${2:-2048}
OUTDIR=~/pki/$DOMAIN
mkdir -p $OUTDIR
echo '[+] Generating private key...'
openssl genrsa -out $OUTDIR/server.key $KEY_SIZE
echo '[+] Generating CSR...'
openssl req -new -key $OUTDIR/server.key \
-out $OUTDIR/server.csr \
-subj "/C=IN/ST=Maharashtra/L=Mumbai/O=MyOrg/CN=$DOMAIN"
echo "[+] CSR generated: $OUTDIR/server.csr"
echo "[+] Verify CSR:"
openssl req -text -noout -in $OUTDIR/server.csr | grep -E "Subject:|DNS:"
```

### Step 1: Make Executable and Run

```bash
chmod +x auto-csr.sh && ./auto-csr.sh mysite.example.com 4096
```

> **❗ Best Practice**
>
> Parameterize all scripts — domain, key size, validity — so they're reusable across environments. Store generated keys in a secrets manager, not on disk.

---

## Task 2: GitHub Actions — TLS Certificate Workflow

Create the following GitHub Actions workflow:

```yaml
# .github/workflows/cert-check.yml
name: Certificate Health Check
on:
schedule:
- cron: '0 6 * * 1' # Every Monday 6 AM
workflow_dispatch:
jobs:
cert-check:
runs-on: ubuntu-latest
steps:
- name: Check Certificate Expiry
run: |
DOMAIN=mysite.example.com
DAYS=$(( ($(date -d "$(echo | openssl s_client \
-connect $DOMAIN:443 2>/dev/null \
| openssl x509 -noout -enddate \
| cut -d= -f2)" +%s) - $(date +%s)) / 86400 ))
echo "Days until expiry: $DAYS"
if [ $DAYS -lt 30 ]; then
echo "WARNING: Certificate expires in $DAYS days!"
exit 1
fi
- name: Send Slack Alert
if: failure()
uses: slackapi/slack-github-action@v1
with:
payload: '{"text": "Certificate expiry warning on mysite!"}'
```

> **📄 Source Reference:** Security Administration & Support Engineer Lab Guide | Page 19 of 27

---

## Task 3: Ansible — Certificate Deployment Playbook

Create the following Ansible playbook:

```yaml
# deploy-cert.yml
---
- name: Deploy SSL Certificate to Web Servers
hosts: webservers
become: true
vars:
cert_src: ./certs/server.crt
key_src: ./certs/server.key
chain_src: ./certs/ca-chain.crt
tasks:
- name: Copy SSL certificate
copy:
src: '{{ cert_src }}'
dest: /etc/ssl/certs/server.crt
mode: '0644'
- name: Copy private key
copy:
src: '{{ key_src }}'
dest: /etc/ssl/private/server.key
mode: '0600'
- name: Copy CA chain
copy:
src: '{{ chain_src }}'
dest: /etc/ssl/certs/ca-chain.crt
mode: '0644'
- name: Test NGINX config
command: nginx -t
- name: Reload NGINX
service:
name: nginx
state: reloaded
```

### Step 1: Run the Playbook

```bash
ansible-playbook deploy-cert.yml -i inventory.ini --diff
```

> **❗ Best Practice**
>
> Use Ansible Vault to encrypt private keys in playbooks:
>
> ```bash
> ansible-vault encrypt server.key
> ```
>
> Never store unencrypted private keys in Git repositories.

> **📄 Source Reference:** Security Administration & Support Engineer Lab Guide | Page 20 of 27

---

## Task 4: Python — Certificate Expiry Monitor

The following Python script monitors certificate expiry across multiple domains:

```python
#!/usr/bin/env python3
# cert-monitor.py
import ssl, socket, datetime, smtplib
from email.message import EmailMessage
DOMAINS = ['mysite.example.com', 'api.example.com', 'admin.example.com']
WARN_DAYS = 30
def check_cert(domain, port=443):
ctx = ssl.create_default_context()
with ctx.wrap_socket(socket.socket(), server_hostname=domain) as s:
s.connect((domain, port))
cert = s.getpeercert()
expiry = datetime.datetime.strptime(cert['notAfter'], '%b %d %H:%M:%S %Y %Z')
days_left = (expiry - datetime.datetime.utcnow()).days
return days_left, expiry
for domain in DOMAINS:
try:
days, expiry = check_cert(domain)
status = 'WARNING' if days < WARN_DAYS else 'OK'
print(f'[{status}] {domain}: {days} days left (expires {expiry.date()})')
except Exception as e:
print(f'[ERROR] {domain}: {e}')
```

> **❗ Best Practice**
>
> Schedule this script via cron:
>
> ```bash
> 0 8 * * * python3 /opt/scripts/cert-monitor.py >> /var/log/cert-monitor.log 2>&1
> ```
