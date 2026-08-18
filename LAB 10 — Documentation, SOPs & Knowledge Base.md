# LAB 10 — Documentation, SOPs & Knowledge Base

> **Create professional runbooks, SOPs, RCA templates, and customer-facing KB articles**

## Objective

Build the documentation foundation for enterprise certificate support — SOPs, RCA templates, onboarding checklists, and knowledge base articles.

---

## Task 1: Write a Certificate Renewal SOP

**Standard Operating Procedures** ensure consistent handling of recurring tasks.

### Template Structure

* **Title:** SSL/TLS Certificate Renewal SOP
* **Owner:** Security Engineering Team
* **Trigger:** Certificate expiring within 60 days (auto-alert from monitoring)
* **Scope:** All production SSL/TLS certificates managed by team
* **Prerequisites:** Access to CLM platform, CA portal, and target server

### SOP Steps

**Step 1.** Identify the certificate from expiry report or CLM alert.

**Step 2.** Verify domain ownership and business unit contact.

**Step 3.** Generate new CSR using approved key size (2048+ RSA or 256 ECDSA).

**Step 4.** Submit CSR to CA via CLM platform or CA portal.

**Step 5.** Complete domain/org validation if required by CA.

**Step 6.** Download and verify the issued certificate (chain, SANs, expiry).

**Step 7.** Deploy to target server(s) following platform-specific procedure.

**Step 8.** Verify deployment: test with `openssl s_client` and SSL checker tool.

**Step 9.** Update CLM platform with new certificate details.

**Step 10.** Close the ticket and notify business unit of completion.

> **❗ Best Practice**
>
> SOPs should include rollback steps. If a certificate deployment fails, document how to quickly revert to the previous cert to minimize downtime.

---

## Task 2: Incident RCA Template

**Root Cause Analysis** document for certificate-related outages:

| Field                | Details                                                                    |
| -------------------- | -------------------------------------------------------------------------- |
| **Incident ID**      | INC-2024-XXX                                                               |
| **Date/Time**        | YYYY-MM-DD HH:MM UTC                                                       |
| **Duration**         | X hours Y minutes                                                          |
| **Severity**         | P1 / P2 / P3                                                               |
| **Affected Service** | mysite.example.com (HTTPS)                                                 |
| **Root Cause**       | SSL certificate expired — renewal alert suppressed due to misconfiguration |

> **📄 Source Reference:** Security Administration & Support Engineer Lab Guide | Page 24 of 27

| Field               | Details                                                    |
| ------------------- | ---------------------------------------------------------- |
| **Timeline**        | Alert at 08:00 → Acknowledged at 08:15 → Resolved at 09:30 |
| **Immediate Fix**   | Deployed renewed certificate; restarted web server         |
| **Long-term Fix**   | Fixed monitoring alert; added 90-day pre-expiry reminder   |
| **Lessons Learned** | Add secondary monitoring via external SSL check tool       |

---

## Task 3: Customer Onboarding Checklist

Use this checklist when onboarding enterprise customers onto your CLM service:

* [ ] Gather customer domain inventory (all FQDNs, SANs, wildcard needs)
* [ ] Document server environments (Apache/NGINX/IIS, OS, cloud platform)
* [ ] Confirm certificate authority preference (DigiCert, Sectigo, Let's Encrypt, private CA)
* [ ] Set up CLM platform access (user accounts, policy folders, API keys)
* [ ] Configure CA connectors in CLM platform
* [ ] Run initial discovery scan to find existing certificates
* [ ] Import existing certificates into CLM for lifecycle tracking
* [ ] Test certificate request → issuance → installation workflow end-to-end
* [ ] Set up expiry alerting (30/60/90 day thresholds)
* [ ] Deliver documentation: API guide, escalation contacts, renewal SOP
* [ ] Schedule 30-day check-in call with customer

---

## Task 4: Production Rollout Runbook

**Migration of certificate from old CA to new CA / CLM system:**

```text id="x7q9n3"
PRODUCTION CERTIFICATE MIGRATION RUNBOOK
=========================================

Pre-checks (T-48 hours):
[ ] New certificate tested in staging environment
[ ] Chain verified with external SSL checker
[ ] Key-cert match confirmed (openssl md5 fingerprints)
[ ] Rollback plan documented
[ ] Change window approved by CAB

Execution (T=0 - Change Window):
[ ] Take snapshot/backup of current server config
[ ] Deploy new certificate per platform SOP
[ ] Test HTTPS connectivity (curl -vI, browser)
[ ] Verify chain: openssl s_client -showcerts
[ ] Run SSL Labs test (ssllabs.com)
[ ] Confirm certificate in CLM platform updated

Post-deployment (T+1 hour):
[ ] Monitor application logs for TLS errors
[ ] Verify monitoring/alerting resumed
[ ] Update CMDB/asset management record
[ ] Close change ticket with completion notes
```

---

## Task 5: Knowledge Base Article — Customer-Facing

> **📄 Source Reference:** Security Administration & Support Engineer Lab Guide | Page 25 of 27

### KB Article Template: `How to Request an SSL Certificate`

**Title:** How to Request an SSL/TLS Certificate

**Category:** Certificate Lifecycle Management | **Audience:** Customer

### Overview

This article guides you through requesting a new SSL/TLS certificate through our portal.

### Prerequisites

* Active account
* Domain ownership
* Web server access

### Steps

**Step 1.** Generate a CSR on your server (see **KB-001: Generating a CSR**).

**Step 2.** Log in to our portal at `portal.example.com > Certificates > New Request`.

**Step 3.** Paste your CSR, select certificate type and validity period.

**Step 4.** Complete domain validation (DV) — approve the email sent to `admin@yourdomain.com`.

**Step 5.** Download the issued certificate and CA bundle from the portal.

**Step 6.** Install on your server following our platform-specific guide (**KB-010 to KB-015**).

### Related Articles

* **KB-001:** CSR Generation
* **KB-010:** Apache Install
* **KB-011:** NGINX Install
* **KB-020:** Troubleshooting

> **❗ Best Practice**
>
> Every KB article should have: clear title, prerequisites, step-by-step instructions with screenshots, and links to related articles. Review quarterly for accuracy.
