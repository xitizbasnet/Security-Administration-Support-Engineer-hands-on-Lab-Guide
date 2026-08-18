# LAB 02 — Certificate Installation on Web Servers

> **Deploy SSL/TLS certificates on Apache, NGINX, and IIS — with verification**

## Objective

Install and configure TLS certificates on the three most common web server platforms used in enterprise environments.

---

## Task 1: Install SSL Certificate on Apache (Linux)

### Step 1: Enable the SSL Module

```bash
sudo a2enmod ssl && sudo systemctl restart apache2
```

### Step 2: Copy Certificate Files to a Secure Location

```bash
sudo cp server.crt /etc/ssl/certs/mysite.crt
sudo cp server.key /etc/ssl/private/mysite.key
sudo cp ca-chain.crt /etc/ssl/certs/ca-chain.crt
```

### Step 3: Edit or Create the VirtualHost Configuration

```bash
sudo nano /etc/apache2/sites-available/mysite-ssl.conf
```

Configure the VirtualHost:

```apache
<VirtualHost *:443>
ServerName mysite.example.com
SSLEngine on
SSLCertificateFile /etc/ssl/certs/mysite.crt
SSLCertificateKeyFile /etc/ssl/private/mysite.key
SSLCertificateChainFile /etc/ssl/certs/ca-chain.crt
SSLProtocol all -SSLv2 -SSLv3 -TLSv1 -TLSv1.1
SSLCipherSuite HIGH:!aNULL:!MD5
DocumentRoot /var/www/html
</VirtualHost>
```

### Step 4: Enable the Site and Test the Configuration

```bash
sudo a2ensite mysite-ssl.conf
sudo apachectl configtest
sudo systemctl reload apache2
```

> **❗ Best Practice**
>
> Always run `apachectl configtest` before reloading. A syntax error in TLS config will take the site offline.

---

## Task 2: Install SSL Certificate on NGINX

### Step 1: Create Combined Certificate (Certificate + Chain)

```bash
cat server.crt ca-chain.crt > mysite-bundle.crt
```

### Step 2: Edit NGINX Server Block

```bash
sudo nano /etc/nginx/sites-available/mysite
```

Configure the server block:

```nginx
server {
listen 443 ssl;
server_name mysite.example.com;
ssl_certificate /etc/ssl/certs/mysite-bundle.crt;
ssl_certificate_key /etc/ssl/private/mysite.key;
ssl_protocols TLSv1.2 TLSv1.3;
ssl_ciphers ECDHE-RSA-AES256-GCM-SHA512:DHE-RSA-AES256-GCM-SHA512;
ssl_prefer_server_ciphers on;
ssl_session_cache shared:SSL:10m;
ssl_session_timeout 10m;
add_header Strict-Transport-Security 'max-age=31536000' always;
location / { root /var/www/html; }
}
```

> **📄 Source Reference:** Security Administration & Support Engineer Lab Guide | Page 5 of 27

### Step 3: Test and Reload

```bash
sudo nginx -t && sudo systemctl reload nginx
```

> **❗ Best Practice**
>
> Bundle cert + chain in NGINX (unlike Apache where you specify separately). Missing the chain causes 'certificate not trusted' errors for some clients.

---

## Task 3: Install Certificate on IIS (Windows Server)

### Step 1: Import PFX Certificate via PowerShell

```powershell
Import-PfxCertificate -FilePath C:\Certs\server.pfx `
-CertStoreLocation Cert:\LocalMachine\My `
-Password (ConvertTo-SecureString 'YourPassword123' -AsPlainText -Force)
```

### Step 2: Bind Certificate to HTTPS Site (PowerShell)

```powershell
Import-Module WebAdministration
$cert = Get-ChildItem Cert:\LocalMachine\My | Where-Object {$_.Subject -like '*mysite*'}
New-WebBinding -Name 'Default Web Site' -Protocol https -Port 443 -HostHeader mysite.example.com
$bind = Get-WebBinding -Name 'Default Web Site' -Protocol https
$bind.AddSslCertificate($cert.Thumbprint, 'My')
```

### Step 3: Or via IIS Manager GUI

**Site > Bindings > Add > HTTPS > Select Certificate**

### Step 4: Verify with PowerShell

```powershell
Get-WebBinding -Name 'Default Web Site' | Where-Object {$_.Protocol -eq 'https'}
```

> **❗ Best Practice**
>
> On IIS, always ensure the private key has permissions for the IIS_IUSRS / NETWORK SERVICE account. A missing permission causes HTTP 403.16 errors.

---

## Task 4: Verify Certificate Installation

### Step 1: Test with `curl`

```bash
curl -vI https://mysite.example.com
```

For self-signed certificates:

```bash
curl -k -vI https://mysite.example.com
```

### Step 2: Test with OpenSSL `s_client`

```bash
openssl s_client -connect mysite.example.com:443 -servername mysite.example.com
```

### Step 3: Online Tools (in Browser)

* `ssllabs.com`
* `whatsmychaincert.com`

### Step 4: Check HSTS Headers

```bash
curl -I https://mysite.example.com | grep Strict
```

---

## Task 5: HTTP to HTTPS Redirect

### Apache — Add to Port 80 VirtualHost

```apache
<VirtualHost *:80>
ServerName mysite.example.com
Redirect permanent / https://mysite.example.com/
</VirtualHost>
```

> **📄 Source Reference:** Security Administration & Support Engineer Lab Guide | Page 6 of 27

### NGINX — Add Server Block

```nginx
server {
listen 80;
server_name mysite.example.com;
return 301 https://$host$request_uri;
}
```

> **❗ Best Practice**
>
> Use 301 (permanent) redirect for production, 302 (temporary) only during testing. 301 redirects are cached by browsers and search engines.
