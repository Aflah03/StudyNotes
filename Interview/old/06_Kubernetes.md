# Kubernetes (K8s) — Learn-From-Scratch & Interview Guide

> A teaching guide, not just a cheat sheet. It builds up from *why* Kubernetes exists to *how* each piece works, with example YAML and the `kubectl` commands you'd actually run. Read top to bottom to learn it; use the Q&A bank and cheat sheet at the end to revise.
>
> **Context for your HPE interview:** a fresher round is unlikely to grill you deeply on K8s internals, but containers/orchestration is core to HPE's world and shows up in cloud discussions. Knowing the architecture and the main objects (Pod, Deployment, Service) confidently is the realistic target; the deeper sections are for genuine understanding.

---

## PART A — THE FUNDAMENTALS

## 1. What Is Kubernetes and Why Does It Exist?

**One-line answer:** Kubernetes is an open-source **container orchestration platform** that automates the deployment, scaling, networking, and management of containerized applications across a cluster of machines.

**The problem it solves.** Containers (via Docker) let you package an app with its dependencies so it runs consistently anywhere. That's great for *one* container on *one* machine. But real systems run hundreds of containers across many machines, and then hard questions appear:
- Which machine should each container run on?
- What happens when a container crashes, or a whole machine dies?
- How do containers find and talk to each other when they keep moving and their IPs change?
- How do you scale up during a traffic spike and back down after?
- How do you roll out a new version without downtime, and roll back if it breaks?

Doing all this by hand doesn't scale. **Kubernetes automates it.** You tell it the *desired state* ("run 5 copies of this app, expose it on port 80, keep it healthy"), and Kubernetes continuously makes reality match — this is its defining idea.

**History note:** Kubernetes grew out of Google's internal system "Borg," was open-sourced in 2014, and is now governed by the **CNCF (Cloud Native Computing Foundation)**. "K8s" = K + 8 letters + s.

**The core principle — declarative + reconciliation.** You don't give step-by-step commands (imperative); you *declare* the end state in a manifest, and Kubernetes' **control loops** constantly compare desired vs actual state and act to close the gap. If you say "5 replicas" and one pod dies, Kubernetes notices actual=4, desired=5, and starts a new one. This is **self-healing**.

---

## 2. Containers Recap (the layer below K8s)

Kubernetes orchestrates containers, so a quick recap:
- A **container** packages an app plus its dependencies, sharing the host OS kernel — lightweight and fast compared to a VM (which ships a full OS).
- **Docker** builds container **images** (immutable templates) and runs containers from them; images live in a **registry** (Docker Hub, Amazon ECR).
- Kubernetes itself is runtime-agnostic — it talks to a container runtime through the **CRI (Container Runtime Interface)**; common runtimes are **containerd** and **CRI-O**. (Docker as a runtime was deprecated in K8s 1.20+, but Docker-built images still run fine — they follow the OCI standard.)

---

## PART B — ARCHITECTURE

## 3. Cluster Architecture — Control Plane + Worker Nodes

A Kubernetes **cluster** = a set of machines (**nodes**) split into the **control plane** (the brain that makes decisions) and **worker nodes** (where your app containers actually run).

### Control Plane components (the brain)
- **kube-apiserver** — the front door of the cluster. Every command (from `kubectl`, controllers, nodes) goes through the API server. It validates requests and is the only component that talks to etcd.
- **etcd** — a distributed, consistent key-value store that holds the *entire cluster state* (the single source of truth). If etcd is lost without backup, the cluster's state is lost — so it's backed up carefully.
- **kube-scheduler** — decides *which node* a newly created pod should run on, based on resource requests, constraints, affinity rules, and taints.
- **kube-controller-manager** — runs the **control loops** (controllers) that drive actual state toward desired state — e.g., the node controller, replication controller, endpoints controller.
- **cloud-controller-manager** — integrates with the underlying cloud provider (creating load balancers, volumes, etc.).

### Worker Node components (where work runs)
- **kubelet** — the agent on each node. It takes pod specs from the API server and ensures the described containers are running and healthy on its node.
- **kube-proxy** — maintains network rules on each node so traffic can reach the right pods; implements the Service abstraction (routing/load-balancing).
- **Container runtime** — the software that actually runs containers (containerd, CRI-O).

