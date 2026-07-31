# Cloud Computing — In-Depth Interview Guide (HPE)

> This is the **deep-study** version. It explains not just *what* each concept is but *how* it works and *why* it exists, with the level of detail that lets you handle follow-up questions ("okay, but how does a load balancer actually decide?", "what's the difference between a security group and a NACL?"). Use the short guide for last-minute revision; use this to actually understand. Big Q&A bank at the end.

---

## PART A — FOUNDATIONS

## 1. What Cloud Computing Is, and Why It Emerged

**Definition:** Cloud computing is the on-demand delivery of IT resources — compute, storage, databases, networking, software — over the internet, provisioned programmatically and billed by usage, instead of owning and operating physical infrastructure.

**The problem it solves (the historical "why"):** Before cloud, if you wanted to launch a web app you had to *forecast* peak demand, *buy* servers to meet it, wait weeks for delivery and setup, and then pay for that hardware whether it was busy or idle. You either over-provisioned (wasted money) or under-provisioned (crashed under load). Cloud decouples capacity from ownership: you rent exactly what you need, when you need it, and give it back when you don't.

**The core economic shift — CapEx to OpEx:**
- **CapEx (Capital Expenditure)** — large upfront investment in owned assets (servers, data centers) that depreciate.
- **OpEx (Operational Expenditure)** — ongoing pay-as-you-go cost, treated as a running expense.
- Cloud turns a big fixed cost into a variable cost that scales with actual usage. This is why startups can launch globally with almost no upfront capital.

**The five deep benefits:**
1. **Elastic capacity** — match supply to demand automatically.
2. **Speed & agility** — provision in minutes, experiment cheaply, fail fast.
3. **Global reach** — deploy near users worldwide instantly.
4. **Economies of scale** — providers buy hardware and power at a scale no single company can match, passing savings on.
5. **Focus** — you stop managing undifferentiated heavy lifting (power, cooling, hardware) and focus on your actual product.

---

## 2. The 5 Essential Characteristics (NIST) — Explained Deeply

1. **On-demand self-service** — you provision resources yourself through a console/API/CLI, automatically, with no human on the provider's side. This automation is what makes the cloud "elastic" in practice.
2. **Broad network access** — resources are reachable over the network via standard protocols from any device (laptop, phone, server).
3. **Resource pooling (multi-tenancy)** — the provider pools physical resources and serves many customers from the same hardware, dynamically assigning and reassigning capacity. You generally don't know or care which physical machine you're on. Isolation between tenants is enforced by virtualization.
4. **Rapid elasticity** — capacity can grow and shrink quickly, sometimes automatically, appearing effectively unlimited to the customer.
5. **Measured service** — usage is metered (compute-seconds, GB stored, requests, data transferred) and you pay for exactly that. Metering also enables monitoring and transparency.

---

## PART B — SERVICE & DEPLOYMENT MODELS

## 3. Service Models — IaaS / PaaS / SaaS (and beyond)

The dividing line is the **responsibility boundary**: how much the provider manages versus you.

**The full stack, top to bottom:** Application → Data → Runtime → Middleware → OS → Virtualization → Servers → Storage → Networking.

- **On-premise:** you manage the entire stack.
- **IaaS (Infrastructure as a Service):** provider manages networking, storage, servers, and virtualization; **you** manage OS and everything above (runtime, apps, data). You get raw building blocks. *Example: AWS EC2, Azure Virtual Machines, Google Compute Engine.* Use when you need maximum control or are migrating existing systems.
- **PaaS (Platform as a Service):** provider also manages the OS, runtime, and middleware; **you** manage only your application and data. You push code; scaling and patching are handled. *Example: Google App Engine, AWS Elastic Beanstalk, Heroku, Azure App Service.* Use when developers want to focus purely on code.
- **SaaS (Software as a Service):** provider manages everything; **you** just use the software over the internet. *Example: Gmail, Salesforce, Microsoft 365, Dropbox.* Use for finished software you don't want to build or run.

**The "as-a-Service" family (be aware of these):**
- **FaaS (Function as a Service)** — serverless functions; a subset of the serverless model (AWS Lambda).
- **CaaS (Container as a Service)** — managed container orchestration (AWS ECS/EKS, Azure AKS).
- **DBaaS (Database as a Service)** — managed databases (Amazon RDS, DynamoDB).
- **XaaS** — "anything as a service," the umbrella term.

**The Pizza-as-a-Service analogy (great to say aloud):** homemade = on-prem; take-and-bake = IaaS; delivery = PaaS; dining out = SaaS.

---

## 4. Deployment Models — Deeper

