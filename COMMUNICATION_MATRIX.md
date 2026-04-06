# Communication Matrix

**Cross-Account Communication Rules**

---


---

## Default Policy

**DEFAULT DENY** — All communication not explicitly listed in this matrix is **BLOCKED**.

**Enforcement Mechanisms:**
1. **Security Groups:** Default-deny inbound rules
2. **CEN Route Policies:** Selective route propagation
3. **Cloud Firewall:** Deep packet inspection in Network Hub
4. **Service Control Policies:** Account-level preventive controls

---

## Visual Overview

```
                         ┌────────────────┐
                         │    INTERNET    │
                         └───────┬────────┘
                                 │
           ┌─────────────────────▼──────────────────────┐
           │         EDGE SECURITY LAYER                │
           │  Anti-DDoS Pro → WAF → CDN (static)        │
           └─────────────────────┬──────────────────────┘
                                 │
           ┌─────────────────────▼──────────────────────┐
           │      NETWORK HUB (ecom-network)            │
           │  All cross-account & egress traffic        │
           │  routed through here                       │
           └─────────────────────┬──────────────────────┘
                                 │ CEN
           ┌─────────────────────┼──────────────────────┐
           │                     │                      │
     ┌─────▼──────┐       ┌─────▼──────┐       ┌──────▼──────┐
     │  PAYMENT   │◄─────►│   ORDER    │       │  CATALOG    │
     │   (Prod)   │       │   (Prod)   │◄─────►│   (Prod)    │
     └────────────┘       └─────┬──────┘       └──────┬──────┘
                                │                      │
                                └──────────┬───────────┘
                                           │
                                    ┌──────▼──────┐
                                    │    USER     │
                                    │   (Prod)    │
                                    └─────────────┘
                                           ▲
                                           │
                                    ┌──────┴──────┐
                                    │   SHARED    │
                                    │  SERVICES   │
                                    └─────────────┘

     ┌────────────┐                              ┌─────────────┐
     │  SECURITY  │ ◄─── monitors all            │    DATA     │
     │  ACCOUNT   │      (read-only)             │  PLATFORM   │ ◄─── events only
     └────────────┘                              └─────────────┘

     ┌────────────┐
     │   AUDIT    │ ◄─── receives logs from all (write-only)
     └────────────┘

     ┌────────────┐
     │  SANDBOX   │ ──── fully isolated (no CEN)
     └────────────┘

     ┌──────────────────┐
     │  LONDON (DR)     │ ◄─── async replication from Frankfurt
     └──────────────────┘
```

---

## Detailed Communication Rules

### Core Infrastructure Communication

| Source | Destination | Ports/Protocol | Direction | Rule | Business Justification |
|--------|-------------|----------------|-----------|------|------------------------|
| **All Production → Network Hub** |
| Any production account | Network Hub | ANY | Outbound | Allowed | Internet egress via centralized NAT Gateway |
| **Why:** Centralized egress = single point for DLP, IP allowlisting, cost savings ($50K/year) |
| **All Accounts → Security Account** |
| Any account | Security Account | TCP/9090 (Prometheus), TCP/9200 (Elasticsearch) | Outbound | Allowed | Metrics and security events forwarding to SIEM |
| **Why:** Centralized threat correlation, compliance monitoring (SOC2 requirement) |
| **All Accounts → Audit Account** |
| Any account | Audit Account | TCP/514 (Syslog), TCP/6514 (Syslog TLS), TCP/443 (HTTPS) | Outbound (write-only) | Allowed | ActionTrail, VPC Flow Logs, application logs |
| **Why:** Immutable compliance logs, 7-year retention (PCI-DSS req 10.5), forensic evidence |
| **All Accounts → Shared Services** |
| Any account | Shared Services | TCP/443 (HTTPS), TCP/22 (SSH) | Bidirectional | Allowed | CI/CD (GitLab), monitoring dashboards, artifact registry |
| **Why:** Centralized tooling avoids duplication, cost savings ($77K/month) |

---

### Production Cross-Domain Communication

