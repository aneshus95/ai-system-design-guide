# SAA-C03 Core Services Cheatsheet

A dense, scannable rapid-review reference covering every commonly-tested AWS service on the SAA-C03 exam — what each service is, when to reach for it, and the "if you see X, pick Y" pattern-matches that decide most questions. For deeper treatment of each exam domain, see the four domain lesson pages (01-design-secure-architectures, 02-design-resilient-architectures, 03-design-high-performing-architectures, 04-design-cost-optimized-architectures).

---

## 1. Compute

| Service | What it is (one line) | Use it when |
|---|---|---|
| **EC2** | Resizable virtual machines in the cloud | You need full OS-level control, custom software, GPU/HPC, persistent long-running workloads |
| **EC2 Auto Scaling** | Automatically adds/removes EC2 instances based on demand or schedule | You need elasticity, HA across AZs, or cost control by scaling in during low traffic |
| **ELB – ALB** | Layer-7 HTTP/HTTPS load balancer with path/host/header routing and WebSocket support | Microservices, containers, API routing, gRPC, authentication offload |
| **ELB – NLB** | Layer-4 TCP/UDP/TLS load balancer with static IPs and ultra-low latency | Real-time gaming, VoIP, financial trading, whitelisting static IPs, PrivateLink endpoints |
| **ELB – GWLB** | Transparent Layer-3 bump-in-the-wire for 3rd-party virtual network appliances | Routing traffic through firewalls / IDS / IPS before it reaches your workload |
| **Lambda** | Serverless function-as-a-service; runs code up to 15 min per invocation | Event-driven processing, API backends, data transformation, scheduled jobs with no idle cost |
| **ECS** | AWS-managed container orchestration for Docker on EC2 or Fargate | Running containers without managing Kubernetes; tight AWS integration |
| **EKS** | Managed Kubernetes control plane | You already use Kubernetes, need portability, or require Kubernetes-native tooling |
| **Fargate** | Serverless compute engine for containers (used with ECS or EKS) | You want containers without managing EC2 instances; pay-per-task pricing |
| **Batch** | Managed batch computing — provisions optimal compute for queued jobs | HPC, ETL, genomics, rendering — jobs with defined start/end that run on a schedule or trigger |
| **Elastic Beanstalk** | PaaS that provisions EC2, ELB, ASG, and RDS for web applications | Deploying web apps quickly without infrastructure expertise; supports Java, Node, Python, .NET, PHP, Ruby, Go, Docker |
| **Lightsail** | Simplified VPS with fixed monthly pricing (compute + storage + transfer bundled) | Simple websites, dev/test, small apps, WordPress; low operational complexity |
| **Outposts** | AWS-managed hardware rack placed in your on-premises data center | Regulatory/latency requirements that demand compute physically on-premises with AWS APIs |

---

## 2. Storage

### 2a. Amazon S3 and Storage Classes

| Service / Class | What it is (one line) | Use it when |
|---|---|---|
| **S3 Standard** | General-purpose object storage; 3-AZ redundancy | Frequently accessed data, web assets, active datasets |
| **S3 Intelligent-Tiering** | Auto-moves objects between access tiers based on usage patterns | Unknown or changing access patterns; avoids retrieval fees |
| **S3 Standard-IA** | Infrequent access, lower storage cost, retrieval fee applies | Backups, disaster recovery data accessed < 1× per month |
| **S3 One Zone-IA** | Same as Standard-IA but single AZ; lower cost | Reproducible infrequent data; non-critical secondary backups |
| **S3 Glacier Instant Retrieval** | Archive with millisecond retrieval; 90-day min storage | Medical images, news media archives retrieved occasionally |
| **S3 Glacier Flexible Retrieval** | Archive; retrieval in minutes to hours; lower cost | Long-term backups, compliance archives, annual retrieval |
| **S3 Glacier Deep Archive** | Cheapest storage; retrieval in 12–48 hours | 7–10 year regulatory retention, rarely if ever retrieved |
| **S3 on Outposts** | Object storage on-premises via Outposts | Data residency requirements, local app access |

### 2b. Block, File, and Other Storage