- **Public cloud** — infrastructure owned and operated by a third-party provider, shared across many tenants over the internet. Cheapest, most scalable, least control. *(AWS, Azure, GCP.)*
- **Private cloud** — cloud infrastructure dedicated to one organization. Can be on-premise or hosted. More control, security, and compliance; higher cost and less elasticity. HPE specializes here.
- **Hybrid cloud** — combines public and private with orchestration and data/workload movement between them. Keep sensitive/regulated workloads private, burst to public for scale. The dominant enterprise pattern.
- **Community cloud** — shared by organizations with common needs/regulations (e.g., government agencies, banks under the same compliance regime).
- **Multi-cloud** — using more than one public provider (e.g., AWS + Azure). Reasons: avoid vendor lock-in, use best-of-breed services, meet data-residency needs, improve resilience.

**Cloud bursting** — a hybrid pattern where an app runs in a private cloud normally but "bursts" into the public cloud when demand spikes.

**Vendor lock-in** — the difficulty/cost of moving off a provider once you depend on its proprietary services. Mitigated by open standards, containers, and multi-cloud design.

> **HPE angle:** HPE's entire thesis is that *not everything belongs in the public cloud.* Data gravity (huge datasets are expensive/slow to move), latency, compliance/sovereignty, and predictable cost for steady workloads all argue for private/hybrid. GreenLake brings a public-cloud *experience* (self-service, pay-per-use) to on-premise infrastructure.

---

## PART C — THE CORE TECHNOLOGY

## 5. Virtualization — The Engine Under the Cloud

**What it is:** technology that abstracts physical hardware so that one physical machine can run multiple isolated virtual machines, each believing it has its own dedicated hardware. This is what makes *resource pooling* and *multi-tenancy* possible.

**Hypervisor (Virtual Machine Monitor)** — the software layer that creates, runs, and isolates VMs, allocating CPU, memory, and I/O.
- **Type 1 (bare-metal)** — runs directly on physical hardware; the VMs sit on top. Faster, more secure, used in data centers. *Examples: VMware ESXi, Microsoft Hyper-V, KVM, Xen.*
- **Type 2 (hosted)** — runs as an application on top of a host OS. Convenient for desktops/testing, more overhead. *Examples: VirtualBox, VMware Workstation.*

**Full vs para-virtualization (deeper follow-up):**
- **Full virtualization** — the guest OS runs unmodified; the hypervisor emulates hardware. Broad compatibility.
- **Para-virtualization** — the guest OS is modified/aware it's virtualized and cooperates with the hypervisor, reducing overhead.
- **Hardware-assisted virtualization** — CPU features (Intel VT-x, AMD-V) accelerate virtualization, now standard.

**Types of virtualization beyond servers:** storage virtualization, network virtualization (the basis of SDN and VPCs), and desktop virtualization (VDI).

---

## 6. Containers & Orchestration — In Depth

### Containers vs VMs (the core distinction)
A **VM virtualizes the hardware** — each VM ships a full guest OS, so it's heavy (GBs) and slow to boot (minutes). A **container virtualizes the operating system** — containers share the host's OS kernel and package only the application plus its dependencies, so they're light (MBs) and start in seconds/milliseconds.

| | VM | Container |
|---|-----|-----------|
| Virtualizes | Hardware | OS |
| Includes | Full guest OS | App + dependencies only |
| Size / boot | GBs / minutes | MBs / seconds |
| Isolation | Stronger (own kernel) | Weaker (shared kernel) |
| Density per host | Lower | Much higher |

Because containers share the kernel, isolation is slightly weaker than VMs — which is why security-sensitive workloads sometimes run containers *inside* VMs.

### Docker (the container runtime/tooling)
- **Image** — an immutable, layered template containing the app and everything it needs. Built from a **Dockerfile**.
- **Layers** — images are built in stacked layers; shared layers are cached and reused, saving space and speeding builds.
- **Container** — a running instance of an image.
- **Registry** — where images are stored/shared (Docker Hub, Amazon ECR).

### Kubernetes (K8s) — the orchestrator
When you run hundreds of containers across many machines, you need orchestration for scheduling, scaling, self-healing, networking, and rollouts. Kubernetes is the standard.

**Architecture — Control Plane (the brain):**
- **API Server** — the front door; all commands go through it.
- **etcd** — the distributed key-value store holding cluster state (the source of truth).
- **Scheduler** — decides which node a new pod runs on.
- **Controller Manager** — runs control loops that drive actual state toward desired state.

**Architecture — Worker Nodes (where work runs):**
- **kubelet** — agent that ensures containers are running as instructed.
- **kube-proxy** — handles networking/routing to pods.
- **Container runtime** — actually runs the containers.

