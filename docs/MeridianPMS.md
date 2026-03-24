# Meridian PMS — Solution Description

Meridian PMS is a patient management system that consolidates medical records, appointment scheduling, secure messaging, telehealth, and medication management into a single platform accessible by healthcare providers and patients.

The system handles Protected Health Information (PHI) and Personally Identifiable Information (PII), placing it under HIPAA regulatory requirements.

## Capabilities

**Unified Patient Record** — Centralised repository for patient medical records with role-based access. Real-time updates across all authorised providers.

**Secure Messaging** — HIPAA-compliant messaging between healthcare providers for consultations, coordination, and care handoffs.

**Appointment Scheduling** — Integrated scheduling with automated reminders to reduce no-shows.

**Collaborative Care Plans** — Multi-specialist care plan creation and tracking for patients with complex conditions.

**Telehealth** — Amara Telehealth integration for virtual consultations and remote patient monitoring via WebRTC video conferencing.

**Medication Management** — ScriptGuard integration for prescription tracking, dosage management, and adherence alerts.

**Analytics** — Clarion Analytics integration for patient outcome reporting and clinical decision support.

**Patient Portal** — Patient-facing interface for viewing medical records, scheduling appointments, and receiving health recommendations.

---

# Meridian PMS — Technology Solution Design

## 1. Overview

Meridian PMS uses a microservices architecture deployed on AWS. The platform is publicly facing and stores PHI/PII data. Healthcare organisations access the system through a web portal.

## 2. Components

| Component | Technology | Notes |
|-----------|------------|-------|
| **Frontend** | React.js on Apache/EC2 | Maintained by the Clinical Systems Team |
| **Backend** | Node.js with Express.js | RESTful API layer |
| **Database** | MariaDB on EC2 | Stores patient records, appointments, medications, user data (name, address, medical history) |
| **Integration** | RESTful APIs | Internal service communication and third-party integrations |
| **Telehealth** | WebRTC + Socket.IO | Video conferencing and real-time messaging |

## 3. Data Flow

**Patient Information** — Records are created and updated by healthcare providers through the API. Medication data synchronises with ScriptGuard via REST API.

**Telehealth** — Appointment scheduling triggers notifications. Video sessions are established via WebRTC with Socket.IO handling real-time messaging.

## 4. Security

**Authentication** — Amazon Cognito User Pools with SSO federation to healthcare provider identity systems. JWT-based session management. Two-factor authentication enforced.

**Encryption** — TLS/SSL for data in transit. AES encryption at rest for sensitive data in MariaDB and S3.

**Compliance** — System is designed to meet HIPAA Security Rule requirements (§164.308, §164.312). See the accompanying threat model for a detailed gap analysis.

## 5. Third-Party Integrations

| Integration | Purpose | Protocol |
|-------------|---------|----------|
| ScriptGuard | Medication tracking and adherence alerts | REST API with API key authentication |
| Amara Telehealth | Virtual consultations and remote monitoring | WebRTC |
| Clarion Analytics | Patient outcome analytics and clinical reporting | REST API |

## 6. Deployment

**Cloud Platform** — AWS (us-east-1 region). EC2 instances for application and database tiers. S3 for document storage. Lambda for serverless processing.

**CI/CD** — GitHub Actions for build, test, and deployment pipelines.

## 7. Monitoring and Logging

**Monitoring** — Prometheus for infrastructure and application metrics.

**Logging** — ELK Stack (Elasticsearch, Logstash, Kibana) for centralised log aggregation and search.

**Metrics** — System performance, error rates, API response times, and authentication events.
