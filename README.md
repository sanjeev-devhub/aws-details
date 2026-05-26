# AWS Interview Preparation — Java AWS Developer

> A complete, deep-dive prep guide for the AWS technical round after clearing Java.
> Each topic follows the same structure: **Concept → Real-world usage → Interview Q&A → Common mistakes → Production example → Architecture (text diagram) → Java/Spring Boot code → Follow-up cross-questions → Difficulty level.**

---

## Table of Contents

**PART 1 — CORE AWS SERVICES**
1. EC2
2. S3
3. RDS
4. DynamoDB
5. Lambda
6. API Gateway
7. VPC
8. IAM
9. CloudWatch
10. CloudFormation / Terraform (IaC)

**PART 2 — JAVA + AWS INTEGRATION**
**PART 3 — MICROSERVICES ARCHITECTURE**
**PART 4 — DEVOPS + CI/CD**
**PART 5 — SECURITY**
**PART 6 — SCENARIO-BASED QUESTIONS**
**PART 7 — TROUBLESHOOTING**
**PART 8 — Final Tips & Cheat Sheet**

---

# PART 1 — CORE AWS SERVICES

---

## 1. EC2 (Elastic Compute Cloud)

### 1.1 Concept

EC2 provides resizable virtual machines (instances) running on AWS hypervisors. You choose CPU, memory, storage, networking, and OS. EC2 is the foundation of compute on AWS — most managed services run on EC2 underneath.

Key building blocks:
- **Instance** — a virtual server
- **AMI** (Amazon Machine Image) — the template (OS + pre-installed software) used to launch an instance
- **EBS** — block storage attached to instances (like a disk)
- **Security Group** — virtual firewall at instance level (stateful)
- **Key pair** — SSH access credentials
- **Elastic IP** — static public IPv4
- **ENI** (Elastic Network Interface) — virtual NIC

### 1.2 Instance Types

EC2 instance families are named like `m5.large`, where:
- **m** = family (general purpose)
- **5** = generation
- **large** = size

| Family | Use case | Example |
|---|---|---|
| **t** (T3, T4g) | Burstable, low-cost, dev/test, small APIs | t3.medium |
| **m** (M5, M6i, M7g) | General purpose (balanced CPU:RAM) | m5.large |
| **c** (C5, C6i, C7g) | Compute-optimized — CPU-heavy (encoding, batch, gaming) | c5.xlarge |
| **r** (R5, R6i) | Memory-optimized — in-memory caches, large JVMs, Redis | r5.large |
| **x / u** | High-memory — SAP HANA, large in-memory DBs | x1e.32xlarge |
| **i / d** | Storage-optimized — NoSQL, data warehousing | i3.large |
| **p / g / inf / trn** | GPU / ML / inference | p4d.24xlarge |
| **a / g (Graviton)** | ARM-based, 20–40% cheaper for same perf | m6g.large, c7g.large |

**Java tip**: Spring Boot microservices usually start with `t3.medium` or `m5.large`. Heap-heavy services → R family. For cost optimization, Graviton (ARM) is great if your dependencies have ARM builds.

### 1.3 Auto Scaling Group (ASG)

ASG automatically adjusts the number of EC2 instances based on demand.

Components:
- **Launch Template** — defines AMI, instance type, security groups, user data
- **Min / Max / Desired capacity**
- **Scaling policies**:
  - **Target tracking** — keep CPU at 60% (most common)
  - **Step scaling** — add 2 instances if CPU > 70%, add 4 if > 90%
  - **Scheduled** — scale up at 9 AM Mon–Fri
- **Health checks** — EC2 status or ELB health
- **Cooldown period** — prevents flapping

### 1.4 Load Balancer (ELB)

| Type | Layer | Use case |
|---|---|---|
| **ALB** (Application LB) | L7 (HTTP/HTTPS) | Microservices, path/host routing, WebSockets, gRPC |
| **NLB** (Network LB) | L4 (TCP/UDP) | Ultra-low latency, static IP, millions of req/sec |
| **GWLB** (Gateway LB) | L3 | Inserting firewalls/IDS into traffic path |
| **CLB** (Classic) | L4/L7 | Legacy — avoid for new workloads |

ALB features: path-based routing (`/api/users` → service A, `/api/orders` → service B), host-based routing, sticky sessions, target groups, WAF integration, SSL/TLS termination.

### 1.5 Security Groups

- **Stateful** — if you allow inbound traffic, the response is automatically allowed out
- **Allow rules only** (no explicit deny — denial is implicit)
- Attached to ENIs (instances), not subnets
- Can reference other SGs (e.g., "allow port 8080 from `sg-app`")

vs **NACLs** (Network ACLs):
- **Stateless** — must allow in AND out separately
- Allow AND deny rules
- Attached to subnets
- Evaluated in rule-number order

### 1.6 AMI

AMI = OS + pre-installed software + config. Two flavors:
- **EBS-backed** — root volume on EBS, can stop/start (most common)
- **Instance store-backed** — ephemeral, lost on stop

**Golden AMI pattern**: Pre-bake your JDK, Tomcat/Spring Boot, agents (CloudWatch, Datadog) into a custom AMI using **Packer**. Faster boot than installing at runtime via user-data.

### 1.7 Pricing Models

| Model | Discount | Use case |
|---|---|---|
| **On-Demand** | 0% (baseline) | Unpredictable workloads, dev/test |
| **Reserved Instances (RI)** | up to ~72% | Steady-state production (commit 1 or 3 yrs) |
| **Savings Plans** | up to ~72% | Like RI but more flexible (commit to $/hr spend) |
| **Spot** | up to ~90% | Stateless, fault-tolerant: batch jobs, CI, big data |
| **Dedicated Host / Instance** | premium | BYOL licensing, compliance |

**Spot gotcha**: AWS can terminate with 2-minute warning. Use for stateless workers; never for stateful DBs.

### 1.8 Placement Groups

| Type | Behavior | Use case |
|---|---|---|
| **Cluster** | All instances in same rack | HPC, low latency |
| **Spread** | Instances on different hardware | Critical small workloads (max 7 per AZ) |
| **Partition** | Logical partitions on distinct racks | HDFS, Cassandra, Kafka |

### 1.9 High Availability vs Fault Tolerance

- **High Availability (HA)** = system stays up during failure (some downtime tolerated)
- **Fault Tolerance (FT)** = zero downtime even during failure (more expensive)

**HA recipe on EC2**:
- ASG across **≥2 AZs**
- ALB in front
- RDS Multi-AZ
- Stateless app servers, session in ElastiCache/Redis

### 1.10 Real-World Usage Example

> "We run our Spring Boot order service on an ASG of 4–20 `m5.large` instances across 3 AZs in `ap-south-1`. An ALB distributes traffic. CloudWatch alarms scale based on `RequestCountPerTarget`. We use 60% Reserved Instances + 40% On-Demand for the steady baseline, with Spot for batch jobs at night."

### 1.11 Interview Questions & Ideal Answers

**Q1. (Beginner) What is the difference between Stop and Terminate an EC2 instance?**
> **Stop**: instance shuts down, EBS volume persists, public IP is released (Elastic IP retained). You can start again.
> **Terminate**: instance is deleted; root EBS volume is deleted by default (`DeleteOnTermination=true`); cannot be recovered.

**Q2. (Intermediate) How does ASG decide which instance to terminate during scale-in?**
> Default termination policy:
> 1. AZ with most instances
> 2. Instance with oldest launch template/config
> 3. Instance closest to next billing hour (legacy)
> 4. Random
> You can override with `OldestInstance`, `NewestInstance`, `OldestLaunchConfiguration`, etc., or use instance protection.

**Q3. (Intermediate) Difference between ALB and NLB?**
> ALB is L7, understands HTTP, supports path/host routing, headers, WebSockets, target groups for Lambda/ECS/IP. NLB is L4 (TCP/UDP/TLS), static IP, preserves source IP, handles millions of req/sec with very low latency. Use NLB for gRPC w/ static IP, financial trading apps; ALB for typical REST microservices.

**Q4. (Advanced) Your EC2 instance can't reach the internet. How do you debug?**
> Walk the network path systematically:
> 1. Is it in a public or private subnet? Check route table — does `0.0.0.0/0` route to an IGW (public) or NAT (private)?
> 2. Does the instance have a public IP (public subnet) or is it behind NAT?
> 3. Security Group — does outbound allow 80/443? (Default allows all outbound)
> 4. NACL — does it allow ephemeral return ports (1024–65535)?
> 5. OS-level firewall (iptables, ufw)?
> 6. DNS resolution working? (`nslookup`)
> 7. VPC endpoints — if you only need S3/DynamoDB, a Gateway Endpoint avoids public internet entirely.

**Q5. (Architect) Design a fault-tolerant deployment for a stateless Spring Boot API serving 50K RPS.**
> - Multi-AZ ASG (≥3 AZs) with `m5.large` minimum 12, max 60
> - ALB with WAF, TLS termination, HTTP/2
> - CloudFront in front for caching static and TLS at edge
> - Stateless app — sessions in ElastiCache Redis cluster (Multi-AZ)
> - RDS Aurora with reader endpoint + 2 read replicas
> - Use SQS for any async work to decouple spikes
> - Use Route 53 health checks with failover to a DR region
> - Use Reserved Instances for ~70% baseline + Spot for stateless background workers

### 1.12 Common Mistakes

- Opening SG to `0.0.0.0/0` on port 22 — **never** in production
- Putting databases in public subnets
- Using single AZ "to save cost" — AWS reimburses nothing for AZ failure
- Forgetting to set `DeleteOnTermination=false` for important data volumes
- Using On-Demand for steady production loads (cost waste)
- Not using IMDSv2 (token-based) — IMDSv1 is vulnerable to SSRF

### 1.13 Architecture Diagram (Text)

```
                          Internet
                              │
                       ┌──────▼──────┐
                       │  Route 53   │  (DNS, health checks)
                       └──────┬──────┘
                              │
                       ┌──────▼──────┐
                       │ CloudFront  │  (CDN + WAF)
                       └──────┬──────┘
                              │
                  ┌───────────▼────────────┐
                  │   ALB (Multi-AZ)        │
                  │   /api/* → ASG-API      │
                  │   /web/* → ASG-Web      │
                  └─────┬─────────────┬─────┘
                        │             │
              ┌─────────▼───┐   ┌─────▼─────────┐
              │  AZ-1a      │   │  AZ-1b        │
              │  EC2 × 2    │   │  EC2 × 2      │
              │  (ASG)      │   │  (ASG)        │
              └──────┬──────┘   └──────┬────────┘
                     │                 │
                     └────────┬────────┘
                              ▼
                       ┌─────────────┐
                       │  RDS        │  Multi-AZ
                       │  + Replicas │
                       └─────────────┘
```

### 1.14 Java / Spring Boot Code

**Reading EC2 instance metadata (IMDSv2) from Spring Boot:**

```java
@Component
public class Ec2MetadataService {
    private final RestTemplate restTemplate = new RestTemplate();
    private static final String METADATA_BASE = "http://169.254.169.254";

    public String getInstanceId() {
        // IMDSv2 — get session token first
        HttpHeaders tokenHeaders = new HttpHeaders();
        tokenHeaders.set("X-aws-ec2-metadata-token-ttl-seconds", "21600");
        ResponseEntity<String> tokenResp = restTemplate.exchange(
            METADATA_BASE + "/latest/api/token",
            HttpMethod.PUT,
            new HttpEntity<>(tokenHeaders),
            String.class);
        String token = tokenResp.getBody();

        HttpHeaders dataHeaders = new HttpHeaders();
        dataHeaders.set("X-aws-ec2-metadata-token", token);
        return restTemplate.exchange(
            METADATA_BASE + "/latest/meta-data/instance-id",
            HttpMethod.GET,
            new HttpEntity<>(dataHeaders),
            String.class).getBody();
    }
}
```

**Graceful shutdown for ASG termination (Spring Boot):**

```yaml
# application.yml
server:
  shutdown: graceful
spring:
  lifecycle:
    timeout-per-shutdown-phase: 30s
```

Set ASG lifecycle hook on `EC2_INSTANCE_TERMINATING` to give Spring Boot time to drain.

### 1.15 Follow-up Cross-Questions

- What's the difference between an EBS volume and an instance store?
- How would you migrate an EC2-based monolith to ECS/EKS?
- Explain the boot sequence of an EC2 instance — when does user-data run?
- How does ASG warm-up period affect target tracking?
- What is enhanced networking (ENA, SR-IOV)?

### 1.16 Difficulty Tags
Beginner: instance types, SG basics. Intermediate: ASG, ALB vs NLB, pricing models. Advanced: HA design, placement groups, Spot strategies. Architect: multi-region failover, cost-optimized hybrid pricing.

---

## 2. S3 (Simple Storage Service)

### 2.1 Concept

S3 is object storage — files (objects) stored in buckets, accessed via HTTP API. Designed for **11 nines of durability** (99.999999999%) and **4 nines availability** (varies by class). Unlimited storage; individual objects up to 5 TB.

Key facts:
- Bucket names are **globally unique**
- Objects = key (path) + value (bytes) + metadata
- Strong read-after-write consistency (since Dec 2020)
- Regional service (data stays in region unless replicated)

### 2.2 Storage Classes

| Class | Use case | Retrieval | Cost |
|---|---|---|---|
| **Standard** | Hot data, frequent access | ms | $$$ |
| **Intelligent-Tiering** | Unknown/changing access patterns | ms | auto-optimized |
| **Standard-IA** | Infrequent access, must be available | ms | $$ |
| **One Zone-IA** | Re-creatable, infrequent, 1 AZ | ms | $ |
| **Glacier Instant** | Archive, ms retrieval | ms | $ |
| **Glacier Flexible** | Archive, minutes-hours retrieval | min–hr | $ |
| **Glacier Deep Archive** | Long-term archive (compliance) | hours | $/10 |

**Lifecycle rule example**: Standard → Standard-IA after 30 days → Glacier after 90 → Delete after 7 years.

### 2.3 Versioning

