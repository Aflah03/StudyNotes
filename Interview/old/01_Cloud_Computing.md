# Cloud Computing — Conceptual Interview Guide (HPE)

> How to use this: Read top to bottom once for understanding, then use the **Interview Q&A** section at the end to self-test. Every core term is defined the way you'd say it out loud to an interviewer — short, correct, and confident. HPE's own cloud product (**GreenLake**) is covered near the end, because interviewers love it when you know their product.

---

## 1. What is Cloud Computing?

**One-line answer:** Cloud computing is the on-demand delivery of computing resources — servers, storage, databases, networking, software — over the internet, on a pay-as-you-go basis, instead of owning and maintaining physical hardware yourself.

The key idea is that you rent computing instead of buying it. A cloud provider (AWS, Azure, Google Cloud, HPE) owns huge data centers, and you consume slices of that capacity when you need them.

**Why it matters (the business case):**
- **No upfront capital cost** — you don't buy servers; you pay operating expense (OpEx not CapEx).
- **Speed** — spin up a server in minutes instead of weeks.
- **Scale on demand** — grow when traffic spikes, shrink when it drops.
- **Global reach** — deploy in data centers around the world instantly.
- **Managed maintenance** — the provider handles hardware failures, power, cooling, and often patching.

---

## 2. The 5 Essential Characteristics (NIST definition)

Interviewers sometimes ask "what defines the cloud?" The standard answer is the five characteristics from the NIST definition:

1. **On-demand self-service** — you provision resources yourself, automatically, without human interaction from the provider.
2. **Broad network access** — resources are available over the network from any standard device.
3. **Resource pooling** — the provider's resources are pooled and shared among many customers (multi-tenancy), dynamically assigned.
4. **Rapid elasticity** — resources can scale out and in quickly, appearing "unlimited" to the customer.
5. **Measured service** — usage is metered, so you pay only for what you consume.

---

## 3. Service Models: IaaS, PaaS, SaaS (MOST asked topic)

Think of it as **how much the provider manages vs. how much you manage.** The classic analogy is **Pizza as a Service.**

| Model | You manage | Provider manages | Example |
|-------|-----------|------------------|---------|
| **On-premise** | Everything | Nothing | Your own data center |
| **IaaS** (Infrastructure) | OS, runtime, apps, data | Servers, storage, networking, virtualization | AWS EC2, Azure VMs |
| **PaaS** (Platform) | Apps, data | Everything below the app (OS, runtime, scaling) | Google App Engine, Heroku, AWS Elastic Beanstalk |
| **SaaS** (Software) | Just use it | Everything | Gmail, Salesforce, Office 365 |

**How to explain each:**

- **IaaS** — You get raw virtual machines, storage, and networks. You install the OS and everything above it. Maximum control, maximum responsibility. Use when you need custom infrastructure.
- **PaaS** — You get a ready platform to deploy your code. You don't manage servers or OS patching; you just push your application. Use when developers want to focus only on code.
- **SaaS** — Fully finished software delivered over the internet. You just log in and use it. No installation, no maintenance.

**The Pizza analogy (great to say aloud):**
- On-premise = make pizza at home (you buy everything).
- IaaS = take-and-bake (they provide ingredients, you cook).
- PaaS = pizza delivery (they cook, you provide the table and drinks).
- SaaS = dining out (they do everything).

---

## 4. Deployment Models

- **Public Cloud** — Infrastructure owned by a third-party provider and shared across many customers over the public internet (AWS, Azure, GCP). Cheapest, most scalable, least control.
- **Private Cloud** — Cloud infrastructure dedicated to a single organization, either on-premise or hosted. More control and security, higher cost. (HPE specializes heavily here.)
- **Hybrid Cloud** — A mix of public and private, with orchestration between them. Keep sensitive data private, burst to public cloud for scale. Very common in enterprises.
- **Community Cloud** — Shared by several organizations with common concerns (e.g., banks under the same regulation).
- **Multi-cloud** — Using more than one public cloud provider (e.g., AWS + Azure) to avoid vendor lock-in.