| Service | What it is (one line) | Use it when |
|---|---|---|
| **EBS gp3** | General-purpose SSD; 3,000 IOPS baseline, independently scalable IOPS/throughput | Default block storage for EC2; boot volumes, most databases; prefer over gp2 for new volumes |
| **EBS io2 / io2 Block Express** | Provisioned IOPS SSD; up to 256,000 IOPS; 99.999% durability | Mission-critical databases (Oracle, SAP HANA) needing consistent sub-ms latency |
| **EBS st1** | Throughput-optimized HDD; high sequential throughput, lower cost | Big data, Kafka, log processing — large sequential reads/writes, not random I/O |
| **EBS sc1** | Cold HDD; lowest EBS cost | Infrequently accessed large sequential datasets; cold data |
| **Instance Store** | Ephemeral NVMe/SSD physically attached to the host | Temporary scratch space, buffer caches; data lost on stop/terminate |
| **EFS** | Managed NFS file system; scales automatically; Multi-AZ | Shared Linux file storage across many EC2/Lambda/container instances; CMS, shared configs |
| **FSx for Windows File Server** | Managed Windows SMB/NTFS file system backed by SSD | Windows workloads needing AD integration, DFS, SMB shares |
| **FSx for Lustre** | High-performance parallel file system integrated with S3 | HPC, ML training, rendering; burst throughput up to hundreds of GB/s |
| **FSx for NetApp ONTAP** | Multi-protocol (NFS/SMB/iSCSI) with ONTAP features | Lift-and-shift NetApp workloads, multi-OS shared storage |
| **FSx for OpenZFS** | Managed OpenZFS file system | ZFS snapshot/clone workflows, Linux NFS, consistent latency |
| **Storage Gateway – File** | NFS/SMB gateway caching local data backed by S3 | On-premises file access with S3 durable backend |
| **Storage Gateway – Volume** | iSCSI block volumes backed by S3 snapshots | On-premises block storage with EBS snapshot DR |
| **Storage Gateway – Tape** | Virtual tape library backed by S3/Glacier | Replacing physical tape backup with cloud archiving |
| **AWS Backup** | Centralized, policy-driven backup across AWS services | Compliance, cross-account and cross-Region backup policies |
| **DataSync** | Automated data transfer between on-premises storage and AWS | Migration or ongoing sync of NFS/SMB/HDFS/S3-compatible to S3/EFS/FSx |
| **Snow Family – Snowcone** | 8 TB ruggedized edge device | Small data transfers or edge compute in disconnected/harsh environments |
| **Snow Family – Snowball Edge** | 80–210 TB device with compute; available in Storage Optimized or Compute Optimized | Petabyte data migration; edge ML inference when connectivity is limited |
| **Snow Family – Snowmobile** | Shipping container-scale data transfer (up to 100 PB) | Exabyte-scale data center migration |

---

## 3. Database

| Service | What it is (one line) | Use it when |
|---|---|---|
| **RDS** | Managed relational DB service supporting MySQL, PostgreSQL, MariaDB, Oracle, SQL Server, Db2 | Standard OLTP relational workloads; want managed patching, backups, Multi-AZ — Multi-AZ standby is NOT readable; use Read Replicas for read scaling |
| **Aurora** | AWS-engineered MySQL/PostgreSQL-compatible relational DB; 5× MySQL perf, 3× PostgreSQL | High-throughput relational workloads; Global Database for multi-Region HA — shared distributed storage; up to 15 readable replicas |
| **Aurora Serverless v2** | Aurora that auto-scales compute capacity in fine-grained ACUs | Variable/unpredictable workloads, dev/test, multi-tenant SaaS |
| **DynamoDB** | Serverless key-value and document NoSQL; single-digit millisecond at any scale | Massive scale key-value/document access, session stores, IoT, gaming leaderboards |
| **DAX** | In-memory write-through cache for DynamoDB; microsecond reads | DynamoDB read-heavy workloads needing sub-millisecond latency |
| **ElastiCache (Redis)** | Managed Redis in-memory cache; persistent, replication, Multi-AZ, Pub/Sub | Session management, leaderboards, complex data structures, pub/sub, geospatial — choose Redis when you need persistence, replication, or Multi-AZ failover |
| **ElastiCache (Memcached)** | Managed Memcached; simple, multi-threaded, no persistence | Simple distributed object caching without persistence or replication needs — data is lost on node restart; no Multi-AZ |
| **Redshift** | Petabyte-scale columnar data warehouse; Redshift Serverless available | OLAP analytics, BI dashboards, large SQL queries over structured data |
| **Neptune** | Managed graph database (Gremlin/SPARQL/openCypher) | Social networks, knowledge graphs, fraud detection, recommendation engines |
| **DocumentDB** | MongoDB-compatible managed document database | Migrating MongoDB workloads to AWS; JSON document storage |
| **Keyspaces** | Serverless Apache Cassandra-compatible managed service | Migrating Cassandra workloads; wide-column, high-volume write workloads |
| **Timestream** | Serverless time-series database | IoT telemetry, operational metrics, DevOps time-series data |
| **QLDB** | Fully managed immutable, cryptographically verifiable ledger database | Audit trails, supply chain, financial transaction history requiring tamper-evidence |
| **RDS Proxy** | Fully managed connection pool proxy for RDS/Aurora | Lambda→RDS connections (Lambda creates too many connections); improves failover time |

---

## 4. Networking and Content Delivery