- Once enabled, **cannot be fully disabled** (only suspended)
- Each PUT creates a new version
- DELETE creates a "delete marker" (object isn't actually removed)
- Combine with **MFA Delete** for compliance
- Costs more — every version is stored

### 2.4 Lifecycle Policies

Automate transitions and deletions:

```json
{
  "Rules": [{
    "ID": "ArchiveOldLogs",
    "Status": "Enabled",
    "Filter": { "Prefix": "logs/" },
    "Transitions": [
      { "Days": 30, "StorageClass": "STANDARD_IA" },
      { "Days": 90, "StorageClass": "GLACIER" }
    ],
    "Expiration": { "Days": 2555 },
    "NoncurrentVersionExpiration": { "NoncurrentDays": 30 }
  }]
}
```

### 2.5 Encryption

- **SSE-S3** — AWS-managed keys (default since Jan 2023, free)
- **SSE-KMS** — AWS KMS keys (audit trail, key rotation)
- **SSE-C** — customer-provided keys
- **Client-side encryption** — encrypt before upload

In transit: enforce HTTPS via bucket policy `"aws:SecureTransport": "true"`.

### 2.6 Presigned URLs

Time-limited URLs to grant temporary access without making the bucket public. Common use: let users upload directly to S3 from browser, skipping your app server (saves bandwidth & memory).

### 2.7 Multipart Upload

Required for objects > 5 GB; recommended for > 100 MB.
- Splits into parts (5 MB – 5 GB each)
- Parallel upload → faster
- Resume on failure → robust
- Use `TransferManager` in Java SDK; abort incomplete uploads via lifecycle rule (else you pay for them).

### 2.8 Cross-Region Replication (CRR) / Same-Region Replication (SRR)

- Requires versioning on both source and destination
- Async — eventually consistent
- Use cases: DR, compliance (data in EU & US), latency reduction
- New objects only by default; use **S3 Batch Replication** for existing.

### 2.9 Static Website Hosting

Bucket as a website (`index.html`, `error.html`). Use **CloudFront + S3 (OAC)** in front for HTTPS, CDN, custom domain.

### 2.10 Interview Q&A

**Q1. (Beginner) Difference between S3 and EBS?**
> S3 is object storage accessed via HTTPS, region-scoped, unlimited, durable, multi-AZ by default. EBS is block storage attached to a single EC2 instance, like a virtual hard disk, AZ-scoped.

**Q2. (Intermediate) How would you let a mobile app upload large videos directly to S3?**
> Use **presigned URLs** with `PutObject` permission, time-limited (e.g., 15 min). For very large files, generate presigned URLs for each part of a multipart upload. App uploads directly, never proxying through your server.

**Q3. (Intermediate) How do you prevent accidental deletion of critical objects?**
> 1. Enable **versioning** — deletes are recoverable
> 2. Enable **MFA Delete** for the bucket
> 3. Apply **Object Lock** in compliance mode (WORM) — even root can't delete during retention
> 4. Use **bucket policies** to deny `s3:DeleteObject` from anyone except a specific role
> 5. Replicate critical data via CRR

**Q4. (Advanced) Explain S3 consistency model.**
> S3 provides **strong read-after-write consistency** for all operations (since Dec 2020) — PUT, overwrites, and DELETE are immediately visible to subsequent GETs and LISTs in any region. Historically only new-object PUTs were strongly consistent; that limitation is gone.

**Q5. (Architect) Design a system to ingest 10 TB/day of clickstream data with cost-efficient storage and analytics.**
> - Kinesis Firehose → S3 Standard (raw landing zone, partitioned `year=/month=/day=/hour=`)
> - Lambda triggered on `s3:ObjectCreated` to validate/enrich → write to a curated bucket in Parquet
> - Athena/Redshift Spectrum for ad-hoc queries on Parquet
> - Lifecycle: raw → IA after 30 days → Glacier after 90; curated stays in Standard
> - CRR to DR region for compliance
> - Glue Catalog for schema, Glue ETL for batch jobs

### 2.11 Common Mistakes

- Making buckets public (huge breach risk); enable **S3 Block Public Access** at account level
- Not enabling versioning until *after* an accidental delete
- Forgetting to abort incomplete multipart uploads (silent cost)
- Listing with `ListObjectsV2` for millions of keys (slow) — use S3 Inventory or Athena
- Putting bucket region in latency-critical path without VPC endpoint (avoid NAT cost)

### 2.12 Architecture Example

```
Browser ── POST presigned URL ──▶  S3 Bucket
                                     │
                          (ObjectCreated event)
                                     ▼
                              Lambda (resize/scan)
                                     │
                            ┌────────┴─────────┐
                            ▼                  ▼
                      DynamoDB metadata    SNS notify
```

### 2.13 Java / Spring Boot — S3 Operations

**Dependency** (`pom.xml`):
```xml
<dependency>
  <groupId>software.amazon.awssdk</groupId>
  <artifactId>s3</artifactId>
  <version>2.25.0</version>
</dependency>
```

**Service class**:
```java
@Service
public class S3Service {
    private final S3Client s3;
    private final S3Presigner presigner;
    private final String bucket;

    public S3Service(@Value("${app.s3.bucket}") String bucket) {
        this.bucket = bucket;
        this.s3 = S3Client.builder().region(Region.AP_SOUTH_1).build();
        this.presigner = S3Presigner.builder().region(Region.AP_SOUTH_1).build();
    }

    public void upload(String key, byte[] data) {
        s3.putObject(PutObjectRequest.builder()
                .bucket(bucket).key(key)
                .serverSideEncryption(ServerSideEncryption.AES256)
                .build(),
            RequestBody.fromBytes(data));
    }

    public URL generatePresignedUploadUrl(String key, Duration ttl) {
        PutObjectRequest objectRequest = PutObjectRequest.builder()
                .bucket(bucket).key(key).build();
        PresignedPutObjectRequest req = presigner.presignPutObject(
            PutObjectPresignRequest.builder()
                .signatureDuration(ttl)
                .putObjectRequest(objectRequest)
                .build());
        return req.url();
    }

    public byte[] download(String key) {
        ResponseBytes<GetObjectResponse> resp = s3.getObjectAsBytes(
            GetObjectRequest.builder().bucket(bucket).key(key).build());
        return resp.asByteArray();
    }
}
```

### 2.14 Follow-up Cross-Questions

- How do you secure an S3 bucket end-to-end?
- What is S3 Transfer Acceleration?
- Difference between bucket policy and IAM policy?
- How does S3 Object Lambda work?
- Explain S3 Select and when you'd use it.
- What is a Gateway Endpoint vs Interface Endpoint for S3?

### 2.15 Difficulty Tags
Beginner: storage classes, versioning. Intermediate: lifecycle, presigned URLs, CRR. Advanced: consistency model, multipart strategy. Architect: data lake design, cost tiering at scale.

---

## 3. RDS (Relational Database Service)

### 3.1 Concept

Managed relational DB service. AWS handles provisioning, patching, backups, failover. You still manage schema, queries, indexes. Supported engines: MySQL, PostgreSQL, MariaDB, Oracle, SQL Server, **Aurora** (AWS-native).

### 3.2 Multi-AZ

- A **synchronous** standby replica in a different AZ
- Automatic failover (~60–120 sec) on primary failure, AZ outage, instance class change, OS patching
- The standby is **NOT readable** (it's just for HA, not load distribution)
- Same DNS endpoint — Spring Boot doesn't need code changes
- Costs ~2× single-AZ

### 3.3 Read Replicas

- **Asynchronous** replication (some lag, usually < 1 sec)
- Up to 5 (15 for Aurora) per primary
- Cross-region replicas possible (DR + read locality)
- Can be promoted to primary (manual)
- Use for read-heavy workloads (reporting, analytics)

> **Multi-AZ ≠ Read Replica.** Multi-AZ = HA (failover, not readable). Read replica = scaling reads (eventual consistency, can be promoted).

### 3.4 Backups

- **Automated backups**: daily full snapshot + 5-min transaction logs, retained 1–35 days, point-in-time restore (PITR)
- **Manual snapshots**: kept until you delete them
- Backups are encrypted if the DB is encrypted
- Stored in S3 internally (not visible to you)

### 3.5 Failover Behavior

Failover triggers:
- Primary AZ outage
- Primary instance failure
- Manual reboot with failover
- Storage failure
- Instance class modification

During failover (~60–120s), CNAME is updated to point to the standby. Your app should retry with exponential backoff.

### 3.6 Aurora

AWS-native engine, compatible with MySQL/PostgreSQL.
- **5× MySQL / 3× PostgreSQL throughput**
- Storage **auto-scales 10 GB → 128 TB**, shared across 6 copies in 3 AZs
- Up to **15 read replicas** with sub-10ms lag
- **Aurora Global Database** — cross-region replication < 1 sec lag
- **Aurora Serverless v2** — auto-scales ACUs based on load
- **Backtrack** (MySQL only) — rewind DB without restore

### 3.7 Connection Pooling

JVM apps open many connections. Each RDS connection has memory cost. Solutions:
- **HikariCP** in Spring Boot (default) — set `maximumPoolSize` carefully
- **RDS Proxy** — pools connections across many app instances, transparent failover (< 1s vs 60s direct), reduces DB connections by 95%+
- For Lambda **always** use RDS Proxy (Lambda concurrency × connections per fn = DB connection storm)

### 3.8 Performance Tuning

- Use **Performance Insights** + **Enhanced Monitoring**
- Right-size instance (memory > working set)
- Use **GP3** EBS (configurable IOPS, cheaper than IO1)
- **Parameter groups** — tune `max_connections`, `work_mem`, `shared_buffers`
- Indexes, EXPLAIN ANALYZE, kill long queries
- Read replicas for read scaling, but watch replication lag
- Cache aggressively with ElastiCache (Redis) in front

### 3.9 Interview Q&A

**Q1. (Beginner) Multi-AZ vs Read Replica?**
> Multi-AZ is for **HA** — synchronous standby in different AZ, not readable, automatic failover. Read replicas are for **read scaling** — asynchronous, readable, can be promoted manually.

**Q2. (Intermediate) Why use RDS Proxy?**
> 1) Reduces DB connection count from app fleets; 2) Sub-1-sec failover vs ~60s without; 3) Critical for Lambda to avoid connection exhaustion; 4) Auth via IAM/Secrets Manager.

**Q3. (Advanced) Your read-replica lag is 30s during peak. How do you debug?**
> Check `ReplicaLag` CloudWatch metric. Causes:
> - Long-running transactions on primary (logical replication can't apply faster than single thread on replica)
> - Replica instance smaller than primary → CPU/IO bottleneck
> - Network saturation between AZs
> - Heavy queries on replica blocking apply
> Fixes: scale up replica, split long writes, use Aurora (parallel replication), avoid huge transactions, use logical replication slots carefully.

**Q4. (Architect) Design a globally available DB layer for an e-commerce app serving India + US.**
> **Aurora Global Database**: primary in `ap-south-1`, secondary in `us-east-1`. Sub-second replication. Writes go to primary; US reads served locally from secondary. Promote US in DR (~< 1 min RTO). Add Redis (ElastiCache Global Datastore) for hot cart/session data. Use DynamoDB Global Tables for things needing local writes (e.g., user profile updates).

### 3.10 Common Mistakes

- Treating Multi-AZ standby as a read replica
- Hardcoding the writer endpoint in code (use Route 53 / Aurora endpoint abstractions)
- Hikari `maximumPoolSize` too high (DB can't handle it)
- Forgetting to test failover (chaos engineering)
- No alarm on `FreeableMemory`, `ReplicaLag`, `DatabaseConnections`

### 3.11 Spring Boot Configuration

```yaml
spring:
  datasource:
    url: jdbc:postgresql://prod-db.cluster-xyz.ap-south-1.rds.amazonaws.com:5432/orders
    username: ${DB_USER}        # from Secrets Manager
    password: ${DB_PASSWORD}
    hikari:
      maximum-pool-size: 20
      minimum-idle: 5
      connection-timeout: 30000
      idle-timeout: 600000
      max-lifetime: 1800000
      leak-detection-threshold: 60000
  jpa:
    properties:
      hibernate.jdbc.batch_size: 50
      hibernate.order_inserts: true
```

For reader/writer routing:
```yaml
spring:
  datasource:
    writer:
      url: jdbc:postgresql://prod-db.cluster-xyz.ap-south-1.rds.amazonaws.com:5432/orders
    reader:
      url: jdbc:postgresql://prod-db.cluster-ro-xyz.ap-south-1.rds.amazonaws.com:5432/orders
```
Use `AbstractRoutingDataSource` to route `@Transactional(readOnly=true)` to reader.

### 3.12 Follow-up Cross-Questions

- What is Aurora's storage architecture?
- Explain Aurora Serverless v2 ACUs.
- How does RDS handle minor and major version upgrades?
- What's the difference between snapshot restore and PITR?
- How would you migrate on-prem Oracle to RDS Postgres?

---

## 4. DynamoDB

### 4.1 Concept

Fully managed, serverless NoSQL key-value + document DB. Single-digit ms latency at any scale. No servers to manage; pay for read/write capacity and storage.

Data model:
- **Table** → **Items** (rows) → **Attributes** (columns, can be nested)
- Each item identified by a **Primary Key**
- Schemaless except for the key

### 4.2 Partition Key & Sort Key

- **Partition Key (PK)** — hash key; determines which partition stores the item
- **Sort Key (SK)** — optional; orders items within a partition
- **Composite key** = PK + SK

Example:
```
PK = userId (string)
SK = orderId (string)
```
This lets you query "all orders for a user, sorted by orderId" efficiently.

**Hot partition problem**: if one PK gets disproportionate traffic, that partition throttles. Design keys to spread load (suffix sharding, write sharding).

### 4.3 GSI vs LSI

- **LSI (Local Secondary Index)** — same PK, different SK. Created **only at table creation**. Max 5.
- **GSI (Global Secondary Index)** — different PK & SK. Created anytime. Max 20. Eventually consistent.

**Rule of thumb**: prefer **GSI**. LSIs share throughput with base table and can't be added later.

### 4.4 Capacity Modes

- **On-Demand** — pay per request; auto-scales instantly; great for unpredictable workloads
- **Provisioned** — set RCU/WCU; cheaper if steady; combine with auto-scaling
- 1 **WCU** = 1 write/sec of ≤ 1 KB
- 1 **RCU** = 1 strongly consistent read/sec of ≤ 4 KB (or 2 eventually consistent)

### 4.5 Scaling

DynamoDB scales horizontally by partitioning. Each partition = 1000 WCU / 3000 RCU / 10 GB max. Partitions split automatically. **Adaptive capacity** helps absorb mild hot keys.

### 4.6 TTL

Set an attribute (e.g., `expireAt` epoch seconds). DynamoDB deletes items within ~48 hours of expiry (no extra cost). Great for session tokens, OTPs, ephemeral caches.

### 4.7 Streams

Change Data Capture (CDC) — every insert/update/delete creates a stream record (retained 24 hrs). Consumers:
- Lambda triggers (most common)
- Kinesis Data Streams adapter

Use cases: cross-region replication, search index sync (to OpenSearch), event sourcing, materialized views.

### 4.8 Use Cases — DynamoDB vs RDS

| Choose DynamoDB when | Choose RDS when |
|---|---|
| Massive scale, predictable access patterns | Complex queries, joins, aggregations |
| Single-digit ms latency required | Transactions across many entities |
| Serverless, no admin | ACID with strong consistency |
| Key-value or document data | Relational data with rich schema |
| Variable/spiky traffic (on-demand) | Reports, BI, analytics |

### 4.9 Interview Q&A

**Q1. (Beginner) Difference between Query and Scan?**
> **Query** uses a PK (and optional SK condition) — O(1)-ish, only reads matching partition, efficient. **Scan** reads the entire table — slow and expensive. Avoid scan; design indexes to support all access patterns.

**Q2. (Intermediate) How would you model a one-to-many relationship?**
> Use composite PK + SK. Example: `PK=USER#123, SK=PROFILE` for the user record; `PK=USER#123, SK=ORDER#789` for each order. Single query on `PK=USER#123` returns the user and all orders.

**Q3. (Advanced) Hot partition keeps throttling. What do you do?**
> 1) Add randomness/sharding to PK (e.g., `userId#0..9`) and fan out reads.
> 2) Switch to on-demand or raise provisioned + auto-scaling.
> 3) Use **DAX** (in-memory cache for DynamoDB) — microsecond reads for hot items.
> 4) Re-examine access pattern; maybe a different PK design avoids the hot key entirely.

**Q4. (Architect) Design a chat application's message store in DynamoDB.**
> Table: `Messages`
> - PK = `chatRoomId`
> - SK = `timestamp#messageId`
> - Attributes: sender, content, attachments
> - GSI1: PK = `userId`, SK = `timestamp` → "all messages by a user"
> - TTL on ephemeral DMs (e.g., disappearing messages)
> - Stream → Lambda → push notifications + OpenSearch index
> - On-demand capacity for spiky traffic