**The flow when you deploy something:** you `kubectl apply` a manifest → API server validates and stores desired state in etcd → the scheduler picks nodes for the pods → the kubelet on each chosen node tells the runtime to start the containers → controllers keep watching and reconciling.

---

## PART C — CORE OBJECTS (WORKLOADS)

## 4. Pods — The Smallest Unit

A **Pod** is the smallest deployable unit in Kubernetes — *not* a container. A pod wraps **one or more containers** that are tightly coupled and share:
- the same **network namespace** (same IP address; they reach each other via `localhost`),
- the same **storage volumes**,
- the same lifecycle (scheduled together on one node).

Most pods have a single container; multi-container pods use patterns like the **sidecar** (a helper container, e.g., a logging or proxy agent alongside the main app).

**Crucial property — pods are ephemeral and disposable.** They can die and be replaced at any time, and a replacement gets a *new IP*. You rarely create bare pods directly; you let higher-level controllers manage them (so they get recreated on failure). Because IPs change, you never talk to a pod by its IP — you use a **Service** (Section 7).

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  labels:
    app: web
spec:
  containers:
    - name: nginx
      image: nginx:1.27
      ports:
        - containerPort: 80
```

## 5. The Workload Controllers

You almost always manage pods through a controller, not directly:

- **ReplicaSet** — ensures a specified number of identical pod replicas are running at all times. If one dies, it makes another. (You usually don't create this directly — a Deployment manages it for you.)
- **Deployment** — the most common workload object for **stateless** apps. It manages ReplicaSets and adds **rolling updates and rollbacks**. You update the image, and it gradually replaces old pods with new ones with zero downtime; if something's wrong you roll back to a previous version.
- **StatefulSet** — for **stateful** apps that need stable, unique identities and stable storage (databases, message brokers). Pods get stable names (`web-0`, `web-1`), are created/deleted in order, and keep their own persistent volume.
- **DaemonSet** — ensures a copy of a pod runs on **every node** (or a subset). Used for node-level agents: log collectors, monitoring agents, network plugins.
- **Job** — runs pods to **completion** for a batch task (runs once, ensures success), then stops.
- **CronJob** — runs Jobs on a **schedule** (like cron), e.g., a nightly backup.

**Deployment vs StatefulSet (common question):** Deployments are for stateless, interchangeable pods (any replica is as good as another). StatefulSets are for stateful pods that need stable identity and storage and ordered operations.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-deployment
spec:
  replicas: 3                    # desired state: 3 pods
  selector:
    matchLabels:
      app: web
  template:                      # the pod template
    metadata:
      labels:
        app: web
    spec:
      containers:
        - name: nginx
          image: nginx:1.27
          ports:
            - containerPort: 80
```

## 6. Labels, Selectors & Namespaces

- **Labels** — key-value tags attached to objects (`app: web`, `env: prod`). They're how Kubernetes groups and finds things.
- **Selectors** — queries that match labels. A Service or Deployment uses a selector to know which pods it manages/routes to. This loose coupling via labels is fundamental to how K8s wires things together.
- **Annotations** — like labels but for non-identifying metadata (not used for selection).
- **Namespaces** — virtual clusters within a physical cluster, used to isolate teams/environments (e.g., `dev`, `staging`, `prod`) and to scope names and resource quotas.

---

## PART D — NETWORKING

## 7. Services — Stable Networking for Ephemeral Pods

Because pods come and go with changing IPs, a **Service** provides a **stable IP and DNS name** that load-balances across a dynamic set of pods (chosen by a label selector). Clients talk to the Service; the Service routes to healthy pods.

**Service types:**
- **ClusterIP (default)** — exposes the Service on an internal IP reachable *only inside the cluster*. For internal service-to-service communication.
- **NodePort** — opens a static port on every node's IP; external traffic to `NodeIP:NodePort` reaches the Service. Simple external access, mostly for dev/testing.
- **LoadBalancer** — provisions an external cloud load balancer (on AWS/Azure/GCP) with a public IP pointing to the Service. The standard way to expose a service to the internet in the cloud.
- **ExternalName** — maps the Service to an external DNS name (a CNAME), for referencing external services.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-service
spec:
  type: ClusterIP
  selector:
    app: web            # routes to pods labeled app=web
  ports:
    - port: 80          # the Service's port
      targetPort: 80    # the pod's port