| Service | What it is (one line) | Use it when |
|---|---|---|
| **VPC** | Logically isolated virtual network in AWS; define CIDRs, subnets, routing | Always — every workload in AWS runs inside a VPC |
| **Public subnet** | Subnet with route to Internet Gateway | Resources that must be directly reachable from the internet (ALB, NAT Gateway, bastion) |
| **Private subnet** | Subnet with no direct internet route | EC2 backends, databases, application tier — not directly internet-facing |
| **Route Table** | Rules that determine where network traffic from a subnet is directed | Controlling traffic flow: local, IGW, NAT GW, VGW, TGW, endpoints |
| **Internet Gateway (IGW)** | Horizontally scaled, redundant gateway enabling internet access for a VPC | Required for public subnets to send/receive internet traffic |
| **NAT Gateway** | Managed NAT enabling private subnet outbound internet; stateful | Private EC2 instances needing outbound internet (patches, APIs); prefer managed NAT GW over NAT instances |
| **Security Groups** | Stateful instance-level firewall; allow rules only; evaluates all rules | Controlling inbound/outbound traffic at the ENI/instance level |
| **NACLs** | Stateless subnet-level firewall; allow and deny rules; evaluated in order | Additional layer of defense at subnet boundary; explicitly blocking IPs |
| **VPC Endpoint – Gateway** | Free route-table-based endpoint for S3 and DynamoDB | Private EC2→S3 or EC2→DynamoDB without internet/NAT Gateway; no cost — S3 and DynamoDB only; works via route table, no ENI |
| **VPC Endpoint – Interface (PrivateLink)** | ENI with private IP for 100+ AWS services and customer services | Private access to AWS services (SSM, KMS, etc.) or expose your own service to other VPCs — costs ~$0.01/hr/AZ + $0.01/GB; use Gateway endpoint for S3/DynamoDB instead |
| **VPC Peering** | Direct private connection between two VPCs; non-transitive | Connecting a small number of VPCs; one-to-one; no transitive routing |
| **Transit Gateway (TGW)** | Regional hub-and-spoke router connecting VPCs, VPNs, Direct Connect | Hub for many VPCs or accounts; replaces complex peering mesh; supports multicast |
| **Direct Connect (DX)** | Dedicated private 1/10/100 Gbps connection from on-premises to AWS | Consistent high-bandwidth, low-latency hybrid connectivity; compliance; large data transfer |
| **Site-to-Site VPN** | IPsec VPN tunnel over public internet from on-premises to VGW or TGW | Quick/cheap hybrid connectivity; backup path for Direct Connect |
| **Route 53** | Authoritative DNS with health checks and traffic routing policies | Domain registration, public/private DNS, health-based failover, latency routing |
| **CloudFront** | Global CDN with 400+ edge locations; caches HTTP/HTTPS content | Static/dynamic content acceleration, S3 OAC, DDoS at edge, media streaming |
| **Global Accelerator** | Anycast static IPs routed over AWS backbone; Layer 4 TCP/UDP | Non-HTTP protocols, dynamic application acceleration, consistent IPs, instant failover |
| **API Gateway** | Fully managed REST, HTTP, and WebSocket API management | Frontend for Lambda, backend services; auth, throttling, caching, versioning |
| **App Mesh** | AWS service mesh using Envoy proxy for microservice traffic control | Observability, traffic shaping, mTLS between ECS/EKS/EC2 microservices |

---

## 5. Security, Identity, and Compliance

| Service | What it is (one line) | Use it when |
|---|---|---|
| **IAM** | Manage users, groups, roles, and policies for AWS access control | Always — every AWS permission starts with IAM |
| **IAM Identity Center (SSO)** | Centralized SSO for AWS accounts and SAML 2.0 apps | Multi-account environments; integrate corporate IdP (Okta, Azure AD) with AWS |
| **Organizations / SCP** | Group accounts into OUs with Service Control Policies as permission guardrails | Multi-account governance; prevent accounts from leaving policy boundaries |
| **STS (AssumeRole)** | Issues temporary security credentials for federated/cross-account access | Cross-account access, EC2/Lambda assuming roles, IdP federation |
| **Cognito User Pools** | User directory for sign-up/sign-in with JWT tokens | App authentication — replace custom auth code; MFA, social login |
| **Cognito Identity Pools** | Exchange Cognito/federated identity for temporary AWS credentials | Grant mobile/web users direct AWS service access (S3 upload, DynamoDB) |
| **KMS** | Managed encryption key service; hardware-backed | Encrypting EBS, S3, RDS, Secrets Manager; envelope encryption; key rotation |
| **CloudHSM** | Dedicated HSM cluster — you control keys, AWS has no access | FIPS 140-2 Level 3 compliance; offload SSL/TLS; custom key management |
| **Secrets Manager** | Stores, rotates, and audits secrets (DB passwords, API keys) | Auto-rotation of DB credentials; cross-account secret sharing; audit via CloudTrail |
| **SSM Parameter Store** | Hierarchical key-value store for config and secrets; free tier | App configuration, non-rotating secrets; integrate with SSM Automation/Run Command |
| **ACM** | Provisions and renews SSL/TLS certificates for AWS services | HTTPS on ALB, CloudFront, API Gateway — free managed certs, auto-renewal |
| **WAF** | Web application firewall with managed rules; Layer 7 | Block SQL injection, XSS, bot traffic on CloudFront, ALB, API Gateway, AppSync |
| **Shield Standard** | Always-on L3/L4 DDoS protection; free | Automatically applied to all AWS resources |
| **Shield Advanced** | Enhanced DDoS protection; 24/7 DRT access; cost protection | Business/Enterprise accounts needing SLA-backed DDoS protection and war-room support |
| **Network Firewall** | Managed stateful network firewall for VPC traffic; Suricata rule engine | Deep packet inspection, IDS/IPS, centralized egress filtering across VPCs |
| **GuardDuty** | Intelligent threat detection using ML on VPC Flow Logs, CloudTrail, DNS logs | Detecting compromised instances, unusual API calls, cryptocurrency mining |
| **Inspector** | Automated vulnerability scanning for EC2, Lambda, and ECR container images | CVE scanning, CIS benchmarks, software vulnerability assessment |
| **Macie** | ML-powered sensitive data discovery and classification in S3 | GDPR/PCI compliance; find PII, credentials, financial data in S3 buckets |
| **Security Hub** | Aggregates findings from GuardDuty, Inspector, Macie, Config into one dashboard | Centralized security posture management across accounts and Regions |
| **Config** | Records resource configuration history and evaluates compliance against rules | Drift detection, compliance auditing, change tracking of AWS resources |
| **CloudTrail** | Records all AWS API calls (who did what, when, from where) | Security audit, compliance, incident investigation; always enable in all Regions |
| **Detective** | Analyzes and visualizes security findings to identify root cause | Forensic investigation after GuardDuty/Security Hub alerts; root-cause analysis |