> **HPE angle:** HPE's whole strategy is *hybrid cloud* — bringing a cloud-like experience to on-premise and private infrastructure. If asked "why hybrid?", say: data sovereignty/compliance, latency, cost predictability, and avoiding lock-in.

---

## 5. Virtualization (the foundation of the cloud)

**What it is:** Virtualization is the technology that lets one physical machine run multiple isolated virtual machines (VMs), each with its own OS. It's what makes resource pooling possible.

- **Hypervisor** — software that creates and runs VMs by sitting between hardware and the VMs, allocating resources.
  - **Type 1 (bare-metal)** — runs directly on hardware. Faster, used in data centers. Examples: VMware ESXi, Microsoft Hyper-V, KVM, Xen.
  - **Type 2 (hosted)** — runs on top of a host OS. Used on personal machines. Examples: VirtualBox, VMware Workstation.

**Why virtualization enables cloud:** It lets providers slice one big physical server into many VMs, giving each customer isolation while maximizing hardware utilization.

---

## 6. Containers vs. Virtual Machines (very common question)

| Aspect | Virtual Machine | Container |
|--------|-----------------|-----------|
| Isolation level | Full OS per VM | Shares host OS kernel |
| Size | GBs (includes full OS) | MBs (just app + dependencies) |
| Startup time | Minutes | Seconds/milliseconds |
| Overhead | High | Low |
| Portability | Less portable | Highly portable |

**Key line:** A VM virtualizes the *hardware* (each VM has its own OS). A container virtualizes the *operating system* (containers share the host kernel but isolate the application). Containers are lighter and start faster.

- **Docker** — the most popular tool for building and running containers.
- **Kubernetes (K8s)** — the orchestration platform that manages containers at scale: scheduling, scaling, self-healing, load balancing across many machines.

---

## 7. Scalability, Elasticity & Availability

These terms get mixed up. Be precise:

- **Scalability** — the ability to handle growing load by adding resources.
  - **Vertical scaling (scale up)** — add more power (CPU/RAM) to a single machine. Limited by hardware ceiling; usually needs downtime.
  - **Horizontal scaling (scale out)** — add more machines and distribute the load. Preferred in cloud; near-unlimited.
- **Elasticity** — automatic scaling *up and down* in real time based on demand. (Auto-scaling groups do this.) Scalability is the capability; elasticity is doing it automatically and both directions.
- **High Availability (HA)** — the system stays up despite failures, usually via redundancy (multiple servers, multiple zones). Measured in "nines" (99.99% uptime).
- **Fault Tolerance** — the system keeps working *with no interruption* even when a component fails (stronger than HA).
- **Disaster Recovery (DR)** — the plan/process to restore service after a major outage. Key metrics: **RTO** (Recovery Time Objective — how fast you recover) and **RPO** (Recovery Point Objective — how much data you can afford to lose).

- **Load Balancer** — distributes incoming traffic across multiple servers so no single one is overwhelmed. Enables both HA and horizontal scaling.

---

## 8. Core Cloud Service Categories

**Compute** — running your code/apps.
- Virtual machines (AWS EC2, Azure VM)
- Containers (ECS, EKS, AKS, GKE)
- Serverless functions (AWS Lambda, Azure Functions)

**Storage** — three main types:
- **Object storage** — stores files as objects with metadata, accessed via API. Massively scalable, cheap. Best for images, backups, static files. (AWS S3, Azure Blob.)
- **Block storage** — raw volumes attached to a VM like a hard disk. Low latency, for databases and OS disks. (AWS EBS.)
- **File storage** — shared file system accessed over a network (NFS/SMB). (AWS EFS.)

**Database** — managed databases:
- **Relational (SQL)** — RDS, Aurora, Azure SQL.
- **NoSQL** — DynamoDB, MongoDB Atlas, Cosmos DB.