| Source | Destination | Ports | Direction | Rule | Business Flow | Measured Latency |
|--------|-------------|-------|-----------|------|---------------|------------------|
| **Payment ↔ Order** | TCP/8080 (HTTPS) | Bidirectional | mTLS required |
| **Flow:** User checkout → Order creates pending order → calls Payment API → Payment authorizes card → Order confirmed |
| **Why bidirectional:** Payment verification (Order→Payment), transaction reconciliation (Payment→Order) |
| **SLA:** P99 < 150ms | **Actual:** 125ms (hub adds 1.8ms) |
| **Order → Catalog** | TCP/8080 (HTTPS) | One-way | Allowed |
| **Flow:** User adds item to cart → Order validates current price from Catalog (prevent price manipulation) |
| **Why one-way:** Order consumes product data, Catalog doesn't need order data (async via Kafka events) |
| **SLA:** P99 < 100ms | **Actual:** 78ms |
| **Order → User** | TCP/8080 (HTTPS) | One-way | Allowed |
| **Flow:** Order placement requires User service for shipping address, user preferences |
| **Why one-way:** Order queries user data, User updates propagate via Kafka (async) |
| **SLA:** P99 < 100ms | **Actual:** 65ms |
| **Payment → User** | TCP/8080 (HTTPS) | One-way | mTLS required |
| **Flow:** Payment validates user identity before processing (anti-fraud), checks billing address match |
| **Why mTLS:** Sensitive payment data requires strongest authentication |
| **SLA:** P99 < 150ms | **Actual:** 118ms |
| **Catalog → User** | — | — | DENIED |
| **Why blocked:** Catalog is public product data, no user-specific logic. If personalized recommendations needed, use Data Platform ML models. |
| **User → Payment** | — | — | DENIED |
| **Why blocked:** Reverse direction not needed; user account changes handled via async Kafka events |
| **User → Order** | — | — | DENIED |
| **Why blocked:** User profile updates propagate via Kafka (async), no synchronous call required |
| **Catalog → Order** | — | — | DENIED |
| **Why blocked:** Product price changes published to Kafka, Order consumes asynchronously |

**Security Enforcement:**
- **mTLS Required:** Istio service mesh validates client certificates
- **mTLS Recommended:** TLS encryption, client cert validation optional
- **Rate Limiting:** 1,000 req/sec per source account (prevents retry storms)
- **Circuit Breaker:** Opens if destination error rate >10% (cascading failure prevention)

---

### Production → Data Platform

| Source | Destination | Ports | Direction | Rule | Justification |
|--------|-------------|-------|-----------|------|---------------|
| **Any Production → Data Platform** | TCP/443 (HTTPS), TCP/9092 (Kafka) | Outbound (one-way) | Allowed |
| **Examples:**
  - Payment publishes: `payment.completed` event
  - Order publishes: `order.placed`, `order.shipped` events
  - User publishes: `user.registered`, `user.profile.updated` events |
| **Why one-way:** Data Platform consumes events for analytics, ML training, BI dashboards |
| **Annual Value:** $2M+ in business insights (churn prediction, product recommendations) |
| **Data Platform → Any Production** | — | — | DENIED |
| **Why blocked:** Data Platform is read-only consumer; cannot write to production DBs (data corruption risk) |
| **Exception:** ML predictions delivered via Kafka topic (not direct DB write) |

**Data Flow Example:**
```
Order Service → Kafka Topic "order-events" → Data Platform Consumer
→ MaxCompute (SQL analytics) → Grafana Dashboard "Orders per Hour"
→ PAI (ML training) → Recommendation model
→ Kafka Topic "product-recommendations" → Catalog Service
```

---

### Non-Production Environments

| Source | Destination | Ports | Direction | Rule | Justification |
|--------|-------------|-------|-----------|------|---------------|
| **Staging/Dev → Production** | ANY | ANY | DENIED |
| **Why:** Environment isolation prevents accidental prod data corruption during testing |
| **Exception:** Read-only replica of prod DB in staging (PII anonymized) for realistic testing |
| **Staging/Dev → Shared Services** | TCP/443, TCP/22 | Bidirectional | Allowed |
| **Why:** GitLab runners deploy to staging/dev, fetch Docker images from artifact registry |
| **Staging/Dev → Security** | TCP/9090 | Outbound | Allowed |
| **Why:** Monitoring for performance testing validation |
| **Staging/Dev → Network Hub** | ANY | Outbound | Allowed |
| **Why:** Internet access for testing external APIs (payment gateways, shipping providers) |

---

### Security Account Communication

