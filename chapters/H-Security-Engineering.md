# H. Security Engineering (Principal Engineer Depth)

This chapter includes:

- Web + backend security fundamentals
- API authentication & authorization
- OAuth2, OIDC, JWT internals
- CSRF, XSS, SSRF, SQL injection
- Secure session management
- Backend data security (encryption, hashing)
- Cloud security (IAM, KMS, VPC)
- Secrets management
- API hardening patterns
- Pentesting mindset
- Threat modeling

---

# 1. SECURITY MINDSET (REALITY CHECK)

Security ≠ feature.  
Security = **design principle**.

Principles to internalize:

- Zero Trust  
- Defense in Depth  
- Least Privilege  
- Fail Secure  
- Secure Defaults  
- Never trust user input (Ever.)  
- Logs cannot contain secrets  

---

# 2. AUTHENTICATION (IDENTITY)

Main mechanisms:

- Password-based auth  
- OAuth2  
- OpenID Connect  
- API keys  
- Mutual TLS  
- Identity providers (Okta, Auth0, AWS Cognito)

---

# 2.1 Password Storage (Best Practice)

**Never store passwords as plain text.**

Use:

```
bcrypt (preferred)
argon2 (new standard)
scrypt
```

Never use:
- MD5  
- SHA1  
- unsalted hashes  

---

# 3. AUTHORIZATION (ACCESS CONTROL)

Types:

### 1. Role-based Access Control (RBAC)

```
Admin → can edit anything
User → limited
```

### 2. Attribute-based Access Control (ABAC)

```
allow if (user.country == resource.country)
```

### 3. Policy-based Access Control (PBAC)

Use engines like **OPA / Rego**.

---

# 4. JWT (DEEP EXPLANATION)

JWT = self-contained, signed token.

Structure:

```
header.payload.signature
```

Header:
```json
{
  "alg": "HS256",
  "typ": "JWT"
}
```

Payload:
```json
{
  "sub": "user123",
  "iat": 1698093,
  "exp": 1698293,
  "roles": ["user"]
}
```

Signed using:

- HMAC (shared secret)
- RSA/ECDSA (public/private keys)

---

## 4.1 JWT Pitfalls

### ❌ Pitfall 1 — Long expiration time
If JWT compromised → long damage window.

### ❌ Pitfall 2 — No server-side revocation
Stateless tokens cannot be revoked unless:

- maintain blacklist cache  
- use short-lived access tokens  
- use refresh tokens with rotation  

### ❌ Pitfall 3 — Overstuffed claims  
Never put passwords, emails, or PII in JWT.

### ❌ Pitfall 4 — "none" algorithm attack  
Some JWT libraries historically allowed:

```
alg": "none"
```

Always enforce allowed algorithms explicitly.

---

# 4.2 Refresh Tokens (Best Practice)

Flow:

```
Client → Access Token (short TTL, 5–15m)
Client → Refresh Token (long TTL, 30 days)
Server → Rotates refresh token on each use
```

Store refresh tokens:

- HttpOnly secure cookies  
- encrypted at rest  

---

# 5. SESSION MANAGEMENT

Do not implement sessions manually.  
Framework-provided session managers are safer.

Best practices:

- HttpOnly  
- secure flag  
- SameSite=strict  
- rotate session IDs on login  
- server-side sessions if possible  

---

# 6. API SECURITY (DEEP)

Key items:

- Strict authentication  
- Strict rate limiting  
- Input validation  
- No unauthenticated endpoints  
- No default credentials  

---

## 6.1 REST API Hardening Checklist

✔ Validate all input  
✔ Strict Content-Type checking  
✔ Output escaping  
✔ Query parameter sanitization  
✔ Rate limiting  
✔ IP throttling  
✔ Logging & auditing  
✔ No stack traces in responses  

---

# 7. TOP VULNERABILITIES (OWASP TOP 10 MASTER SUMMARY)

---

## 7.1 SQL Injection

Bad:
```java
"SELECT * FROM user WHERE name='" + name + "'"
```

Good:
```java
preparedStatement.setString(1, name);
```

ORMs don’t guarantee safety — HQL/JPQL must also use parameters.

---

## 7.2 XSS (Cross-site scripting)

Bad:
```html
<div>${username}</div>
```

Good:
```
HTML encode all output
Use templating that escapes by default
```

---

## 7.3 CSRF

Fix:
- SameSite cookies  
- CSRF tokens  
- double submit cookies  

---

## 7.4 SSRF (Server-side request forgery)

Attackers can force server to make requests:

```
http://169.254.169.254/latest/meta-data/
```

Mitigations:

- block internal IP ranges  
- validate URLs  
- use allow-list domain patterns  
- use egress firewalls  

---

## 7.5 RCE (Remote Code Execution)

Mitigations:
- avoid deserialization of user input  
- disable dynamic script engines  
- validate file uploads  

---

# 8. CLOUD SECURITY (AWS/GCP/AKS)

Critical cloud security principles:

### 8.1 IAM (Identity Access Management)
Least privilege.

Example:

Bad:
```
s3:*
```

Good:
```
s3:GetObject on arn:bucket/user-uploads/*
```

---

## 8.2 VPC Security

- private subnets for databases  
- no public RDS/EKS  
- use NAT gateway for egress  
- security groups restrict inbound ports  

---

## 8.3 Secrets Management

Never store secrets in:
- GitHub  
- Docker images  
- environment variables (in plain text)

Use:

- AWS Secrets Manager  
- GCP Secret Manager  
- HashiCorp Vault  

---

# 9. ENCRYPTION (DEEP)

## 9.1 Encryption In Transit
TLS 1.2+  
Force HTTPS everywhere.

## 9.2 Encryption At Rest
- AES-256  
- KMS-managed keys  

## 9.3 Hashing
For passwords:

- bcrypt  
- argon2  
- pbkdf2  

---

# 10. FILE UPLOAD SECURITY

Dangers:
- RCE  
- path traversal  
- malware injection  

Mitigations:
- never expose file system paths  
- always verify MIME type + extension  
- store uploads in isolated bucket  
- virus scan  

---

# 11. API RATE LIMITING & THROTTLING

Rate limit strategies:

- token bucket  
- leaky bucket  
- fixed window  
- sliding window  

Edge-level:

- Cloudflare  
- AWS API Gateway  
- NGINX  

---

# 12. ZERO-TRUST ARCHITECTURE

Principles:

- authenticate every request  
- encrypt every connection  
- authorizations enforced on every hop  
- no implicit trust  

---

# 13. THREAT MODELING

STRIDE model:

```
S: Spoofing
T: Tampering
R: Repudiation
I: Information disclosure
D: Denial of service
E: Elevation of privilege
```

Apply during design reviews.

---

# 14. AUDIT LOGGING

Every critical operation must be logged:

- login  
- password change  
- funds transfer  
- data modification  
- access to sensitive resources  

Audit logs must:

- be immutable  
- include user ID  
- include trace ID  
- go to secure sink  

---

# 15. SECURING MICROservices

Key principles:

- use mTLS between services  
- enforce authz even inside the cluster  
- encrypt service-to-service traffic  
- use service mesh for certificates  
- rotate secrets automatically  
- remove debug endpoints  

---

# 16. SECURE CODING PRINCIPLES

- validate input  
- escape output  
- principle of least privilege  
- no hardcoded secrets  
- no shell execution on user input  
- prefer whitelists over blacklists  

---

# END OF CHAPTER H
