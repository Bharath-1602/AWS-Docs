## 🔐 ACM — SSL/TLS Certificates

### 💡 What is ACM?

**AWS Certificate Manager (ACM)** is a managed service that handles the entire lifecycle of SSL/TLS certificates for your AWS resources — requesting, validating, deploying, and automatically renewing them before they expire.

Without ACM, you would need to manually purchase certificates from a Certificate Authority, install them on each server, track expiry dates, and renew them every year. ACM eliminates all of that overhead entirely.

```
Traditional Certificate Process:        With ACM:
──────────────────────────────          ──────────
Buy cert from CA (~$100/year)     →     Free with AWS services
Install manually on each server   →     Attach to ALB in 2 clicks
Track expiry yourself             →     Auto-renews before expiry
Renew every year                  →     Never expires if attached
Update config on renewal          →     Zero-downtime renewal
```

The most important thing to understand: **ACM certificates are completely free** when used with integrated AWS services like Application Load Balancers, CloudFront distributions, and API Gateway. You only pay for the underlying AWS service, never for the certificate itself.

---

### 🔑 Key Concepts

| Term | Plain English Explanation |
|---|---|
| **SSL/TLS** | The encryption protocol that makes HTTPS work — scrambles data between browser and server so nobody can intercept it |
| **TLS Termination** | The ALB decrypts incoming HTTPS traffic and forwards plain HTTP to your backend instances — instances don't need to handle encryption |
| **End-to-End Encryption** | HTTPS all the way from browser → ALB → backend instances — traffic is never decrypted in transit |
| **DNS Validation** | Proving domain ownership by adding a specific CNAME record to your DNS — recommended method |
| **Email Validation** | Proving domain ownership by clicking a link sent to admin emails like admin@yourdomain.com |
| **Wildcard Certificate** | A single certificate that covers a domain and ALL its subdomains (*.example.com covers www, api, mail, shop, etc.) |
| **Certificate ARN** | The unique identifier AWS assigns to your certificate — used when attaching it to ALB or CloudFront |

---

### 🏗️ Why Use ACM?

```
4 Core Reasons:

1. 🔒 HTTPS is Required Today
   Browsers show "Not Secure" on HTTP sites
   Google penalizes non-HTTPS sites in search rankings
   Users don't trust sites without the padlock icon

2. 💰 Completely Free
   No annual certificate fees
   No renewal costs
   Only pay for the AWS service it's attached to

3. ♻️ Auto-Renewal
   ACM renews certificates automatically before expiry
   No reminders, no manual work, no downtime from expired certs

4. 🔗 Native AWS Integration
   Works directly with ALB, CloudFront, API Gateway
   Route 53 + ACM = one-click DNS validation
   No manual installation on servers
```

---

### 🚀 Hands-On: Request, Validate, and Attach a Certificate

#### What We're Building

```
Browser (HTTPS)
      │
      │  Port 443 — encrypted with ACM certificate
      ▼
Application Load Balancer
      │  Certificate: *.example.com (from ACM)
      │  TLS terminates here
      │
      │  Port 80 — plain HTTP internally
      ▼
EC2 Instances / Target Group
      (backend never deals with encryption)

Also:
HTTP :80 → ALB → 301 Redirect → HTTPS :443
(users typing http:// automatically get upgraded)
```

---

#### 📋 Step 1 — Request a Certificate

**Navigate to ACM:**
- AWS Console → Search **"Certificate Manager"** → Open ACM
- Make sure you are in the **correct region** — certificates are region-specific

> ⚠️ If you are attaching the certificate to CloudFront, you must request it in **us-east-1 (N. Virginia)** regardless of where your other resources are. For ALB, request it in the same region as your ALB.

- Click **"Request a certificate"**
- Select **"Request a public certificate"** → Click **"Next"**

**Add Your Domain Names:**

| Domain | Type | Covers |
|---|---|---|
| `example.com` | Exact match | Root domain only |
| `*.example.com` | Wildcard | All subdomains (www, api, shop, mail, etc.) |

> Best practice: always request both `example.com` AND `*.example.com` together in a single certificate. This covers your root domain and every subdomain with one cert.

**Validation Method:**

| Method | How It Works | When to Use |
|---|---|---|
| **DNS Validation** ✅ | Add a CNAME record to your DNS | Recommended — works with Route 53 automatically |
| **Email Validation** | Click link in email to admin@yourdomain.com | Use only if you don't control the DNS |

- Select **"DNS validation"** → Click **"Request"**

---

#### 📋 Step 2 — Validate Domain Ownership

After requesting, ACM gives you a CNAME record that proves you own the domain.

**What the validation record looks like:**

```
ACM gives you this CNAME to add to DNS:

Name:   _a79865eb4cd1a6ab56f5b50a..example.com
Value:  _1234567890abcdef.acm-validations.aws.

This record proves to AWS that you control example.com
```