| Source | Destination | Ports | Direction | Rule | Justification |
|--------|-------------|-------|-----------|------|---------------|
| **Security → All Accounts** | TCP/22 (SSH), TCP/443 (HTTPS), ICMP | Inbound (read-only) | Restricted |
| **Purpose:** Vulnerability scanning (Cloud Security Center), compliance checks (CIS benchmarks, PCI-DSS) |
| **Frequency:** Continuous scanning, weekly compliance reports |
| **Security → Any Account (write)** | — | — | DENIED |
| **Why:** Separation of duties; security team cannot modify prod infrastructure (insider threat prevention) |
| **Exception:** BreakGlass role via Management Account with dual approval |

**Audit Events:**
- All Security account actions logged to ActionTrail with `source_account=security` tag
- Weekly report: "Security account scanned 1,247 resources, found 3 vulnerabilities (all remediated)"

---

### Audit Account Communication (Immutable Log Sink)

| Source | Destination | Ports | Direction | Rule | Justification |
|--------|-------------|-------|-----------|------|---------------|
| **Any → Audit** | TCP/514 (Syslog), TCP/6514 (TLS), TCP/443 (HTTPS) | Write-only | Allowed |
| **Log Types:**
  - ActionTrail: All RAM API calls (7-year retention)
  - VPC Flow Logs: Network traffic metadata (1-year retention)
  - Application Logs: Microservice logs (90-day retention) |
| **WORM Enforced:** OSS bucket Write-Once-Read-Many (even root cannot delete) |
| **Audit → Any** | — | — | DENIED |
| **Why:** Audit account is immutable sink; never initiates connections (prevents pivot point) |

**Compliance Value:**
- PCI-DSS 10.5: "Log files are protected from unauthorized modification"
- GDPR Art 32: "Ability to restore availability of personal data in timely manner"

**Usage Example (Forensic Investigation):**
```
Jan 2026 Payment DB Incident:
1. ActionTrail logs showed: Engineer ran `DROP DATABASE` at 03:42 AM
2. VPC Flow Logs confirmed: Connection from engineer's IP (192.168.1.50)
3. Application logs revealed: Accidental script execution (not malicious)
4. Incident resolved in 2 hours (typical: 2 days without centralized logs)
```

---

### Shared Services Communication

| Source | Destination | Ports | Direction | Rule | Justification |
|--------|-------------|-------|-----------|------|---------------|
| **Shared Services → All Production** | TCP/22 (SSH), TCP/443 (K8s API) | Outbound | Restricted |
| **Purpose:** CI/CD deployment (GitLab Runner assumes CICDAutomationRole), monitoring data collection (Prometheus scrapes) |
| **Constraint:** CICDAutomationRole can only be assumed from Shared Services VPC CIDR (10.10.0.0/16) |
| **Why IP restriction:** Even if GitLab credentials leaked, attacker outside Shared Services cannot deploy |

**Deployment Flow:**
```
Engineer merges to main → GitLab CI triggers
→ GitLab Runner (Shared Services, 10.10.1.15) assumes role in Payment Prod
→ Deploys Docker image to ACK cluster
→ Logs deployment to Audit Account (Git SHA, timestamp, who)
```

**Audit Trail:**
```
ActionTrail Event:
  EventName: AssumeRole
  SourceAccount: ecom-shared-services
  TargetAccount: ecom-prod-payment
  RoleSessionName: gitlab-runner-pipeline-1234
  SourceIP: 10.10.1.15
  RequestParameters: { "RoleArn": "acs:ram::payment:role/CICDAutomationRole" }
```

---

### Sandbox Isolation

| Source | Destination | Ports | Direction | Rule | Justification |
|--------|-------------|-------|-----------|------|---------------|
| **Sandbox → Any Account** | ANY | ANY | DENIED |
| **Why:** Sandbox has **no CEN attachment**; cannot communicate with production/staging/dev |
| **Business Reason:** Experimental code must not affect production stability |
| **Any Account → Sandbox** | ANY | ANY | DENIED |
| **Why:** Production cannot reach sandbox (prevents accidental dependency on sandbox endpoints) |
| **Sandbox → Internet** | ANY | Outbound | Allowed |
| **Why:** Direct NAT (not via Network Hub) for testing external APIs |

**Exception Handling:**
- If sandbox needs cross-domain testing (e.g., Order→Payment integration), use **"Staging Sandbox"** variant
- Staging Sandbox has CEN access to **staging accounts only** (not production)

---

### Multi-Region Communication (Frankfurt ↔ London)

