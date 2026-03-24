# Meridian PMS Threat Model

**Version:** 1.0  
**Date:** December 19, 2025  
**Classification:** Internal – Security Sensitive  
**Author:** Security Architecture Team  
**Review Status:** Draft

---

## Executive Summary

Meridian PMS is a cloud-hosted healthcare patient management system built on AWS infrastructure using a microservices architecture. The platform enables healthcare organizations to manage patient records, appointments, medications, and telehealth sessions through a publicly accessible web portal. Given its handling of Protected Health Information (PHI) and Personally Identifiable Information (PII), the system falls under HIPAA regulatory requirements and represents a high-value target for threat actors.

This threat model assesses the security posture of Meridian PMS's core components: Amazon Cognito authentication, JWT session management, MariaDB patient data storage, S3 file storage, real-time WebRTC/Socket.IO communications, and third-party API integrations. The analysis identifies 15 prioritized threats across authentication bypass, data exposure, API abuse, and infrastructure compromise categories.

**Key Findings:** The current architecture has significant gaps in JWT validation hardening, S3 access controls, WebRTC TURN server authentication, and audit logging completeness. Eight high-risk threats require immediate remediation to achieve acceptable HIPAA compliance posture. The third-party integration with ScriptGuard introduces supply chain risk that is not adequately addressed by current controls.

---

## Architecture Diagram

```mermaid
flowchart TB
    subgraph Internet["Internet (Untrusted)"]
        User[Healthcare Staff]
        Patient[Patient Portal Users]
        ExtAPI[ScriptGuard API]
    end

    subgraph TrustBoundary1["DMZ - Public Facing"]
        WAF[AWS WAF]
        ALB[Application Load Balancer]
    end

    subgraph TrustBoundary2["Application Tier"]
        React[React Frontend<br/>Apache/EC2]
        NodeAPI[Node.js API<br/>Express.js]
        Lambda[AWS Lambda<br/>Serverless Functions]
        WebRTC[WebRTC Gateway<br/>Telehealth Video]
        SocketIO[Socket.IO Server<br/>Real-time Messaging]
    end

    subgraph TrustBoundary3["Authentication"]
        Cognito[Amazon Cognito<br/>User Pools + 2FA]
        JWT[JWT Token Service]
    end

    subgraph TrustBoundary4["Data Tier"]
        MariaDB[(MariaDB<br/>Patient Records)]
        S3[(AWS S3<br/>File Storage)]
    end

    subgraph TrustBoundary5["Monitoring"]
        Prometheus[Prometheus]
        ELK[ELK Stack]
    end

    User --> WAF
    Patient --> WAF
    WAF --> ALB
    ALB --> React
    React --> NodeAPI
    NodeAPI --> Cognito
    Cognito --> JWT
    NodeAPI --> MariaDB
    NodeAPI --> S3
    NodeAPI --> Lambda
    NodeAPI --> WebRTC
    NodeAPI --> SocketIO
    NodeAPI --> ExtAPI
    NodeAPI --> Prometheus
    NodeAPI --> ELK
    Lambda --> MariaDB
    Lambda --> S3
```

---

## Assets Inventory

| Asset | Description | Data Classification | Owner | Storage Location |
|-------|-------------|---------------------|-------|------------------|
| Patient Records | Name, DOB, SSN, address, medical history | PHI/PII – Critical | Data Team | MariaDB |
| Medical Documents | Lab results, imaging, prescriptions | PHI – Critical | Data Team | S3 |
| User Credentials | Passwords, MFA secrets, session tokens | Sensitive | Security Team | Cognito |
| Appointment Data | Scheduling, provider assignments | PHI – High | Application Team | MariaDB |
| Medication Records | Prescriptions, dosages, refill history | PHI – Critical | Data Team | MariaDB |
| Telehealth Sessions | Video streams, chat transcripts | PHI – Critical | Application Team | WebRTC/S3 |
| Audit Logs | Access logs, authentication events | Sensitive | Security Team | ELK Stack |
| API Keys | Third-party integration credentials | Sensitive | DevOps Team | AWS Secrets Manager |
| JWT Signing Keys | Token signing/verification keys | Critical | Security Team | Cognito/EC2 |