---

## 6. Application Integration

| Service | What it is (one line) | Use it when |
|---|---|---|
| **SQS** | Fully managed pull-based message queue; standard (at-least-once) or FIFO (exactly-once, ordered) | Decoupling producers/consumers; buffering bursts; async job processing |
| **SNS** | Fully managed pub/sub push messaging; fan-out to many subscribers | Broadcasting events to multiple endpoints (Lambda, SQS, email, HTTP); fan-out pattern |
| **EventBridge** | Serverless event bus with content-based routing rules; SaaS integrations; scheduler | Event-driven architectures; routing AWS service events; SaaS event integration; scheduled invocations |
| **Step Functions** | Visual serverless workflow orchestration using state machines | Multi-step Lambda orchestration, human approval workflows, retry/error handling |
| **Amazon MQ** | Managed Apache ActiveMQ and RabbitMQ broker | Migrating on-premises apps using JMS/AMQP/MQTT/STOMP without re-architecting |
| **AppFlow** | No-code integration service for SaaS data flows (Salesforce, Slack, etc.) | Transferring SaaS data to S3, Redshift, or other AWS services without custom ETL |
| **SWF** | Fully managed workflow orchestration with human activity tasks (older service) | Legacy workflows requiring external process coordination; otherwise prefer Step Functions |

---

## 7. Analytics

| Service | What it is (one line) | Use it when |
|---|---|---|
| **Kinesis Data Streams** | Real-time ordered, replayable data stream; shards | Real-time analytics, log ingestion; multiple consumers need same stream; replay needed |
| **Kinesis Data Firehose** | Managed delivery stream to S3, Redshift, OpenSearch, Splunk; near-real-time | Simplest path to land streaming data in a destination; no consumer code needed |
| **Kinesis Managed Flink (formerly Data Analytics)** | Managed Apache Flink for real-time SQL/stream processing | Real-time stream processing, aggregations, anomaly detection |
| **Athena** | Serverless SQL on data in S3 using Presto; pay per query | Ad-hoc S3 data analysis, log queries, data lake exploration without loading to a warehouse |
| **Glue** | Serverless ETL service with Data Catalog | Data transformation, schema discovery, ETL pipelines; catalog for Athena/Redshift Spectrum |
| **EMR** | Managed Hadoop/Spark/Hive/Presto cluster on EC2 or Serverless | Big data processing, ML at scale, custom Spark jobs on large datasets |
| **OpenSearch Service** | Managed OpenSearch (Elasticsearch fork); full-text search and log analytics | Log analytics (ELK stack), full-text search, application search, real-time dashboards |
| **MSK (Managed Streaming for Kafka)** | Fully managed Apache Kafka | Existing Kafka workloads; high-throughput event streaming with Kafka ecosystem |
| **QuickSight** | Serverless BI and visualization service; ML Insights | Business dashboards, embedded analytics, sharing reports without a BI server |
| **Lake Formation** | Simplifies building, securing, and sharing data lakes on S3 | Centralized permissions and governance over a data lake; column/row-level security |

---

## 8. Management and Cost

| Service | What it is (one line) | Use it when |
|---|---|---|
| **CloudWatch** | Monitoring, metrics, logs, alarms, dashboards, and events for AWS resources | Any monitoring/alerting need; custom metrics from EC2; log aggregation from Lambda/ECS |
| **CloudFormation** | Infrastructure-as-Code using JSON/YAML templates (stacks) | Repeatable, version-controlled AWS infrastructure; drift detection |
| **Systems Manager (SSM)** | Operational management: Run Command, Patch Manager, Session Manager, Parameter Store, Automation | Managing EC2 fleets without SSH; patching; secure shell-less access; config management |
| **Trusted Advisor** | Automated checks for cost, performance, security, fault tolerance, service limits | Identify unused resources, open security groups, underutilized EC2; free tier limited |
| **Cost Explorer** | Visualizes AWS costs/usage over time; forecasting; Reserved Instance recommendations | Analyzing past spend patterns, right-sizing, RI/SP coverage reports |
| **Budgets** | Sets cost/usage/RI/SP budget thresholds with alerts | Proactive cost control; alert when spend exceeds threshold; automated actions |
| **CUR (Cost and Usage Report)** | Granular hourly/daily billing data in S3 (CSV/Parquet) | Detailed cost allocation, chargeback, custom analytics with Athena/QuickSight |
| **Compute Optimizer** | ML-based recommendations for right-sizing EC2, Lambda, EBS, ECS on Fargate | Right-sizing to reduce over-provisioning; uses CloudWatch metrics as inputs |
| **Control Tower** | Landing zone service that sets up multi-account AWS environment with guardrails | Automating multi-account setup, enforcing baseline governance via SCPs and Config rules |
| **Health Dashboard** | Personalized view of AWS service health and scheduled changes affecting your account | Proactive notification of AWS events impacting your resources; integrate with EventBridge |