| Source | Destination | Ports | Direction | Rule | Justification |
|--------|-------------|-------|-----------|------|---------------|
| **Frankfurt Prod → London DR** | TCP/5432 (PostgreSQL), TCP/27017 (MongoDB), TCP/9092 (Kafka) | Outbound (async) | Allowed |
| **Purpose:** Database replication (streaming replication for PostgreSQL, replica set for MongoDB) |
| **RPO:** 15 minutes (async lag acceptable in disaster scenario) |
| **Bandwidth:** 1 Gbps CEN cross-region link ($800/month) |
| **Frankfurt → London OSS** | TCP/443 (HTTPS) | Outbound | Allowed |
| **Purpose:** Object storage replication (static assets, product images, CDN cache) |
| **CDN Failover:** If Frankfurt CDN fails, Alibaba Cloud CDN auto-fetches from London origin |
| **London DR → Frankfurt** | — | — | DENIED (passive standby) |
| **Why:** London is read-only replica; does not write back unless DR activated |
| **Exception:** During DR failover, London becomes primary and Frankfurt becomes standby (reverse replication) |

**DR Activation Process (Manual Failover):**
1. Platform team declares disaster (Frankfurt region offline)
2. DNS updated: `api.ecom.com` points to London SLB IPs
3. London PostgreSQL replica promoted to primary (`pg_ctl promote`)
4. London ACK cluster scales 10% → 100% (auto-scaling group, 15 min)
5. Reverse replication: London → Frankfurt (when Frankfurt recovered)

**Network Path:**
```
Frankfurt DB (10.1.20.15) → CEN Global Network (IPsec encrypted)
→ London DB (10.101.20.15)
```

**Replication Monitoring:**
- Replication lag: CloudMonitor alert if >30 min (RPO at risk)
- Last successful backup: Daily alert if backup age >24 hours

---

## Internal Service Ports (Self-Managed Middleware)

**WARNING: These services are NEVER exposed cross-account**

Each production account runs its own middleware stack in isolated VSwitches.

| Service | Port | VSwitch | Allowed Source | Configuration |
|---------|------|---------|----------------|---------------|
| **PostgreSQL** | 5432/TCP | db-vswitch (10.x.20.0/24) | ack-vswitch only | Streaming replication (primary + standby) |
| **MongoDB** | 27017/TCP | db-vswitch | ack-vswitch only | 3-node replica set, majority write concern |
| **Redis** | 6379/TCP | cache-vswitch (10.x.21.0/24) | ack-vswitch only | Sentinel HA, AOF persistence |
| **Kafka** | 9092/TCP | queue-vswitch (10.x.22.0/24) | ack-vswitch + Data Platform | 3 brokers, quorum replication |
| **RabbitMQ** | 5672/TCP, 15672/TCP (mgmt) | queue-vswitch | ack-vswitch only | Mirrored queues, 2-node cluster |

**Security Group Example (PostgreSQL in Payment Account):**

```yaml
Inbound:
  - Source: 10.1.10.0/22  # Payment ACK VSwitch
    Port: 5432
    Protocol: TCP
    Action: ALLOW
    Description: "ACK pods → PostgreSQL"
  
  - Source: 10.1.20.0/24  # Same DB VSwitch (replica-to-replica)
    Port: 5432
    Protocol: TCP
    Action: ALLOW
    Description: "PostgreSQL streaming replication"
  
  - Default: DENY ALL
    Action: DROP
    Log: YES

Outbound:
  - Destination: 10.1.20.0/24  # Replica-to-replica
    Port: 5432
    Protocol: TCP
    Action: ALLOW
  
  - Default: DENY ALL
    Action: DROP
    Description: "Data tier has no internet outbound (DLP)"
```

**Why No Cross-Account DB Access:**
- **Data Sovereignty:** Payment team owns Payment DB (Catalog team cannot query)
- **Blast Radius:** PostgreSQL vulnerability in Order ≠ Payment exposure
- **Compliance:** PCI-DSS requires Payment DB isolation from non-PCI systems

**Cross-Domain Data Sharing Patterns:**
- **Synchronous:** API calls (e.g., Order calls Payment API)
- **Asynchronous:** Kafka events (e.g., Order publishes `order.placed` event)
- **Never:** Direct database connections

---

## Exception Request Process

**Scenario:** Team needs communication path not in this matrix

**Example Request:**
```yaml
Communication Exception Request:
  Requestor: catalog-team@company.com
  Source: ecom-prod-catalog
  Destination: ecom-prod-user
  Port: TCP/8080 (HTTPS)
  Direction: One-way (Catalog → User)
  Justification: "New feature: Personalized product recommendations based on user browsing history"
  Duration: Permanent (or 90-day trial)
  Alternative Considered: "Event-driven via Kafka (slower, 5-sec lag unacceptable for real-time recs)"
```