### 4.10 Common Mistakes

- Using Scan in production
- Designing without thinking through access patterns first (DDB rewards single-table design)
- Forgetting GSIs cost extra writes (every write to base table → write to each GSI)
- No TTL on ephemeral data → table bloat
- Strong consistency everywhere (2× RCU; usually eventually consistent is fine)

### 4.11 Java / Spring Boot — Enhanced Client

```java
@DynamoDbBean
public class Order {
    private String userId;
    private String orderId;
    private String status;
    private Long createdAt;

    @DynamoDbPartitionKey public String getUserId() { return userId; }
    @DynamoDbSortKey public String getOrderId() { return orderId; }
    // setters, getters...
}

@Service
public class OrderRepository {
    private final DynamoDbTable<Order> table;

    public OrderRepository(DynamoDbEnhancedClient enhancedClient) {
        this.table = enhancedClient.table("Orders",
            TableSchema.fromBean(Order.class));
    }

    public void save(Order o) { table.putItem(o); }

    public Order find(String userId, String orderId) {
        return table.getItem(Key.builder()
            .partitionValue(userId).sortValue(orderId).build());
    }

    public List<Order> ordersForUser(String userId) {
        return table.query(QueryConditional.keyEqualTo(
            Key.builder().partitionValue(userId).build()))
            .items().stream().toList();
    }
}
```

### 4.12 Follow-up Cross-Questions

- What is DAX and when would you use it?
- Explain DynamoDB transactions (TransactWriteItems).
- How do Global Tables handle write conflicts?
- What is single-table design? Pros/cons.
- How does DynamoDB pricing differ between on-demand and provisioned?

---

## 5. Lambda

### 5.1 Concept

Serverless compute — upload code, AWS runs it on demand. Pay per invocation + GB-seconds. No servers, OS, or scaling to manage.

Limits:
- 15 min max execution
- 10 GB memory (vCPU scales proportionally)
- 512 MB – 10 GB `/tmp`
- 6 MB sync payload (request/response), 256 KB async event
- 1000 concurrent executions per account (soft limit, raisable)

### 5.2 Cold Start

When a new container is provisioned: download code → init runtime → init your code → handler runs. Cold start = the init phase delay.

Cold start factors:
- **Runtime**: Java/.NET slowest (1–3 sec), Python/Node fastest (~100ms), Go/Rust fast
- **Package size**: bigger zip = slower download
- **VPC**: faster now (ENI sharing), but still slightly slower
- **Init code**: static blocks, Spring context — minimize!

Mitigations:
- **Provisioned Concurrency** — keep N warm containers ready
- **SnapStart** (Java) — snapshot post-init, restore in < 200ms. Free on Java 11/17/21.
- Avoid Spring Boot full container — use Spring Cloud Function + GraalVM native image, or Micronaut/Quarkus for Lambda-optimized frameworks
- Lazy-load heavy SDKs

### 5.3 Memory Optimization

CPU scales linearly with memory. 1769 MB = 1 full vCPU. Often **more memory = faster = cheaper** (GB-seconds drop). Use **AWS Lambda Power Tuning** (state machine) to find the sweet spot.

### 5.4 Timeout Handling

Set timeout slightly above worst-case latency. For downstream calls, set client timeout < Lambda timeout. Use exponential backoff with jitter on retries. Idempotent handlers are critical (Lambda may retry on async events).

### 5.5 Event-Driven Architecture

Lambda triggers:
- API Gateway (sync HTTP)
- ALB
- S3 (object created/deleted)
- SQS (poll-based, batch)
- SNS (push, fan-out)
- EventBridge (cron, custom buses)
- DynamoDB Streams / Kinesis (stream-based)
- Cognito, IoT, CloudWatch Logs, etc.

### 5.6 Java Lambda Best Practices

1. **Use SnapStart** — biggest cold-start win for Java
2. **Reuse SDK clients** outside the handler (static / instance fields) — they survive between invocations
3. **Avoid Spring Boot** unless using SnapStart + Spring Cloud Function
4. **Prefer Java 21** + ZGC for short-lived tasks
5. **Use `aws-lambda-java-events`** for typed events
6. **Externalize config** — env vars, Parameter Store, Secrets Manager
7. **Structured logging** — JSON, with correlation IDs
8. **Use Powertools for Lambda Java** (logging, metrics, tracing, idempotency)

### 5.7 Integration with API Gateway / SQS / SNS

- **API Gateway → Lambda**: sync, 29s API GW timeout (raise to 30s default; up to 5 min via HTTP API). Return `APIGatewayProxyResponse`.
- **SQS → Lambda**: poll-based, batch up to 10 (FIFO) / 10000 (Standard). Errors return entire batch to queue unless `ReportBatchItemFailures` is used. Use DLQ on the queue.
- **SNS → Lambda**: async push. Failures → Lambda's own DLQ.

### 5.8 Interview Q&A

**Q1. (Beginner) What is a cold start and how do you reduce it for Java?**
> First invocation of a new Lambda container: AWS provisions, downloads code, inits JVM, runs your initializer. For Java this is 1–3 sec. Mitigations: **SnapStart** (snapshot post-init, restore in ~200 ms), **Provisioned Concurrency**, smaller jars, lazy loading, native image (GraalVM).

**Q2. (Intermediate) How does Lambda scale, and what is reserved/provisioned concurrency?**
> Lambda spins up one container per concurrent request, up to account limit. **Reserved concurrency** caps how many can run for that function (also reserves slots). **Provisioned concurrency** pre-initializes N containers so they're always warm (you pay for them idle).

**Q3. (Advanced) Lambda is timing out connecting to RDS. Why?**
> Most likely **connection storm** — each Lambda creates its own DB connections; at scale you exhaust `max_connections`. Fix: use **RDS Proxy** in front of RDS; configure short connection timeout in app; reuse `DataSource` outside handler; consider Aurora Serverless v2.

**Q4. (Architect) Design an image-processing pipeline using Lambda.**
> S3 upload → S3 event → SQS (decouple + retry + DLQ) → Lambda (thumbnail + EXIF strip) → DynamoDB metadata → SNS to downstream subscribers. Use Lambda destinations for async success/failure routing. Provisioned concurrency = 10 for predictable latency; on-demand for bursts.

### 5.9 Common Mistakes

- Creating SDK client *inside* handler (re-created every cold start)
- No DLQ → silent message loss
- Long timeouts hide real issues (set conservatively)
- Spring Boot fat jar on Lambda without SnapStart (multi-second cold starts)
- Not making handlers idempotent → duplicate processing
- Holding DB connections forever (no pool eviction)

### 5.10 Java Lambda Example (with SnapStart-friendly init)

```java
public class OrderHandler implements RequestHandler<SQSEvent, Void> {

    // Initialized once, reused across invocations
    private static final DynamoDbClient ddb = DynamoDbClient.builder()
            .region(Region.AP_SOUTH_1)
            .build();
    private static final ObjectMapper mapper = new ObjectMapper();

    @Override
    public Void handleRequest(SQSEvent event, Context ctx) {
        for (SQSEvent.SQSMessage msg : event.getRecords()) {
            try {
                Order order = mapper.readValue(msg.getBody(), Order.class);
                ddb.putItem(PutItemRequest.builder()
                    .tableName("Orders")
                    .item(Map.of(
                        "userId",  AttributeValue.fromS(order.userId()),
                        "orderId", AttributeValue.fromS(order.orderId()),
                        "status",  AttributeValue.fromS("RECEIVED")))
                    .build());
            } catch (Exception e) {
                ctx.getLogger().log("Failed: " + e.getMessage());
                throw new RuntimeException(e); // returns msg to queue, eventually DLQ
            }
        }
        return null;
    }
}
```

Enable SnapStart in template:
```yaml
Resources:
  OrderFunction:
    Type: AWS::Serverless::Function
    Properties:
      Runtime: java21
      SnapStart:
        ApplyOn: PublishedVersions
```

### 5.11 Follow-up Cross-Questions

- Difference between sync, async, and stream invocations?
- What is Lambda destinations vs DLQ?
- Explain Lambda execution role vs resource policy.
- When would you choose Step Functions over a chain of Lambdas?
- How does Lambda's ENI lifecycle work in a VPC?

---

## 6. API Gateway

### 6.1 Concept

Managed API front door. Handles routing, auth, throttling, caching, transformations, monitoring for REST/HTTP/WebSocket APIs. Backed by Lambda, HTTP endpoints, ECS, EC2, or AWS services.

### 6.2 REST API vs HTTP API vs WebSocket API

| Feature | REST API | HTTP API | WebSocket |
|---|---|---|---|
| Cost | $$$ (3.5x) | $ (cheapest) | $$ |
| Latency | Higher | Lower | N/A |
| Caching | Yes | No | No |
| Request validation | Yes | No | No |
| WAF integration | Yes | No (via CloudFront) | No |
| API keys / usage plans | Yes | No | No |
| Private endpoints | Yes | No | No |
| JWT authorizer | Lambda only | Native | Lambda |
| Use case | Enterprise, complex | Simple proxy, microservices | Real-time chat/notifs |

Default for new microservices: **HTTP API** (cheaper, faster). Use REST API only if you need features it has.

### 6.3 Authentication

- **IAM** — SigV4 signed requests (service-to-service)
- **Cognito User Pools** — OAuth2 / OIDC user auth
- **Lambda authorizer** — custom JWT, OAuth, API key
- **JWT authorizer** (HTTP API native) — validates JWT against JWKS
- **API keys + Usage plans** — quota & rate limit per customer
- **Resource policy** — restrict by source IP/VPC

### 6.4 Throttling & Rate Limiting

- **Account-level**: default 10,000 RPS, 5,000 burst
- **Stage-level**: override per stage
- **Method-level**: override per route
- **Per-API-key** via usage plans (REST API)
- Returns **429 Too Many Requests** when exceeded

### 6.5 Caching (REST API only)

- Per stage, 0.5 GB – 237 GB
- Reduces backend calls + improves latency
- Cache key = method + path + selected query/header params
- Costs extra (hourly)
- Invalidate via `Cache-Control: max-age=0` header (require permission)

### 6.6 Lambda Integration

- **Proxy integration** (most common) — entire request/response passed to/from Lambda as JSON
- **Non-proxy** — use mapping templates (VTL) to transform — flexible but harder to maintain

### 6.7 Interview Q&A

**Q1. (Beginner) Difference between REST API and HTTP API?**
> HTTP API is newer, ~70% cheaper, faster, supports JWT natively, but lacks features like caching, request validation, usage plans, edge-optimized endpoints, and WAF. Use HTTP API for most new builds; REST API when you need the extra features.

**Q2. (Intermediate) How do you implement rate limiting per customer?**
> Use REST API with **API keys + Usage plans**. Each customer gets an API key; usage plan sets quota (e.g., 10K req/day) and throttle (e.g., 100 RPS). For HTTP API, do this at Lambda authorizer + DynamoDB counter, or in front via WAF rate-based rules.

**Q3. (Advanced) How would you secure an API exposing PII?**
> 1. TLS 1.2+ only (custom domain + ACM cert)
> 2. JWT auth (Cognito or IdP) with short-lived tokens
> 3. WAF rules: SQLi/XSS, rate-based, geo-blocks
> 4. Resource policy to restrict source IPs (corporate VPN)
> 5. CloudWatch + X-Ray for audit + tracing
> 6. Encrypt sensitive request/response in transit + at rest (KMS)
> 7. Private API via VPC endpoint if internal
> 8. Mutual TLS (mTLS) for B2B partners
> 9. Secrets in Secrets Manager, never in env vars
> 10. Logging excludes PII (use parameter filtering)

**Q4. (Architect) Design an API platform serving 10K RPS with multi-tenant isolation.**
> CloudFront + WAF in front → API Gateway HTTP API → ALB → ECS Fargate microservices. Per-tenant API keys + usage plans on REST API (if needed). Identify tenants via JWT claim; isolate data via row-level security or per-tenant DDB partition. Multi-region with Route 53 latency routing.

### 6.8 Common Mistakes

- Using REST API when HTTP API would do (3x cost)
- Not setting throttling → runaway cost from misbehaving clients
- Mapping templates with hardcoded logic (move to Lambda)
- Logging full requests including PII/secrets
- No WAF in front of public APIs

### 6.9 Architecture Diagram

```
Client → CloudFront(+WAF) → API Gateway (HTTP API, JWT authorizer)
                                     │
                            ┌────────┼─────────┐
                            ▼        ▼         ▼
                       Lambda    ALB→ECS   VPC Link→NLB
                                     │
                                  Service
```

### 6.10 Spring Boot Behind API Gateway

API Gateway HTTP API → VPC Link → internal NLB → ECS service.

```java
@RestController
@RequestMapping("/orders")
public class OrderController {
    @GetMapping("/{id}")
    public Order get(@PathVariable String id,
                     @RequestHeader("X-Trace-Id") String traceId) {
        MDC.put("traceId", traceId);
        return service.findById(id);
    }
}
```

API GW can pass JWT claims as headers (`X-User-Id`, `X-Tenant-Id`); Spring Boot reads them via `@RequestHeader` or a filter.

### 6.11 Follow-up Cross-Questions

- Edge-optimized vs Regional vs Private endpoint?
- How do mapping templates work? Show a VTL example.
- What's a Lambda authorizer TOKEN vs REQUEST?
- How do you do API versioning?
- Canary deployments in API Gateway?

---

## 7. VPC (Virtual Private Cloud)

### 7.1 Concept

A logically isolated network in AWS — your own private cloud. Defined by a CIDR block (e.g., `10.0.0.0/16`). Spans one region, multiple AZs.

### 7.2 Subnets

A subnet is a CIDR sub-range within one AZ.
- **Public subnet** — route table has a route to an **Internet Gateway (IGW)**
- **Private subnet** — no direct IGW route; outbound via **NAT Gateway**
- **Isolated subnet** — no internet at all (DBs, internal-only services)

Best practice: 3 AZs × (public, private-app, private-db) = 9 subnets.

### 7.3 Internet Gateway (IGW)

Horizontally scaled, redundant VPC component. One per VPC. Makes a subnet "public" when route table has `0.0.0.0/0 → igw-xxxx`.

### 7.4 NAT Gateway

Allows private subnets to make outbound internet requests (patch updates, external APIs) while blocking inbound.
- Managed (vs NAT instance)
- AZ-scoped — deploy one per AZ for HA
- Costs: ~$32/month + data processing $0.045/GB
- **Cost trap**: huge data transfer through NAT is expensive — use VPC Endpoints for AWS services

### 7.5 Route Tables

Determine where traffic from a subnet goes.
- One route table per subnet (or use default)
- Routes are evaluated **most-specific first**
- Common entries: `local` (VPC CIDR), `0.0.0.0/0 → IGW or NAT`, peered VPC, VPN/Transit Gateway

### 7.6 NACL (Network ACL)

Subnet-level **stateless** firewall.
- Rules numbered, evaluated in order
- Both allow and deny rules
- Must allow ephemeral ports (1024–65535) for return traffic
- Default NACL: allow all in/out; custom NACL: deny all in/out

### 7.7 Security Group vs NACL — Side-by-side