```

## 8. Ingress — Smart HTTP Routing

Creating a LoadBalancer per service gets expensive and doesn't do URL-based routing. **Ingress** is an object that manages **external HTTP(S) access**, providing host- and path-based routing (`/api` → one service, `/shop` → another), TLS termination, and a single entry point.

Ingress needs an **Ingress Controller** (e.g., NGINX Ingress, Traefik) actually running in the cluster to implement the rules — the Ingress object alone does nothing without a controller.

## 9. The Kubernetes Network Model & DNS

- **Flat network model:** every pod gets its own IP, and every pod can reach every other pod directly *without NAT*, across nodes. This is implemented by a **CNI (Container Network Interface)** plugin (Calico, Flannel, Cilium).
- **Cluster DNS (CoreDNS):** Services get DNS names, so pods reach a service by name (e.g., `web-service.default.svc.cluster.local`) instead of an IP.
- **NetworkPolicy** — rules that control which pods can talk to which (a pod-level firewall). By default all pods can talk to each other; NetworkPolicies restrict that for security.

---

## PART E — CONFIGURATION & STORAGE

## 10. ConfigMaps & Secrets

Keep configuration out of your container images so the same image works across environments:
- **ConfigMap** — stores non-sensitive config as key-value pairs (URLs, feature flags), injected into pods as environment variables or mounted files.
- **Secret** — stores sensitive data (passwords, API keys, TLS certs). Similar to ConfigMaps but intended for confidential data. **Note:** by default Secrets are only **base64-encoded, not encrypted** — you should enable encryption at rest and RBAC to protect them properly.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  DB_HOST: "db.internal"
  LOG_LEVEL: "info"
```

## 11. Storage — Volumes, PV, PVC, StorageClass

Containers have ephemeral filesystems — data is lost when a pod restarts. Kubernetes storage fixes this:
- **Volume** — storage attached to a pod, outliving container restarts but tied to the pod's lifecycle (e.g., `emptyDir` is temporary).
- **PersistentVolume (PV)** — a piece of storage in the cluster provisioned by an admin or dynamically; exists independently of any pod.
- **PersistentVolumeClaim (PVC)** — a *request* for storage by a user/pod ("I need 10Gi, read-write"). Kubernetes binds the claim to a matching PV. This decouples *who needs storage* from *how it's provisioned*.
- **StorageClass** — defines a "class" of storage (e.g., SSD vs HDD, which cloud disk type) and enables **dynamic provisioning** — a PVC automatically creates a PV on demand, no manual admin step.

**Mental model:** a Pod uses a PVC (the request), which binds to a PV (the actual storage), whose type is defined by a StorageClass.

---

## PART F — SCALING, SCHEDULING & HEALTH

## 12. Scaling

- **Manual scaling** — set `replicas` on a Deployment (or `kubectl scale`).
- **Horizontal Pod Autoscaler (HPA)** — automatically adds/removes *pod replicas* based on observed metrics (CPU, memory, or custom metrics). The most common autoscaler.
- **Vertical Pod Autoscaler (VPA)** — automatically adjusts the *CPU/memory requests* of pods (makes each pod bigger/smaller).
- **Cluster Autoscaler** — adds/removes *nodes* in the cluster when pods can't be scheduled (not enough capacity) or nodes are underused.

**How they combine:** HPA scales pods out; when there aren't enough nodes to place them, the Cluster Autoscaler adds nodes.

## 13. Scheduling — How Pods Land on Nodes

The **scheduler** places each pod on a node in two phases: **filtering** (which nodes *can* run this pod — enough resources, matches constraints) and **scoring** (which of those is *best*).

Controls you can use:
- **Resource requests & limits** — `requests` is what a container is guaranteed (used for scheduling); `limits` is the maximum it may use (throttled/killed if exceeded). Setting these well is critical for stability.
- **nodeSelector / Node Affinity** — steer pods toward nodes with certain labels (e.g., GPU nodes).
- **Taints & Tolerations** — a *taint* on a node repels pods; only pods with a matching *toleration* can be scheduled there (e.g., reserve GPU nodes for ML workloads).
- **Pod Affinity/Anti-Affinity** — schedule pods near (or away from) other pods (e.g., spread replicas across nodes for availability).