---

## Risk Scoring Methodology

Risk ratings follow the OWASP Risk Rating Methodology: **Risk = Likelihood x Impact**.

**Likelihood** considers exploitability (skill required, tooling availability, public knowledge of the attack vector) and exposure (whether the component is internet-facing, requires authentication, or sits behind existing controls).

**Impact** considers the scope of data at risk (PHI record count, data sensitivity classification), HIPAA regulatory exposure (specific §164 subsections violated), and service availability consequences.

| Rating | Criteria |
|--------|----------|
| **High** | Exploitable by an unauthenticated or low-skill attacker against an internet-facing component, OR directly exposes PHI/PII, OR bypasses a primary security control (authentication, authorization, encryption) |
| **Medium** | Requires authenticated access or specific preconditions to exploit, AND exposes system metadata or operational data rather than PHI directly, OR degrades a defence-in-depth layer without fully bypassing it |

---

## Threat Register

| ID | Category | Threat | Component | Likelihood | Impact | Risk | Status |
|----|----------|--------|-----------|------------|--------|------|--------|
| AUTH-01 | Spoofing | JWT algorithm confusion—attacker substitutes RS256 with HS256 using public key as secret | JWT Authentication | High — public key is retrievable from Cognito JWKS endpoint; attack is well-documented (CVE-2015-9235) | High — complete authentication bypass, full access to any user session including PHI | **High** | Not Addressed |
| AUTH-02 | Spoofing | Cognito user enumeration via differential response timing on login attempts | Amazon Cognito | Medium — requires scripted timing analysis against a public endpoint; Cognito has partial built-in protections | Medium — reveals valid usernames but does not directly expose PHI; enables targeted credential attacks | **Medium** | Not Addressed |
| AUTH-03 | Elevation of Privilege | JWT token theft via XSS allows session hijacking with full user privileges | Frontend/JWT | High — tokens stored in localStorage are accessible to any injected script; XSS vectors are common in React apps without strict CSP | High — stolen token grants full session access including PHI read/write | **High** | Partial |
| AUTH-04 | Repudiation | Insufficient logging of authentication failures prevents breach detection | ELK Stack | Medium — logging gap is passive; attacker benefits only if already exploiting another vulnerability | Medium — delays breach detection but does not directly expose data; violates §164.312(b) audit requirements | **Medium** | Not Addressed |
| DATA-01 | Information Disclosure | S3 bucket misconfiguration exposes patient documents publicly | S3 Storage | High — S3 bucket misconfigurations are actively scanned by automated tools; no authentication required if public | High — direct exposure of PHI documents (lab results, prescriptions, imaging) to the internet | **High** | Not Addressed |
| DATA-02 | Information Disclosure | MariaDB connection strings hardcoded in application code or environment variables | Node.js API | High — credentials in source code or env vars are extractable via code repository access, SSRF, or log leakage | High — database credentials grant direct access to all patient records | **High** | Not Addressed |
| DATA-03 | Tampering | SQL injection in patient search endpoint modifies or extracts PHI | MariaDB/API | High — patient search is a public-facing endpoint; SQLi is well-understood and tooling (sqlmap) is freely available | High — enables bulk extraction or modification of patient records; violates §164.312(c)(1) integrity controls | **High** | Not Addressed |
| API-01 | Elevation of Privilege | IDOR vulnerability allows patients to access other patients' records via predictable IDs | Patient API | High — sequential or predictable IDs require no special tooling; any authenticated user can enumerate | High — cross-patient PHI access; violates §164.312(a)(1) access controls | **High** | Not Addressed |
| API-02 | Denial of Service | Lack of rate limiting enables API abuse and resource exhaustion | Node.js API | Medium — requires sustained automated requests; WAF provides partial protection at the network layer | Medium — service degradation affects availability but does not expose PHI directly; impacts §164.312(a)(1) availability requirements | **Medium** | Not Addressed |
| API-03 | Spoofing | ScriptGuard API key compromise enables unauthorized medication data access | Third-party Integration | High — static API key without rotation; single key compromise exposes the integration permanently | High — unauthorized access to medication data (PHI); potential for data manipulation affecting patient safety | **High** | Not Addressed |
| RTC-01 | Information Disclosure | WebRTC TURN server lacks authentication, allowing unauthorized relay usage | WebRTC Gateway | High — unauthenticated TURN endpoints are discoverable via port scanning; abuse requires no credentials | High — exposes telehealth video streams (PHI); enables bandwidth abuse for traffic relay | **High** | Not Addressed |
| RTC-02 | Information Disclosure | Socket.IO connections lack origin validation, enabling cross-site hijacking | Socket.IO Server | Medium — requires attacker to deliver a malicious page to a logged-in user (social engineering dependency) | Medium — exposes real-time messaging content; limited to the session of the targeted user rather than bulk data | **Medium** | Not Addressed |
| INFRA-01 | Elevation of Privilege | Lambda function IAM roles overly permissive, enabling lateral movement | AWS Lambda | Medium — requires initial compromise of a Lambda function or its invocation path; not directly internet-facing | Medium — enables access to adjacent AWS resources (S3, MariaDB) but scope depends on the over-provisioned permissions | **Medium** | Not Addressed |
| INFRA-02 | Information Disclosure | Prometheus metrics endpoint exposed without authentication leaks system info | Prometheus | Medium — endpoint must be network-reachable; default Prometheus port (9090) is commonly scanned | Medium — exposes system performance data, internal hostnames, and error rates; useful for reconnaissance but not direct PHI access | **Medium** | Not Addressed |
| INFRA-03 | Tampering | GitHub Actions workflow injection via malicious PR modifies deployment | CI/CD Pipeline | High — public repositories accept PRs from any GitHub user; workflow_run and pull_request_target triggers are well-known vectors | High — code execution in deployment pipeline enables supply chain compromise; attacker can inject malicious code into production | **High** | Not Addressed |