---

## 9. If You See X → Pick Y: Disambiguation Table

| If the scenario says… | The correct answer is… | Why / nuance |
|---|---|---|
| Need stateful firewall at **instance level** | **Security Group** | SG is stateful — return traffic is auto-allowed, allow rules only; NACL is stateless so you'd have to manually open ephemeral return ports (1024–65535) and it supports explicit Deny, not SG. |
| Need to **explicitly DENY** specific IPs at subnet level | **NACL** | NACLs support Deny rules evaluated in number order; SGs are allow-only. Because NACLs are stateless, you must also allow outbound ephemeral ports (1024–65535) for return traffic — the classic exam gotcha. |
| Fan-out: **one event → multiple consumers** simultaneously | **SNS → multiple SQS queues** | SNS is push-based pub/sub: one publish delivers to all subscribers simultaneously. SQS alone serves only one consumer group; each SQS queue behind SNS gets an independent copy. |
| Decouple producer/consumer, handle **burst traffic, one consumer** | **SQS** | SQS buffers/pull for one consumer group; SNS pushes to many simultaneously. Use SQS when you need backpressure or async processing, not fan-out. |
| Route events from **AWS services or SaaS** based on content rules | **EventBridge** | EventBridge does content-based routing/filtering across 90+ AWS sources and SaaS partners, plus scheduling. SNS fans out without content filtering; SQS queues without routing. |
| Real-time **streaming, multiple consumers, replay** capability | **Kinesis Data Streams** | Ordered shard log; each consumer reads at its own position; up to 365-day replay retention. SQS deletes a message once consumed — no replay, no parallel independent readers. Firehose delivers to a single destination without replay. |
| HTTP/HTTPS routing by **path or hostname** (microservices) | **ALB** | ALB is Layer-7 and supports path-based and host-based routing rules; NLB is Layer-4 only and routes by IP/port with no HTTP awareness. |
| **Static IP**, ultra-low latency, TCP/UDP, or **PrivateLink** | **NLB** | NLB is Layer-4 with static Elastic IPs and is the only ELB type that can be a PrivateLink target. ALB has dynamic IPs and no native PrivateLink support. |
| Shared **Linux** file system across many instances | **EFS** | EFS is NFS-based, multi-AZ, and scales automatically; EBS is single-instance block storage (except Multi-Attach io2 in limited use cases). |
| Shared **Windows** file system, AD integration | **FSx for Windows** | FSx for Windows is SMB/NTFS with native Active Directory and DFS Namespaces support; EFS does not support SMB or Windows ACLs. |
| High-performance parallel FS for **HPC/ML** | **FSx for Lustre** | Sub-ms latency, hundreds of GB/s throughput, native S3 integration as a data repository; EFS cannot match Lustre's raw HPC throughput. |
| Block storage for **EC2**, best price/performance | **EBS gp3** | gp3 lets you independently provision IOPS and throughput without paying for more capacity, and is cheaper than gp2 at equivalent performance; io2 is for missions needing guaranteed high IOPS beyond gp3's 16,000 IOPS ceiling. |
| **Guaranteed IOPS** for mission-critical DB | **EBS io2 Block Express** | Up to 256,000 IOPS, 99.999% durability, sub-ms latency; gp3 maxes at 16,000 IOPS and 1,000 MB/s throughput. |
| **Infrequent large sequential** reads (log archive) | **EBS st1** | st1 is a throughput-optimized HDD tuned for sequential reads/writes; io2/gp3 are more expensive SSDs suited for random I/O, not large sequential cold data. |
| **RDS Multi-AZ** vs **Read Replica** | Multi-AZ = **HA/failover** (sync standby, not readable); Read Replica = **read scalability** (async, readable) | Multi-AZ standby is synchronously replicated but cannot serve reads — it exists only for automatic AZ-failure failover. Read Replicas are readable but use async replication (can lag) and do not auto-promote on primary failure. |
| MySQL/PostgreSQL needing **higher throughput than RDS** | **Aurora** | Aurora's shared distributed storage delivers 5× MySQL and 3× PostgreSQL throughput vs. standard RDS, supports up to 15 readable replicas, and auto-heals storage; standard RDS does not. |
| **Variable/unpredictable** relational workload, want to scale to zero | **Aurora Serverless v2** | Aurora Serverless v2 auto-scales in fine-grained ACUs and can scale to near-zero; standard Aurora has a fixed instance size and incurs cost even at idle. |
| **Key-value at massive scale**, single-digit ms, serverless | **DynamoDB** | DynamoDB is a fully serverless NoSQL store with single-digit-ms latency at any scale and Global Tables for multi-Region active-active; RDS cannot scale horizontally without significant re-architecture. |
| DynamoDB **read-heavy**, need **microsecond** latency | **DAX** | DAX is a write-through in-memory cache sitting in front of DynamoDB, delivering microsecond reads with no application code change; ElastiCache Redis requires application-level cache logic and doesn't natively understand DynamoDB. |
| In-memory cache needing **persistence, replication, Pub/Sub** | **ElastiCache Redis** | Redis supports persistence (AOF/RDB), Multi-AZ replication, Pub/Sub, and rich data structures; Memcached is volatile — data is lost on node restart and has no replication. |
| Simple **stateless cache**, multi-thread, no persistence needed | **ElastiCache Memcached** | Memcached is multi-threaded and simpler; Redis adds persistence and replication overhead. If a node dies, losing cache data is acceptable here — choose Memcached to minimize cost and complexity. |
| Static content delivery to **global users** with caching | **CloudFront** | CloudFront is a CDN that caches HTTP/HTTPS content at 400+ edge locations; Global Accelerator does NOT cache — it optimizes network routing for dynamic/non-cacheable traffic. |
| TCP/UDP global acceleration, **static anycast IPs**, non-HTTP | **Global Accelerator** | Global Accelerator provides two static anycast IPs and routes TCP/UDP over the AWS backbone; CloudFront only handles HTTP/HTTPS and uses dynamically changing IPs unsuitable for IP allowlisting. |
| **Private EC2 → S3** without NAT/internet (free) | **S3 Gateway VPC Endpoint** | Gateway endpoint is completely free and works via route table entry for S3 and DynamoDB only; Interface endpoint (PrivateLink) costs ~$0.01/hr/AZ + $0.01/GB — never pay for Interface when Gateway suffices. |
| Private access to **AWS service** (SSM, KMS, ECR) from VPC | **Interface VPC Endpoint (PrivateLink)** | Interface endpoints cover 100+ services via an ENI in your subnet; Gateway endpoints only support S3 and DynamoDB. Interface endpoints cost ~$0.01/hr/AZ + $0.01/GB, so use Gateway for S3/DynamoDB. |
| Expose **your own service** privately to other VPCs/accounts | **PrivateLink / Endpoint Service** (backed by NLB) | PrivateLink keeps traffic on the AWS network; VPC Peering requires non-overlapping CIDRs and is not scalable for exposing a service to many consumers. |
| Connect **many VPCs** in hub-and-spoke (transitive routing) | **Transit Gateway** | TGW is a regional hub that supports transitive routing and cross-Region peering; VPC Peering is non-transitive — each pair needs its own peering connection, which doesn't scale. |
| Connect **two VPCs** directly, small scale | **VPC Peering** | VPC Peering is simple and free within a Region for a small number of VPCs; Transit Gateway adds cost and is overkill for two VPCs. |
| **Consistent, dedicated** hybrid connection (high bandwidth) | **Direct Connect** | DX is a physical dedicated private line with consistent low latency; Site-to-Site VPN runs over the public internet with variable latency and lower max bandwidth. |
| **Quick, cheap** hybrid connection or DX backup | **Site-to-Site VPN** | VPN is provisioned in minutes over the internet at low cost; Direct Connect takes weeks to provision and requires physical cross-connect. |
| App authentication (sign-up/sign-in) returning **JWT** | **Cognito User Pool** | User Pools are a user directory that handles sign-up/sign-in and issues JWT tokens (ID, access, refresh); Identity Pools do not authenticate users — they exchange tokens for temporary AWS credentials. |
| Mobile app needs **temporary AWS credentials** to access S3/DDB | **Cognito Identity Pool** | Identity Pools exchange a User Pool JWT (or social IdP token) for short-term STS credentials; User Pools only issue JWT tokens, not AWS credentials. |
| **Auto-rotate** DB passwords and audit access | **Secrets Manager** | Secrets Manager has native rotation for RDS/Aurora/Redshift and costs $0.40/secret/month; SSM Parameter Store has no built-in rotation — you'd need to build custom Lambda automation. |
| **Config and non-rotating secrets** cheaply | **SSM Parameter Store** | Standard parameters are free; Secrets Manager charges $0.40/secret/month even for secrets that never rotate. Use Parameter Store for app config and non-rotating values. |
| Detect **threats** in AWS account (compromised creds, crypto mining) | **GuardDuty** | GuardDuty uses ML on VPC Flow Logs, CloudTrail, and DNS logs to detect behavioral threats; Inspector scans for software vulnerabilities (CVEs), not behavioral anomalies. |
| **Vulnerability scanning** on EC2 AMIs, ECR images, Lambda | **Inspector** | Inspector does continuous CVE/CIS benchmark scanning; GuardDuty detects runtime threats and unusual API behavior, not software vulnerabilities. |
| Find **PII/sensitive data** in S3 buckets | **Macie** | Macie uses ML to classify and detect PII/sensitive data in S3; GuardDuty detects threats but does not scan object content for sensitive data. |
| **Forensic investigation** after a security alert | **Detective** | Detective builds a graph of entity behavior over time for root-cause analysis; GuardDuty generates the initial finding but does not provide investigation tooling. |
| **Cross-account** EC2→another account's S3 | **STS AssumeRole** | EC2 instance profile assumes a cross-account IAM role via STS; no long-term credentials are stored. Resource-based S3 bucket policies alone cannot grant the calling identity temporary credentials. |
| **Backup & Restore** DR (cheapest, longest RTO) | Cheapest: backup data to S3/Glacier; restore on demand | RTO: hours–days; RPO: hours. Lowest cost because no standby infrastructure runs — you pay only for storage until disaster strikes. |
| **Pilot Light** DR (minimal footprint, faster than B&R) | Core DB running; minimal compute; scale out on DR event | RTO: hours; RPO: minutes. Only the critical data tier is live; compute must be provisioned on failover, unlike Warm Standby where it already runs scaled-down. |
| **Warm Standby** DR (reduced fleet always running) | Scaled-down prod in DR Region; scale to full on failure | RTO: minutes; RPO: seconds. A reduced-capacity stack is always running and can scale up quickly; costs more than Pilot Light because compute is always on. |
| **Multi-site Active-Active** DR (lowest RTO/RPO) | Full traffic in multiple Regions simultaneously | RTO: near zero; RPO: near zero; highest cost. All Regions serve live traffic; no failover needed — traffic shifts via Route 53 or Global Accelerator. |
| **Reserved Instances** vs **Savings Plans** vs **Spot** | RI = commit to specific instance family/size/Region; SP = commit to $/hr (flexible across types/Regions/Fargate/Lambda); Spot = up to 90% off, interruptible with 2-min warning | Steady workload needing specific capacity → RI (deepest discount for that exact type); steady + flexible or includes Fargate/Lambda → Savings Plans; fault-tolerant/interruptible batch → Spot. |
| **Analyze past costs** and forecast | **Cost Explorer** | Cost Explorer provides historical trend visualizations, RI/SP coverage reports, and forecasts; Budgets is prospective (alert when threshold is crossed), not analytical. |
| **Alert when budget threshold crossed** | **Budgets** | Budgets sends proactive alerts and can trigger automated actions (Lambda/SNS/IAM policy); Cost Explorer is for analysis and forecasting, not alerting. |
| **Detailed line-item billing** for custom analytics | **CUR** | CUR delivers hourly/daily granular line-item data to S3 queryable with Athena; Cost Explorer shows visualizations but not raw per-resource hourly granularity. |
| Route 53 **blue/green or A/B traffic split** | **Weighted routing** | Weighted routing assigns a 0–255 weight to each record; gradually shift traffic by adjusting weights — a weight of 0 removes a record from rotation without deleting it. |
| Route 53 **active-passive failover** | **Failover routing** with health check | Failover routing promotes the secondary record only when the primary's health check fails; Weighted routing cannot detect health and always distributes traffic. |
| Route 53 serve users from **closest Region** | **Latency routing** | Latency routing measures actual AWS-measured latency between the user's location and each Region; Geolocation routing uses IP-to-geography mapping and doesn't measure real latency. |
| Route 53 serve users **by country/continent** | **Geolocation routing** | Geolocation routing maps source IP to a geographic location; a default record is required or unmatched locations get NODATA. Latency routing doesn't support geographic-restriction requirements. |
| Route 53 all records **equally** (load balance with DNS) | **Multivalue Answer** | Multivalue Answer returns up to 8 healthy records and is health-check-aware; it is NOT a substitute for ELB (no connection persistence or Layer-7 features). |
| **NAT Gateway** vs **S3 Gateway Endpoint** for private EC2→S3 | **Gateway Endpoint is free**; NAT Gateway charges per GB | Gateway endpoint eliminates NAT data-processing costs ($0.045/GB) for S3 and DynamoDB traffic; NAT Gateway is still needed for other internet-bound traffic but should not be used for S3/DynamoDB. |