| | Security Group | NACL |
|---|---|---|
| Scope | ENI / instance | Subnet |
| State | Stateful | Stateless |
| Rules | Allow only | Allow + Deny |
| Evaluation | All evaluated | Numbered, in order |
| Applies | One SG to many ENIs | One NACL to one subnet |
| Use | Primary firewall | Defense in depth (e.g., block specific IPs) |

### 7.8 VPC Peering

Connect two VPCs (same or different account/region). Traffic stays on AWS backbone, no IGW. **Not transitive** — A↔B and B↔C does not mean A↔C. For many VPCs, use **Transit Gateway** instead.

### 7.9 VPC Endpoints

Access AWS services privately, without traversing the internet.

| Type | Services | Cost |
|---|---|---|
| **Gateway** | S3, DynamoDB | Free |
| **Interface** (PrivateLink) | Most AWS services + your own | $/hour + $/GB |

Use cases: keep traffic on AWS network; avoid NAT data costs; meet compliance (no public internet).

### 7.10 AWS PrivateLink

Expose your own service (behind NLB/ALB) to other VPCs/accounts via interface endpoints, without VPC peering. Provider exposes a "service"; consumers create an interface endpoint to it. Used heavily for B2B SaaS within AWS.

### 7.11 Interview Q&A

**Q1. (Beginner) Difference between IGW and NAT Gateway?**
> IGW allows bidirectional internet traffic to/from public subnets. NAT GW allows only **outbound** internet traffic from private subnets (inbound is blocked).

**Q2. (Intermediate) Security Group vs NACL?**
> SG is stateful, attached to ENIs, allow-only. NACL is stateless, attached to subnets, allow + deny, evaluated by number. NACLs are useful for blocking specific IP ranges; SGs do day-to-day filtering.

**Q3. (Advanced) Reduce data transfer cost from EC2 → S3.**
> Add a **Gateway VPC Endpoint** for S3. Traffic stays inside AWS network, no NAT processing, no internet egress. Endpoint is free; you only pay the (much cheaper) intra-region transfer.

**Q4. (Architect) Design VPC for a 3-tier app across 3 AZs.**
> VPC `10.0.0.0/16`. In each of 3 AZs: public subnet `/24` (ALB, NAT GW), private-app `/22` (ECS/EC2), private-db `/24` (RDS). Public RT → IGW; app RT → NAT in same AZ (avoid cross-AZ NAT cost); db RT → no internet. SGs: ALB-sg (80/443 from 0.0.0.0/0), app-sg (8080 from alb-sg), db-sg (5432 from app-sg). Gateway endpoints for S3 + DynamoDB. Interface endpoints for Secrets Manager, KMS, ECR.

### 7.12 Common Mistakes

- All resources in public subnet (security risk)
- One NAT GW for all AZs (single point of failure + cross-AZ data charge)
- Overlapping CIDRs (blocks future peering)
- Too-small CIDR (run out of IPs at scale)
- No VPC endpoints (expensive NAT traffic for AWS APIs)
- NACL blocking ephemeral return ports (mystery timeouts)

### 7.13 Architecture Diagram

```
VPC 10.0.0.0/16
│
├── AZ-1a
│   ├── Public 10.0.1.0/24  → IGW    [ALB, NAT-1a]
│   ├── App    10.0.10.0/22 → NAT-1a [ECS tasks]
│   └── DB     10.0.20.0/24 → (no internet) [RDS primary]
│
├── AZ-1b
│   ├── Public 10.0.2.0/24  → IGW
│   ├── App    10.0.14.0/22 → NAT-1b
│   └── DB     10.0.21.0/24 → (no internet) [RDS standby]
│
└── AZ-1c ...
```

### 7.14 Follow-up Cross-Questions

- What is Transit Gateway and when do you use it?
- Difference between PrivateLink and VPC Peering?
- How do you debug network issues — VPC Flow Logs, Reachability Analyzer?
- What is a VPC Lattice?
- IPv6 in VPC?

---

## 8. IAM (Identity and Access Management)

### 8.1 Concept

Global, free service that controls **who can do what** in AWS. Built on:
- **Users** — long-term human credentials
- **Groups** — collection of users
- **Roles** — temporary credentials, assumed by services/users
- **Policies** — JSON documents describing permissions

### 8.2 Policy Types

| Type | Where | Use |
|---|---|---|
| **Identity-based** | Attached to user/role/group | Permissions granted to identity |
| **Resource-based** | Attached to resource (S3 bucket, KMS key) | Who can access this resource (cross-account) |
| **Permission boundary** | Attached to user/role | Max permissions (limit) |
| **SCP** (Service Control Policy) | Org/OU/account | Org-wide guardrails |
| **Session policy** | Passed in AssumeRole | Further restrict session |
| **ACL** | S3 (legacy) | Avoid |

### 8.3 Policy Structure

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "AllowS3Read",
    "Effect": "Allow",
    "Action": ["s3:GetObject", "s3:ListBucket"],
    "Resource": [
      "arn:aws:s3:::my-bucket",
      "arn:aws:s3:::my-bucket/*"
    ],
    "Condition": {
      "StringEquals": { "aws:RequestedRegion": "ap-south-1" },
      "IpAddress":   { "aws:SourceIp": "203.0.113.0/24" }
    }
  }]
}
```

### 8.4 Roles (the crux)

A role is an identity with temporary credentials, assumed by:
- EC2 / ECS task / Lambda (via instance/task role)
- Federated users (SSO, SAML, OIDC)
- Other AWS accounts (cross-account access)

Each role has:
- **Trust policy** — who can assume it
- **Permission policy** — what the role can do

### 8.5 Least Privilege

Start with **deny all**, grant minimum permissions to function. Use:
- IAM Access Analyzer (generates policies from CloudTrail)
- `aws iam simulate-principal-policy`
- Scoped resources (specific ARNs, not `*`)
- Conditions to narrow further

### 8.6 Temporary Credentials & STS

**Security Token Service (STS)** issues short-lived credentials.
- `AssumeRole` — assume a role in same or other account
- `AssumeRoleWithSAML` — federated via SAML
- `AssumeRoleWithWebIdentity` — Cognito, Google, OIDC
- `GetSessionToken` — MFA-required session

Credentials = access key + secret key + **session token** (must be sent on every request).

### 8.7 Cross-Account Access

Pattern:
1. Account A creates a role; trust policy allows Account B's principal.
2. Account B's user/role calls `sts:AssumeRole` with role ARN.
3. STS returns temp credentials valid for Account A.

Add **External ID** for third-party access (confused deputy prevention).

### 8.8 Interview Q&A

**Q1. (Beginner) Difference between IAM user and role?**
> A **user** has long-lived credentials, ties to a person. A **role** has temporary creds, is assumed by services or other principals. Best practice: humans use roles via SSO; services use instance/task roles. Avoid IAM users in production wherever possible.

**Q2. (Intermediate) How does an EC2 instance get permissions?**
> Attach an **instance profile** (containing a role) to the EC2. The instance metadata service (IMDSv2) exposes temporary credentials at `169.254.169.254`. AWS SDK auto-retrieves and refreshes them. App code doesn't need static keys.

**Q3. (Advanced) An explicit Allow and explicit Deny both apply. What happens?**
> **Explicit Deny always wins.** IAM evaluation order: deny by SCP → deny by resource policy → deny by identity policy → allow (any source) → otherwise implicit deny.

**Q4. (Architect) Design a permissions model for a 200-engineer company across dev/stage/prod accounts.**
> Use **AWS Organizations** + **SCPs** for guardrails (deny dangerous actions org-wide). **AWS SSO / Identity Center** for human access — engineers federated, assume role per account. **Permission sets** per job function (dev, qa, sre, sec). Service accounts as IAM roles (no users). Cross-account access via `AssumeRole`. Boundaries on every role to prevent privilege escalation. CloudTrail aggregated to a Security account.

### 8.9 Common Mistakes

- Long-lived access keys in code/repo
- `"Resource": "*"` everywhere
- Same role used by dev and prod
- Granting `iam:*` to developers
- No MFA on root
- Storing root credentials anywhere (root should be locked away)

### 8.10 Java / Spring Boot — Using Roles

With instance/task roles, the SDK auto-discovers credentials:

```java
S3Client s3 = S3Client.builder()
    .region(Region.AP_SOUTH_1)
    .credentialsProvider(DefaultCredentialsProvider.create())
    .build();
```

`DefaultCredentialsProvider` checks env vars → system props → profile → container → instance role.

**Cross-account access**:
```java
StsClient sts = StsClient.create();
AssumeRoleRequest req = AssumeRoleRequest.builder()
        .roleArn("arn:aws:iam::222222222222:role/CrossAccountRead")
        .roleSessionName("orders-service")
        .externalId("UNIQUE-EXTERNAL-ID")
        .build();
Credentials creds = sts.assumeRole(req).credentials();

AwsSessionCredentials session = AwsSessionCredentials.create(
        creds.accessKeyId(), creds.secretAccessKey(), creds.sessionToken());
S3Client crossAccountS3 = S3Client.builder()
        .credentialsProvider(StaticCredentialsProvider.create(session))
        .build();
```

### 8.11 Follow-up Cross-Questions

- What is an IAM permissions boundary and when do you use one?
- Difference between an SCP and an IAM policy?
- How does ABAC (tag-based) differ from RBAC?
- What is IRSA (IAM Roles for Service Accounts) in EKS?
- How would you detect overly permissive policies?

---

## 9. CloudWatch

### 9.1 Concept

AWS's observability suite: metrics, logs, alarms, dashboards, events, synthetic monitoring, ServiceLens (X-Ray + RUM).

### 9.2 Metrics

- Numeric data points emitted by services or your code
- **Namespaces** group metrics (`AWS/EC2`, `MyApp/Orders`)
- **Dimensions** = key-value labels (InstanceId, FunctionName)
- **Standard resolution** (1 min) free for AWS metrics; **High resolution** (1 sec) custom only, costs more
- Retention: 1 min for 15 days, 5 min for 63 days, 1 hr for 15 months

### 9.3 Logs

- **Log group** → **Log stream** → **Log events**
- Sources: Lambda (auto), EC2 (via agent), ECS (awslogs driver), VPC Flow Logs, etc.
- **Retention**: default = "Never expire" (cost trap!); set explicitly (1 day to 10 years)
- **Insights** — SQL-like log queries
- **Metric filters** — extract numeric metrics from logs
- **Subscription filters** — stream to Kinesis/Lambda for real-time processing

### 9.4 Alarms

- Based on metric crossing threshold for N periods
- States: `OK`, `ALARM`, `INSUFFICIENT_DATA`
- Actions: SNS, Auto Scaling, EC2 actions (reboot/stop)
- **Composite alarms** — combine multiple alarms with AND/OR
- **Anomaly detection** — ML-based threshold

### 9.5 Dashboards

JSON-defined; widgets: line, stacked, number, log query, text. Cross-region. Export as PDF/share read-only.

### 9.6 Log Retention

**Always set retention** on log groups — by default they keep forever and cost adds up fast. Standard: 30 days for dev, 90 days for prod, 7 years for compliance-relevant.

### 9.7 Monitoring Microservices

Pattern: every service emits:
- **RED** metrics: Rate, Errors, Duration
- **USE** metrics: Utilization, Saturation, Errors (for infra)
- Structured JSON logs with `traceId`, `userId`, `service`
- **X-Ray** traces across services
- Custom metrics via **EMF (Embedded Metric Format)** — emit metrics in log JSON, CloudWatch auto-extracts

### 9.8 Interview Q&A

**Q1. (Beginner) Difference between CloudWatch Metrics and Logs?**
> Metrics = time-series numeric data, aggregated for dashboards/alarms. Logs = text events, queryable, can be transformed into metrics via metric filters or EMF.

**Q2. (Intermediate) How do you alert on p99 latency > 500ms?**
> Emit latency as a histogram metric (EMF). Create a CloudWatch alarm on `p99` statistic > 500 for 3 of 5 periods. Action → SNS topic → PagerDuty.

**Q3. (Advanced) Your microservice generates 100 GB logs/day. Cost is too high.**
> 1) Set retention (default forever); 2) Reduce log verbosity in prod (INFO not DEBUG); 3) Stream selective logs to S3 via Firehose (cheaper long-term storage); 4) Use **log compression**; 5) Sample noisy logs; 6) Use metric filters to convert log signals to metrics (cheaper).

**Q4. (Architect) Design end-to-end observability for 30 microservices.**
> Three pillars: **metrics** (CloudWatch / Prometheus + Grafana), **logs** (CloudWatch / OpenSearch / Datadog), **traces** (X-Ray / OpenTelemetry → Jaeger). Standardized OpenTelemetry SDK across services emits to ADOT (AWS Distro for OpenTelemetry) collector. Correlation IDs propagated end-to-end. ServiceLens or Datadog APM for cross-service maps. PagerDuty for paging; runbooks linked from alerts.

### 9.9 Common Mistakes

- Forgetting to set log retention
- High-resolution custom metrics for everything (expensive)
- No structured logging (hard to query)
- Alarming on raw counters (alarm on rate instead)
- No `INSUFFICIENT_DATA` handling (alarms silent during outages)

### 9.10 Spring Boot → CloudWatch

Use **Micrometer** with the CloudWatch registry, or **Powertools for AWS Lambda** for Lambda.

```xml
<dependency>
  <groupId>io.micrometer</groupId>
  <artifactId>micrometer-registry-cloudwatch2</artifactId>
</dependency>
```

```java
@Bean
public CloudWatchMeterRegistry cloudWatchMeterRegistry() {
    return new CloudWatchMeterRegistry(
        new CloudWatchConfig() {
            @Override public String get(String key) { return null; }
            @Override public String namespace() { return "OrdersService"; }
            @Override public Duration step() { return Duration.ofMinutes(1); }
        },
        Clock.SYSTEM,
        CloudWatchAsyncClient.create());
}
```

Logback JSON encoder for structured logs:
```xml
<encoder class="net.logstash.logback.encoder.LogstashEncoder">
  <customFields>{"service":"orders","env":"prod"}</customFields>
</encoder>
```

### 9.11 Follow-up Cross-Questions

- What is EMF and why is it cheaper than PutMetricData?
- How does X-Ray sampling work?
- Difference between CloudWatch Events and EventBridge?
- How do you alert on log patterns (e.g., "ERROR" rate)?
- Explain Container Insights for ECS/EKS.

---

## 10. CloudFormation / Terraform (IaC)

### 10.1 Concept

**Infrastructure as Code (IaC)** — declare infra in version-controlled files; tools provision exactly what's declared. Eliminates "click-ops", enables reproducibility, peer review, rollback.

### 10.2 CloudFormation

- AWS-native, YAML/JSON
- **Stack** = collection of resources from a template
- **Change sets** = preview before apply
- **Drift detection** = detect manual changes
- **Nested stacks** + **StackSets** (multi-account/region)
- **CDK** (Cloud Development Kit) — write infra in Java/TS/Python, compiles to CloudFormation
- Free (you pay only for resources)

### 10.3 Terraform

- HashiCorp, multi-cloud (AWS, GCP, Azure, Kubernetes, etc.)
- HCL (HashiCorp Configuration Language)
- **State file** (S3 + DynamoDB lock for teams)
- **Modules** — reusable building blocks
- Larger community, better ecosystem outside AWS
- Terraform Cloud / Enterprise for collaboration

### 10.4 Stack Design

Layer by lifecycle:
1. **Network** (VPC, subnets, IGW, NAT) — rarely changes
2. **Data** (RDS, DynamoDB, S3) — careful changes
3. **Compute** (ECS, ASG, Lambda) — frequent
4. **App** (API Gateway, services) — most frequent

Each layer in its own stack — independent deploys, smaller blast radius.

### 10.5 Parameterization

CloudFormation:
```yaml
Parameters:
  Environment:
    Type: String
    AllowedValues: [dev, stage, prod]