**Key objects:**
- **Pod** — the smallest deployable unit; one or more tightly coupled containers sharing network/storage.
- **Deployment** — declares the desired number of pod replicas and manages rolling updates/rollbacks.
- **Service** — a stable network endpoint / load balancer for a set of pods (pods are ephemeral; Services give them a fixed address).
- **Namespace** — logical partition of the cluster.
- **Ingress** — manages external HTTP(S) access into the cluster.

**The core idea — declarative + reconciliation:** you declare the *desired* state ("I want 5 replicas"), and Kubernetes continuously reconciles reality toward it — restarting crashed pods (self-healing), rescheduling on node failure, and scaling.

---

## PART D — THE BUILDING BLOCKS

## 7. Compute

- **Virtual machines / instances** — the workhorse; you choose instance types optimized for general purpose, compute, memory, storage, or GPU (for AI/ML).
- **Containers** — see above (ECS, EKS, AKS, GKE).
- **Serverless / functions** — see Section 12.
- **Auto Scaling Group** — a set of instances that grows/shrinks by policy.
  - **Scaling policies:** *target tracking* (keep CPU at, say, 50%), *step scaling* (add N instances at thresholds), *scheduled scaling* (scale up before known peaks).

## 8. Storage — The Three Types (and how to choose)

- **Object storage** — data stored as objects (data + metadata + unique ID) in a flat namespace, accessed via HTTP API. Virtually unlimited, cheap, highly durable. Not a file system; you can't mount it like a disk. *Best for: images, video, backups, static websites, data lakes.* *Example: Amazon S3, Azure Blob.*
- **Block storage** — raw volumes split into fixed-size blocks, attached to a single VM and formatted with a file system, behaving like a physical disk. Low latency, high performance. *Best for: databases, OS boot disks, transactional workloads.* *Example: Amazon EBS.*
- **File storage** — a shared hierarchical file system accessed over a network (NFS/SMB) by multiple machines at once. *Best for: shared application files, lift-and-shift apps expecting a file system.* *Example: Amazon EFS, Azure Files.*

**Storage tiers (cost vs access):** hot (frequent access, higher cost) → cool/infrequent → cold/archive (cheapest, retrieval delay, e.g., Amazon S3 Glacier). **Lifecycle policies** move data between tiers automatically to save cost.