---

## Glossary

| Term | Simple explanation | Purpose |
|---|---|---|
| **AZ (Availability Zone)** | One or more discrete data centers with redundant power/networking within a Region | Unit of fault isolation; deploy across AZs for HA |
| **Region** | Geographic cluster of 3+ AZs (e.g., us-east-1) | Fault isolation, data residency, latency proximity |
| **VPC** | Virtual Private Cloud — your logically isolated network in AWS | Network boundary; controls IP ranges, routing, security |
| **CIDR** | Classless Inter-Domain Routing notation (e.g., 10.0.0.0/16) | Define IP address ranges for VPCs and subnets |
| **IAM Role** | Identity with permissions assumed by services/users; no long-term credentials | Grant EC2/Lambda/cross-account access using temporary credentials |
| **IAM Policy** | JSON document defining allowed/denied actions on resources | Attached to users, groups, or roles to control access |
| **SCP (Service Control Policy)** | Organization-level policy setting maximum permissions for member accounts | Guardrails — even admin cannot exceed SCP limits |
| **KMS** | Key Management Service — managed CMKs for envelope encryption | Encrypt data at rest across S3, EBS, RDS, Secrets Manager, etc. |
| **CMK** | Customer Master Key — KMS key used to encrypt data keys | Root of trust for envelope encryption; can be AWS-managed or customer-managed |
| **RTO** | Recovery Time Objective — max acceptable downtime after a disaster | DR strategy selection: lower RTO = higher cost |
| **RPO** | Recovery Point Objective — max acceptable data loss (in time) | Determines backup/replication frequency |
| **IOPS** | Input/Output Operations Per Second — measure of storage I/O performance | Key metric for EBS volume type selection |
| **Throughput** | Amount of data transferred per second (MB/s or GB/s) | Key metric for sequential workloads (st1, EFS, FSx Lustre) |
| **ENI** | Elastic Network Interface — virtual NIC in a VPC | Attached to EC2; has private IP; security groups applied here |
| **EIP** | Elastic IP — static public IPv4 address | Assign to EC2/NAT GW to maintain consistent public IP |
| **ACU** | Aurora Capacity Unit — measure of compute for Aurora Serverless | Auto-scales in fractions; pay per second of ACU consumption |
| **AMI** | Amazon Machine Image — template for EC2 instances (OS + config) | Launch identical instances; share across accounts/Regions |
| **ASG** | Auto Scaling Group — collection of EC2 managed as a unit | Scale out/in based on policies; span multiple AZs |
| **Target Group** | Collection of targets (EC2, Lambda, IP) for ELB to route to | Health-checked endpoints behind ALB/NLB |
| **Origin Access Control (OAC)** | CloudFront mechanism to restrict S3 bucket access to CloudFront only | Prevents direct S3 URL access; replaces deprecated OAI |
| **FIPS 140-2** | US government cryptographic module standard (Level 2 = KMS, Level 3 = CloudHSM) | Compliance requirement for regulated industries |
| **SRD** | Scalable Reliable Datagram — AWS network protocol for EC2 Enhanced Networking | Lower latency, higher throughput compared to TCP for EC2 |
| **Spot Instance** | Unused EC2 capacity at up to 90% discount; can be interrupted with 2-min notice | Fault-tolerant batch, CI/CD, stateless workloads; not for critical persistent tasks |
| **Savings Plan** | Commit to $/hour usage for 1 or 3 years; flexible across instance types/Regions | Compute SP: most flexible; EC2 Instance SP: highest discount |
| **Reserved Instance** | Commit to specific instance family, size, Region for 1 or 3 years | Predictable steady-state workloads; convertible RI allows type changes |
| **Placement Group** | Logical grouping affecting EC2 physical placement: Cluster, Spread, Partition | Cluster = low latency HPC; Spread = HA isolation; Partition = large distributed |
| **VGW** | Virtual Private Gateway — VPN endpoint on AWS side; attaches to VPC | Terminate Site-to-Site VPN or Direct Connect on AWS side |
| **DGW** | Direct Connect Gateway — connect one DX connection to multiple VPCs across Regions | Consolidate DX connections; pairs with TGW for transitive routing |
| **TTL** | Time To Live — DNS: seconds a record is cached; S3/CloudFront: seconds object is cached | Lower TTL = faster updates but more DNS queries |
| **Alias Record** | Route 53-specific DNS extension; maps hostname to AWS resource (free, no TTL needed) | Use instead of CNAME at zone apex (e.g., example.com → ALB) |

