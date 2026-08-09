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
| **RDS** | Managed relational DB service supporting MySQL, PostgreSQL, MariaDB, Oracle, SQL Server, Db2 | Standard OLTP relational workloads; want managed patching, backups, Multi-AZ |
| **Aurora** | AWS-engineered MySQL/PostgreSQL-compatible relational DB; 5× MySQL perf, 3× PostgreSQL | High-throughput relational workloads; Global Database for multi-Region HA |
| **Aurora Serverless v2** | Aurora that auto-scales compute capacity in fine-grained ACUs | Variable/unpredictable workloads, dev/test, multi-tenant SaaS |
| **DynamoDB** | Serverless key-value and document NoSQL; single-digit millisecond at any scale | Massive scale key-value/document access, session stores, IoT, gaming leaderboards |
| **DAX** | In-memory write-through cache for DynamoDB; microsecond reads | DynamoDB read-heavy workloads needing sub-millisecond latency |
| **ElastiCache (Redis)** | Managed Redis in-memory cache; persistent, replication, Multi-AZ, Pub/Sub | Session management, leaderboards, complex data structures, pub/sub, geospatial |
| **ElastiCache (Memcached)** | Managed Memcached; simple, multi-threaded, no persistence | Simple distributed object caching without persistence or replication needs |
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
| **VPC Endpoint – Gateway** | Free route-table-based endpoint for S3 and DynamoDB | Private EC2→S3 or EC2→DynamoDB without internet/NAT Gateway; no cost |
| **VPC Endpoint – Interface (PrivateLink)** | ENI with private IP for 100+ AWS services and customer services | Private access to AWS services (SSM, KMS, etc.) or expose your own service to other VPCs |
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

