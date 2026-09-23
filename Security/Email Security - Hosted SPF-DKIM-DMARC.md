Tags: #email #security #dns #sysadmin #proofpoint 


This document describes how **SPF, DKIM, and DMARC** are implemented when moving email security controls to **Proofpoint Hosted Email Defense**, and how **alignment** affects DMARC pass/fail outcomes.

> Proofpoint is not the only option, it is just what we are currently using
---
## Core Email Identity Concepts (Important)
Every email has **three relevant domain identities**:

|Identity|Where it lives|Used by|
|---|---|---|
|**From**|Message header (user‑visible)|DMARC|
|**Return‑Path (MAIL FROM)**|SMTP envelope|SPF|
|**DKIM `d=` domain**|DKIM-Signature header|DKIM + DMARC|

> **DMARC alignment compares SPF or DKIM results against the From domain**

### Super Important
DMARC requires a pass and an align on at least one SPF or DKIM.
- A pass on SPF but not an align is a fail in DMARC

---
## Authentication Protocols

### SPF (Sender Policy Framework)
- DNS TXT record
- Defines **which servers may send mail**
- Evaluated against the **Return‑Path domain**
- Subject to a **10 DNS lookup limit**

SPF **does not authenticate the From header**.

---
### DKIM (DomainKeys Identified Mail)
- Uses **asymmetric cryptography**
- Message is signed by the sender
- Public key published in DNS under `_domainkey`
- Signature references a domain via `d=`

DKIM **protects message integrity and sender domain identity**.

---
### DMARC (Domain‑based Message Authentication, Reporting & Conformance)
- DNS TXT record at `_dmarc.domain`
- Performs:
    - **Alignment checks**
    - **Policy enforcement**
    - **Reporting**
- Does **no authentication** itself

> DMARC consumes SPF and DKIM results.

---
## Alignment (Critical Concept)

### What alignment means
Alignment checks whether the **authenticated domain matches the From domain**.

- SPF alignment → Return‑Path domain = From
- DKIM alignment → `d=` domain = From

Alignment modes:
- **Relaxed (`r`)** → subdomains allowed _(recommended)_
- **Strict (`s`)** → exact match only

---
### SPF alignment example

```
From: user@company.com
Return‑Path: bounce@vendor-mailer.com
```

- ✅ SPF passes (vendor authorized)
- ❌ SPF alignment fails
- ❌ DMARC does not accept SPF

---
### DKIM alignment example
```
DKIM-Signature: d=company.com
From: user@company.com
```

- ✅ DKIM validates
- ✅ DKIM aligns
- ✅ DMARC passes (even if SPF fails)

> This is why **DKIM alignment is the preferred path** for third‑party senders.

---
## DNS Records

### SPF

**Typical (non‑Proofpoint)**
```
example.com TXT
v=spf1 include:_spf.google.com include:spf.protection.outlook.com -all
```

**Proofpoint**
```
example.com TXT
v=spf1 include:%{ir}.%{v}.%{d}.spf.has.pphosted.com ~all
```

>Proofpoint handles lookup optimization internally
>Terraform requires escaping `%` → `%%`

---
### DMARC

**Typical**
```
_dmarc.example.com TXT
v=DMARC1; p=none; rua=mailto:dmarc@example.com; fo=1
```

**Proofpoint**
```
_dmarc.example.com CNAME
_dmarc.example.com.dmarc.has.pphosted.com
```

> p=none | p=quarantine | p=reject

---
### DKIM

**Typical**
```
selector1._domainkey.example.com TXT
v=DKIM1; k=rsa; p=MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8A...
```
> the d=example.com is not in DNS, the email server adds that to the email headers

**Proofpoint (delegated mode)**

- DKIM is different that SPF/DMARC. You create a NS record and point that to Proofpoint. You don't modify the selectors. 
- Create selectors in PP > create NS records in DNS > wait 1 week > remove selectors from DNS

```
_domainkey.example.com NS ns1-dkim.has.pphosted.com  
_domainkey.example.com NS ns2-dkim.has.pphosted.com  
```

**Super Important**
 - Delegating `_domainkey.example.com` to Proofpoint disables ALL existing DKIM selectors. 
 - This would break any AWS SES that is being used as a sender with SES for example. Below shows a CNAME for a selector that points to SES
```
3dxshqup7s4r35x5ms7nznycfxe6dih7._domainkey.example.com CNAME
3dxshqup7s4r35x5ms7nznycfxe6dih7.dkim.amazonses.com
```

**Proofpoint (nondelegated mode)**
```
3dxshqup7s4r35x5ms7nznycfxe6dih7._domainkey.example.com CNAME
3dxshqup7s4r35x5ms7nznycfxe6dih7._domainkey.proofpoint.com
```

---
## What Google (and others) require

```
SPF  → who can send (Return‑Path)
DKIM → message signature (d=)
DMARC → policy + alignment
```

**DMARC passes if:**

```
(SPF OR DKIM) passes
AND
that result aligns with From domain
```

---
## “1.5 Pass” Explained (Informal Term)

> _Note: “1.5” is industry shorthand, not an RFC term._

- **1.0** → One aligned pass → DMARC passes
- **+0.5** → Other method passes without alignment

### Recommended pattern

- ✅ DKIM: pass + aligned
- ✅ SPF: pass (unaligned)

---

## Bottom Line

- DMARC requires **at least one aligned authentication**
- **Aligned DKIM is the most reliable strategy**
- Proofpoint simplifies SPF, DKIM, and DMARC management
- DNS changes are minimized after initial setup