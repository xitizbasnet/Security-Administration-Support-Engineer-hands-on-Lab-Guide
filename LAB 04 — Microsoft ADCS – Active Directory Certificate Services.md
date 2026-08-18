# LAB 04 — Microsoft ADCS – Active Directory Certificate Services

> **Configure enterprise PKI with Windows Server ADCS, templates, and auto-enrollment**

## Objective

Install and configure AD CS, create custom certificate templates, enroll certificates manually and via GPO auto-enrollment.

---

## Task 1: Install AD CS Role (Windows Server)

### Step 1: Install via PowerShell

Run PowerShell as **Administrator**:

```powershell
Install-WindowsFeature -Name AD-Certificate, ADCS-Cert-Authority,
ADCS-Web-Enrollment -IncludeManagementTools
```

### Step 2: Configure as Enterprise Root CA

```powershell
Install-AdcsCertificationAuthority -CAType EnterpriseRootCA
-CACommonName 'MyOrg-RootCA' -KeyLength 2048
-HashAlgorithmName SHA256 -ValidityPeriod Years -ValidityPeriodUnits 10 -Force
```

### Step 3: Verify CA Is Running

```powershell
certutil -ping
```

> **❗ Best Practice**
>
> For production, use a **2-tier PKI: offline Root CA + online Issuing CA**. Never expose the Root CA to the network. Keep it powered off when not in use.

---

## Task 2: Create Certificate Templates

### Step 1: Open Certificate Templates Console

Run:

```text
certtmpl.msc
```

### Step 2: Duplicate an Existing Template

For example, duplicate the **Web Server** template:

**Right-click `Web Server` > Duplicate Template > Windows Server 2016 or later**

### Step 3: Configure the Template

Configure the following settings:

* **General tab:** Set name `MyWebServer`, validity 2 years, renewal 6 weeks.
* **Subject Name tab:** Select **Supply in the request** for web servers.
* **Extensions tab:** Add **Application Policies > Server Authentication**.
* **Security tab:** Add **Domain Computers** with **Read + Enroll** permissions.
* **Cryptography tab:** Key size 2048, Provider: RSA.

### Step 4: Publish the Template on the CA

**Via CA Console:**

```text
Right-click Certificate Templates > New > Certificate Template to Issue
```

**Or PowerShell:**

```powershell
add-CATemplate -TemplateName 'MyWebServer'
```

> **❗ Best Practice**
>
> Always duplicate existing templates — never modify the built-in ones. Test templates in a lab before publishing to production.

---

## Task 3: Request Certificate via MMC

### Step 1: Open Certificate Manager

Run:

```text
certmgr.msc
```

for the **user** certificate store, or:

```text
certlm.msc
```

for the **computer** certificate store.

### Step 2: Navigate to Certificate Enrollment

Navigate to:

```text
Personal > Certificates > All Tasks > Request New Certificate
```

### Step 3: Select the Certificate Template

Select the enrollment policy, choose your custom template, and fill in the required details.

> **📄 Source Reference:** Security Administration & Support Engineer Lab Guide | Page 10 of 27

### Step 4: PowerShell Certificate Request

```powershell
$cert = Get-Certificate -Template 'MyWebServer'
-DnsName 'mysite.example.com'
-CertStoreLocation Cert:\LocalMachine\My
$cert.Certificate.Thumbprint
```

---

## Task 4: Auto-Enrollment via Group Policy

### Step 1: Open Group Policy Management Console

Run:

```text
gpmc.msc
```

Create or edit a GPO.

### Step 2: Navigate to Auto-Enrollment Policy

Navigate to:

```text
Computer Config > Policies > Windows Settings > Security Settings > Public Key Policies
```

### Step 3: Configure Certificate Auto-Enrollment

Open:

```text
Certificate Services Client – Auto-Enrollment
```

Set **Configuration Model** to **Enabled**.

Check both **renewal** and **update** boxes.

### Step 4: Force GPO Update and Verify

Force a Group Policy update:

```cmd
gpupdate /force
```

Run certificate auto-enrollment:

```cmd
certutil -autoenroll -q
```

List all certificates in the computer store:

```powershell
Get-ChildItem Cert:\LocalMachine\My
```

> **❗ Best Practice**
>
> Auto-enrollment requires the CA to be reachable from all domain computers. Check firewall rules for TCP 443 (HTTPS) to the CA web enrollment URL.

---

## Task 5: Export & Backup the CA

### Step 1: Back Up CA Keys and Database

Back up the CA private keys:

```cmd
certutil -backupKey C:\CABackup
```

Back up the CA database:

```cmd
certutil -backupDB C:\CABackup
```

### Step 2: Export CA Certificate for Distribution

Export the CA certificate:

```cmd
certutil -ca.cert C:\CABackup\RootCA.cer
```

**Import on client machines:**

```cmd
certutil -addstore Root C:\CABackup\RootCA.cer
```

### Step 3: Schedule Regular Backups

Schedule regular CA backups using **Task Scheduler**.