**Durability vs availability (important distinction):**
- **Durability** — the probability your data is *not lost* (e.g., S3's "eleven nines," 99.999999999%), achieved by replicating across devices/facilities.
- **Availability** — the probability the data is *accessible right now* (e.g., 99.99% uptime).
- You can have high durability but temporarily low availability (data is safe but momentarily unreachable).

## 9. Databases in the Cloud — With Scaling Patterns

**Managed relational (SQL):** RDS, Aurora, Azure SQL, Cloud SQL — strong consistency, ACID, joins; the provider handles patching, backups, failover.

**NoSQL types:** document (MongoDB, DynamoDB), key-value (Redis, DynamoDB), column-family (Cassandra), graph (Neo4j). Flexible schema, horizontal scale, often eventual consistency.

**Scaling patterns (deep, commonly probed):**
- **Vertical scaling** — bigger DB server. Simple but capped and often needs downtime.
- **Read replicas** — copies of the primary that serve read traffic, offloading the primary. Great for read-heavy apps; replicas are usually eventually consistent.
- **Replication** — keeping copies of data on multiple nodes for availability and read scaling. *Synchronous* (strong consistency, higher latency) vs *asynchronous* (lower latency, possible lag).
- **Sharding (horizontal partitioning)** — splitting data across multiple databases by a *shard key* (e.g., user ID range or hash). Each shard holds a subset. Scales writes and storage, but adds complexity (cross-shard queries, rebalancing, hot shards).
- **Caching** — putting a fast in-memory store (Redis, Memcached) in front of the DB to serve hot data and cut latency/load.
  - **Cache strategies:** *cache-aside* (app checks cache, falls back to DB, then populates), *write-through* (write to cache and DB together), *write-back* (write to cache, flush to DB later).
  - **Cache invalidation** — the hard part: keeping cached data fresh (TTL expiry, explicit invalidation).

**CAP theorem (for distributed data stores):** during a network **P**artition you can guarantee at most one of **C**onsistency (every read sees the latest write) or **A**vailability (every request gets a response). Since partitions are unavoidable at scale, you trade C vs A. Traditional RDBMS lean **CP**; many NoSQL stores lean **AP**. **BASE** (Basically Available, Soft state, Eventual consistency) is the NoSQL counterpart to ACID.

---

## PART E — NETWORKING (often under-prepared — study this)

## 10. Cloud Networking Internals

- **VPC (Virtual Private Cloud)** — your own logically isolated private network within the provider's cloud, with a chosen IP range (CIDR block).
- **Subnets** — subdivisions of a VPC's IP range, usually split into:
  - **Public subnet** — has a route to the internet (via an Internet Gateway); hosts things like load balancers/bastion hosts.
  - **Private subnet** — no direct inbound internet route; hosts databases/app servers.
- **Internet Gateway (IGW)** — connects a VPC to the public internet.
- **NAT Gateway** — lets instances in a *private* subnet reach out to the internet (for updates) while blocking unsolicited inbound connections.
- **Route table** — rules that decide where network traffic goes.
- **VPC Peering** — private connection between two VPCs.
- **VPN / Direct Connect (dedicated link)** — connect on-premise networks to the cloud (VPN over internet; Direct Connect is a dedicated private line — lower latency, more consistent).

**Security groups vs NACLs (classic follow-up):**
- **Security Group** — a *stateful* virtual firewall at the *instance* level. Stateful means if you allow an inbound request, the response is automatically allowed out. Only supports *allow* rules.
- **NACL (Network ACL)** — a *stateless* firewall at the *subnet* level. Stateless means inbound and outbound rules are evaluated independently (you must allow both directions). Supports *allow and deny* rules.

**DNS** — translates domain names to IP addresses (e.g., Amazon Route 53), and can also do health-based routing.

**CDN (Content Delivery Network)** — a network of geographically distributed **edge locations** that cache content close to users, cutting latency and offloading the origin server. *Example: CloudFront, Cloudflare, Akamai.*

**Load Balancers — with the actual algorithms:**
- **Layer 4 (transport/TCP)** load balancer — routes by IP/port, very fast, protocol-agnostic.
- **Layer 7 (application/HTTP)** load balancer — routes by content (URL path, host header, cookies); enables path-based routing to microservices.
- **Balancing algorithms:** *round robin* (each server in turn), *least connections* (send to the least busy), *IP hash* (same client → same server, for session stickiness), *weighted* (bigger servers get more traffic).
- **Health checks** — the LB stops sending traffic to instances that fail health checks, enabling high availability.

---

## PART F — RESILIENCE, SCALE & ARCHITECTURE

## 11. Scalability, Elasticity, Availability, Resilience

- **Scalability** — the ability to handle growing load by adding resources.
  - **Vertical (scale up)** — add CPU/RAM to one machine; simple, but a hardware ceiling and usually downtime.
  - **Horizontal (scale out)** — add more machines behind a load balancer; near-unlimited, the cloud-preferred approach; requires stateless design.
- **Elasticity** — automatic scaling *up and down* in real time with demand (auto-scaling). Scalability is the capability; elasticity is doing it automatically, both directions.
- **Availability** — proportion of time the system is operational, measured in "nines":
  - 99% ≈ 3.65 days downtime/year; 99.9% ≈ 8.76 hours; 99.99% ≈ 52.6 minutes; 99.999% ≈ 5.26 minutes.
- **Fault tolerance** — continues operating with *no interruption* when a component fails (stronger than HA, via full redundancy).
- **Resilience** — the system recovers gracefully from failures.
- **Redundancy** — duplicate components so there's no single point of failure.

**HA patterns:**
- **Availability Zones (AZs)** — isolated data centers within a region; deploy across multiple AZs so one facility failure doesn't take you down.
- **Regions** — geographically separate; multi-region deployment survives an entire region outage and reduces latency for global users.
- **Active-passive** (standby failover) vs **active-active** (all nodes serve traffic).

**Disaster Recovery (DR) — strategies from cheapest/slowest to most expensive/fastest:**
1. **Backup & Restore** — restore from backups after disaster (highest RTO/RPO).
2. **Pilot Light** — a minimal core always running, scaled up when needed.
3. **Warm Standby** — a scaled-down but running copy, quickly scaled up.
4. **Multi-site Active-Active** — full duplicate running live (lowest RTO/RPO, highest cost).
- **RTO (Recovery Time Objective)** — how fast you must recover. **RPO (Recovery Point Objective)** — how much data loss (in time) you can tolerate.

## 12. Serverless & Event-Driven Computing

**Serverless** — you run code/logic without provisioning or managing servers; the provider auto-scales (even to zero) and bills per execution.
- **FaaS** — event-triggered functions (AWS Lambda, Azure Functions). Triggered by HTTP requests, file uploads, queue messages, schedules.
- **Pros:** no server management, automatic fine-grained scaling, pay only for execution time, cheap for spiky/low-traffic workloads.
- **Cons:**
  - **Cold start** — first invocation after idle has startup latency (the runtime must spin up).
  - **Execution time limits** — not for long-running jobs.
  - **Statelessness** — functions shouldn't hold state between invocations.
  - **Vendor lock-in** and harder local debugging/observability.
- **Note:** "serverless" means *you* don't manage servers — they still exist.

## 13. Microservices & Modern Application Architecture

- **Monolith** — one large deployable unit. Simple to build and deploy initially; but scaling means duplicating the whole app, one bug can take everything down, and it's hard to evolve.
- **Microservices** — the app is decomposed into small, independent services, each owning one capability and its own data, communicating over APIs. Independently deployable and scalable; fault-isolated; teams work in parallel. Costs: distributed-systems complexity, network latency, data consistency across services, and harder debugging.

**Communication:**
- **Synchronous** — REST/gRPC request-response; simple but couples services in time (caller waits).
- **Asynchronous** — message queues / event streaming (RabbitMQ, Kafka, SQS); the sender doesn't wait, improving decoupling and resilience.
- **Message Queue** — buffers messages between producer and consumer, smoothing spikes and decoupling components.
- **Pub/Sub** — publishers emit events; multiple subscribers react independently.

**Supporting patterns:**
- **API Gateway** — single entry point for clients that routes to microservices, handling auth, rate limiting, and aggregation.
- **Service Mesh** (e.g., Istio) — infrastructure layer handling service-to-service communication, security (mTLS), and observability without app code changes.
- **Circuit breaker** — stops calling a failing service to prevent cascading failures.
- **Idempotency** — designing operations so retrying them doesn't cause duplicate effects (crucial with async retries).

---

## PART G — OPERATIONS, SECURITY & COST

## 14. DevOps, CI/CD & Infrastructure as Code

- **DevOps** — culture + practices merging Development and Operations to ship faster and more reliably via automation, collaboration, and feedback loops.
- **CI (Continuous Integration)** — developers merge frequently; every merge triggers automated build + tests, catching issues early.
- **CD (Continuous Delivery)** — code is always in a deployable state, released with a manual approval. **Continuous Deployment** — every passing change auto-deploys to production.
- **Pipeline stages (typical):** source → build → test → deploy → monitor.
- **IaC (Infrastructure as Code)** — define infrastructure in declarative config/code (Terraform, CloudFormation) so environments are versioned, reproducible, and reviewable. Eliminates manual "click-ops" drift.
- **GitOps** — Git as the single source of truth for infrastructure; changes flow through pull requests.
- **Blue-green deployment** — run two environments; switch traffic to the new one instantly, roll back by switching back.
- **Canary deployment** — release to a small % of users first, then ramp up if healthy.

## 15. Cloud Security — In Depth

**Shared Responsibility Model:** the provider is responsible for security **of** the cloud (physical facilities, hardware, host virtualization, network backbone); the customer is responsible for security **in** the cloud (their data, IAM, OS patching in IaaS, app code, network/firewall config, encryption). The exact split shifts by service model — in SaaS the provider covers most of it; in IaaS you cover much more.

**Identity & Access Management (IAM):**
- **Authentication** (who you are) vs **Authorization** (what you can do).
- **Principle of least privilege** — grant the minimum access needed.
- **Roles vs users** — roles grant temporary permissions to services/workloads (better than long-lived keys).
- **MFA** — additional verification factor beyond a password.

**Encryption:**
- **At rest** — data stored encrypted on disk.
- **In transit** — data encrypted while moving over the network (TLS/HTTPS).
- **KMS (Key Management Service)** — manages and protects encryption keys; envelope encryption (a data key encrypted by a master key).
- **Symmetric** (one shared key, fast) vs **asymmetric** (public/private key pair, used in TLS handshake).

**Network & perimeter security:**
- **Firewalls / security groups / NACLs** — control traffic (see Section 10).
- **WAF (Web Application Firewall)** — filters malicious HTTP traffic (SQL injection, XSS).
- **DDoS protection** — absorbs/deflects volumetric attacks (AWS Shield, Cloudflare).
- **Zero Trust** — "never trust, always verify"; authenticate and authorize every request regardless of network location, rather than trusting anything "inside the perimeter."

**Compliance & data residency:** regulations (GDPR, HIPAA, data-sovereignty laws) dictate where and how data is stored/processed — a major driver of private/hybrid cloud.

## 16. Cost Management (FinOps)

**Pricing models:**
- **On-demand / pay-as-you-go** — no commitment, most flexible, highest per-unit cost.
- **Reserved / committed use** — commit 1–3 years for a large discount; best for steady baseline load.
- **Spot / preemptible** — deeply discounted spare capacity that can be reclaimed anytime; ideal for fault-tolerant, interruptible batch jobs.

**Optimization techniques:** rightsizing (matching instance size to actual usage), auto-scaling, using spot for non-critical work, storage tiering/lifecycle policies, shutting down idle resources, and monitoring spend with budgets/alerts. **FinOps** is the discipline of managing cloud cost as a shared engineering responsibility. **TCO (Total Cost of Ownership)** compares the full cost of cloud vs on-prem including people, power, and maintenance — not just the sticker price.

---

## PART H — DESIGN, MIGRATION & OBSERVABILITY

## 17. Cloud Design Principles (Well-Architected Framework pillars)

A strong, senior-sounding framework. The pillars:
1. **Operational Excellence** — run and monitor systems, automate, improve continuously.
2. **Security** — protect data and systems (least privilege, encryption, traceability).
3. **Reliability** — recover from failure, scale to meet demand, avoid single points of failure.
4. **Performance Efficiency** — use resources efficiently, choose the right service, scale appropriately.
5. **Cost Optimization** — avoid unnecessary spend, match supply to demand.
6. **Sustainability** — minimize environmental impact (efficient resource use).

Core design mantras: **design for failure** (assume components will fail), **decouple components**, **make services stateless** (so they scale horizontally), **automate everything**, and **think elastically**.

## 18. Cloud Migration — The 6 R's

When moving existing systems to the cloud, each workload gets one strategy:
1. **Rehost ("lift and shift")** — move as-is to cloud VMs; fastest, least optimized.
2. **Replatform ("lift, tinker, and shift")** — minor optimizations (e.g., move to a managed DB) without rewriting.
3. **Repurchase** — switch to a different product, usually SaaS (e.g., drop a self-hosted CRM for Salesforce).
4. **Refactor / Re-architect** — redesign the app to be cloud-native (e.g., break a monolith into microservices/serverless); most effort, most benefit.
5. **Retire** — decommission what's no longer needed.
6. **Retain** — keep some workloads on-premise (often the hybrid case).

## 19. Monitoring & Observability

- **Monitoring** — tracking known metrics and alerting on thresholds.
- **Observability** — being able to understand a system's internal state from its outputs, so you can debug *unknown* problems. Its three pillars:
  - **Metrics** — numeric time-series (CPU, latency, request rate).
  - **Logs** — timestamped event records.
  - **Traces** — the path of a single request across services (distributed tracing), crucial for microservices.
- **SLI / SLO / SLA:** an **SLI** is a measured indicator (e.g., latency); an **SLO** is your internal target (e.g., 99.9% under 200ms); an **SLA** is the contractual promise to customers with penalties.

## 20. Edge Computing & Emerging Areas

**Edge computing** — processing data near where it's generated (IoT devices, sensors, local gateways) instead of round-tripping to a central cloud. Cuts latency and bandwidth, supports real-time use cases (autonomous systems, industrial IoT) and data-locality requirements. The cloud and edge work together: edge for immediate processing, cloud for heavy analytics and coordination. This is a major HPE focus area.

---

## PART I — HPE-SPECIFIC DEEP DIVE

**HPE GreenLake** — HPE's flagship, a **consumption-based, as-a-service platform** that delivers a cloud experience (self-service provisioning, pay-per-use metering, managed operations) for infrastructure that can live **on-premise, at the edge, or in a colocation facility** — not just in the public cloud. The pitch: cloud economics and agility *without* forcing every workload into a public cloud.

**Why HPE bets on hybrid:**
- **Data gravity** — large datasets are costly and slow to move; compute should come to the data.
- **Latency** — some workloads need to be physically close to users/machines.
- **Compliance & sovereignty** — regulated data may be required to stay in specific locations/on-prem.
- **Predictable cost** — steady workloads can be cheaper on owned/consumption infrastructure than metered public cloud.
- **Avoiding lock-in.**

**Relevant HPE portfolio context:** HPE Alletra (storage), HPE Cray (HPC/supercomputing for AI), HPE ProLiant (servers), and — after the Juniper acquisition — a major networking stack (Aruba + Juniper Mist AI) that underpins AI data-center fabrics. Tie your cloud interest to their hybrid + AI-infrastructure story.

---

## PART J — COMPREHENSIVE Q&A BANK

**Q: Define cloud computing and the shift it represents.**
On-demand delivery of computing resources over the internet on a pay-per-use basis. It shifts spending from CapEx (buying and owning hardware upfront) to OpEx (variable, usage-based cost) and decouples capacity from ownership.

**Q: What are the five NIST characteristics?**
On-demand self-service, broad network access, resource pooling (multi-tenancy), rapid elasticity, and measured service.

**Q: Explain IaaS vs PaaS vs SaaS by responsibility.**
IaaS: provider manages hardware and virtualization, you manage OS and up. PaaS: provider also manages OS/runtime, you manage only app and data. SaaS: provider manages everything, you just use the software.

**Q: Public vs private vs hybrid, and when to use each?**
Public: shared, cheapest, most scalable — good for variable/general workloads. Private: dedicated, controlled, compliant — good for sensitive/regulated workloads. Hybrid: both with orchestration — keep sensitive data private and burst to public for scale.

**Q: What is a hypervisor and the two types?**
Software that creates and isolates VMs and allocates hardware. Type 1 runs on bare metal (data centers); Type 2 runs on a host OS (desktops/testing).

**Q: Container vs VM — and why containers are lighter.**
A VM virtualizes hardware and includes a full guest OS (heavy, slow boot). A container virtualizes the OS, sharing the host kernel and packaging only the app and dependencies (light, fast boot), so you get much higher density.

**Q: What does Kubernetes do, and what's in its control plane?**
It orchestrates containers at scale — scheduling, scaling, self-healing, networking, rollouts. Control plane: API server (front door), etcd (state store), scheduler (placement), controller manager (reconciliation loops). Worker nodes run kubelet, kube-proxy, and the container runtime.

**Q: What is a Pod, and why do we need Services?**
A Pod is the smallest deployable unit — one or more containers sharing network/storage. Pods are ephemeral with changing IPs, so a Service provides a stable endpoint and load-balances across the matching pods.

**Q: Explain Kubernetes' declarative model.**
You declare desired state; Kubernetes continuously reconciles actual state toward it — restarting failed pods, rescheduling on node failure, and scaling replicas.

**Q: Object vs block vs file storage?**
Object: data as objects via API, unlimited and cheap, for media/backups (S3). Block: raw volumes attached to one VM like a disk, low latency, for databases (EBS). File: shared network file system for multiple machines (EFS).

**Q: Durability vs availability?**
Durability is the probability data isn't lost (via replication). Availability is the probability it's accessible right now (uptime). You can have high durability with temporarily low availability.

**Q: How do you scale a database?**
Vertically (bigger server, capped), read replicas (offload reads), replication (copies for availability), sharding (partition data by a shard key to scale writes/storage), and caching (in-memory layer like Redis for hot data).

**Q: What is sharding and its downside?**
Splitting data horizontally across multiple databases by a shard key. It scales writes and storage but complicates cross-shard queries, rebalancing, and can create hot shards.

**Q: Explain cache-aside vs write-through.**
Cache-aside: the app reads the cache, falls back to the DB on a miss, then populates the cache. Write-through: writes go to cache and DB together, keeping them consistent at the cost of write latency.

**Q: What is the CAP theorem?**
In a distributed store, during a network partition you can guarantee only one of consistency or availability. Since partitions are unavoidable, you trade C against A; RDBMS often choose CP, many NoSQL stores choose AP.

**Q: What is a VPC and why split into public/private subnets?**
A VPC is your isolated virtual network in the cloud. Public subnets (with an internet gateway route) host internet-facing components like load balancers; private subnets host databases/app servers with no direct inbound internet access, improving security.

**Q: Security group vs NACL?**
A security group is a stateful firewall at the instance level with allow-only rules (return traffic is auto-allowed). A NACL is a stateless firewall at the subnet level supporting allow and deny rules, evaluating inbound and outbound independently.

**Q: What does a NAT gateway do?**
It lets instances in a private subnet initiate outbound internet connections (e.g., for updates) while blocking unsolicited inbound connections.

**Q: How does a load balancer decide where to send traffic?**
Using algorithms like round robin, least connections, IP hash (session stickiness), or weighted distribution, and it uses health checks to avoid routing to unhealthy instances. Layer 4 balances by IP/port; Layer 7 balances by HTTP content like URL path.

**Q: What is a CDN and how does it help?**
A network of edge locations that cache content near users, reducing latency and offloading the origin server.

**Q: Horizontal vs vertical scaling, and which does cloud prefer?**
Vertical adds power to one machine (capped, downtime). Horizontal adds more machines behind a load balancer (near-unlimited, needs stateless design). Cloud prefers horizontal.

**Q: Scalability vs elasticity?**
Scalability is the ability to grow with load; elasticity is automatically scaling up and down in real time with demand.

**Q: What do the "nines" mean?**
Availability percentages: 99.9% is about 8.76 hours of downtime a year; 99.99% about 52 minutes; 99.999% about 5 minutes.

**Q: Availability Zone vs Region?**
An AZ is an isolated data center within a region; deploying across AZs survives a single facility failure. Regions are geographically separate; multi-region survives a whole-region outage and reduces global latency.

**Q: Explain DR strategies from cheapest to most robust.**
Backup & restore, pilot light, warm standby, and multi-site active-active — trading lower cost/higher RTO-RPO for higher cost/near-zero RTO-RPO.

**Q: RTO vs RPO?**
RTO is how fast you must recover after an outage; RPO is how much data loss (in time) you can tolerate.

**Q: What is serverless, and its main drawbacks?**
Running code without managing servers, auto-scaled and billed per execution (e.g., Lambda). Drawbacks: cold-start latency, execution time limits, statelessness, and vendor lock-in.

**Q: Monolith vs microservices — trade-offs?**
Monolith: simple to build/deploy but hard to scale selectively and fragile to single failures. Microservices: independently deployable/scalable and fault-isolated, but add distributed-systems complexity, latency, and data-consistency challenges.

**Q: Synchronous vs asynchronous communication?**
Synchronous (REST/gRPC) is request-response and couples services in time. Asynchronous (message queues/events) decouples them, improving resilience and smoothing spikes.

**Q: What is an API Gateway?**
A single entry point for clients that routes requests to backend microservices and handles cross-cutting concerns like auth, rate limiting, and aggregation.

**Q: Explain the shared responsibility model.**
The provider secures the cloud (physical, hardware, infrastructure); the customer secures what's in the cloud (data, IAM, config, app). The split depends on the service model.

**Q: What is the principle of least privilege?**
Granting each identity only the minimum permissions needed to do its job, reducing the blast radius if credentials are compromised.

**Q: Encryption at rest vs in transit?**
At rest protects stored data on disk; in transit protects data moving over the network using TLS/HTTPS.

**Q: What is Zero Trust?**
A security model that trusts nothing by default and verifies every request regardless of network location, instead of assuming anything inside the perimeter is safe.

**Q: On-demand vs reserved vs spot pricing?**
On-demand: flexible, no commitment, priciest per unit. Reserved: 1–3 year commitment for a discount, for steady load. Spot: cheap spare capacity that can be reclaimed anytime, for interruptible workloads.

**Q: What is IaC and why does it matter?**
Defining infrastructure in versioned, declarative code (Terraform, CloudFormation) so environments are reproducible, reviewable, and free of manual configuration drift.

**Q: Blue-green vs canary deployment?**
Blue-green switches all traffic between two full environments (instant rollback by switching back). Canary releases to a small percentage first and ramps up if healthy.

**Q: What are the Well-Architected pillars?**
Operational excellence, security, reliability, performance efficiency, cost optimization, and sustainability.

**Q: Name the 6 R's of migration.**
Rehost, replatform, repurchase, refactor, retire, retain.

**Q: Metrics vs logs vs traces?**
Metrics are numeric time-series; logs are event records; traces follow a single request across services. Together they provide observability.

**Q: SLI vs SLO vs SLA?**
An SLI is a measured indicator, an SLO is your internal target for it, and an SLA is the contractual commitment to customers with penalties.

**Q: What is edge computing and why does it matter?**
Processing data near where it's generated instead of sending it all to a central cloud, reducing latency and bandwidth and supporting real-time and data-locality needs.

**Q: What is HPE GreenLake? (HPE-specific)**
HPE's consumption-based, as-a-service platform that brings a cloud experience — self-service and pay-per-use — to on-premise, edge, and colocation infrastructure, for customers needing control, low latency, or compliance.

**Q: Why would an enterprise choose hybrid over pure public cloud?**
Data gravity, latency, compliance and data sovereignty, predictable cost for steady workloads, and avoiding vendor lock-in — keep sensitive/steady workloads private and burst to public for scale.

---

## PART K — RAPID-RECALL CHEAT SHEET
- CapEx→OpEx; 5 NIST traits: on-demand, broad access, pooling, elasticity, measured.
- IaaS/PaaS/SaaS = responsibility ladder. Public/Private/Hybrid/Community/Multi.
- Virtualization → hypervisor (Type 1 bare-metal / Type 2 hosted).
- VM virtualizes hardware; container virtualizes OS. Docker builds; Kubernetes orchestrates (control plane: API server, etcd, scheduler, controllers).
- Storage: object (S3) / block (EBS) / file (EFS). Durability ≠ availability.
- DB scaling: vertical, read replicas, replication, sharding, caching. CAP: pick C or A under partition.
- Networking: VPC → subnets (public/private), IGW, NAT, route tables. Security group = stateful/instance; NACL = stateless/subnet. LB L4 vs L7; algorithms: round robin, least connections, IP hash, weighted.
- Scale horizontally + stateless. Nines = uptime. AZ vs Region. DR: backup→pilot light→warm standby→active-active.
- Serverless = no server mgmt, pay-per-run, cold starts. Micro vs monolith; sync vs async (queues).
- DevOps + CI/CD + IaC. Blue-green vs canary.
- Security = shared responsibility; least privilege; encrypt at rest + in transit; Zero Trust.
- Cost: on-demand/reserved/spot; FinOps; TCO.
- Well-Architected: OpEx-excellence, security, reliability, performance, cost, sustainability.
- Migration 6 R's. Observability = metrics/logs/traces. SLI/SLO/SLA.
- HPE = hybrid + GreenLake (consumption on-prem/edge) + AI infrastructure. Drivers: data gravity, latency, compliance, cost, lock-in.