---

## References

- [AWS Certified Solutions Architect – Associate (SAA-C03) Official Exam Guide](https://d1.awsstatic.com/training-and-certification/docs-sa-assoc/AWS-Certified-Solutions-Architect-Associate_Exam-Guide_C03.pdf)
- [AWS Certified Solutions Architect – Associate (SAA-C03) Exam Guide (HTML)](https://docs.aws.amazon.com/aws-certification/latest/solutions-architect-associate-03/solutions-architect-associate-03.html)
- [Amazon S3 Storage Classes](https://aws.amazon.com/s3/storage-classes/)
- [Amazon EBS Volume Types](https://docs.aws.amazon.com/ebs/latest/userguide/ebs-volume-types.html)
- [Amazon EFS – How It Works](https://docs.aws.amazon.com/efs/latest/ug/how-it-works.html)
- [Gateway Endpoints for Amazon S3](https://docs.aws.amazon.com/vpc/latest/privatelink/vpc-endpoints-s3.html)
- [VPC Interface Endpoint vs Gateway Endpoint – Tutorials Dojo](https://tutorialsdojo.com/vpc-interface-endpoint-vs-gateway-endpoint-in-aws/)
- [Disaster Recovery Options in the Cloud – AWS Whitepaper](https://docs.aws.amazon.com/whitepapers/latest/disaster-recovery-workloads-on-aws/disaster-recovery-options-in-the-cloud.html)
- [DR Part IV: Multi-Site Active/Active – AWS Architecture Blog](https://aws.amazon.com/blogs/architecture/disaster-recovery-dr-architecture-on-aws-part-iv-multi-site-active-active/)
- [Amazon ElastiCache FAQs (Redis vs Memcached)](https://aws.amazon.com/elasticache/faqs/)
- [ElastiCache Redis vs Memcached – Jayendra Patil](https://jayendrapatil.com/aws-elasticache-redis-vs-memcached/)
- [AWS EBS Volume Types Comparison – Jayendra Patil](https://jayendrapatil.com/aws-ebs-volume-types/)
- [AWS Reliability Pillar – Disaster Recovery Strategies](https://docs.aws.amazon.com/wellarchitected/latest/reliability-pillar/rel_planning_for_recovery_disaster_recovery.html)
- [AWS PrivateLink and VPC Endpoints Overview](https://docs.aws.amazon.com/vpc/latest/privatelink/privatelink-access-aws-services.html)
- [Amazon Route 53 Routing Policies](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/routing-policy.html)
- [AWS Cost Management – Cost Explorer](https://docs.aws.amazon.com/cost-management/latest/userguide/ce-what-is.html)
- [AWS Budgets](https://docs.aws.amazon.com/cost-management/latest/userguide/budgets-managing-costs.html)
- [AWS Cost and Usage Report](https://docs.aws.amazon.com/cur/latest/userguide/what-is-cur.html)

---

*Part of the [AWS Solutions Architect Associate course](README.md) · SAA-C03.*