---

## Mitigations

### AUTH-01: Pin JWT Algorithm in Verification

**Threat:** JWT algorithm confusion attack allows signature bypass

**Recommendation:** Explicitly restrict accepted algorithms during token verification. Never allow the algorithm to be derived from the token header alone.

```javascript
const jwt = require('jsonwebtoken');

const verifyToken = (token) => {
  return jwt.verify(token, publicKey, {
    algorithms: ['RS256'], // Explicitly pin algorithm
    issuer: 'https://cognito-idp.us-east-1.amazonaws.com/YOUR_POOL_ID',
    audience: 'YOUR_CLIENT_ID'
  });
};
```

**Additional Controls:**
- Rotate signing keys quarterly via Cognito key rotation
- Monitor for tokens with unexpected algorithm headers in ELK

---

### AUTH-03: Implement Secure Token Storage

**Threat:** XSS-based JWT theft enables session hijacking

**Recommendation:** Store tokens in HttpOnly cookies instead of localStorage. Implement Content Security Policy headers.

```javascript
// Express.js cookie configuration
res.cookie('accessToken', token, {
  httpOnly: true,
  secure: true,
  sameSite: 'strict',
  maxAge: 3600000 // 1 hour
});
```

**CSP Header Configuration:**

```
Content-Security-Policy: default-src 'self'; script-src 'self'; connect-src 'self' wss://your-domain.com
```

---

### DATA-01: Enforce S3 Private Access

**Threat:** Public bucket exposure leaks patient documents