Resources:
  Bucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketName: !Sub "orders-${Environment}-${AWS::AccountId}"
```

Terraform:
```hcl
variable "environment" {
  type = string
  validation {
    condition = contains(["dev", "stage", "prod"], var.environment)
    error_message = "Must be dev, stage, or prod."
  }
}

resource "aws_s3_bucket" "orders" {
  bucket = "orders-${var.environment}-${data.aws_caller_identity.current.account_id}"
}
```

### 10.6 Rollback Handling

- **CloudFormation**: by default rolls back on failure. Disable for debugging with `--disable-rollback`. Stack states: `CREATE_FAILED`, `ROLLBACK_COMPLETE` (must delete to retry).
- **Terraform**: does NOT auto-rollback. Failed apply leaves partial state. You must fix and re-apply, or `terraform destroy` and start over.

### 10.7 Drift Detection

- CloudFormation: `detect-stack-drift` API
- Terraform: `terraform plan` shows drift implicitly (compares state to actual)
- Enforce "no manual changes" via SCPs and CI/CD

### 10.8 Interview Q&A

**Q1. (Beginner) CloudFormation vs Terraform?**
> CFN is AWS-only, native, free, no state file to manage. Terraform is multi-cloud, larger community, has state file (you manage), better ecosystem for non-AWS resources. Most teams pick Terraform for portability + ecosystem.

**Q2. (Intermediate) How does Terraform handle state?**
> Local by default → not safe for teams. Use **remote backend** (S3 + DynamoDB lock). S3 stores `terraform.tfstate`; DynamoDB provides locking so two engineers can't apply simultaneously. Enable versioning + encryption on the bucket.

**Q3. (Advanced) You changed a resource manually in console. How do you reconcile?**
> Terraform: `terraform plan` shows drift; either revert the manual change or import / update HCL to match. CloudFormation: run drift detection; update template to reflect changes, then update stack.

**Q4. (Architect) Design a multi-account, multi-region IaC strategy.**
> Use **AWS Organizations** with separate accounts per env (dev/stage/prod) and per business unit. **CDK or Terraform** with modules per layer. CI/CD pipeline (CodePipeline / GitHub Actions) per account assumes deploy role via OIDC. Shared modules versioned in artifact registry. **StackSets** (CFN) or **Terragrunt** (TF) for fanning out to many accounts. Policy-as-code (OPA, cfn-nag, tfsec) gates every PR.

### 10.9 Common Mistakes

- Manual console changes after IaC adoption
- Monolithic stack (one stack for everything) → slow updates, large blast radius
- State file in Git (security disaster — has secrets)
- No `terraform plan` review before apply
- Hardcoding account IDs / regions
- Forgetting to lifecycle critical resources (`prevent_destroy`)

### 10.10 Java + IaC: CDK Example

```java
public class OrderServiceStack extends Stack {
    public OrderServiceStack(Construct scope, String id) {
        super(scope, id);

        Vpc vpc = Vpc.Builder.create(this, "Vpc")
                .maxAzs(3).build();

        Cluster cluster = Cluster.Builder.create(this, "Cluster")
                .vpc(vpc).build();

        ApplicationLoadBalancedFargateService.Builder.create(this, "OrderSvc")
                .cluster(cluster)
                .cpu(512).memoryLimitMiB(1024)
                .desiredCount(3)
                .taskImageOptions(ApplicationLoadBalancedTaskImageOptions.builder()
                    .image(ContainerImage.fromRegistry("orders:latest"))
                    .containerPort(8080)
                    .build())
                .publicLoadBalancer(true)
                .build();
    }
}
```

### 10.11 Follow-up Cross-Questions

- What is CDK and how does it relate to CloudFormation?
- Difference between `terraform import` and a CFN resource import?
- How would you secure secrets in IaC?
- Explain StackSets vs Service Catalog.
- What is policy-as-code? Tools?

---

# PART 2 — JAVA + AWS INTEGRATION

This is where most Java AWS Developer interviews focus. Expect deep code questions.

---

## 2.1 Spring Boot Deployment on AWS

### Deployment Options (from least to most managed)

| Option | When |
|---|---|
| **EC2 + jar** | Legacy / simple / full control |
| **Elastic Beanstalk** | Quick app deploys, less DevOps |
| **ECS on EC2** | Container orchestration, control over hosts |
| **ECS on Fargate** | Serverless containers (recommended default) |
| **EKS** | Kubernetes ecosystem, multi-cloud aspiration |
| **Lambda + SnapStart** | Event-driven, low traffic, spiky |
| **App Runner** | Simple HTTPS API from container, zero infra |

### Recommended: ECS Fargate

**Why**: no servers to patch, scales to thousands of tasks, integrates with ALB, CloudWatch, IAM. Cost-effective for most Spring Boot services.

### Build Pipeline

1. Maven/Gradle build → fat jar
2. Docker build (multi-stage for small image)
3. Push to **ECR** (Elastic Container Registry)
4. ECS service update (new task definition revision)
5. ECS rolls out (rolling/blue-green/canary)

### Dockerfile (multi-stage, JDK 21, layered jar)

```dockerfile
# Stage 1: Build
FROM eclipse-temurin:21-jdk-alpine AS build
WORKDIR /app
COPY pom.xml mvnw ./
COPY .mvn .mvn
RUN ./mvnw dependency:go-offline
COPY src src
RUN ./mvnw package -DskipTests && \
    java -Djarmode=layertools -jar target/*.jar extract

# Stage 2: Runtime
FROM eclipse-temurin:21-jre-alpine
WORKDIR /app
COPY --from=build /app/dependencies/ ./
COPY --from=build /app/spring-boot-loader/ ./
COPY --from=build /app/snapshot-dependencies/ ./
COPY --from=build /app/application/ ./
EXPOSE 8080
ENV JAVA_OPTS="-XX:+UseG1GC -XX:MaxRAMPercentage=75 -XX:+UseContainerSupport"
ENTRYPOINT ["sh", "-c", "java $JAVA_OPTS org.springframework.boot.loader.launch.JarLauncher"]
```

### Task Definition (ECS Fargate)

```json
{
  "family": "orders-service",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "1024",
  "memory": "2048",
  "executionRoleArn": "arn:aws:iam::111:role/ecsTaskExecutionRole",
  "taskRoleArn":      "arn:aws:iam::111:role/ordersTaskRole",
  "containerDefinitions": [{
    "name": "orders",
    "image": "111.dkr.ecr.ap-south-1.amazonaws.com/orders:1.4.2",
    "portMappings": [{ "containerPort": 8080 }],
    "secrets": [
      { "name": "DB_PASSWORD",
        "valueFrom": "arn:aws:secretsmanager:ap-south-1:111:secret:prod/db-AbCdEf" }
    ],
    "environment": [
      { "name": "SPRING_PROFILES_ACTIVE", "value": "prod" }
    ],
    "logConfiguration": {
      "logDriver": "awslogs",
      "options": {
        "awslogs-group": "/ecs/orders",
        "awslogs-region": "ap-south-1",
        "awslogs-stream-prefix": "orders"
      }
    },
    "healthCheck": {
      "command": ["CMD-SHELL", "wget -qO- http://localhost:8080/actuator/health || exit 1"],
      "interval": 30, "timeout": 5, "retries": 3, "startPeriod": 60
    }
  }]
}
```

---

## 2.2 Microservices Architecture on AWS

### Reference Pattern

```
                  ┌────────────┐
                  │  Route 53  │
                  └─────┬──────┘
                        │
                  ┌─────▼──────┐
                  │ CloudFront │
                  │ + WAF      │
                  └─────┬──────┘
                        │
                  ┌─────▼─────────┐
                  │  API Gateway  │ ── JWT (Cognito) / mTLS
                  │   or ALB      │
                  └─────┬─────────┘
                        │
        ┌───────────────┼────────────────┐
        ▼               ▼                ▼
  ┌──────────┐    ┌──────────┐     ┌──────────┐
  │ Orders   │    │ Users    │     │ Payments │  ECS Fargate
  │ Service  │    │ Service  │     │ Service  │
  └────┬─────┘    └────┬─────┘     └────┬─────┘
       │               │                │
       │  (SQS/SNS/EventBridge for async)
       ▼               ▼                ▼
  ┌──────────┐    ┌──────────┐     ┌──────────┐
  │ Aurora   │    │ DynamoDB │     │ Aurora   │
  └──────────┘    └──────────┘     └──────────┘

  Shared: Secrets Manager, Parameter Store, KMS, S3, ElastiCache, X-Ray
```

### Key Decisions

| Question | Default answer |
|---|---|
| Sync or async between services? | Async (SQS/SNS/EventBridge) wherever possible |
| One DB or per service? | **One DB per service** (true microservice) |
| Service discovery? | **AWS Cloud Map** or ECS Service Connect; or just internal ALB |
| Authentication between services? | mTLS / IAM SigV4 / signed JWT |

---

## 2.3 ECS vs EKS vs EC2

| Aspect | ECS | EKS | EC2 + manual |
|---|---|---|---|
| Orchestrator | AWS proprietary | Kubernetes | None |
| Learning curve | Easy | Steep | Easy infra, hard ops |
| Portability | AWS-only | Multi-cloud | Anywhere |
| Networking | awsvpc (1 ENI/task) | CNI plugins | Manual |
| Cost (control plane) | Free | $0.10/hr | Free |
| Best for | AWS-native teams, simpler ops | K8s expertise, multi-cloud, complex ecosystems | Legacy, niche needs |

**Default for Java microservices on AWS**: ECS Fargate. Move to EKS if you need Helm charts, operators, multi-cloud, or your team is already deep in K8s.

---

## 2.4 API Deployment Strategies

| Strategy | Risk | Speed |
|---|---|---|
| **All-at-once** | High (downtime, no rollback safety) | Fast |
| **Rolling** | Medium (some old + new during deploy) | Medium |
| **Blue/Green** | Low (instant switch, instant rollback) | Slow (double infra) |
| **Canary** | Lowest (5% → 25% → 100%) | Slowest |

ECS supports rolling natively; blue/green via **CodeDeploy**. API Gateway supports canary on stages.

---

## 2.5 SQS / SNS with Spring Boot

### SQS

Fully managed message queue. Two types:
- **Standard** — at-least-once, best-effort ordering, unlimited TPS
- **FIFO** — exactly-once, strict ordering, 300 TPS (3000 with batching)

Features: dead-letter queue (DLQ), visibility timeout, long polling, message retention (up to 14 days).

### SNS

Pub/sub. Topics with multiple subscribers (SQS, Lambda, HTTP, email, SMS). Fanout pattern: SNS → multiple SQS queues, each consumer processes independently.

### Spring Cloud AWS Setup

```xml
<dependency>
  <groupId>io.awspring.cloud</groupId>
  <artifactId>spring-cloud-aws-starter-sqs</artifactId>
  <version>3.1.1</version>
</dependency>
```

### Producer

```java
@Service
public class OrderEventPublisher {
    private final SqsTemplate sqsTemplate;
    private final SnsTemplate snsTemplate;

    public void publishOrderCreated(OrderCreatedEvent e) {
        // Direct to SQS
        sqsTemplate.send(to -> to.queue("order-events")
            .payload(e)
            .header("eventType", "OrderCreated"));

        // Or fan-out via SNS
        snsTemplate.convertAndSend("order-events-topic", e,
            Map.of("eventType", "OrderCreated"));
    }
}
```

### Consumer

```java
@Component
public class OrderEventConsumer {

    @SqsListener("order-events")
    public void handle(@Payload OrderCreatedEvent event,
                       @Header("MessageId") String messageId) {
        log.info("Processing order {}, msg {}", event.orderId(), messageId);
        // ... idempotent business logic
    }
}
```

### DLQ

Configure on the queue (CloudFormation/Terraform):
```hcl
resource "aws_sqs_queue" "main" {
  name = "order-events"
  visibility_timeout_seconds = 30
  redrive_policy = jsonencode({
    deadLetterTargetArn = aws_sqs_queue.dlq.arn
    maxReceiveCount     = 5
  })
}
```

---

## 2.6 AWS SDK for Java

Two versions:
- **v1** (`com.amazonaws:aws-java-sdk-*`) — deprecated as of Dec 2025
- **v2** (`software.amazon.awssdk:*`) — current, async/non-blocking, modular, smaller

Always use **v2** for new code. Key features: builder pattern, immutable models, async clients (`*AsyncClient`), Netty-based HTTP, pluggable HTTP client (Apache, URLConnection, CRT).

### Bill of Materials (BOM) — keeps versions aligned

```xml
<dependencyManagement>
  <dependencies>
    <dependency>
      <groupId>software.amazon.awssdk</groupId>
      <artifactId>bom</artifactId>
      <version>2.25.0</version>
      <type>pom</type>
      <scope>import</scope>
    </dependency>
  </dependencies>
</dependencyManagement>
```

### Sync vs Async

```java
// Sync
S3Client s3 = S3Client.create();
ResponseBytes<GetObjectResponse> resp = s3.getObjectAsBytes(...);

// Async
S3AsyncClient s3a = S3AsyncClient.create();
CompletableFuture<ResponseBytes<GetObjectResponse>> fut =
    s3a.getObject(req, AsyncResponseTransformer.toBytes());
fut.thenAccept(r -> log.info("ok"));
```

---

## 2.7 Async Processing

For decoupling, retry, traffic shaping.

Patterns:
- **Fire-and-forget**: producer → SQS → consumer. Producer doesn't wait.
- **Request-response**: producer → SQS (work queue) → consumer; consumer puts result in response queue or DDB; producer polls.
- **Pub/sub**: SNS → multiple SQS.
- **Saga**: choreography (events) or orchestration (Step Functions).

Spring Boot `@Async` for in-process tasks; SQS/SNS for inter-service.

---

## 2.8 Retry Mechanisms

### SDK retries

AWS SDK v2 retries automatically with exponential backoff + jitter. Configure:

```java
S3Client.builder()
    .overrideConfiguration(b -> b
        .retryStrategy(SdkDefaultRetryStrategy.standardRetryStrategy()
            .toBuilder().maxAttempts(5).build()))
    .build();
```

### App-level retries (Spring Retry)

```java
@Retryable(
    retryFor = TransientException.class,
    maxAttempts = 4,
    backoff = @Backoff(delay = 500, multiplier = 2, random = true))
public Order callPaymentService(String orderId) { ... }

@Recover
public Order recover(TransientException e, String orderId) {
    return Order.pendingPayment(orderId);
}
```

### Resilience4j

More features (circuit breaker, bulkhead, rate limiter, time limiter).

```java
@CircuitBreaker(name = "paymentSvc", fallbackMethod = "fallback")
@Retry(name = "paymentSvc")
@TimeLimiter(name = "paymentSvc")
public CompletableFuture<PaymentResp> charge(ChargeReq req) { ... }
```

---

## 2.9 Idempotency

Critical for at-least-once delivery (SQS, retries). Patterns:

### Idempotency key

Client sends `Idempotency-Key` header. Server stores result keyed by it; replays return same response.

```java
@PostMapping("/orders")
public ResponseEntity<Order> create(
        @RequestHeader("Idempotency-Key") String key,
        @RequestBody OrderRequest req) {

    Optional<Order> existing = idempotencyStore.get(key);
    if (existing.isPresent()) return ResponseEntity.ok(existing.get());

    Order order = orderService.create(req);
    idempotencyStore.save(key, order, Duration.ofHours(24));
    return ResponseEntity.status(201).body(order);
}
```

Store: DynamoDB with TTL is ideal.

### Powertools Idempotency (works in Spring Boot too)

```java
@Idempotent
public OrderResponse processOrder(@IdempotencyKey String key, OrderRequest req) {
    ...
}
```

Stores hashed payload in DynamoDB; concurrent identical calls deduplicated.

---

## 2.10 Distributed Tracing

**AWS X-Ray** (or OpenTelemetry → X-Ray / Jaeger / Datadog).

```xml
<dependency>
  <groupId>com.amazonaws</groupId>
  <artifactId>aws-xray-recorder-sdk-spring</artifactId>
</dependency>
```

```java
@Bean
public Filter tracingFilter() {
    return new AWSXRayServletFilter("orders-service");
}

@Bean
public AWSXRayRecorderBuilder xrayBuilder() {
    return AWSXRayRecorderBuilder.standard()
        .withPlugin(new ECSPlugin())
        .withSegmentListener(new SLF4JSegmentListener());
}
```

Propagate `X-Amzn-Trace-Id` header across services (OpenFeign interceptor / WebClient filter).

---

## 2.11 OpenFeign with AWS

OpenFeign for declarative HTTP clients.

```java
@FeignClient(name = "payments",
             url = "${app.payments.base-url}",
             configuration = AwsSigV4FeignConfig.class)
public interface PaymentsClient {
    @PostMapping("/charges")
    ChargeResponse charge(ChargeRequest req);
}
```

If calling a private API Gateway with IAM auth, add a SigV4 signing interceptor:

```java
public class AwsSigV4RequestInterceptor implements RequestInterceptor {
    private final Aws4Signer signer = Aws4Signer.create();
    private final AwsCredentialsProvider creds = DefaultCredentialsProvider.create();

    @Override
    public void apply(RequestTemplate tpl) {
        // Build SdkHttpFullRequest from tpl, sign with signer, copy headers back to tpl
        // (full impl ~30 lines — keep handy as a snippet)
    }
}
```

Add Resilience4j on Feign for circuit breaker + retry.

---

## 2.12 Secrets Manager Integration

Stores secrets with automatic rotation (RDS, Aurora native rotation). Encrypted with KMS, audited via CloudTrail.

### Spring Cloud AWS

```xml
<dependency>
  <groupId>io.awspring.cloud</groupId>
  <artifactId>spring-cloud-aws-starter-secrets-manager</artifactId>
</dependency>
```

```yaml
spring:
  config:
    import: aws-secretsmanager:/prod/orders/db,/prod/orders/api-keys
```

Spring loads secrets into `Environment` at startup. Reference like any property: `${db.password}`.

### Manual access

```java
SecretsManagerClient sm = SecretsManagerClient.create();
String secretJson = sm.getSecretValue(r -> r.secretId("prod/orders/db"))
        .secretString();
DbCreds creds = mapper.readValue(secretJson, DbCreds.class);
```

### Rotation

Lambda-based; AWS provides templates for RDS, Aurora, DocumentDB. For others, write your own with `createSecret → setSecret → testSecret → finishSecret` steps.

---

## 2.13 Parameter Store (SSM Parameter Store)

Like Secrets Manager but for config + light secrets. Cheaper. No automatic rotation. Hierarchical paths (`/prod/orders/feature-x`).

| | Secrets Manager | Parameter Store |
|---|---|---|
| Auto-rotation | Yes | No |
| Cost | $0.40/secret + API calls | Free (standard) / $0.05/advanced |
| Versioning | Yes | Yes |
| KMS encryption | Mandatory | Optional (SecureString) |
| Max value size | 64 KB | 4 KB (standard), 8 KB (advanced) |
| Cross-account | Yes | Yes |

```yaml
spring:
  config:
    import: aws-parameterstore:/prod/orders/
```

Now `prod.orders.featureX` → `${featureX}`.

---

## 2.14 Production-Ready Architecture Example

**Use case**: E-commerce order processing — submission → inventory check → payment → fulfillment → notification

```
Customer
   │
   ▼
Route 53 → CloudFront → API GW HTTP API (JWT via Cognito)
   │
   ▼
ALB ──► Orders Service (ECS Fargate, ASG 4-30, Java 21 SB 3.x)
            │
            ├── Aurora PostgreSQL (Multi-AZ + 2 readers via RDS Proxy)
            ├── ElastiCache Redis (session, hot cart cache)
            └── EventBridge ──► "order.created" event
                    │
                    ├──► SQS-payments  ──► Payments Service (ECS) ──► Stripe
                    ├──► SQS-inventory ──► Inventory Service (ECS) ──► DynamoDB
                    └──► SQS-notify    ──► Notify Lambda ──► SES/SNS

Observability: X-Ray, CloudWatch (EMF metrics, structured logs)
Secrets: Secrets Manager (DB), Parameter Store (config)
CI/CD: GitHub Actions ──► ECR ──► CodeDeploy (blue/green on ECS)
Security: WAF on CloudFront, SG layered, IAM least privilege, KMS for at-rest
DR: cross-region Aurora Global, S3 CRR, DynamoDB Global Tables
```

---

## 2.15 Java + AWS Interview Q&A (Cross-cutting)

**Q1. (Intermediate) How do you give your Spring Boot app credentials in production?**
> Use IAM **task role** (ECS) or **instance role** (EC2). AWS SDK auto-discovers via `DefaultCredentialsProvider`. No keys in code or env vars.

**Q2. (Intermediate) Your service reads a config value from Parameter Store on every request — what's wrong?**
> Latency + cost. Cache it. Use Spring Cloud AWS auto-config (loads at startup) or wrap SSM client with Caffeine + TTL. Subscribe to Parameter Store change events for invalidation if values change.

**Q3. (Advanced) Two services need to call each other; how do you secure that?**
> Options:
> 1. **Internal ALB** + SG so only `service-a-sg` can talk to `service-b-sg`. Layer JWT (issued by Cognito) on top.
> 2. **VPC Lattice** or **App Mesh** with mTLS.
> 3. **PrivateLink** if cross-account.
> Avoid: public endpoints with API keys.

**Q4. (Architect) Your SQS consumer is processing slowly and visibility timeout is expiring, causing reprocessing.**
> Multiple fixes:
> - Extend visibility timeout dynamically (`ChangeMessageVisibility`) while long jobs run
> - Split job into smaller messages
> - Scale consumer fleet (more workers / threads)
> - Make handler idempotent so duplicates don't break correctness
> - Move long-running work to Step Functions

---

# PART 3 — MICROSERVICES ARCHITECTURE

---

## 3.1 Service Discovery

How services find each other.

| Approach | How | Pros / cons |
|---|---|---|
| **Internal ALB / NLB** | DNS to LB → tasks | Simple, mature; SPOF per LB; cost |
| **AWS Cloud Map** | Service registry; DNS or API | Native, integrates ECS, free |
| **ECS Service Connect** | Sidecar (Envoy), service mesh-lite | mTLS, retries, observability built-in |
| **Eureka (Spring Cloud Netflix)** | App-level | Portable; you operate it |
| **Consul** | HashiCorp | Multi-cloud; you operate it |

Default: **ECS Service Connect** or **Cloud Map + ALB**.

---

## 3.2 API Gateway Pattern

A single entry point for clients; routes to microservices, handles cross-cutting concerns (auth, rate limiting, logging, transformation).

On AWS: API Gateway, ALB, or self-built (Spring Cloud Gateway). Sometimes layer them: CloudFront → API Gateway → internal ALB.

**Backend for Frontend (BFF)**: separate gateway per client type (mobile, web, partner) — each shapes responses for that client.

---

## 3.3 Circuit Breaker — Resilience4j

Prevents cascading failures.

States: **Closed** (normal) → **Open** (fail fast on error threshold) → **Half-Open** (test recovery).

```yaml
resilience4j:
  circuitbreaker:
    instances:
      payments:
        sliding-window-size: 50
        minimum-number-of-calls: 20
        failure-rate-threshold: 50
        wait-duration-in-open-state: 30s
        permitted-number-of-calls-in-half-open-state: 5
        automatic-transition-from-open-to-half-open-enabled: true
  retry:
    instances:
      payments:
        max-attempts: 3
        wait-duration: 500ms
        exponential-backoff-multiplier: 2
```

```java
@CircuitBreaker(name = "payments", fallbackMethod = "fallbackCharge")
@Retry(name = "payments")
@TimeLimiter(name = "payments")
public CompletableFuture<PaymentResponse> charge(PaymentRequest req) {
    return paymentsClient.chargeAsync(req);
}

private CompletableFuture<PaymentResponse> fallbackCharge(
        PaymentRequest req, Throwable t) {
    log.warn("Payments unavailable, queueing", t);
    sqsTemplate.send("payment-retry", req);
    return CompletableFuture.completedFuture(PaymentResponse.queued());
}
```

---

## 3.4 Distributed Transactions

ACID across services is hard. Two patterns:

### Two-Phase Commit (2PC)
Avoid. Blocks, doesn't scale, requires distributed coordinator. Not for cloud microservices.

### Saga
Long-running transaction as a sequence of local transactions, with compensating actions on failure.

**Choreography** — services react to events:
```
OrderCreated → Inventory.reserve → InventoryReserved
                                         ↓
                                  Payment.charge → PaymentCompleted
                                                          ↓
                                                  Order.confirm
On failure → emit "OrderFailed" → each prior step compensates
```

**Orchestration** — a central orchestrator (Step Functions) calls each service and handles failures.

---

## 3.5 Step Functions for Orchestration

AWS-managed workflow service. Define state machines in JSON/YAML. Built-in error handling, retries, parallelism, human approval.

```json
{
  "StartAt": "ReserveInventory",
  "States": {
    "ReserveInventory": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:...:function:ReserveInventory",
      "Retry": [{ "ErrorEquals": ["States.ALL"], "MaxAttempts": 3 }],
      "Catch": [{ "ErrorEquals": ["States.ALL"], "Next": "Compensate" }],
      "Next": "ChargePayment"
    },
    "ChargePayment": {
      "Type": "Task",
      "Resource": "arn:aws:states:::lambda:invoke",
      "Parameters": {
        "FunctionName": "ChargePayment",
        "Payload.$": "$"
      },
      "Catch": [{ "ErrorEquals": ["States.ALL"], "Next": "ReleaseInventory" }],
      "Next": "ConfirmOrder"
    },
    "ConfirmOrder": { "Type": "Task", "Resource": "...", "End": true },
    "ReleaseInventory": { "Type": "Task", "Resource": "...", "Next": "Compensate" },
    "Compensate": { "Type": "Fail" }
  }
}
```

Standard workflows: long-running (up to 1 year), exactly-once. Express workflows: high-volume short tasks, at-least-once.

---

## 3.6 Event-Driven Design

Services react to events, decoupled in time and identity.

Building blocks on AWS:
- **EventBridge** — event bus, schema registry, content-based routing, partner integrations (Stripe, Shopify)
- **SNS** — pub/sub, fan-out
- **SQS** — point-to-point, buffering
- **Kinesis** — high-volume streaming, ordered, replayable
- **MSK** (Managed Kafka) — Kafka-compatible

Event types: domain events (`OrderPlaced`), integration events (`InventoryReserved`), notifications (`EmailRequired`).

---

## 3.7 CQRS Basics

**Command Query Responsibility Segregation** — separate read and write models.

- Writes → command model (normalized, transactional, e.g., Aurora)
- Reads → query model (denormalized, fast, e.g., OpenSearch, DynamoDB)
- Sync via events (write emits event, read model subscribes)

When to use: very different read/write patterns, reads dominate, complex reporting. Don't use everywhere — adds complexity.

---

## 3.8 Kafka vs SQS

| | Kafka (MSK / self-hosted) | SQS |
|---|---|---|
| Model | Distributed log | Queue |
| Ordering | Per partition | FIFO queue (300 TPS) |
| Replay | Yes (retention configurable) | No (once consumed, gone) |
| Throughput | Millions/sec | Effectively unlimited |
| Consumer model | Pull, consumer groups | Pull, competing consumers |
| Delivery | At-least-once / exactly-once | At-least-once / FIFO exactly-once |
| Ops cost | High (even MSK) | Zero (fully managed) |
| Best for | Event streaming, CDC, analytics pipeline | Decoupling services, work queues |

**Rule of thumb**: SQS for "do this work once"; Kafka for "stream of events many consumers replay".

---

## 3.9 Database Per Service

Each microservice owns its DB; others access via API/events, never direct DB.

Benefits: independent scaling, schema evolution, polyglot persistence (use Aurora for orders, DynamoDB for sessions, OpenSearch for search).

Challenges: cross-service queries (CQRS read model), distributed transactions (Saga), data sync.

---

## 3.10 Centralized Logging

Aggregate logs from all services in one place.

Options:
- **CloudWatch Logs + Insights**
- **OpenSearch (managed)** — Elasticsearch-compatible, Kibana UI
- **Third-party**: Datadog, New Relic, Splunk

Pipeline: app → stdout → log driver (awslogs / firelens) → CloudWatch / Firehose → S3 / OpenSearch.

Structured JSON, correlation IDs, log levels by environment, retention policies, sensitive-data redaction.

---

## 3.11 Microservices Interview Q&A

**Q1. (Intermediate) Choreography vs orchestration?**
> Choreography: services react to events independently. No central coordinator. Loose coupling but harder to reason about / monitor. Orchestration: a central state machine (Step Functions) calls services. Easier to visualize, debug, and modify, but introduces a coordinator dependency.

**Q2. (Advanced) How do you handle a service failing mid-saga?**
> Each local transaction has a compensating transaction. On failure, the orchestrator (or each prior service via event) executes compensations in reverse order. Important: compensations should be idempotent because saga may retry.

**Q3. (Architect) When would you choose Kafka over SQS?**
> Kafka if you need: log retention (replay), high throughput per partition with ordering, multiple consumer groups reading independently, stream processing (KStreams/Flink), CDC pipelines. SQS for simple decoupling, low ops overhead.

---

# PART 4 — DEVOPS + CI/CD

---

## 4.1 Jenkins Pipeline (Declarative)

```groovy
pipeline {
  agent any
  tools { maven 'mvn-3.9'; jdk 'jdk-21' }
  environment {
    AWS_REGION = 'ap-south-1'
    ECR_REPO = '111111111111.dkr.ecr.ap-south-1.amazonaws.com/orders'
  }
  stages {
    stage('Checkout') { steps { checkout scm } }

    stage('Build & Test') {
      steps {
        sh 'mvn -B clean verify'
        junit 'target/surefire-reports/*.xml'
        jacoco execPattern: 'target/jacoco.exec'
      }
    }

    stage('SonarQube') { steps { withSonarQubeEnv('sq') { sh 'mvn sonar:sonar' } } }

    stage('Docker Build & Push') {
      steps {
        sh '''
          aws ecr get-login-password --region $AWS_REGION \
            | docker login --username AWS --password-stdin $ECR_REPO
          docker build -t $ECR_REPO:$GIT_COMMIT -t $ECR_REPO:latest .
          docker push $ECR_REPO:$GIT_COMMIT
          docker push $ECR_REPO:latest
        '''
      }
    }

    stage('Deploy to ECS') {
      steps {
        sh '''
          aws ecs update-service \
            --cluster prod-cluster --service orders \
            --force-new-deployment --region $AWS_REGION
          aws ecs wait services-stable \
            --cluster prod-cluster --services orders
        '''
      }
    }
  }
  post {
    failure { slackSend channel: '#alerts', message: "Pipeline failed: ${env.BUILD_URL}" }
  }
}
```

Use **OIDC** between Jenkins and AWS for short-lived creds (no static keys).

---

## 4.2 GitHub Actions

```yaml
name: Deploy Orders Service
on:
  push: { branches: [main] }

permissions:
  id-token: write   # for OIDC
  contents: read

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with: { java-version: 21, distribution: temurin, cache: maven }
      - run: mvn -B verify

      - name: Configure AWS Credentials (OIDC)
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::111111111111:role/GHA-Deploy
          aws-region: ap-south-1

      - uses: aws-actions/amazon-ecr-login@v2
        id: ecr

      - name: Build & push
        env:
          REPO: ${{ steps.ecr.outputs.registry }}/orders
          TAG: ${{ github.sha }}
        run: |
          docker build -t $REPO:$TAG -t $REPO:latest .
          docker push $REPO:$TAG
          docker push $REPO:latest

      - name: Update ECS
        run: |
          aws ecs update-service --cluster prod-cluster \
            --service orders --force-new-deployment
```

**OIDC is the secret to safe CI/CD on AWS** — no long-lived keys ever.

---

## 4.3 Dockerization Best Practices

- **Multi-stage builds** — small final image
- **JRE not JDK** at runtime
- **Non-root user** (`USER 1000`)
- **Layered jars** (Spring Boot 2.3+) — fast rebuilds when code changes but deps don't
- **Health check** in Dockerfile or task definition
- **`-XX:MaxRAMPercentage=75 -XX:+UseContainerSupport`** for container-aware JVM
- Scan images: **ECR scanning on push** (free basic, advanced via Inspector)

---

## 4.4 ECS Deployment

### Deployment controllers
- **ECS (rolling)** — default. `minimumHealthyPercent=100`, `maximumPercent=200` for zero-downtime.
- **CODE_DEPLOY (blue/green)** — two target groups, instant cutover.
- **EXTERNAL** — your own controller.

### Service auto scaling
Target tracking on CPU/memory/ALBRequestCountPerTarget. Step scaling for finer control.

---

## 4.5 Deployment Strategies

### Blue/Green
Run two environments. Switch traffic atomically. Easy rollback. Requires double capacity briefly.

AWS: **CodeDeploy** with ECS or ALB target group swap.

### Rolling
Replace instances/tasks gradually. Cheap (no double capacity). Slower rollback.

ECS native rolling deploy.

### Canary
Send 5% traffic to new version → monitor → 25% → 100%. Catches problems with limited blast radius.

AWS: API Gateway stage canary, ALB weighted target groups, App Mesh, or CodeDeploy linear/canary configurations.

---

## 4.6 CodeDeploy & CodePipeline

### CodePipeline
Orchestrates stages: Source (CodeCommit/GitHub/S3) → Build (CodeBuild) → Test → Deploy (CodeDeploy/ECS/CloudFormation/Lambda).

### CodeDeploy
Handles deployment to EC2, on-prem, Lambda, ECS. Hooks: `BeforeInstall`, `AfterAllowTraffic`. Auto-rollback on alarms.

`appspec.yml` for ECS blue/green:
```yaml
version: 0.0
Resources:
  - TargetService:
      Type: AWS::ECS::Service
      Properties:
        TaskDefinition: arn:aws:ecs:...:task-definition/orders:42
        LoadBalancerInfo:
          ContainerName: orders
          ContainerPort: 8080
Hooks:
  - BeforeAllowTraffic: arn:aws:lambda:...:function:smoke-tests
  - AfterAllowTraffic:  arn:aws:lambda:...:function:post-deploy-checks
```

---

## 4.7 Kubernetes Basics on AWS (EKS)

- **EKS** — managed control plane ($0.10/hr). You manage worker nodes (EC2/Fargate).
- **Fargate profiles** — serverless pods (per-pod billing).
- **Karpenter** — modern node autoscaler (replaces Cluster Autoscaler).
- **IRSA** — IAM Roles for Service Accounts, pod-level AWS access.
- **AWS Load Balancer Controller** — provisions ALB/NLB from Ingress/Service.
- **EBS CSI / EFS CSI** — persistent volumes.

Spring Boot on EKS:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata: { name: orders }
spec:
  replicas: 3
  selector: { matchLabels: { app: orders } }
  template:
    metadata: { labels: { app: orders } }
    spec:
      serviceAccountName: orders-sa       # IRSA
      containers:
        - name: orders
          image: 111.dkr.ecr.ap-south-1.amazonaws.com/orders:1.4.2
          ports: [{ containerPort: 8080 }]
          resources:
            requests: { cpu: 250m, memory: 512Mi }
            limits:   { cpu: 1000m, memory: 1Gi }
          readinessProbe: { httpGet: { path: /actuator/health/readiness, port: 8080 }, initialDelaySeconds: 30 }
          livenessProbe:  { httpGet: { path: /actuator/health/liveness,  port: 8080 }, initialDelaySeconds: 60 }
          env:
            - name: SPRING_PROFILES_ACTIVE
              value: prod
```

---

## 4.8 DevOps Interview Q&A

**Q1. (Intermediate) Blue/green vs canary?**
> Blue/green: instant 0%→100% switch between two identical environments. Fast rollback (switch back). Costly (double infra). Canary: gradual traffic shift (5→25→100%) over time to limit blast radius if something is wrong. Catches latent issues but slower to fully roll out.

**Q2. (Advanced) How do you secure CI/CD pipelines?**
> OIDC between CI and AWS (no static keys). Separate deploy roles per env with least privilege. Signed commits, branch protection, code review. Scan images (ECR scanning, Trivy). Scan IaC (cfn-nag, tfsec, Checkov). Secrets via Secrets Manager, never in repo. Audit via CloudTrail. Pre-prod approval gates for prod.

**Q3. (Architect) Design a pipeline supporting 50+ microservices across dev/stage/prod.**
> Monorepo or polyrepo + shared workflow templates (GitHub Actions reusable workflows or Jenkins shared libraries). Build matrix: each PR runs unit + lint + scan; merge to main → dev deploy. Promotion via tagged releases. ArgoCD or CodePipeline for env promotion. Shared Terraform modules. Trunk-based dev. Observability dashboards per service.

---

# PART 5 — SECURITY

---

## 5.1 Encryption at Rest & In Transit

### At Rest
- **S3**: SSE-S3 (default), SSE-KMS, SSE-C, client-side
- **EBS**: AES-256 with KMS keys (enable by default at account level)
- **RDS / Aurora**: KMS encryption (must enable at creation)
- **DynamoDB**: KMS-encrypted by default
- **Lambda**: env vars encrypted at rest with KMS

### In Transit
- TLS 1.2+ everywhere
- ACM (AWS Certificate Manager) for free public certs
- ACM Private CA for internal mTLS
- Enforce HTTPS via bucket policy, ALB listener, CloudFront viewer protocol

---

## 5.2 KMS (Key Management Service)

Centralized managed encryption keys.

Key types:
- **AWS-managed keys** (`aws/s3`, `aws/rds`) — free, no control
- **Customer-managed keys (CMK)** — full control: policies, rotation, deletion
- **Customer-provided keys (XKS)** — external HSM

Patterns:
- **Envelope encryption** — KMS encrypts a data key; data key encrypts data
- **Key policies + IAM** — both must allow access (key policy is primary)
- **Automatic rotation** — yearly for CMKs
- **Multi-region keys** — same key material in multiple regions

```java
KmsClient kms = KmsClient.create();
// Generate data key
GenerateDataKeyResponse dk = kms.generateDataKey(r -> r
    .keyId("alias/orders-key").keySpec(DataKeySpec.AES_256));
SecretKeySpec secret = new SecretKeySpec(dk.plaintext().asByteArray(), "AES");
// Encrypt data with secret (AES/GCM), store dk.ciphertextBlob alongside
// To decrypt: kms.decrypt(ciphertextBlob) → plaintext key → decrypt data
```

---

## 5.3 Secrets Manager

(Covered in Part 2.12.) Key points for interview:
- Auto-rotation via Lambda
- Audit through CloudTrail
- Integrated with RDS/Aurora native rotation
- Resource policies for cross-account
- Replicate to other regions for DR

---

## 5.4 WAF (Web Application Firewall)

Layer 7 firewall. Attaches to CloudFront, ALB, API Gateway, App Runner, AppSync.

Rule types:
- **Managed rule groups** — AWS-managed (Core, Bot Control, IP reputation), Marketplace
- **Custom rules** — IP allow/deny, geo block, size limits, regex
- **Rate-based rules** — block IPs exceeding threshold (DDoS-lite)
- **CAPTCHA / Challenge** actions

---

## 5.5 Shield

- **Shield Standard** — automatic, free, against common L3/L4 DDoS
- **Shield Advanced** — $3000/month, app-layer DDoS, DRT support, cost protection during attack, WAF included

---

## 5.6 IAM Best Practices

1. Lock down root, enable MFA
2. Use Identity Center / SSO for humans
3. Roles, not users
4. Least privilege; iterate with Access Analyzer
5. Permission boundaries on developer-created roles
6. SCPs for org-wide guardrails
7. Rotate access keys (or eliminate)
8. CloudTrail enabled in all accounts → central security account
9. Tag-based access control (ABAC)
10. Detect privilege escalation paths regularly

---

## 5.7 Token-Based Authentication

JWT (JSON Web Token) most common.

Structure: `header.payload.signature` (base64url).

Verify: signature using issuer's public key (JWKS endpoint), check `iss`, `aud`, `exp`, `nbf`.

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {
    @Bean
    public SecurityFilterChain chain(HttpSecurity http) throws Exception {
        http
          .authorizeHttpRequests(a -> a
              .requestMatchers("/actuator/health").permitAll()
              .requestMatchers("/api/admin/**").hasAuthority("SCOPE_admin")
              .anyRequest().authenticated())
          .oauth2ResourceServer(o -> o.jwt(jwt -> jwt
              .jwkSetUri("https://cognito-idp.ap-south-1.amazonaws.com/<userPoolId>/.well-known/jwks.json")));
        return http.build();
    }
}
```

---

## 5.8 OAuth2 / OIDC

- **OAuth2** — authorization framework (access to resources)
- **OIDC** — authentication layer on top of OAuth2 (who the user is)
- AWS provider: **Cognito User Pools** (OIDC IdP) + **Cognito Identity Pools** (federate to AWS creds)
- Alt: Auth0, Okta, Keycloak

Flows: Authorization Code + PKCE (web/mobile), Client Credentials (machine-to-machine).

---

## 5.9 Secure API Design Checklist

- [ ] TLS 1.2+ enforced
- [ ] AuthN: JWT/OAuth2/mTLS
- [ ] AuthZ: scopes/roles per endpoint
- [ ] Input validation (Bean Validation `@Valid`)
- [ ] Output encoding (XSS prevention)
- [ ] Parameterized queries (no SQL/NoSQL injection)
- [ ] Rate limiting per identity (not just IP)
- [ ] WAF rules (OWASP Top 10)
- [ ] CORS configured strictly
- [ ] Security headers (CSP, HSTS, X-Content-Type-Options)
- [ ] Secrets in Secrets Manager
- [ ] Structured audit logs, no PII
- [ ] Versioning (`/v1/`) for backward compatibility
- [ ] Pagination + size limits (no unbounded responses)
- [ ] Idempotency keys for mutations

---

## 5.10 Security Interview Q&A

**Q1. (Beginner) Difference between Security Group and WAF?**
> SG is L3/L4 (IP/port) — network-level filtering. WAF is L7 (HTTP) — inspects request content (SQLi, XSS, bot signatures, geo). Layered defense.

**Q2. (Intermediate) Where do you store DB passwords?**
> Secrets Manager. App fetches at startup via task role permissions. Auto-rotation enabled. Never in code, env files in repo, or container images.

**Q3. (Advanced) How would you protect against credential leakage if a developer's laptop is compromised?**
> No long-lived keys on laptops — use AWS SSO with short-lived role sessions. MFA mandatory. CloudTrail alerts on `GetSessionToken` from unusual IPs. Detect leaked keys via GitGuardian / Trufflehog. Permission boundaries limit blast radius. Rotate root + service roles after suspected compromise.

**Q4. (Architect) Design end-to-end security for a fintech API.**
> Network: VPC with no public subnets for app/db; CloudFront + WAF + Shield Advanced; private API GW. Identity: Cognito + MFA; JWT with short TTL + refresh; mTLS for B2B. Encryption: KMS CMKs with rotation; TLS 1.3; field-level encryption for PII. Access: least privilege IAM; permission boundaries; SCPs; Access Analyzer. Audit: CloudTrail (org trail) + Config + GuardDuty + Security Hub; central security account. Compliance: PCI DSS / SOC 2 controls; quarterly access reviews; pen tests.

---

# PART 6 — SCENARIO-BASED QUESTIONS

These are the high-leverage questions that typically tip the interview. Practice telling each as a 2-3 minute story with **decisions + tradeoffs**.

---

## 6.1 How would you design scalable microservices on AWS?

**Answer template (STAR-style)**:

**Components:**
- **Compute**: ECS Fargate (or EKS) — auto-scaling tasks across 3 AZs
- **Edge**: CloudFront + WAF + Route 53 with health-check failover
- **API layer**: API Gateway HTTP API (JWT auth) or internal ALB
- **Data**:
  - Aurora PostgreSQL (Multi-AZ + read replicas) for transactional
  - DynamoDB for key-value at scale
  - ElastiCache Redis for hot data, session
  - S3 for blobs + data lake
  - OpenSearch for search
- **Async**: EventBridge (event router) + SQS (work queues) + Step Functions (orchestration)
- **Config/secrets**: Parameter Store + Secrets Manager
- **Observability**: CloudWatch metrics+logs, X-Ray traces, EMF custom metrics
- **CI/CD**: GitHub Actions → ECR → CodeDeploy blue/green
- **Security**: IAM roles, KMS, WAF, Shield, GuardDuty, Security Hub
- **DR**: cross-region replicas (Aurora Global, DynamoDB Global, S3 CRR)

**Principles**:
- One DB per service
- Stateless services; state in caches/DBs
- Async between services wherever possible (event-driven)
- Idempotent handlers
- Circuit breakers + retries with jitter
- Auto-scale on RED metrics (rate, errors, duration)
- IaC for everything

---

## 6.2 How do you handle sudden traffic spikes?

1. **At the edge**: CloudFront absorbs static + cacheable; API Gateway/ALB scales transparently.
2. **Auto-scaling**: ECS/EKS scale on target tracking (CPU, ALBRequestCountPerTarget). Use **predictive scaling** + warm pools.
3. **Buffer**: insert SQS in front of any slow consumer — spike soaks into queue; consumer drains at its pace.
4. **Burstable resources**: provisioned concurrency on Lambda; DynamoDB on-demand; Aurora Serverless v2.
5. **Cache aggressively**: ElastiCache for hot keys; CloudFront for HTTP responses; DAX for DynamoDB.
6. **Rate limit / shed load**: WAF rate-based rules and per-tenant API key throttling protect the system.
7. **Async non-critical work**: notifications, analytics → background queues so user-facing latency unaffected.

**Real example**: "During product launch, RPS jumped from 2K to 40K in 90 seconds. We had pre-warmed the ALB, set ECS scale-out cooldown to 30s, predictive scaling on, and SQS in front of the inventory service. p99 latency stayed under 800ms."

---

## 6.3 How do you troubleshoot high latency?

Methodical traversal:

1. **Where is the latency?** Use X-Ray service map / Datadog APM → which service / span.
2. **Network**: ALB target response time vs request processing time. VPC Flow Logs for packet drops.
3. **App**: Thread dumps for blocked threads, GC logs for long pauses, CPU profiler for hot methods.
4. **DB**: Performance Insights for slow queries, lock waits. Replica lag.
5. **Downstream**: external API latency. Add circuit breaker if it's flaky.
6. **Cold starts**: Lambda — enable SnapStart or provisioned concurrency.
7. **Connection pool exhaustion**: Hikari metrics show waiters > 0.
8. **Cross-AZ chattiness**: services calling each other across AZs (data transfer + latency).

---

## 6.4 How do you secure APIs?

(See Part 5.9 checklist.)

Top 5 to mention:
1. TLS + JWT (short TTL)
2. WAF with managed rule groups + rate limiting
3. Input validation + parameterized queries
4. Secrets in Secrets Manager
5. CloudTrail + GuardDuty + alerting on anomalies

---

## 6.5 How do you reduce AWS cost?

Cost optimization is in 4 buckets:

### Right-size
- CloudWatch + Compute Optimizer → flag oversized EC2/RDS/EBS
- Move EBS gp2 → gp3 (~20% cheaper)
- Graviton (ARM) for 20-40% saving where compatible

### Buy smarter
- Savings Plans / Reserved Instances for steady baseline
- Spot for stateless batch / CI workers
- Spot Fleet diversified across instance types and AZs

### Architectural
- S3 lifecycle: Standard → IA → Glacier; Intelligent-Tiering for unknown patterns
- Lambda for low-traffic services
- DynamoDB on-demand for spiky; provisioned + auto-scale for steady
- VPC Gateway Endpoints for S3/DDB to skip NAT costs
- CloudFront caching to reduce origin requests
- Delete unused EBS snapshots, old AMIs, idle ELBs
- Right-size RDS, use Aurora Serverless v2 for variable load
- Compress + tier logs (set retention!)

### Visibility
- **Cost Explorer**, **Cost Anomaly Detection**, **AWS Budgets** with alerts
- Tag everything (Environment, Team, CostCenter) → cost allocation
- **CUR (Cost & Usage Report)** in S3 → Athena / QuickSight for deep analysis

---

## 6.6 How do you migrate a monolith to microservices?

**Strangler Fig pattern**:

1. **Discover & document** existing monolith — endpoints, DB tables, dependencies, transactions
2. **Identify seams** — bounded contexts (DDD), high-change vs stable, high-risk vs low-risk
3. **Build a routing layer** (API Gateway / Spring Cloud Gateway) in front of monolith
4. **Extract one service at a time**:
   - Carve out feature → new microservice (its own DB)
   - Route layer redirects to new service for that feature
   - Old code becomes dead, removed
5. **Data**: dual-write during transition, then cutover. Or CDC (DMS) to sync.
6. **Repeat** until monolith is hollow / gone
7. **Observability** from day 1 (X-Ray, structured logs, dashboards)

**Avoid**: Big bang rewrite. Microservices without ops maturity (you'll multiply pain).

---

## 6.7 What happens if one availability zone fails?

If your architecture is multi-AZ:
- ALB stops routing to that AZ
- ASG launches replacements in healthy AZs
- RDS Multi-AZ fails over to standby (~60s)
- ElastiCache replicas promote
- ECS tasks reschedule
- Aurora promotes a reader in a healthy AZ
- DynamoDB / S3 / Lambda / SQS unaffected (multi-AZ by default)

What you need to verify:
- NAT GW per AZ (else cross-AZ failover storms NAT costs)
- ASG min capacity per AZ, not just total
- App handles connection retries gracefully
- DR runbook tested with **Fault Injection Service (FIS)** for chaos engineering

---

## 6.8 How would you process millions of events?

**Architecture**: Kinesis Data Streams (ordered, partitioned, replayable) or Kafka (MSK).

```
Producers (1000s) ──► Kinesis Data Streams (N shards)
                            │
                ┌───────────┼───────────┐
                ▼           ▼           ▼
        KCL App / Lambda  Firehose   Analytics
        (real-time proc)    │       (Flink/Spark)
                            ▼
                    S3 (raw) → Glue → Athena/Redshift
```

Considerations:
- **Shards**: 1 MB/s in, 2 MB/s out per shard. Calculate based on throughput.
- **Partition key**: well-distributed to avoid hot shards
- **Idempotent consumers**: at-least-once delivery
- **Checkpointing**: KCL handles for you
- **Backpressure**: enhanced fan-out for multiple consumers without throttling
- **Replay**: 24h default retention (extendable to 365 days)

For < 10K msg/s with simple decoupling: SQS is enough.

---

## 6.9 How would you design a highly available architecture?

(Combines previous answers.)

**Tiered HA**:
- **Region**: pick one primary; DR in secondary
- **AZs**: ≥3 for all stateful + stateless
- **Stateless**: ASG/ECS across AZs, ALB
- **Stateful**: Multi-AZ DBs, read replicas, cache replicas
- **DNS**: Route 53 with health checks + failover routing
- **DR strategy** (RTO/RPO):
  - **Backup & restore** (hours)
  - **Pilot light** (minutes)
  - **Warm standby** (seconds–minutes)
  - **Multi-region active-active** (zero RTO)
- **Test**: GameDays + AWS Fault Injection Service

---

# PART 7 — TROUBLESHOOTING

---

## 7.1 Debugging Production Issues — Method

1. **Confirm the symptom** — what's actually failing? Reproducible?
2. **Identify scope** — one user, one service, one AZ, all?
3. **Recent changes** — deploys, config, infra (use CodeDeploy/CloudTrail history)
4. **Check dashboards** — RED metrics, error rates, latency by service
5. **Drill down** — X-Ray to find slowest span; logs to find error messages
6. **Hypothesis → test** — minimum change to test theory
7. **Fix forward or rollback** — bias to fastest mitigation
8. **Post-incident review** — root cause, prevention

---

## 7.2 CloudWatch Logs Insights — Killer Queries

```
fields @timestamp, @message, level, traceId, latencyMs
| filter level = "ERROR"
| stats count() by errorType
| sort count desc
```

Latency percentiles:
```
filter @message like /HTTP/
| parse @message /latency=(?<latency>\d+)/
| stats avg(latency), pct(latency, 50), pct(latency, 95), pct(latency, 99) by bin(5m)
```

Find the slowest endpoints:
```
fields @timestamp, path, latencyMs
| filter @message like /HTTP_REQUEST/
| stats avg(latencyMs) as avgL, pct(latencyMs, 99) as p99 by path
| sort p99 desc
| limit 10
```

---

## 7.3 Memory Leaks (Java)

Symptoms: heap grows, GC pauses lengthen, OOM eventually.

Tools:
- `-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/dumps`
- `jcmd <pid> GC.heap_dump /tmp/heap.hprof`
- **Eclipse MAT** for analysis
- JFR (Java Flight Recorder) for in-flight diagnosis

Common Java/Spring causes:
- Static caches without eviction
- ThreadLocal holding references
- ClassLoader leaks after hot reload
- HTTP client not closed
- Listener / observer not deregistered
- Hibernate L2 cache misconfigured
- Logging dumping huge objects

Fix: identify dominators, bounded caches (Caffeine with `maximumSize` + TTL), close resources (`try-with-resources`).

---

## 7.4 CPU Spikes

Tools:
- `top` / `htop` on host
- **CloudWatch Container Insights** for ECS/EKS
- **Async-profiler** flame graphs (`./profiler.sh -d 60 -f cpu.svg <pid>`)
- JFR + JMC

Common causes:
- Tight loop / unbounded retry
- Inefficient queries (N+1 with JPA)
- JSON ser/deser of huge payloads
- Regex backtracking
- GC thrashing (insufficient heap)

---

## 7.5 Connection Timeout Issues

Layer-by-layer:

1. **DNS** — Route 53 health, TTL too long?
2. **TCP** — SG, NACL, route table, NAT GW healthy?
3. **TLS** — cert expired? cipher mismatch?
4. **HTTP** — backend healthy? slow? returning 5xx?
5. **App** — connection pool exhausted? thread pool exhausted?
6. **Downstream** — RDS, external API, Redis?

Check ALB **target response time** vs **request processing time**. If target time is low but client sees timeout → likely client-side issue (DNS, retries).

---

## 7.6 Lambda Timeout Issues

- Increase timeout (but find root cause)
- Increase memory (more CPU)
- Check VPC: ENI creation delay (rare now)
- Check downstream: RDS connection storm, S3 throttling
- Use **RDS Proxy** for DB
- Async work → don't synchronously chain Lambdas; use Step Functions / EventBridge
- Cold starts → SnapStart / Provisioned Concurrency
- `X-Ray` traces show where the time goes

---

## 7.7 Database Bottlenecks

- **Performance Insights** + slow query logs
- **EXPLAIN ANALYZE** suspect queries — missing indexes? sequential scans?
- **Connection pool** size vs DB `max_connections`
- **Replica lag** (`ReplicaLag` metric) — long writes / under-sized replica
- **Locks** — `pg_locks` / `SHOW PROCESSLIST`
- **IOPS** — switch to gp3 with custom IOPS, or io1/io2 for high need
- **CPU** — scale up instance class
- **Memory** — buffer pool size, working set fits in RAM?

---

## 7.8 ECS Task Failures

Where to look:
- **ECS console** → service → events tab → why tasks stopped
- **Stopped task** detail → "Stopped reason" (OOM, exit code 137 = SIGKILL, 139 = SIGSEGV)
- **CloudWatch Logs** for the container
- **awslogs driver** must be configured + IAM allowed
- **Health check** failing → check endpoint returns 200 within timeout
- **Image pull errors** → ECR permissions, image tag exists
- **CPU/memory limits** too low → bump task definition
- **Security group / subnet** prevents Pull from ECR (no internet, no VPC endpoint)

---

## 7.9 Deployment Rollback Strategies

| Strategy | Rollback |
|---|---|
| Rolling | New deploy that re-rolls previous task definition |
| Blue/green | Switch ALB back to blue target group (seconds) |
| Canary | Stop traffic shift; pull back to 0% on new |
| Database migrations | Forward-only with safe migrations (additive); store backups |

**Always**:
- Keep last known good task definition revision tagged
- CloudWatch alarms tied to CodeDeploy auto-rollback
- Smoke tests in deploy hooks
- Feature flags decouple deploy from release

---

## 7.10 Troubleshooting Interview Q&A

**Q1. (Intermediate) Pod/task keeps restarting in ECS — how do you debug?**
> 1. Check **stopped reason** in console — OOM, exit code, health check fail. 2. CloudWatch Logs for app errors. 3. Health check endpoint actually returning 200? 4. Task CPU/memory limits sufficient? 5. Dependencies (DB, Redis) reachable from task SG? 6. ECR image pull permissions?

**Q2. (Advanced) Database CPU is at 90%. Walk me through the diagnosis.**
> 1. Performance Insights → top SQL by load and waits.
> 2. EXPLAIN ANALYZE the heaviest queries → missing index? Bad plan?
> 3. Check `pg_stat_activity` for long transactions or locks.
> 4. Connections — pool exhausted?
> 5. Workload change? Sudden traffic spike or new feature?
> 6. Fix: add index, optimize query, route reads to replica, scale up, add cache.
> 7. Long-term: tag heavy queries, add SLO monitoring.

---

# PART 8 — Final Tips & Cheat Sheet

---

## 8.1 Interview Cheat Sheet

| When asked | Always mention |
|---|---|
| EC2 | Multi-AZ, ASG, ALB, SG, role-based auth |
| S3 | Versioning, lifecycle, encryption, Block Public Access |
| RDS | Multi-AZ ≠ read replica, RDS Proxy, Performance Insights |
| DynamoDB | Access patterns first, GSI > LSI, no Scans |
| Lambda | SnapStart for Java, idempotency, DLQ |
| API Gateway | HTTP API default, JWT, WAF in front |
| VPC | Public/private/isolated, NAT per AZ, VPC endpoints |
| IAM | Roles not users, least privilege, explicit deny wins |
| CloudWatch | Set log retention, EMF metrics, X-Ray |
| IaC | Modules per layer, no state in Git, OIDC for CI |
| Microservices | One DB per service, async via SQS/EventBridge, idempotent |
| Security | TLS+JWT, WAF, KMS, Secrets Manager, GuardDuty |

---

## 8.2 Common "Why?" Answers to Memorize

- **Why Multi-AZ?** Survive AZ failure with automatic failover. RPO ~0, RTO ~60–120s.
- **Why RDS Proxy?** Connection pooling for fleets/Lambda; sub-1s failover.
- **Why SQS in front of consumer?** Decouple, buffer spikes, retry, DLQ.
- **Why VPC endpoint?** Private access to AWS services; avoid NAT cost & internet.
- **Why event-driven?** Loose coupling, independent scaling, replay capability.
- **Why idempotency?** At-least-once delivery means duplicates; correctness requires it.
- **Why blue/green?** Zero-downtime deploys + instant rollback.
- **Why KMS envelope encryption?** Performance (KMS only protects small key) + audit.

---

## 8.3 Java-Specific Gotchas

- AWS SDK v2 client creation is expensive — make them singletons
- Spring Boot fat jar on Lambda = slow cold start; use SnapStart or Lambda Web Adapter for migration
- Hikari `maximumPoolSize` should account for total replicas (don't exceed DB limits)
- `DefaultCredentialsProvider` automatically picks instance/task role
- Use **MDC** for correlation IDs across logs (`traceId`)
- Always set `connectionTimeout` AND `socketTimeout` on HTTP clients
- For long Lambda invocations, extend SQS `VisibilityTimeout`

---

## 8.4 Final Preparation Plan (7 days)

| Day | Focus |
|---|---|
| 1 | Part 1 (EC2, S3, RDS, DynamoDB) — drill Q&A, draw diagrams |
| 2 | Part 1 (Lambda, API GW, VPC, IAM, CloudWatch, IaC) |
| 3 | Part 2 — Java + AWS integration; practice code snippets by hand |
| 4 | Part 3 — Microservices; design 3 systems from scratch |
| 5 | Part 4 + 5 — DevOps and Security |
| 6 | Part 6 — Scenarios; rehearse out loud, 2-3 min each |
| 7 | Part 7 — Troubleshooting; mock interview / re-do hardest Qs |

**On interview day**: think out loud, ask clarifying questions, state tradeoffs, draw on whiteboard (text-tree is fine), and always tie answers back to production experience ("In our setup we did X because Y").

---

## 8.5 Closing Note

For each AWS topic, your three-line answer template is:

1. **What it is** (1 sentence)
2. **When you use it / why** (1 sentence with a tradeoff)
3. **Production example** ("In our setup, we used X because Y; the gotcha was Z")

That structure beats memorized definitions. Interviewers want to see **decisions + tradeoffs + scars** — exactly the things you've earned with hands-on work.

Good luck. You've got this.