**Networking:**
- **VPC (Virtual Private Cloud)** — your own isolated private network inside the cloud.
- **Subnets** — segments of a VPC (public vs private).
- **CDN (Content Delivery Network)** — caches content at edge locations near users to reduce latency (CloudFront, Cloudflare).
- **DNS** — maps domain names to IPs (Route 53).

---

## 9. Serverless Computing

**What it is:** You run code without managing any servers. The provider automatically provisions, scales, and bills only for the exact execution time (down to milliseconds).

- **FaaS (Function as a Service)** — AWS Lambda, Azure Functions. Your code runs in response to events (HTTP request, file upload, queue message).
- **Pros:** no server management, automatic scaling, pay-per-execution, cheap for spiky workloads.
- **Cons:** "cold start" latency, execution time limits, harder to debug, potential vendor lock-in.

Note the irony to mention: "serverless doesn't mean no servers — it means *you* don't manage them."

---

## 10. Microservices & Modern Architecture

- **Monolithic architecture** — the entire application is one single codebase/deployment. Simple to start, hard to scale and maintain as it grows.
- **Microservices architecture** — the app is split into small, independent services that each do one thing and communicate over APIs. Each can be developed, deployed, and scaled independently.
  - **Pros:** independent scaling and deployment, technology flexibility, fault isolation.
  - **Cons:** operational complexity, network overhead, harder debugging, data consistency challenges.

- **API (Application Programming Interface)** — the contract that lets services/apps talk to each other. **REST** (uses HTTP, stateless, JSON) is the most common style.

---

## 11. DevOps & CI/CD

- **DevOps** — a culture/practice combining Development and Operations to deliver software faster and more reliably through automation and collaboration.
- **CI (Continuous Integration)** — developers merge code frequently; automated builds and tests run on every merge.
- **CD (Continuous Delivery/Deployment)** — code changes are automatically prepared for (and, in deployment, pushed to) production.
- **IaC (Infrastructure as Code)** — managing infrastructure through code/config files instead of manual setup (Terraform, CloudFormation). Makes environments reproducible.

---

## 12. Cloud Security — The Shared Responsibility Model

This is a favorite question. The key concept:

**Security *of* the cloud** is the provider's job (physical data centers, hardware, host OS, network infrastructure).

**Security *in* the cloud** is the customer's job (your data, access management, OS patching in IaaS, application code, network config).

The split shifts by service model: in SaaS the provider handles almost everything; in IaaS you handle much more.

**Key security concepts to name:**
- **IAM (Identity and Access Management)** — controls who can access what. Follow the **principle of least privilege** (give minimum access needed).
- **Encryption** — at rest (stored data) and in transit (data moving over the network, via TLS/HTTPS).
- **Firewalls / Security Groups** — control inbound/outbound traffic.
- **MFA (Multi-Factor Authentication)** — extra verification beyond passwords.

---

## 13. Cost Models

- **Pay-as-you-go / on-demand** — pay per use, no commitment. Flexible, most expensive per unit.
- **Reserved instances** — commit to 1–3 years for a big discount. Good for steady workloads.
- **Spot instances** — bid on spare capacity, very cheap, but can be reclaimed anytime. Good for fault-tolerant batch jobs.
- **CapEx vs OpEx** — cloud shifts spending from capital expenditure (buying hardware upfront) to operational expenditure (ongoing usage cost).

---

## 14. HPE-Specific Cloud Knowledge (bonus points)

HPE's flagship cloud offering is **HPE GreenLake** — an "as-a-service" / consumption-based platform that brings the **cloud experience to on-premise and hybrid environments.** The idea: you get cloud-style, pay-per-use billing and self-service, but the infrastructure can sit in your own data center for control, compliance, and low latency.

Talking points if HPE cloud comes up:
- HPE bets on **hybrid cloud** — not every workload belongs in the public cloud (data gravity, latency, compliance, cost).
- GreenLake is **consumption-based** — pay for what you use, even on-prem.
- It addresses **data sovereignty** and **edge computing** needs.