**Recommendation:** Enable S3 Block Public Access at account level and enforce secure transport via bucket policy.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyInsecureTransport",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:*",
      "Resource": [
        "arn:aws:s3:::meridian-pms-patient-docs",
        "arn:aws:s3:::meridian-pms-patient-docs/*"
      ],
      "Condition": {
        "Bool": {
          "aws:SecureTransport": "false"
        }
      }
    },
    {
      "Sid": "DenyPublicAccess",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::meridian-pms-patient-docs/*",
      "Condition": {
        "StringNotEquals": {
          "aws:PrincipalAccount": "YOUR_ACCOUNT_ID"
        }
      }
    }
  ]
}
```

**AWS CLI Verification:**

```bash
aws s3api get-public-access-block --bucket meridian-pms-patient-docs
```

---

### DATA-02: Externalize Secrets Management

**Threat:** Hardcoded credentials in code or environment variables

**Recommendation:** Use AWS Secrets Manager with automatic rotation for database credentials.

```javascript
const { SecretsManagerClient, GetSecretValueCommand } = require('@aws-sdk/client-secrets-manager');

const getDbCredentials = async () => {
  const client = new SecretsManagerClient({ region: 'us-east-1' });
  const response = await client.send(
    new GetSecretValueCommand({ SecretId: 'meridian-pms/mariadb/prod' })
  );
  return JSON.parse(response.SecretString);
};
```

---

### DATA-03: Implement Parameterized Queries

**Threat:** SQL injection in patient search endpoints

**Recommendation:** Use parameterized queries exclusively. Enable MariaDB query logging for injection attempt detection.

```javascript
// Using mysql2 with prepared statements
const searchPatients = async (searchTerm) => {
  const [rows] = await pool.execute(
    'SELECT id, name, dob FROM patients WHERE name LIKE ? OR mrn = ?',
    [`%${searchTerm}%`, searchTerm]
  );
  return rows;
};
```

**Input Validation Layer:**

```javascript
const Joi = require('joi');

const patientSearchSchema = Joi.object({
  query: Joi.string().max(100).pattern(/^[a-zA-Z0-9\s\-]+$/).required()
});
```

---

### API-01: Implement Authorization Checks for IDOR Prevention

**Threat:** Insecure Direct Object Reference allows cross-patient data access

**Recommendation:** Validate resource ownership on every request. Implement middleware that verifies the authenticated user has access to the requested patient record.

```javascript
const authorizePatientAccess = async (req, res, next) => {
  const requestedPatientId = req.params.patientId;
  const userId = req.user.sub; // From JWT
  
  const hasAccess = await checkPatientAccess(userId, requestedPatientId);
  
  if (!hasAccess) {
    logger.warn('IDOR attempt', { userId, requestedPatientId, ip: req.ip });
    return res.status(403).json({ error: 'Access denied' });
  }
  next();
};

app.get('/api/patients/:patientId', authorizePatientAccess, getPatientHandler);
```

---

### API-03: Secure Third-Party API Integration

**Threat:** ScriptGuard API key compromise

**Recommendation:** Store API keys in AWS Secrets Manager with rotation. Implement request signing and IP allowlisting.

```javascript
// Rotate keys and use short-lived tokens where supported
const getScriptGuardClient = async () => {
  const apiKey = await getSecretValue('meridian-pms/scriptguard/apikey');
  return axios.create({
    baseURL: 'https://api.scriptguard.io/v2',
    headers: {
      'X-API-Key': apiKey,
      'X-Request-ID': crypto.randomUUID()
    },
    timeout: 5000
  });
};
```

**Network Controls:**
- Configure VPC endpoints for outbound API calls
- Implement egress filtering to allow only ScriptGuard IP ranges
- Log all third-party API requests to ELK

---

### RTC-01: Secure WebRTC TURN Server

**Threat:** Unauthenticated TURN server abuse

**Recommendation:** Implement time-limited TURN credentials generated per session.

```javascript
const generateTurnCredentials = (userId) => {
  const timestamp = Math.floor(Date.now() / 1000) + 3600; // 1 hour validity
  const username = `${timestamp}:${userId}`;
  const credential = crypto
    .createHmac('sha1', TURN_SECRET)
    .update(username)
    .digest('base64');
  
  return {
    urls: ['turn:turn.meridian-pms.com:443?transport=tcp'],
    username,
    credential
  };
};
```

---

### INFRA-03: Secure GitHub Actions Workflows

**Threat:** Workflow injection via malicious pull requests

**Recommendation:** Require approval for workflows from first-time contributors. Pin action versions to SHA.

```yaml
# .github/workflows/deploy.yml
name: Deploy
on:
  push:
    branches: [main]

permissions:
  contents: read
  id-token: write

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: production  # Requires approval
    steps:
      - uses: actions/checkout@b4ffde65f46336ab88eb53be808477a3936bae11 # v4.1.1
      - uses: aws-actions/configure-aws-credentials@e3dd6a429d7300a6a4c196c26e071d42e0343502 # v4.0.2
        with:
          role-to-assume: arn:aws:iam::ACCOUNT:role/github-deploy
          aws-region: us-east-1
```

---

### AUTH-02: Mitigate Cognito User Enumeration

**Threat:** Differential response timing reveals valid usernames

**Recommendation:** Enable Cognito's `PREVENT_USER_EXISTENCE_ERRORS` setting so that authentication failures return identical responses regardless of whether the user exists.

```bash
aws cognito-idp update-user-pool-client \
  --user-pool-id us-east-1_XXXXX \
  --client-id YOUR_CLIENT_ID \
  --prevent-user-existence-errors ENABLED
```

**Additional Controls:**
- Implement constant-time comparison in any custom authentication Lambda triggers
- Rate limit login attempts per source IP via WAF rules to slow enumeration

---

### AUTH-04: Implement Comprehensive Authentication Logging

**Threat:** Insufficient auth failure logging prevents breach detection

**Recommendation:** Log all authentication events — successes, failures, and token operations — with structured fields for correlation. Ship to ELK with alerting on anomalous patterns.

```javascript
const logAuthEvent = (event, req) => {
  logger.info({
    type: 'AUTH_EVENT',
    action: event.action, // 'login_success', 'login_failure', 'token_refresh', 'logout'
    userId: event.userId || 'unknown',
    sourceIp: req.ip,
    userAgent: req.headers['user-agent'],
    timestamp: new Date().toISOString(),
    failureReason: event.reason || null
  });
};
```

**Alerting Rules:**
- Trigger alert on >5 failed login attempts from a single IP within 10 minutes
- Trigger alert on successful login from a new geographic region
- Retain auth logs for minimum 6 years per HIPAA §164.312(b)

---

### API-02: Implement API Rate Limiting

**Threat:** Unrestricted API access enables resource exhaustion

**Recommendation:** Apply rate limits at both the API Gateway and application layers. Use sliding window counters per authenticated user and per source IP for unauthenticated endpoints.

```javascript
const rateLimit = require('express-rate-limit');

const apiLimiter = rateLimit({
  windowMs: 60 * 1000, // 1 minute
  max: 100, // 100 requests per minute per IP
  standardHeaders: true,
  legacyHeaders: false,
  keyGenerator: (req) => req.user?.sub || req.ip, // Per-user if authenticated, per-IP otherwise
  handler: (req, res) => {
    logger.warn('Rate limit exceeded', { userId: req.user?.sub, ip: req.ip });
    res.status(429).json({ error: 'Too many requests' });
  }
});

app.use('/api/', apiLimiter);
```

**Additional Controls:**
- Configure AWS WAF rate-based rules as a first line of defence (2,000 requests per 5 minutes per IP)
- Apply stricter limits to sensitive endpoints: `/api/patients/search` at 20 requests per minute

---

### RTC-02: Validate Socket.IO Connection Origins

**Threat:** Cross-site Socket.IO hijacking via missing origin validation

**Recommendation:** Restrict Socket.IO connections to trusted origins and require JWT authentication on the handshake.

```javascript
const io = require('socket.io')(server, {
  cors: {
    origin: ['https://meridian-pms.com'],
    methods: ['GET', 'POST'],
    credentials: true
  }
});

io.use((socket, next) => {
  const token = socket.handshake.auth.token;
  try {
    const decoded = jwt.verify(token, publicKey, { algorithms: ['RS256'] });
    socket.user = decoded;
    next();
  } catch (err) {
    logger.warn('Socket.IO auth failed', { ip: socket.handshake.address, error: err.message });
    next(new Error('Authentication required'));
  }
});
```

---

### INFRA-01: Apply Least-Privilege Lambda IAM Roles

**Threat:** Over-permissioned Lambda roles enable lateral movement

**Recommendation:** Create per-function IAM roles scoped to the minimum required resources. Avoid wildcard permissions on actions or resources.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject"
      ],
      "Resource": "arn:aws:s3:::meridian-pms-patient-docs/lab-results/*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "secretsmanager:GetSecretValue"
      ],
      "Resource": "arn:aws:secretsmanager:us-east-1:ACCOUNT:secret:meridian-pms/mariadb/prod-*"
    }
  ]
}
```

**Additional Controls:**
- Use IAM Access Analyzer to identify unused permissions and tighten roles
- Enable CloudTrail logging for Lambda invocations to detect anomalous execution patterns
- Review IAM policies quarterly as part of access review cycle

---

### INFRA-02: Secure Prometheus Metrics Endpoint

**Threat:** Unauthenticated Prometheus endpoint leaks system metadata

**Recommendation:** Place Prometheus behind an authentication proxy and restrict network access to the monitoring VPC subnet.

```yaml
# nginx reverse proxy for Prometheus
server {
    listen 9090 ssl;
    server_name prometheus.internal.meridian-pms.com;

    ssl_certificate /etc/ssl/certs/prometheus.crt;
    ssl_certificate_key /etc/ssl/private/prometheus.key;

    location / {
        auth_basic "Prometheus";
        auth_basic_user_file /etc/nginx/.htpasswd;
        proxy_pass http://localhost:9091;
    }
}
```

**Network Controls:**
- Bind Prometheus to localhost (127.0.0.1:9091) and proxy authenticated requests via nginx
- Restrict security group ingress on port 9090 to the monitoring subnet CIDR only
- Scrub sensitive labels (database hostnames, internal IPs) from exported metrics using `metric_relabel_configs`

---

## HIPAA Security Rule Mapping

| HIPAA Requirement | §164 Reference | Current State | Gap |
|-------------------|----------------|---------------|-----|
| Access Control | §164.312(a)(1) | Cognito + JWT implemented | IDOR vulnerability (API-01) undermines access control |
| Audit Controls | §164.312(b) | ELK Stack deployed | Insufficient auth failure logging (AUTH-04) |
| Integrity Controls | §164.312(c)(1) | TLS in transit stated | SQL injection risk (DATA-03) threatens integrity |
| Transmission Security | §164.312(e)(1) | TLS/SSL configured | WebRTC TURN lacks auth (RTC-01) |
| Person Authentication | §164.312(d) | 2FA via Cognito | JWT algorithm confusion (AUTH-01) bypasses auth |
| Encryption | §164.312(a)(2)(iv) | At-rest encryption stated | S3 bucket policy gaps (DATA-01) |
| Contingency Plan | §164.308(a)(7) | Not documented | Backup/recovery procedures needed |
| Security Incident | §164.308(a)(6) | Prometheus monitoring | No incident response playbook documented |

### Priority HIPAA Remediation Actions

1. **Immediate:** Address AUTH-01, DATA-01, API-01 to restore access control integrity
2. **30 Days:** Implement comprehensive audit logging covering all PHI access
3. **60 Days:** Document incident response procedures and conduct tabletop exercise
4. **90 Days:** Complete penetration test and remediate findings

---

## Risk Summary

| Risk Level | Count | Immediate Action Required |
|------------|-------|---------------------------|
| High | 8 | Yes – Remediate before next release |
| Medium | 7 | Plan remediation within 30 days |
| Low | 0 | — |

**Overall Risk Posture:** HIGH – Multiple unaddressed vulnerabilities in authentication and data protection create significant breach risk and HIPAA compliance gaps.

---

## References

- OWASP Top 10 2021: https://owasp.org/Top10/
- OWASP API Security Top 10: https://owasp.org/www-project-api-security/
- NIST SP 800-53 Rev. 5 Security Controls: https://csrc.nist.gov/publications/detail/sp/800-53/rev-5/final
- NIST Cybersecurity Framework: https://www.nist.gov/cyberframework
- HHS HIPAA Security Rule Guidance: https://www.hhs.gov/hipaa/for-professionals/security/guidance/
- AWS Well-Architected Security Pillar: https://docs.aws.amazon.com/wellarchitected/latest/security-pillar/
- CWE-287 Improper Authentication: https://cwe.mitre.org/data/definitions/287.html
- CWE-639 Authorization Bypass Through User-Controlled Key: https://cwe.mitre.org/data/definitions/639.html

---

## Document History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | December 19, 2025 | Security Architecture Team | Initial threat model |

---

*This document contains security-sensitive information. Distribution is limited to authorized personnel with a need to know.*