| If the scenario says… | The correct answer is… | Why |
|---|---|---|
| Need stateful firewall at **instance level** | **Security Group** | SGs are stateful (return traffic auto-allowed); apply to ENIs |
| Need to **explicitly DENY** specific IPs at subnet level | **NACL** | NACLs support deny rules; SGs do not; NACLs are stateless |
| Fan-out: **one event → multiple consumers** simultaneously | **SNS → multiple SQS queues** | SNS fan-out pattern; each SQS queue gets a copy for independent processing |
| Decouple producer/consumer, handle **burst traffic, one consumer** | **SQS** | Queue buffers messages; consumers pull at their own pace |
| Route events from **AWS services or SaaS** based on content rules | **EventBridge** | Native AWS event bus; supports 90+ AWS sources and SaaS partners |
| Real-time **streaming, multiple consumers, replay** capability | **Kinesis Data Streams** | Ordered shard log; multiple consumer groups read independently; 7-day retention |
| HTTP/HTTPS routing by **path or hostname** (microservices) | **ALB** | Layer-7 routing rules; path-based and host-based conditions |
| **Static IP**, ultra-low latency, TCP/UDP, or **PrivateLink** | **NLB** | Layer-4; preserves source IP; static Elastic IPs; PrivateLink target |
| Shared **Linux** file system across many instances | **EFS** | NFS, multi-AZ, concurrent mounts; scales automatically |
| Shared **Windows** file system, AD integration | **FSx for Windows** | SMB/NTFS; Active Directory; DFS Namespaces |
| High-performance parallel FS for **HPC/ML** | **FSx for Lustre** | Sub-ms latency; integrates with S3 as a data repo |
| Block storage for **EC2**, best price/performance | **EBS gp3** | Independently provision IOPS/throughput; cheaper than gp2 at same performance |
| **Guaranteed IOPS** for mission-critical DB | **EBS io2 Block Express** | Up to 256K IOPS; 99.999% durability; sub-ms latency |
| **Infrequent large sequential** reads (log archive) | **EBS st1** | HDD; high throughput; not suited for random I/O |
| **RDS Multi-AZ** vs **Read Replica** | Multi-AZ = **HA/failover** (sync standby, no reads); Read Replica = **read scalability** (async, readable) | Multi-AZ survives AZ failure; Read Replicas reduce read load |
| MySQL/PostgreSQL needing **higher throughput than RDS** | **Aurora** | 5× MySQL, 3× PostgreSQL performance; shared storage cluster |
| **Variable/unpredictable** relational workload, want to scale to zero | **Aurora Serverless v2** | ACU auto-scaling; pay per second; good for dev, multi-tenant |
| **Key-value at massive scale**, single-digit ms, serverless | **DynamoDB** | NoSQL; infinite scale; Global Tables for multi-Region |
| DynamoDB **read-heavy**, need **microsecond** latency | **DAX** | DynamoDB Accelerator; write-through cache; no app code change |
| In-memory cache needing **persistence, replication, Pub/Sub** | **ElastiCache Redis** | Rich data structures, TTL, replication, Multi-AZ, Lua scripts |
| Simple **stateless cache**, multi-thread, no persistence needed | **ElastiCache Memcached** | Multi-threaded; horizontal scaling; no persistence |
| Static content delivery to **global users** with caching | **CloudFront** | CDN; 400+ edge locations; caches at edge; S3/ALB/EC2 origin |
| TCP/UDP global acceleration, **static anycast IPs**, non-HTTP | **Global Accelerator** | Routes over AWS backbone; consistent IP; instant failover; not a CDN |
| **Private EC2 → S3** without NAT/internet (free) | **S3 Gateway VPC Endpoint** | Free; route-table based; only for S3 and DynamoDB |
| Private access to **AWS service** (SSM, KMS, ECR) from VPC | **Interface VPC Endpoint (PrivateLink)** | Creates ENI in subnet; charged hourly; 100+ services supported |
| Expose **your own service** privately to other VPCs/accounts | **PrivateLink / Endpoint Service** (backed by NLB) | Consumer creates interface endpoint; traffic stays in AWS network |
| Connect **many VPCs** in hub-and-spoke (transitive routing) | **Transit Gateway** | Single managed router; transitive; cross-Region peering supported |
| Connect **two VPCs** directly, small scale | **VPC Peering** | Simple, free within Region; non-transitive; limited scale |
| **Consistent, dedicated** hybrid connection (high bandwidth) | **Direct Connect** | Physical dedicated line; consistent latency; avoid public internet |
| **Quick, cheap** hybrid connection or DX backup | **Site-to-Site VPN** | IPsec over internet; minutes to provision; lower cost |
| App authentication (sign-up/sign-in) returning **JWT** | **Cognito User Pool** | User directory; handles auth; issues ID/access tokens |
| Mobile app needs **temporary AWS credentials** to access S3/DDB | **Cognito Identity Pool** | Exchanges Cognito/federated token for STS temporary credentials |
| **Auto-rotate** DB passwords and audit access | **Secrets Manager** | Native rotation for RDS/Aurora/Redshift; costs per secret |
| **Config and non-rotating secrets** cheaply | **SSM Parameter Store** | Free standard tier; no rotation; hierarchical path-based access |
| Detect **threats** in AWS account (compromised creds, crypto mining) | **GuardDuty** | ML on VPC Flow Logs, CloudTrail, DNS; no agents needed |
| **Vulnerability scanning** on EC2 AMIs, ECR images, Lambda | **Inspector** | CVE/CIS; automated continuous scanning; does not analyze behavior |
| Find **PII/sensitive data** in S3 buckets | **Macie** | ML-based data classification; S3-specific |
| **Forensic investigation** after a security alert | **Detective** | Visualizes entity behavior over time; graph-based investigation |
| **Cross-account** EC2→another account's S3 | **STS AssumeRole** | EC2 instance profile assumes role in target account; no long-term creds |
| **Backup & Restore** DR (cheapest, longest RTO) | Cheapest: backup data to S3/Glacier; restore on demand | RTO: hours–days; RPO: hours |
| **Pilot Light** DR (minimal footprint, faster than B&R) | Core DB running; minimal compute; scale out on DR event | RTO: hours; RPO: minutes |
| **Warm Standby** DR (reduced fleet always running) | Scaled-down prod in DR Region; scale to full on failure | RTO: minutes; RPO: seconds |
| **Multi-site Active-Active** DR (lowest RTO/RPO) | Full traffic in multiple Regions simultaneously | RTO: near zero; RPO: near zero; highest cost |
| **Reserved Instances** vs **Savings Plans** vs **Spot** | RI = commit to specific instance type/Region; SP = flexible commit to $/hr usage; Spot = up to 90% discount, interruptible | Spot for fault-tolerant batch; SP for flexible committed use; RI for predictable specific instances |
| **Analyze past costs** and forecast | **Cost Explorer** | Historical trends, RI recommendations, forecasts |
| **Alert when budget threshold crossed** | **Budgets** | Proactive alerts; can trigger Lambda/SNS action |
| **Detailed line-item billing** for custom analytics | **CUR** | Hourly/daily granularity; delivered to S3; query with Athena |
| Route 53 **blue/green or A/B traffic split** | **Weighted routing** | Assign weight 0-255; gradually shift traffic between record sets |
| Route 53 **active-passive failover** | **Failover routing** with health check | Primary record; secondary failover record activated when health check fails |
| Route 53 serve users from **closest Region** | **Latency routing** | Routes based on measured latency between user and AWS Regions |
| Route 53 serve users **by country/continent** | **Geolocation routing** | Match user source IP to geographic location; must have default record |
| Route 53 all records **equally** (load balance with DNS) | **Multivalue Answer** | Returns up to 8 healthy records; not a substitute for ELB |
| **NAT Gateway** vs **S3 Gateway Endpoint** for private EC2→S3 | **Gateway Endpoint is free**; NAT Gateway charges per GB | Use gateway endpoint to eliminate NAT data-processing costs for S3/DynamoDB |

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