**Edge computing** (worth knowing): processing data close to where it's generated (IoT devices, sensors) instead of sending everything to a central cloud. Reduces latency and bandwidth. HPE is active in this space.

---

## 15. Interview Q&A Bank (self-test)

**Q: What is cloud computing in one sentence?**
On-demand delivery of computing resources over the internet on a pay-per-use basis, instead of owning physical hardware.

**Q: Difference between IaaS, PaaS, and SaaS?**
It's about the level of management. IaaS gives raw infrastructure (you manage OS and up). PaaS gives a platform to deploy code (you manage only the app and data). SaaS gives finished software (you just use it).

**Q: Public vs Private vs Hybrid cloud?**
Public is shared, third-party managed, cheapest. Private is dedicated to one organization, more control and security. Hybrid combines both with orchestration between them.

**Q: VM vs Container?**
A VM virtualizes hardware and runs a full OS per instance (heavy, slower boot). A container shares the host OS kernel and isolates just the app (lightweight, fast boot). Docker builds containers; Kubernetes orchestrates them.

**Q: What is a hypervisor?**
Software that creates and manages VMs, allocating hardware resources. Type 1 runs on bare metal (data centers); Type 2 runs on a host OS (personal use).

**Q: Horizontal vs vertical scaling?**
Vertical = add more power to one machine (scale up, has a ceiling). Horizontal = add more machines and distribute load (scale out, preferred in cloud).

**Q: Scalability vs elasticity?**
Scalability is the ability to grow with load. Elasticity is automatically scaling up *and down* in real time based on demand.

**Q: What is high availability vs fault tolerance?**
HA keeps the system up despite failures, usually via redundancy, with minimal downtime. Fault tolerance is stronger — zero interruption when a component fails.

**Q: What is serverless computing?**
Running code without managing servers; the provider auto-scales and you pay per execution (e.g., AWS Lambda). Downside: cold starts and time limits.

**Q: Object vs block vs file storage?**
Object = files as objects via API, massively scalable (S3), for images/backups. Block = raw volumes attached to a VM like a disk (EBS), for databases. File = shared network file system (EFS).

**Q: Explain the shared responsibility model.**
The provider secures the cloud (physical, hardware, infrastructure). The customer secures what's *in* the cloud (data, access, config, app). The split depends on the service model.

**Q: What is a load balancer?**
It distributes incoming traffic across multiple servers to prevent overload and enable high availability and horizontal scaling.

**Q: Monolith vs microservices?**
A monolith is one single deployable unit — simple but hard to scale. Microservices split the app into small independent services communicating via APIs — independently scalable but operationally complex.

**Q: What is a CDN?**
A network of edge servers that cache content close to users, reducing latency and load on the origin server.

**Q: What is auto-scaling?**
Automatically adding or removing compute instances based on real-time demand/metrics like CPU usage.

**Q: What is multi-tenancy?**
Multiple customers sharing the same physical infrastructure while being logically isolated from each other.

**Q: RTO vs RPO?**
RTO is how quickly you must recover after an outage. RPO is how much data loss (measured in time) you can tolerate.

**Q: What is HPE GreenLake?** *(HPE-specific)*
HPE's consumption-based, as-a-service platform that brings a cloud experience — pay-per-use, self-service — to on-premise and hybrid environments, for customers who need control, compliance, or low latency.

**Q: Why would a company choose hybrid cloud over pure public cloud?**
Data sovereignty and compliance, latency-sensitive workloads, predictable costs for steady loads, avoiding vendor lock-in, and keeping sensitive data on-premise while bursting to public cloud for scale.

---

### Quick-recall cheat sheet
- 5 characteristics: on-demand, broad access, resource pooling, elasticity, measured service.
- 3 service models: IaaS, PaaS, SaaS.
- 4 deployment models: public, private, hybrid, community.
- 2 scaling directions: vertical (up), horizontal (out).
- Security = shared responsibility. Provider secures *of* the cloud; you secure *in* the cloud.
- HPE = hybrid cloud + GreenLake + edge.