**Option A — Domain is in Route 53 (Easiest):**
- On the certificate details page → expand **Domains** section
- Click **"Create records in Route 53"** button
- ACM automatically adds the CNAME records to your hosted zone
- Click **"Create records"** → Done ✅

**Option B — Domain is with GoDaddy or Namecheap:**

Go to your registrar's DNS management panel and manually add:

| Field | Value |
|---|---|
| Record Type | CNAME |
| Name | `_a79865eb4cd1a6ab56f5b50a` (the part before your domain) |
| Value | `_1234567890abcdef.acm-validations.aws.` |
| TTL | 300 |

```
⏳ Validation Timeline:

Record added → DNS propagates globally → ACM detects it → Certificate issued

Typical wait time: 5 to 30 minutes
Maximum wait time: 72 hours (DNS propagation across globe)

Status changes:
Pending validation → (wait) → Issued ✅
```

- Monitor status in **ACM → Certificates** — refresh the page every few minutes
- Once status shows **"Issued"** — your certificate is ready to use

---

#### 📋 Step 3 — Attach Certificate to ALB

**Add HTTPS Listener:**
- EC2 → Load Balancers → Select your ALB
- Click the **"Listeners and rules"** tab
- Click **"Add listener"**

| Field | Value |
|---|---|
| Protocol | HTTPS |
| Port | 443 |
| Default action | Forward to → select your target group |
| Default SSL/TLS certificate | Select your ACM certificate from dropdown |

- Click **"Add"** ✅

**Redirect HTTP to HTTPS (Best Practice):**

Users might type `http://example.com` directly. You want them automatically upgraded to HTTPS.

- In the Listeners tab → find the **HTTP : 80** listener
- Click on it → **Edit listener**
- Change the default action to: **Redirect to URL**

| Field | Value |
|---|---|
| Protocol | HTTPS |
| Port | 443 |
| Status code | 301 — Moved Permanently |

- Click **"Save changes"** ✅

```
Result of 301 redirect:

User types:     http://example.com
Browser sends:  GET http://example.com
ALB responds:   301 → https://example.com
Browser follows: GET https://example.com ✅
Browser saves:  Remembers to always use HTTPS next time
```

---

### 🔄 TLS Termination vs End-to-End Encryption

These are two different architectural approaches to where HTTPS encryption lives in your stack.

#### Option A — TLS Termination at ALB (Most Common)

```
Browser ──[HTTPS encrypted]──► ALB ──[HTTP plain]──► EC2 Instances
                                 ↑
                         Decryption happens here
                         ACM certificate lives here

Pros:
  ✅ Simpler backend — EC2 instances don't handle SSL
  ✅ Lower CPU on backend instances (no encryption overhead)
  ✅ ACM handles everything — no certs to install on servers
  ✅ ALB can inspect HTTP headers for routing decisions

Cons:
  ⚠️ Traffic between ALB and EC2 is unencrypted
  ⚠️ (This is inside AWS's private network — generally acceptable)
```

#### Option B — End-to-End Encryption

```
Browser ──[HTTPS encrypted]──► ALB ──[HTTPS re-encrypted]──► EC2 Instances
                                 ↑                              ↑
                         Decrypts here              Re-encrypts here
                         then re-encrypts           (self-signed or private cert)

Pros:
  ✅ Data encrypted at every hop — even inside AWS network
  ✅ Required for strict compliance (PCI-DSS, HIPAA)

Cons:
  ⚠️ Must install certificates on every EC2 instance
  ⚠️ Higher CPU usage on backend servers
  ⚠️ More complex setup and maintenance
  ⚠️ Target group protocol must be set to HTTPS
```

**Which should you use?**

```
Use TLS Termination (Option A) when:
  → Standard web applications
  → Internal backend traffic stays within AWS VPC
  → You want simplicity and lower operational overhead

Use End-to-End Encryption (Option B) when:
  → Strict compliance requirements (healthcare, payments)
  → Regulatory mandates require encrypted data at every layer
  → Your security policy requires no plaintext anywhere
```

---

### 📋 ACM Quick Reference

```
🌍 Region Rules:
   ALB certificate       → Same region as ALB
   CloudFront certificate → MUST be us-east-1 (N. Virginia)

✅ Always Request Both:
   example.com           → Covers root domain
   *.example.com         → Covers all subdomains

🔄 Validation Methods:
   Route 53 + ACM        → One-click automatic (recommended)
   Other registrar       → Manual CNAME record addition

⏳ Wait Times:
   DNS validation        → 5 to 30 minutes typically
   Certificate status    → Pending validation → Issued

💰 Cost:
   Public certificate    → Free with ALB, CloudFront, API Gateway
   Private certificate   → $400/month per CA

🔒 HTTP to HTTPS Redirect:
   HTTP :80 listener → Action: Redirect → HTTPS :443 → 301
   Always do this — never leave HTTP open without redirect
```

---
