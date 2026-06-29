# Kubernetes: A Complete Beginner's Guide

> **How to use this guide:** Read it top to bottom at least once. Every chapter builds on the one before it. The diagrams are the anchors — read the text around each one to understand *why* the diagram looks the way it does, not just *what* it shows.

---

## Table of Contents

- [Chapter 0 — What Is Kubernetes and Why Does It Exist?](#chapter-0--what-is-kubernetes-and-why-does-it-exist)
- [Chapter 1 — Cluster Architecture: The Brain and the Muscle](#chapter-1--cluster-architecture-the-brain-and-the-muscle)
- [Chapter 2 — The Control Loop: How Kubernetes Thinks](#chapter-2--the-control-loop-how-kubernetes-thinks)
- [Chapter 3 — The Pod: The Smallest Unit](#chapter-3--the-pod-the-smallest-unit)
- [Chapter 4 — Core Objects That Support a Pod](#chapter-4--core-objects-that-support-a-pod)
- [Chapter 5 — Workload Controllers: Choosing the Right One](#chapter-5--workload-controllers-choosing-the-right-one)
- [Chapter 6 — Rolling Updates and Rollbacks](#chapter-6--rolling-updates-and-rollbacks)
- [Chapter 7 — Storage: PVC, PV, and StorageClass](#chapter-7--storage-pvc-pv-and-storageclass)
- [Chapter 8 — Networking: Services](#chapter-8--networking-services)
- [Chapter 9 — Ingress: Layer 7 HTTP Routing](#chapter-9--ingress-layer-7-http-routing)
- [Chapter 10 — Network Policy: Firewalls for Pods](#chapter-10--network-policy-firewalls-for-pods)
- [Chapter 11 — Pod Health Probes](#chapter-11--pod-health-probes)
- [Chapter 12 — RBAC: Who Can Do What](#chapter-12--rbac-who-can-do-what)
- [Chapter 13 — End-to-End: One Request's Full Journey](#chapter-13--end-to-end-one-requests-full-journey)
- [Appendix A — How a Pod Gets Scheduled](#appendix-a--how-a-pod-gets-scheduled)
- [Appendix B — kubectl Cheatsheet](#appendix-b--kubectl-cheatsheet)

---

## Chapter 0 — What Is Kubernetes and Why Does It Exist?

### The Problem

Imagine you're running a web application. You package it as a container (using Docker) and start it on a server. Life is good — until:

- The server crashes at 3am. Your app goes down. Customers are angry.
- Traffic spikes. You need 10 copies of your app running. You have to log into 10 servers and start them manually.
- You need to update your app to a new version. You take it down, update it, and bring it back up — causing downtime.
- A container silently gets stuck in a bad state. It's running but not actually responding. You don't notice for an hour.

Managing containers manually across many servers does not scale. It becomes fragile, error-prone, and exhausting.

### The Solution: Kubernetes

**Kubernetes** (often shortened to **K8s**) is an open-source system for automatically managing containers across a cluster of machines.

The key insight is this: **instead of telling Kubernetes what to do step-by-step, you tell it what you want the end result to look like.** Kubernetes figures out how to get there and keeps it that way — forever.

This is called a **declarative model**:

| Imperative (old way) | Declarative (Kubernetes way) |
|---|---|
| "Start 3 copies of my app on servers A, B, and C" | "I want 3 copies of my app running at all times" |
| "If one dies, SSH in and restart it" | Kubernetes automatically restarts it |
| "To update, stop the old version and start the new one" | "I want version 2 running" — Kubernetes handles the transition |

Think of Kubernetes as an **autopilot for your containers**. You set the destination. It flies the plane.

---

## Chapter 1 — Cluster Architecture: The Brain and the Muscle

A Kubernetes **cluster** is a group of machines working together. There are two types of machines in a cluster:

1. **The Control Plane** — the brain. It makes decisions.
2. **Worker Nodes** — the muscle. They actually run your application containers.

![Cluster Architecture: Control Plane and Worker Nodes](./4cluster_architecture_control_plane_and_nodes.png)

### The Control Plane

The control plane is where all the intelligence lives. It never runs your application containers — it just manages everything else.

**`kube-apiserver`** — The single front door to the entire cluster. Every command you type with `kubectl`, every component talking to another component, every automation tool — they all go through the API server. Nothing talks to anything else directly.

**`etcd`** — A fast, distributed key-value database. It is the **only place** the cluster's state is stored. If you have 3 replicas declared for your app, that fact lives in etcd. If etcd is lost, the cluster loses its memory. This is why etcd is always backed up in production.

**`kube-scheduler`** — Watches for newly created pods that don't have a node assigned yet. It looks at available nodes, checks resource requirements, and picks the best fit. Once it assigns a node, it writes that assignment to etcd via the API server.

**`controllers`** — A collection of control loops (reconcilers). Each controller watches one type of object and makes sure the real world matches the declared state. The Deployment controller watches Deployments; the Node controller watches Node health; and so on. They all talk exclusively through the API server.

### Worker Nodes

Worker nodes are where your actual application pods run. Every worker node has three things:

**`kubelet`** — An agent that runs on every node. It watches the API server for pods assigned to its node, then tells the container runtime to start them. It also reports pod status back to the API server. Think of it as the local manager on each machine.

**Container Runtime** — The software that actually runs containers. Kubernetes supports any runtime that implements the Container Runtime Interface (CRI). The most common runtimes are `containerd` and `CRI-O`. Docker itself uses `containerd` under the hood.

**`kube-proxy`** — A network component that runs on every node. It programs network rules (iptables or IPVS) so that traffic sent to a Service's virtual IP gets forwarded to the correct pod. More on this in Chapter 8.

### The Full Picture

Here is what a real cluster looks like with two worker nodes:

```
┌────────────────────────────────────────────────────────────────┐
│                        CONTROL PLANE                           │
│                                                                │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                    kube-apiserver                        │  │
│  │              (all traffic goes through here)             │  │
│  └──────┬──────────────────┬──────────────────┬─────────────┘  │
│         │                  │                  │                │
│  ┌──────▼──────┐   ┌───────▼──────┐   ┌──────▼──────────┐    │
│  │    etcd     │   │kube-scheduler│   │   controllers   │    │
│  │(state store)│   │(places pods) │   │ (reconcile loop)│    │
│  └─────────────┘   └──────────────┘   └─────────────────┘    │
└──────────────────────────┬─────────────────────────────────────┘
                           │  API server ⇄ kubelet
         ┌─────────────────┴────────────────────┐
         │                                      │
┌────────▼──────────────┐           ┌───────────▼───────────┐
│      WORKER NODE 1    │           │      WORKER NODE 2    │
│                       │           │                       │
│  ┌──────────────────┐ │           │  ┌──────────────────┐ │
│  │     kubelet      │ │           │  │     kubelet      │ │
│  │  (runs & reports)│ │           │  │  (runs & reports)│ │
│  └──────────────────┘ │           │  └──────────────────┘ │
│  ┌──────────────────┐ │           │  ┌──────────────────┐ │
│  │   kube-proxy     │ │           │  │   kube-proxy     │ │
│  │ (service routes) │ │           │  │ (service routes) │ │
│  └──────────────────┘ │           │  └──────────────────┘ │
│  ┌────────┐ ┌────────┐│           │  ┌────────┐ ┌────────┐│
│  │  Pod A │ │  Pod B ││           │  │  Pod C │ │  Pod D ││
│  └────────┘ └────────┘│           │  └────────┘ └────────┘│
└───────────────────────┘           └───────────────────────┘
```

---

## Chapter 2 — The Control Loop: How Kubernetes Thinks

Every single thing Kubernetes does can be explained by one pattern: the **reconciliation control loop**.

![Reconciliation Control Loop](./5reconciliation_control_loop.png)

Here is what the loop does:

1. **Desired** — You write a YAML file declaring what you want (e.g., "3 replicas of my app").
2. **Observe** — A controller reads the actual current state of the cluster from the API server.
3. **Compare** — It checks: does actual state match desired state? Is there any "drift"?
4. **Act** — If there is drift, it takes action to close the gap (create a pod, delete a pod, update a config, etc.).
5. **Loop** — It goes back to step 2 and re-checks. Continuously. Forever.

This loop is called **level-triggered** (rather than edge-triggered). It doesn't just react to events — it periodically re-checks the full state. This makes Kubernetes very robust: even if it missed an event, it will catch the drift on the next observation.

### The Thermostat Analogy

This is identical to how a thermostat works:

- **Desired**: you set the temperature to 22°C
- **Observe**: the thermostat reads the current room temperature (18°C)
- **Compare**: 18 ≠ 22, there is drift
- **Act**: turns the heater on
- **Loop**: re-reads temperature every few seconds until 22°C is reached, then turns heater off

If someone opens a window and the room cools down again — the thermostat catches it on the next check and acts again. You never had to tell it "the window is open, please compensate."

Kubernetes works exactly the same way. If a pod dies, a controller notices the count is short and creates a new one. You never have to intervene.

---

## Chapter 3 — The Pod: The Smallest Unit

### What Is a Pod?

A **pod** is the smallest deployable unit in Kubernetes. You never deploy a container directly — you always deploy a pod, and the pod contains your containers.

Every pod gets **one unique IP address** and all containers inside that pod share it. They can talk to each other via `localhost`. From the network's perspective, the pod looks like one machine.

![Inside a Pod: Shared Context](./6inside_a_pod_shared_context.png)

### Containers Inside a Pod

A pod can have more than one container. There are three roles a container can play:

**Init Container** — Runs first, before any other container in the pod. It must complete successfully before the main app is allowed to start. Use it for one-time setup tasks: waiting for a database to be ready, running a database migration, writing a config file to a shared volume. Once it exits successfully, Kubernetes moves on.

**App Container** (main container) — The primary container running your application process. This is what you think of when you think "my app."

**Sidecar Container** — A helper that runs alongside the app container for the entire pod lifetime. Common uses: a log shipper that reads log files and forwards them to a central system, a proxy that handles mTLS for the app, or a metrics exporter. It shares the same network and can share volumes with the app container.

Example to understand, CR(resource type), CRD(blueprint) and Objects(the actual instance).

- **`clusterlogforwarders.logging.openshift.io`** is the **CRD (the blueprint)**. If you want to know what properties, fields, and validation rules define how a forwarder must be structured, you inspect this CRD.
- **`ClusterLogForwarder`** (or the lowercase command shortcut `clusterlogforwarder`) is the **Custom Resource (the class/type)**. By itself, it is just a category definition, not a live instance.
- **`clf01`** is the **Object (the instance)**. It is a specific entity living in your cluster's database that has the type (`kind`) `ClusterLogForwarder`.

### The Complete Alignment

| **Term**   | **Role**                  | **Programming Analogy**                 | **Example**                                 |
| ---------- | ------------------------- | --------------------------------------- | ------------------------------------------- |
| **CRD**    | The Definition Blueprint  | The language specification/schema rules | `clusterlogforwarders.logging.openshift.io` |
| **CR**     | The Resource Type / Class | The Class name                          | `ClusterLogForwarder`                       |
| **Object** | The Actual Live Entity    | The instantiated Object instance        | `clf01`                                     |

### Pod Startup Sequence

```
Pod Lifecycle:
─────────────────────────────────────────────────────────────────
Time ──►

[ init container ]
   runs first ─── exits successfully
                        │
                        ▼
           [ app container ]  ──── runs until pod terminates
           [ sidecar container ] ── runs until pod terminates
                        │
                        ▼
                  [ pod exits ]
─────────────────────────────────────────────────────────────────

If init container FAILS → pod restarts (does not start the app)
If app container FAILS  → pod restarts (sidecar restarts too)
```

### Shared Volumes

Containers in the same pod can share a **volume** — a directory on disk that both containers can read from and write to. This is how the app container and sidecar container exchange data. For example: the app writes log files to `/var/log/app`, and the sidecar reads those files and ships them to a logging service.

---

## Chapter 4 — Core Objects That Support a Pod

No pod is an island. In a real cluster, a pod is surrounded by several supporting objects that give it identity, configuration, network access, and security.

![Kubernetes Objects Around the Pod](./1k8s_objects_around_the_pod.png)

Let's look at each supporting object:

### Namespace

A **namespace** is an isolation boundary within a cluster. Every object in Kubernetes (pods, services, configs) lives inside a namespace. Objects in different namespaces are isolated from each other by default — they can have the same names, can't see each other's secrets, and can have separate access controls.

Think of a namespace as a "virtual sub-cluster." Teams often get their own namespace (e.g., `team-frontend`, `team-backend`) on a shared cluster.

Two namespaces exist by default: `default` (where objects go if you don't specify) and `kube-system` (where Kubernetes' own components live).

### ConfigMap

A **ConfigMap** stores non-sensitive configuration data as key-value pairs. Instead of baking configuration into your container image, you inject it at runtime via a ConfigMap. This means you can use the same image in staging and production with different configs.

```yaml
# Example: a ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  DATABASE_HOST: "postgres.default.svc.cluster.local"
  LOG_LEVEL: "info"
```

The pod mounts this ConfigMap as environment variables or as a file in the container.

### Secret

A **Secret** is identical to a ConfigMap in structure, but it is intended for sensitive values: passwords, API tokens, TLS certificates. Secrets are stored in etcd (optionally encrypted at rest) and Kubernetes restricts who can read them via RBAC.

```yaml
# Example: a Secret
apiVersion: v1
kind: Secret
metadata:
  name: db-password
type: Opaque
data:
  password: c3VwZXJzZWNyZXQ=   # base64-encoded value
```

> **Important:** base64 is encoding, not encryption. It is not secure on its own. Always enable etcd encryption at rest in production.

### ServiceAccount

A **ServiceAccount** is the pod's identity when it talks to the Kubernetes API. Every pod runs as a ServiceAccount. If a pod needs to list other pods, read ConfigMaps, or call the API server for any reason, it authenticates with its ServiceAccount's token.

By default, pods are assigned the `default` ServiceAccount, which has minimal permissions. You create custom ServiceAccounts and grant them specific permissions via RBAC (covered in Chapter 12).

### Service

A **Service** gives a stable network identity to a set of pods. Pods are temporary — they come and go, and their IPs change. A Service is permanent. It has a fixed DNS name and virtual IP that never changes.

The Service selects pods using **labels** — key-value tags attached to pods. For example, the Service `app=backend` selects all pods that have the label `app=backend`. When a pod dies and is replaced, the new pod gets the same label and is automatically added to the Service's endpoint list.

### Deployment

A **Deployment** is the object you create when you want to run a long-running, stateless application and keep a specific number of replicas (copies) running at all times. You say "I want 3 copies of my app." The Deployment creates and manages them. If one dies, the Deployment notices and creates a replacement. More in Chapter 5.

---

## Chapter 5 — Workload Controllers: Choosing the Right One

A **workload controller** is an object that manages pods on your behalf. You don't usually create pods directly — you create a controller and it creates pods for you according to rules you specify.

There are two families of controllers:

![Workload Controller Chains](./2workload_controller_chains.png)

### The Long-Running Path: Deployment → ReplicaSet → Pod

**Deployment** — The top-level object you interact with. You declare which container image to run, how many replicas, resource limits, and update strategy. You rarely change this directly after creation except when rolling out a new image version.

**ReplicaSet** — Created automatically by the Deployment. Its only job is to ensure a specific number of pod replicas are running. You almost never create or edit a ReplicaSet directly — the Deployment manages it for you. When you update a Deployment (e.g., new image), the Deployment creates a *new* ReplicaSet and gradually shifts pods from the old RS to the new one (see Chapter 6).

**Pod** — The actual running container(s). Created and deleted by the ReplicaSet automatically.

### The Task Path: CronJob → Job → Pod

**CronJob** — Runs a **Job** on a schedule (using standard cron syntax: `"0 * * * *"` = every hour). Think of it as a scheduled task runner.

**Job** — Runs one or more pods to completion. Unlike a Deployment, a Job pod is allowed to exit. A Job considers itself done when the required number of pods complete successfully. If a pod fails, the Job retries it.

**Pod** — The task pod. It runs, does its work, then exits with code 0 (success). The Job marks it complete.

### The Full Controller Menu

![Choosing a Workload Controller](./14choosing_a_workload_controller.png)

Here is how to pick the right controller:

| Controller | Use When | Example |
|---|---|---|
| **Deployment** | Stateless app, pods are identical and interchangeable | Web server, API, frontend |
| **StatefulSet** | App needs a stable hostname and/or its own dedicated storage per replica | PostgreSQL, Redis, Kafka, Elasticsearch |
| **DaemonSet** | Exactly one pod must run on every (or selected) node | Log collector, metrics agent, CNI plugin |
| **Job** | Run a task once and exit | Database migration, batch job, one-time report |
| **CronJob** | Run a task on a schedule | Nightly backup, hourly cleanup, daily email |

**StatefulSet details:** Unlike Deployments, StatefulSets give each pod a stable, predictable hostname (`pod-0`, `pod-1`, `pod-2`) and each pod gets its own PersistentVolumeClaim (see Chapter 7). The pods are created and deleted in order. This matters for databases where `pod-0` is the primary and `pod-1`, `pod-2` are replicas — they need to know who they are.

**DaemonSet details:** When a new node joins the cluster, the DaemonSet automatically schedules a pod onto it. When a node is removed, the pod is cleaned up. Perfect for infrastructure-level agents that need to be on every machine.

---

## Chapter 6 — Rolling Updates and Rollbacks

Deploying a new version of your app should never require downtime. Kubernetes handles this with a **rolling update**.

![Deployment Rolling Update](./13deployment_rolling_update.png)

### How a Rolling Update Works

Imagine you have 3 pods running version 1 (v1) of your app and you update the Deployment to use version 2 (v2):

1. **Kubernetes creates a new ReplicaSet** for v2 (initially at 0 replicas).
2. **It scales up the new RS** (adds v2 pods) and **scales down the old RS** (removes v1 pods) in batches.
3. At any moment, both v1 and v2 pods may be running simultaneously.
4. Once all v2 pods are healthy and running, the old RS is scaled to 0.

The diagram shows this in three phases: **before** (3× v1), **during** (1× v1, 2× v2), **after** (3× v2).

### Controlling the Pace

Two settings on the Deployment control how aggressive the rollout is:

**`maxSurge`** — How many *extra* pods above the desired count are allowed during the update. Default is 25%. With 3 replicas and `maxSurge=1`, you can temporarily have 4 pods running (3 desired + 1 surge).

**`maxUnavailable`** — How many pods below the desired count are allowed during the update. Default is 25%. With 3 replicas and `maxUnavailable=1`, you can have as few as 2 pods running at once.

```
Old RS count:  3 ──► 2 ──► 1 ──► 0
New RS count:  0 ──► 1 ──► 2 ──► 3
                     ▲
               pods run in parallel here
```

### Rollback

The old ReplicaSet (v1) is kept at 0 replicas but is **not deleted**. Kubernetes keeps a history of recent ReplicaSets. This means rollback is instant:

```bash
kubectl rollout undo deployment my-app
```

Kubernetes just swaps the ReplicaSet counts again: new RS goes to 0, old RS goes back to 3. No new image pull required.

---

## Chapter 7 — Storage: PVC, PV, and StorageClass

Containers are ephemeral. When a pod restarts, any data written to the container's filesystem is gone. For stateful applications (databases, file stores), you need **persistent storage** — storage that outlives the pod.

![Storage: PVC, PV, StorageClass](./7storage_pvc_pv_storageclass.png)

Kubernetes storage has three layers. It helps to think about them as a hotel analogy:

### StorageClass — The Hotel Chain

A **StorageClass** defines *how* storage gets provisioned. It specifies the provisioner (e.g., AWS EBS, Google Cloud Persistent Disk, a local NFS server) and the configuration parameters (SSD or HDD, replication level, etc.).

The cluster administrator sets up StorageClas ses. As an application developer, you usually just reference the name of an existing StorageClass (`standard`, `fast-ssd`, etc.).

**Analogy:** StorageClass = the hotel chain (Marriott, Hilton, etc.) with its policies, room types, and pricing.

### PersistentVolume (PV) — The Actual Room

A **PersistentVolume** is the actual storage resource: a specific disk, NFS share, cloud volume, etc. It has a capacity (e.g., 10Gi) and an access mode (read-write by one pod, or read-write by many).

PVs can be provisioned manually by an admin, or automatically when a PVC is created (dynamic provisioning via StorageClass).

**Analogy:** PersistentVolume = the actual hotel room. It exists. It has a bed, a bathroom, a specific room number.

### PersistentVolumeClaim (PVC) — The Booking Form

A **PersistentVolumeClaim** is a request for storage made by a pod. The developer specifies: "I need 5Gi of storage with ReadWriteOnce access." Kubernetes finds a matching PV and **binds** it to the PVC. Once bound, that PV is reserved exclusively for this PVC.

The pod references the PVC by name. The pod doesn't care which physical disk backs it — it just mounts the PVC.

**Analogy:** PVC = the hotel booking form. You specify: "I need a room for two, non-smoking, for 3 nights." The hotel (Kubernetes) finds a matching room and assigns it to you.

### Access Modes

| Mode | Abbreviation | Meaning |
|---|---|---|
| ReadWriteOnce | RWO | Mounted read-write by one node at a time |
| ReadOnlyMany | ROX | Mounted read-only by many nodes |
| ReadWriteMany | RWX | Mounted read-write by many nodes (requires special storage like NFS) |

---

## Chapter 8 — Networking: Services

### The Problem Services Solve

Pods are ephemeral and their IPs change every time they are replaced. You cannot hardcode a pod IP in your application config. You need a stable address that always points to healthy pods — even as pods come and go.

That's exactly what a **Service** provides: a stable DNS name and virtual IP that acts as the front door for a set of pods.

### The Three Service Types (Nested Layers)

![Service Types: Nested Layers](./8service_types_nested_layers.png)

The diagram shows the types as nested layers — each outer type wraps and extends the inner one.

**ClusterIP** (innermost layer)
The default service type. Creates a virtual IP that is only reachable from inside the cluster. Other pods use the Service's DNS name (`my-service.my-namespace.svc.cluster.local`) to reach it. You cannot access a ClusterIP service from outside the cluster.

Use this when: you have a backend service (database, API) that only other pods in the cluster need to call.

**NodePort** (middle layer)
Extends ClusterIP by opening a specific port (30000–32767) on **every node** in the cluster. Traffic hitting `<any-node-ip>:<nodeport>` is forwarded to the ClusterIP, which forwards to a pod. This lets you access the service from outside the cluster if you know a node's IP.

Use this when: you're testing on a bare-metal or local cluster that doesn't have a cloud load balancer.

**LoadBalancer** (outermost layer)
Extends NodePort by provisioning an external load balancer from your cloud provider (AWS ELB, GCP Load Balancer, Azure LB, etc.). The cloud LB gets a public IP and routes traffic to the NodePort on the nodes. This is the standard way to expose a service to the internet in cloud environments.

Use this when: you need to expose a service to the internet on a cloud cluster.

### How Traffic Actually Flows

![Packet Path to a Pod](./10packet_path_to_a_pod.png)

The Service IP is **virtual** — there is no server sitting behind it. It is implemented entirely as network rules (iptables or IPVS) programmed by `kube-proxy` on every node.

When a client sends a packet to the Service VIP:

1. The packet arrives at a node.
2. `kube-proxy`'s iptables rules intercept it.
3. The rules perform **DNAT** (Destination Network Address Translation): they rewrite the destination from the Service VIP to a real pod IP.
4. The packet is forwarded to the actual pod.

The pod never knows it was reached via a VIP — it just sees a regular connection. This is transparent to both the client and the pod.

```
Client ──► Service VIP:80 ──► [kube-proxy DNAT] ──► Pod IP:8080
           (no real server)    (iptables rule)       (real server)
```

---

## Chapter 9 — Ingress: Layer 7 HTTP Routing

### The Problem With Using LoadBalancer for Everything

A LoadBalancer service gives you one cloud load balancer per service. If you have 10 services exposed to the internet, you need 10 cloud LBs. Cloud LBs are not free. This becomes expensive quickly.

Also, LoadBalancer operates at Layer 4 (TCP/UDP). It can't look at HTTP headers, hostnames, or URL paths. It can only forward traffic based on IP and port.

### Enter Ingress

An **Ingress** is a Layer 7 (HTTP/HTTPS) router that sits in front of multiple services. You define routing rules based on the **hostname** and **URL path** of incoming HTTP requests.

![Ingress: Layer 7 Routing](./9ingress_l7_routing.png)

With one Ingress (and one cloud LB behind it), you can route:
- `example.com/app` → Service A (frontend pods)
- `example.com/api` → Service B (backend pods)
- `api.example.com` → Service C (API pods on a different subdomain)

All through a single entry point.

### The Ingress Controller

The Ingress resource is just a set of rules stored in Kubernetes. Something must actually implement those rules and handle the traffic. That's the **Ingress Controller** — a pod running in the cluster that reads Ingress rules and acts as the actual reverse proxy.

Common Ingress Controllers:
- **nginx-ingress** — runs NGINX inside the cluster as the proxy
- **Traefik** — a modern, dynamic router popular in container environments
- **AWS ALB Ingress Controller** — provisions AWS Application Load Balancers natively

You install the Ingress Controller once per cluster. All your Ingress resources are then served by it.

```
Internet
    │
    ▼
[ Cloud LB ] ── one LB for the whole cluster
    │
    ▼
[ Ingress Controller Pod ] ── reads Ingress rules, acts as reverse proxy
    │
    ├──► example.com/app  ──► Service: frontend ──► frontend pods
    │
    └──► example.com/api  ──► Service: backend  ──► backend pods
```

---

## Chapter 10 — Network Policy: Firewalls for Pods

### The Default: No Firewall

By default in Kubernetes, every pod can reach every other pod in the cluster — across namespaces, across nodes. If your frontend pod knows the IP of your database pod, it can connect directly to port 5432 with nothing blocking it.

This is fine for simple setups but dangerous in production. A compromised frontend pod could talk directly to the database, or to other sensitive services it has no business reaching.

### NetworkPolicy: Firewall Rules for Pods

A **NetworkPolicy** lets you define who can send traffic to a pod (ingress rules) and who a pod can send traffic to (egress rules), using label selectors.

![NetworkPolicy: Default Deny + Allowlist](./11networkpolicy_default_deny.png)

### The Default-Deny Pattern

The recommended pattern is:

1. Create a **default-deny** policy that blocks all ingress (and optionally egress) to a set of pods.
2. Create **allowlist** policies that explicitly permit specific traffic.

This way, if you forget to write a rule, traffic is blocked rather than allowed. It is a **fail-closed** posture.

Example: protecting a backend pod:

```yaml
# Step 1: Default deny all ingress to pods with label app=backend
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-to-backend
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Ingress

---

# Step 2: Allow only frontend pods to reach backend on port 5432
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - port: 5432
```

Result:
- `frontend` pod → `backend` port 5432: **ALLOWED**
- `other` pod → `backend` port 5432: **BLOCKED**
- `frontend` pod → `backend` port 22: **BLOCKED** (not in the allowlist)

> **Note:** NetworkPolicy enforcement requires a CNI (Container Network Interface) plugin that supports it, such as Calico, Cilium, or Weave Net. The default `kubenet` CNI does not enforce NetworkPolicies.

---

## Chapter 11 — Pod Health Probes

Kubernetes needs to know when your application is healthy. It can't just assume that because a container is running, the application inside is actually working. A process can be running but completely deadlocked.

That's where **probes** come in. You define health checks; Kubernetes runs them and acts accordingly.

![Pod Probes and Their Effects](./12pod_probes_and_their_effects.png)

There are three probes, each with a distinct role:

### 1. `startupProbe` — "Has the app booted yet?"

Some applications take a long time to start (legacy Java apps, model loading, etc.). If you run a livenessProbe too early on such an app, Kubernetes might restart it before it even gets a chance to start — causing an infinite restart loop.

The `startupProbe` solves this. While it is failing (or hasn't succeeded yet), the other two probes are **paused**. Once the `startupProbe` succeeds once, it stops running and the other probes take over.

Effect: **gates the other two probes** until startup is complete.

### 2. `readinessProbe` — "Is the app ready to serve traffic?"

A pod might be started and alive but not yet ready to handle requests. Maybe it's still warming up a cache, connecting to a database, or loading configuration. You don't want traffic sent to it during this time.

The `readinessProbe` controls whether the pod is included in the Service's **endpoints** list. If the probe fails, the pod is removed from the list (no traffic sent to it). If it recovers, it's added back.

Effect: **adds/removes the pod from Service traffic** (no restart).

### 3. `livenessProbe` — "Is the app still alive and not stuck?"

Checks if your running app is still healthy — not deadlocked, not hung on an infinite loop. If this probe fails repeatedly, Kubernetes **restarts the container**.

Effect: **restarts the container** on repeated failure.

### Probe Methods

All three probes support the same check methods:

| Method | What it checks |
|---|---|
| `httpGet` | Sends an HTTP GET to a path/port; success = 2xx or 3xx response |
| `tcpSocket` | Opens a TCP connection to a port; success = connection accepted |
| `exec` | Runs a command inside the container; success = exit code 0 |

### Common Mistake: Mixing Up Probes

| If you use... | Instead of... | What goes wrong |
|---|---|---|
| `livenessProbe` for readiness | `readinessProbe` | Kubernetes restarts pods that are just slow to warm up |
| `readinessProbe` for liveness | `livenessProbe` | Stuck pods never get restarted; they just stop receiving traffic |
| No `startupProbe` for slow apps | `startupProbe` | `livenessProbe` kills the app before it finishes starting |

---

## Chapter 12 — RBAC: Who Can Do What

### The Problem

The Kubernetes API is powerful. Whoever can talk to it can create pods, read secrets, delete deployments, or even shut down the whole cluster. You need fine-grained control over who (or what) is allowed to do what.

Kubernetes uses **Role-Based Access Control (RBAC)** to control access to the API.

![RBAC Permission Chain](./3rbac_permission_chain.png)

### The Three RBAC Objects

**Role** — A set of permissions. It lists which **verbs** (get, list, watch, create, update, delete) are allowed on which **resources** (pods, secrets, configmaps, etc.), within one namespace.

```yaml
# A Role that allows reading pods and configmaps in the "default" namespace
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: default
rules:
- apiGroups: [""]
  resources: ["pods", "configmaps"]
  verbs: ["get", "list", "watch"]
```

**RoleBinding** — Attaches a Role to a subject. The subject is usually a **ServiceAccount**. It says "the ServiceAccount named `my-app-sa` gets the permissions defined in the Role named `pod-reader`."

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: default
subjects:
- kind: ServiceAccount
  name: my-app-sa
  namespace: default
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

**ServiceAccount** — The identity the pod runs as. The pod's token (automatically mounted at `/var/run/secrets/kubernetes.io/serviceaccount/token`) is used to authenticate to the API server. The pod *inherits* all permissions granted to its ServiceAccount.

### The Chain

```
Role             RoleBinding          ServiceAccount        Pod
(permissions) ──► (ties them) ──────► (the identity) ──────► (inherits access)
get, list, watch   role → SA           my-app-sa              runs as my-app-sa
```

### ClusterRole and ClusterRoleBinding

`Role` and `RoleBinding` are **namespace-scoped** — they only apply within one namespace. For cluster-wide permissions (e.g., reading nodes, which are not namespaced), you use `ClusterRole` and `ClusterRoleBinding`. The concept is identical, just broader in scope.

---

## Chapter 13 — End-to-End: One Request's Full Journey

Now let's put it all together. Follow a single HTTP request from a browser all the way to your application pod — and trace every component it touches.

**Scenario:** A user opens `https://example.com/api/products` in their browser. Your Kubernetes cluster is running a backend API.

```
[ Browser ]
     │
     │  HTTPS GET /api/products
     ▼
[ Cloud Load Balancer ]          ← LoadBalancer Service (Chapter 8)
     │                             (public IP, provisions by cloud)
     │  forwards to NodePort
     ▼
[ Worker Node: port 30080 ]      ← NodePort opens this port on every node
     │
     │  kube-proxy DNAT
     ▼
[ ClusterIP virtual IP ]         ← ClusterIP routes to healthy pod endpoints
     │
     │  re-checks endpoints list (only pods with passing readinessProbe)
     ▼
[ Pod IP:8080 ]                  ← Chapter 3: the real pod
     │
     │  NetworkPolicy check:     ← Chapter 10: is this source allowed?
     │  is LB traffic on port 8080 in the allowlist? YES → proceed
     ▼
[ app container :8080 ]          ← Chapter 6: running v2 of your image
     │
     │  reads DATABASE_HOST from ConfigMap   ← Chapter 4
     │  reads DB_PASSWORD from Secret        ← Chapter 4
     │  uses ServiceAccount token to call    ← Chapter 12
     │  the Kubernetes API if needed
     ▼
[ Response flows back ]
     │
     ▼
[ Browser ]  ← sees the product list
```

Notice how many chapters are active simultaneously during a single request:
- **Chapter 1** (architecture): the request lands on a worker node; kubelet and kube-proxy are running there
- **Chapter 4** (objects): the pod uses ConfigMap, Secret, ServiceAccount
- **Chapter 5** (workloads): the pod was created by a Deployment → ReplicaSet
- **Chapter 6** (updates): you can roll out a new pod version with zero downtime for this exact request path
- **Chapter 8** (services): LoadBalancer → NodePort → ClusterIP → pod
- **Chapter 10** (network policy): guards what can reach this pod
- **Chapter 11** (probes): the pod was only added to the endpoint list after its readinessProbe passed
- **Chapter 12** (RBAC): the pod's ServiceAccount controls what it can query from the API

---

## Appendix A — How a Pod Gets Scheduled

When you run `kubectl apply -f my-deployment.yaml`, what actually happens? Here is the full sequence from command to running container:

```
You
 │
 │  kubectl apply -f my-deployment.yaml
 ▼
kube-apiserver
 │  1. Validates the YAML
 │  2. Authenticates you (are you allowed to create a Deployment?)
 │  3. Writes the Deployment object to etcd
 ▼
etcd
 │  Deployment object is now stored. Cluster state updated.
 ▼
Deployment Controller (inside kube-controller-manager)
 │  Watches for Deployment objects. Sees the new one.
 │  Creates a ReplicaSet object. Writes it to etcd via apiserver.
 ▼
ReplicaSet Controller
 │  Watches for ReplicaSets. Sees the new one.
 │  Creates Pod objects (just the spec, not yet running). Writes to etcd via apiserver.
 ▼
kube-scheduler
 │  Watches for Pods with no node assigned.
 │  Evaluates all nodes: CPU/memory available? Taints/tolerations? Affinity rules?
 │  Picks the best node. Updates the Pod object's nodeName field in etcd via apiserver.
 ▼
kubelet (on the chosen node)
 │  Watches for Pod objects assigned to its node.
 │  Sees the new Pod. Calls the container runtime (containerd/CRI-O).
 ▼
Container Runtime
 │  Pulls the container image from the registry (if not cached).
 │  Creates and starts the container.
 ▼
kubelet (continued)
 │  Monitors the container.
 │  Reports pod status back to apiserver (Running, Ready, etc.).
 ▼
Pod is Running ✓
```

**Key insight:** nothing communicates directly with anything else. Every component talks through the API server. The API server writes to etcd. Other components watch etcd for changes. This "watch-and-react" pattern is how the entire system is decoupled — you could restart any one component and it would catch up by re-reading from etcd.

---

## Appendix B — kubectl Cheatsheet

`kubectl` is the command-line tool for interacting with a Kubernetes cluster. You can create objects declaratively (with YAML files via `kubectl apply`) or imperatively (with quick one-liner commands). The cheatsheet below covers the imperative commands — useful for quick tasks, testing, and (especially) the CKA exam.

---

### Workloads & Scheduling

**Pod**
```bash
kubectl run <pod_name> --image=<image_name> --restart=Always
```

**Deployment**
```bash
kubectl create deployment <deployment_name> --image=<image_name> --replicas=<number>
```

**Job**
```bash
kubectl create job <job_name> --image=<image_name> -- <command_to_run>
```

**CronJob**
```bash
kubectl create cronjob <cronjob_name> --image=<image_name> --schedule="<cron_expression>" -- <command_to_run>
```

---

### Networking & Services

**Service (ClusterIP / NodePort / LoadBalancer)**
```bash
kubectl create service <service_type> <service_name> --tcp=<service_port>:<target_port>
```

> Valid `<service_type>` values: `clusterip`, `nodeport`, `loadbalancer`

---

### Configuration & Security

**Namespace**
```bash
kubectl create namespace <namespace_name>
```

**ConfigMap**
```bash
kubectl create configmap <configmap_name> --from-literal=<key1>=<value1> --from-literal=<key2>=<value2>
```

**Secret (Generic / Opaque)**
```bash
kubectl create secret generic <secret_name> --from-literal=<key>=<sensitive_value>
```

**Secret (Docker Registry)**
```bash
kubectl create secret docker-registry <secret_name> \
  --docker-username=<username> \
  --docker-password=<password> \
  --docker-email=<email>
```

**Secret (TLS)**
```bash
kubectl create secret tls <secret_name> --cert=<path_to_cert_file> --key=<path_to_key_file>
```

---

### Cluster Management & Access Control

**ServiceAccount**
```bash
kubectl create serviceaccount <serviceaccount_name>
```

**Role**
```bash
kubectl create role <role_name> --resource=<resource_types> --verb=<actions>
```

**RoleBinding**
```bash
kubectl create rolebinding <binding_name> \
  --role=<role_name> \
  --serviceaccount=<namespace>:<serviceaccount_name>
```

---

### YAML Generation Pattern (Dry-Run)

Add `--dry-run=client -o yaml` to any command above to print the YAML instead of creating the object. Redirect it to a file to get a starter YAML you can then edit:

```bash
kubectl create deployment my-app --image=nginx --replicas=3 --dry-run=client -o yaml > my-deployment.yaml
```

This is the fastest way to generate correct Kubernetes YAML without writing it from scratch.

---

### Essential Inspection Commands

```bash
# List objects
kubectl get pods
kubectl get pods -n <namespace>
kubectl get all

# Describe an object (events, conditions, full spec)
kubectl describe pod <pod_name>
kubectl describe deployment <deployment_name>

# View logs
kubectl logs <pod_name>
kubectl logs <pod_name> -c <container_name>   # specific container in a pod
kubectl logs -f <pod_name>                     # follow (stream) logs

# Execute a command inside a running pod
kubectl exec -it <pod_name> -- /bin/sh

# Rollout management
kubectl rollout status deployment <deployment_name>
kubectl rollout history deployment <deployment_name>
kubectl rollout undo deployment <deployment_name>

# Scale
kubectl scale deployment <deployment_name> --replicas=<number>
```

---

*End of Guide*

---

> **What to learn next:**
> - **Helm** — a package manager for Kubernetes (bundles of YAML called "charts")
> - **Kustomize** — template-free YAML customization built into `kubectl`
> - **Operators** — custom controllers that extend Kubernetes for specific applications (databases, certificates, etc.)
> - **Observability** — Prometheus for metrics, Grafana for dashboards, Loki for logs
> - **Service Mesh** — Istio or Linkerd for advanced traffic management and mutual TLS between pods