## 14. Health Checks (Probes) & Self-Healing

The kubelet uses **probes** to know a container's state:
- **Liveness probe** — "is the app still alive?" If it fails, Kubernetes **restarts** the container (recovers from deadlocks/hangs).
- **Readiness probe** — "is the app ready to receive traffic?" If it fails, the pod is **removed from Service endpoints** (no traffic sent) until it's ready — but not restarted. Prevents sending requests to a pod that's still starting up.
- **Startup probe** — gives slow-starting apps time to boot before liveness checks kick in.

**Self-healing** in Kubernetes = probes + controllers: crashed containers are restarted, failed pods are recreated by their controller, and pods on a dead node are rescheduled elsewhere.

---

## PART G — DEPLOYMENTS, RELEASES & OPERATIONS

## 15. Deployment Strategies

- **Rolling update (default)** — gradually replaces old pods with new ones (controlled by `maxSurge`/`maxUnavailable`), so there's no downtime. Roll back with `kubectl rollout undo`.
- **Recreate** — kills all old pods, then creates new ones (causes downtime; used when versions can't coexist).
- **Blue-green** — run the new version (green) alongside the old (blue), then switch the Service to green; instant rollback by switching back. (Not a built-in strategy — you implement it with Services/labels.)
- **Canary** — send a small percentage of traffic to the new version, ramp up if healthy. (Implemented with multiple deployments or a service mesh / ingress weighting.)

## 16. RBAC & Security Basics

- **RBAC (Role-Based Access Control)** — controls who/what can do which actions on which resources. Key objects: **Role**/**ClusterRole** (a set of permissions) bound to a subject via **RoleBinding**/**ClusterRoleBinding**.
- **ServiceAccount** — an identity for pods/workloads (so an app can authenticate to the API server), distinct from human user accounts.
- **Least privilege**, **NetworkPolicies**, **Secret encryption at rest**, **Pod Security Standards** (restricting what pods can do, e.g., no running as root) are the core security practices.

## 17. The Ecosystem (good to name)

- **Helm** — the "package manager for Kubernetes." A **Helm chart** bundles all the YAML for an app into a reusable, parameterized package you can install/upgrade with one command.
- **Operators** — custom controllers that encode operational knowledge for a specific app (e.g., a database operator that handles backups/failover), extending Kubernetes via **Custom Resource Definitions (CRDs)**.
- **Service Mesh** (Istio, Linkerd) — a layer handling service-to-service traffic, mutual TLS, retries, and observability without changing app code.
- **Managed Kubernetes** — EKS (AWS), AKS (Azure), GKE (Google) run the control plane for you.

---

## PART H — PRACTICAL: kubectl & YAML

## 18. Manifests and the apply Workflow

Everything is defined declaratively in **YAML manifests** with four common top-level fields: `apiVersion`, `kind`, `metadata`, and `spec`. You apply them with `kubectl apply -f file.yaml`, and Kubernetes reconciles toward that state.

## 19. Essential kubectl Commands

```bash
kubectl get pods                      # list pods
kubectl get pods -o wide              # more detail (node, IP)
kubectl get deployments,services      # list multiple resource types
kubectl describe pod <name>           # detailed info + events (great for debugging)
kubectl logs <pod>                    # view a pod's logs
kubectl logs <pod> -c <container>     # logs of a specific container
kubectl exec -it <pod> -- /bin/bash   # shell into a running container
kubectl apply -f manifest.yaml        # create/update from a manifest (declarative)
kubectl delete -f manifest.yaml       # delete resources
kubectl scale deployment web --replicas=5   # scale
kubectl rollout status deployment web       # watch a rollout
kubectl rollout undo deployment web         # roll back
kubectl get events --sort-by=.lastTimestamp # recent cluster events
```

## 20. Basic Troubleshooting Flow

- **Pod stuck `Pending`** → usually the scheduler can't place it: insufficient resources, no matching node, or unbound PVC. Check `kubectl describe pod` events.
- **`CrashLoopBackOff`** → the container starts and crashes repeatedly. Check `kubectl logs` for the app error; often a bad config, missing env var, or failing liveness probe.
- **`ImagePullBackOff` / `ErrImagePull`** → can't fetch the image: wrong image name/tag or missing registry credentials.
- **Service not reachable** → check the Service selector matches pod labels, and that pods pass their readiness probe (only ready pods get traffic).
- **General:** `describe` shows events (why something happened); `logs` shows what the app said; `get -o wide` shows placement.

---

## PART I — Q&A BANK

**Q: What is Kubernetes and what problem does it solve?**
An open-source container orchestration platform that automates deploying, scaling, networking, and managing containers across a cluster. It solves the problem of running many containers across many machines reliably — placement, self-healing, scaling, service discovery, and zero-downtime rollouts.

**Q: What is the declarative model / reconciliation loop?**
You declare the desired state in a manifest; Kubernetes' control loops continuously compare desired vs actual state and take action to close the gap (e.g., recreating a dead pod), which gives self-healing.

**Q: Describe the control plane components.**
API server (front door for all requests), etcd (key-value store holding cluster state), scheduler (places pods on nodes), and controller manager (runs reconciliation loops). Optionally a cloud-controller-manager integrates with the cloud provider.

**Q: What runs on a worker node?**
The kubelet (ensures containers run as specified), kube-proxy (implements Service networking), and a container runtime (containerd/CRI-O).

**Q: What is a Pod, and why isn't it just a container?**
A Pod is the smallest deployable unit — one or more tightly coupled containers sharing an IP, storage, and lifecycle. It's an abstraction above the container that lets helper containers (sidecars) share context with the main one.

**Q: Why are pods considered ephemeral?**
They can be killed and replaced anytime, and replacements get new IPs. So you never rely on a pod's IP — you use a Service for stable access, and let controllers recreate pods.

**Q: Deployment vs ReplicaSet vs StatefulSet?**
A ReplicaSet keeps N identical pods running. A Deployment manages ReplicaSets and adds rolling updates/rollbacks — used for stateless apps. A StatefulSet is for stateful apps needing stable identities, ordered operations, and per-pod persistent storage (databases).

**Q: What is a DaemonSet used for?**
Running one pod copy on every node — for node-level agents like log collectors, monitoring agents, or network plugins.

**Q: Job vs CronJob?**
A Job runs pods to completion for a batch task. A CronJob runs Jobs on a schedule, like a nightly backup.

**Q: What is a Service and why is it needed?**
A stable IP and DNS name that load-balances across a changing set of pods selected by labels. It's needed because pods are ephemeral with changing IPs, so clients need a fixed endpoint.

**Q: Explain the Service types.**
ClusterIP (internal-only, default), NodePort (a static port on every node for external access), LoadBalancer (a cloud load balancer with a public IP), and ExternalName (maps to an external DNS name).

**Q: ClusterIP vs LoadBalancer vs Ingress?**
ClusterIP is internal only. LoadBalancer exposes one service externally via a cloud LB. Ingress provides HTTP/HTTPS routing (host/path-based) through a single entry point and needs an Ingress Controller to work.

**Q: How do labels and selectors work?**
Labels are key-value tags on objects; selectors match those labels. Services and controllers use selectors to find the pods they route to or manage — the loose coupling mechanism of Kubernetes.

**Q: ConfigMap vs Secret?**
Both externalize configuration from images. ConfigMaps hold non-sensitive key-value config; Secrets hold sensitive data. Secrets are only base64-encoded by default, so you should enable encryption at rest and restrict access with RBAC.

**Q: Explain PV, PVC, and StorageClass.**
A PersistentVolume is actual storage in the cluster. A PersistentVolumeClaim is a request for storage that binds to a PV. A StorageClass defines a type of storage and enables dynamic provisioning so a PVC auto-creates a PV.

**Q: Liveness vs readiness probe?**
A failing liveness probe restarts the container (recovers hung apps). A failing readiness probe removes the pod from Service endpoints so no traffic is sent until it's ready, without restarting it.

**Q: How does Kubernetes self-heal?**
Controllers recreate failed pods to maintain desired replicas, the kubelet restarts crashed containers (via liveness probes), and pods on a dead node are rescheduled to healthy nodes.

**Q: What are the autoscalers?**
HPA scales the number of pod replicas based on metrics; VPA adjusts pods' CPU/memory requests; Cluster Autoscaler adds/removes nodes when pods can't be scheduled or nodes are idle.

**Q: Requests vs limits?**
Requests are the resources guaranteed to a container and used by the scheduler for placement; limits are the maximum it can use before being throttled (CPU) or killed (memory).

**Q: What are taints and tolerations?**
A taint on a node repels pods unless a pod has a matching toleration — used to reserve nodes (e.g., GPU nodes) for specific workloads.

**Q: What is a rolling update and how do you roll back?**
Gradually replacing old pods with new ones for zero-downtime deploys, controlled by maxSurge/maxUnavailable. Roll back with `kubectl rollout undo`.

**Q: How would you do a canary or blue-green deploy in K8s?**
Blue-green: run the new version alongside the old and switch the Service selector to it (instant rollback by switching back). Canary: route a small share of traffic to the new version and ramp up — done with multiple deployments or a service mesh/ingress weighting.

**Q: What is the pod network model?**
Every pod gets its own IP and can reach every other pod directly without NAT, implemented by a CNI plugin. Services get DNS names via CoreDNS.

**Q: What is a NetworkPolicy?**
A rule set controlling which pods may communicate — effectively a pod-level firewall. By default all pods can talk; policies restrict it.

**Q: What is RBAC and a ServiceAccount?**
RBAC controls who can perform which actions on which resources, via Roles bound to subjects. A ServiceAccount is a machine identity used by pods to authenticate to the API server.

**Q: What is Helm?**
The package manager for Kubernetes; a Helm chart bundles and parameterizes an app's manifests for repeatable install/upgrade.

**Q: What is an Operator?**
A custom controller (built on CRDs) that encodes operational knowledge for a specific application, automating tasks like backups and failover.

**Q: What's the difference between Docker and Kubernetes?**
Docker builds and runs individual containers; Kubernetes orchestrates many containers across a cluster — scheduling, scaling, networking, self-healing. They're complementary, not competitors.

**Q: How do you debug a CrashLoopBackOff?**
Check `kubectl logs <pod>` for the app error and `kubectl describe pod` for events; it's usually a bad config, missing dependency/env var, or a failing liveness probe.

**Q: What does the scheduler consider when placing a pod?**
Resource requests vs available node capacity, nodeSelector/affinity rules, taints/tolerations, and then it scores the feasible nodes to pick the best.

---

## PART J — CHEAT SHEET
- K8s = container orchestration: automates deploy, scale, network, heal.
- Declarative + reconciliation loop → self-healing. Desired state in YAML, `kubectl apply`.
- Control plane: API server, etcd, scheduler, controller manager. Node: kubelet, kube-proxy, runtime.
- Pod = smallest unit (1+ containers, shared IP/storage), ephemeral.
- Workloads: Deployment (stateless, rolling updates), StatefulSet (stateful, stable identity), DaemonSet (one per node), Job/CronJob (batch/scheduled), ReplicaSet (N replicas).
- Service = stable endpoint over pods: ClusterIP (internal), NodePort, LoadBalancer (external), ExternalName. Ingress = HTTP routing (needs a controller).
- Config: ConfigMap (non-secret), Secret (base64, encrypt at rest). Storage: PVC → PV, StorageClass = dynamic provisioning.
- Scaling: HPA (pods), VPA (pod size), Cluster Autoscaler (nodes). requests vs limits.
- Health: liveness (restart), readiness (traffic gating), startup (slow starts).
- Scheduling: labels/selectors, node affinity, taints/tolerations.
- Deploy strategies: rolling (default), recreate, blue-green, canary.
- Security: RBAC + ServiceAccounts, NetworkPolicy, least privilege.
- Ecosystem: Helm (packaging), Operators/CRDs, service mesh, managed K8s (EKS/AKS/GKE).
- Debug: `describe` (events) + `logs` (app output). Pending = scheduling; CrashLoopBackOff = app/config; ImagePullBackOff = image/creds.