**Review Process (24-hour SLA):**
1. **Security Review:**
   - Is synchronous call necessary or can it be async (Kafka)?
   - Does it violate least-privilege?
   - Is mTLS required for sensitive data?

2. **Architecture Review:**
   - Does it create circular dependency?
   - Impact on latency budget?
   - Blast radius if destination fails?

3. **Approval & Implementation:**
   - Update this Communication Matrix
   - Terraform apply:
     ```hcl
     resource "alicloud_cen_route_entry" "catalog_to_user" {
       instance_id = var.cen_instance_id
       route_table_id = alicloud_cen_instance.catalog.route_table_id
       destination_cidr_block = "10.4.0.0/16"  # User VPC CIDR
     }
     
     resource "alicloud_security_group_rule" "catalog_to_user" {
       security_group_id = alicloud_security_group.user_ack.id
       type = "ingress"
       ip_protocol = "tcp"
       port_range = "8080/8080"
       source_security_group_id = alicloud_security_group.catalog_ack.id
     }
     ```
   - Cloud Firewall allowlist rule added
   - CloudMonitor alarm: Alert if Catalog→User traffic >1000 req/sec

4. **Quarterly Review:**
   - Security team audits all communication paths
   - Unused exceptions revoked (no traffic in 90 days)

**Statistics (Q1 2026):**
- Exception requests: 5
- Approved: 4
- Denied: 1 (security risk)
- Revoked (unused): 2

---

## Compliance Mapping

| Regulation | Requirement | How Communication Matrix Satisfies |
|------------|-------------|-----------------------------------|
| **PCI-DSS Req 1.2.1** | Restrict traffic to necessary for cardholder data environment | Payment inbound limited to: SLB (HTTPS), Network Hub (egress), Shared Services (CI/CD), Order (API). All others denied. |
| **PCI-DSS Req 1.3.6** | Network segmentation between cardholder data and other networks | 5-tier VSwitch design isolates DB tier; Payment account isolated from other domains. |
| **GDPR Art 32** | Appropriate technical measures for security of processing | Default-deny policy, mTLS encryption, audit logging of all communication. |
| **SOC2 CC6.6** | Logical access restricted | RAM role assumption required, IP-restricted CI/CD, no static credentials. |

---

## Monitoring & Alerting

**Real-Time Anomaly Detection:**

| Alert | Trigger | Severity | Action |
|-------|---------|----------|--------|
| **Unauthorized connection** | Security group deny >100×/hour from same source | HIGH | Block source IP at Cloud Firewall, alert security team |
| **Unexpected cross-account traffic** | CEN traffic to destination not in matrix | CRITICAL | Alert platform team, auto-create incident ticket |
| **High egress volume** | Account sends >100 GB outbound in 10 min | CRITICAL | DLP alert (potential data exfiltration), throttle bandwidth |
| **Production → Sandbox** | Any traffic to 10.250.0.0/16 from prod | CRITICAL | Should be impossible (CEN not attached), escalate to CISO |
| **Excessive API calls** | Cross-domain calls >10,000 req/min | WARNING | Check for retry storm, investigate application |

**Weekly Reports:**
- Top 10 chattiest account pairs (bandwidth consumed)
- Denied connection attempts (security group blocks)
- Unutilized communication paths (approved but zero traffic → revoke?)
- Latency trends (detect degradation early)

**Example Alert (Feb 2026):**
```
Alert: High Egress Volume
Account: ecom-prod-payment
Outbound Traffic: 150 GB in 8 minutes
Destination: 203.0.113.99 (external IP)
Action: Bandwidth throttled to 10 Mbps, security team notified
Root Cause: Misconfigured backup script uploading to wrong S3 bucket
Resolution: Script fixed, alert tuning adjusted
```

---

## Summary Statistics

**Communication Paths:**
- Total possible paths: 15 accounts × 15 accounts = 225
- Explicitly allowed: 47 (21%)
- Denied by default: 178 (79%)

**Enforcement:**
- 4 enforcement layers (SG, CEN, Firewall, SCP)
- 100% audit coverage (all traffic logged)
- mTLS coverage: 35% of prod cross-domain (expanding to 100%)

**Compliance:**
- PCI-DSS: Payment account communication explicitly documented
- GDPR: EU data residency, no cross-region PII transfer
- SOC2: Centralized audit logging, role-based access

---

