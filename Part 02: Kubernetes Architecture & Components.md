### **Chapter 4: Kubernetes Architecture**  
Kubernetes follows a **distributed, client-server architecture** with a **Control Plane** (Master) managing **Worker Nodes**. All communication flows *through* the Control Plane.

---

#### **I. Control Plane (Master Node) Components**  
*The "brain" of the cluster. Manages the cluster state, schedules workloads, and responds to events.*

1. **API Server (`kube-apiserver`)**  
   - **What it is**: The **central hub** and *only* component that talks directly to `etcd`. It's the front door to the cluster.  
   - **How it works**:  
     - Exposes the Kubernetes **REST API** (e.g., `https://<cluster-ip>:6443`).  
     - All components (kubectl, controllers, kubelets) communicate *exclusively* via the API Server.  
     - Validates, processes, and authenticates all requests (using RBAC, ABAC, or Webhooks).  
     - Serves as a **stateless load balancer** – multiple instances can run for HA.  
   - **Why it matters**:  
     - **Single source of truth**: All cluster state changes go through it.  
     - **Admission Control**: Plugins like `PodSecurityPolicy` or `ResourceQuota` enforce rules *before* writing to `etcd`.  
     - **Failure Impact**: If down, *no changes* can be made to the cluster (but running pods stay alive).  
   - **Key Flags**: `--advertise-address`, `--secure-port=6443`, `--etcd-servers`.

2. **etcd**  
   - **What it is**: A **distributed, consistent key-value store** (using Raft consensus) acting as Kubernetes' **"database"**.  
   - **How it works**:  
     - Stores *all* cluster data: Pod specs, ConfigMaps, Secrets, Node status, Service definitions.  
     - Data is stored hierarchically (e.g., `/registry/pods/default/nginx-pod`).  
     - Requires **odd-numbered nodes** (3, 5, 7) for fault tolerance (quorum = `(n/2)+1`).  
   - **Why it matters**:  
     - **Critical for HA**: Loss of quorum = cluster outage. Back up `etcd` *daily*!  
     - **Consistency**: Uses **linearizable reads** – guarantees all clients see the latest state.  
     - **Security**: Always run with TLS (`--cert-file`, `--key-file`, `--client-cert-auth`).  
   - **Tools**: `etcdctl` for snapshots (`etcdctl snapshot save backup.db`).

3. **Scheduler (`kube-scheduler`)**  
   - **What it is**: Watches for **newly created Pods** with no assigned node and **binds them to a Worker Node**.  
   - **How it works**:  
     1. **Filters**: Eliminates nodes that *can't* run the Pod (e.g., insufficient CPU, missing taints).  
        - *Predicates*: `PodFitsResources`, `NoDiskConflict`, `HostName`.  
     2. **Scores**: Ranks remaining nodes (e.g., least busy node gets highest score).  
        - *Priorities*: `LeastRequestedPriority`, `BalancedResourceAllocation`.  
     3. **Binds**: Sends `Binding` object to API Server → Pod assigned to a node.  
   - **Why it matters**:  
     - **Custom Schedulers**: You can run multiple schedulers (e.g., for machine learning workloads).  
     - **Failure Impact**: New Pods hang in `Pending` state (existing Pods unaffected).  
     - **Taints & Tolerations**: Key mechanism to control Pod placement (e.g., `dedicated=ml:NoSchedule`).

4. **Controller Manager (`kube-controller-manager`)**  
   - **What it is**: A **process running multiple controllers** that monitor cluster state via API Server and **reconcile desired vs. actual state**.  
   - **Key Controllers**:  
     - **Node Controller**: Checks node health (via `kubelet` heartbeats). Marks nodes `NotReady` if unresponsive.  
     - **Replication Controller**: Ensures `replicas` of a Pod are running. Restarts failed Pods.  
     - **Endpoint Controller**: Populates `Endpoints` object (IPs of Pods behind a Service).  
     - **Service Account Controller**: Creates default ServiceAccounts in new namespaces.  
   - **How it works**:  
     - Each controller runs an **infinite reconciliation loop**:  
       ```python
       while True:
         actual_state = get_current_state()
         desired_state = get_desired_state()
         if actual_state != desired_state:
           make_changes(actual_state, desired_state)
       ```  
   - **Why it matters**:  
     - **Self-healing**: If a node dies, the ReplicaSet controller spins up new Pods elsewhere.  
     - **Stateless**: Can be restarted without data loss (state is in `etcd`).  
     - **Failure Impact**: Cluster loses self-healing (e.g., dead nodes won't be replaced).

5. **Cloud Controller Manager (CCM)**  
   - **What it is**: **Decouples cloud provider-specific logic** from core Kubernetes (introduced in v1.6).  
   - **How it works**:  
     - Runs cloud-specific controllers (e.g., for AWS, GCP, Azure):  
       - **Node Controller**: Sets node labels (e.g., `region=us-east-1`).  
       - **Service Controller**: Creates cloud LoadBalancers for `Service: type=LoadBalancer`.  
       - **Route Controller**: Manages pod network routes (e.g., VPC routes in AWS).  
   - **Why it matters**:  
     - **Vendor neutrality**: Core Kubernetes stays cloud-agnostic.  
     - **Plug-and-play**: Swap cloud providers without rebuilding the cluster.  
     - **Failure Impact**: Cloud resources (LBs, routes) won't auto-provision, but cluster keeps running.

---

#### **II. Worker Node Components**  
*Where your containers run. Managed by the Control Plane.*

1. **Kubelet**  
   - **What it is**: The **primary node agent** that ensures containers run in a Pod.  
   - **How it works**:  
     - Registers the node with the API Server.  
     - **Watches API Server** for Pod assignments *to its node*.  
     - **Manages Pod lifecycle**: Starts/stops containers via the Container Runtime.  
     - **Reports status** (e.g., `Ready`, `NotReady`) via node heartbeats (default: 10s).  
     - **Executes probes**: `livenessProbe`, `readinessProbe`, `startupProbe`.  
   - **Why it matters**:  
     - **No standalone operation**: Kubelet *cannot* function without API Server.  
     - **Pod Spec Source**: Accepts Pod specs from API Server *or* local manifests (`--pod-manifest-path`).  
     - **Critical Failure**: Node marked `NotReady`; Pods evicted after 5m (default).

2. **Kube-proxy**  
   - **What it is**: Maintains **network rules** on nodes to enable Pod-to-Pod communication.  
   - **How it works**:  
     - **Watches API Server** for `Services` and `Endpoints`.  
     - **Sets up rules** to route traffic to Pods:  
       - **`iptables` mode (default)**: Uses Linux `iptables` for DNAT. Simple but slow to update.  
       - **`IPVS` mode**: Uses Linux Virtual Server (LVS). Better for >1k Services (supports weighted round-robin).  
     - **Example**: For `Service: nginx`, routes `:80` → Pod IPs in `Endpoints`.  
   - **Why it matters**:  
     - **Not a proxy**: It configures the *node's* networking (like a network plugin sidekick).  
     - **Failure Impact**: New Services won't route, but existing connections work.

3. **Container Runtime**  
   - **What it is**: Software responsible for **running containers** (e.g., Docker, containerd, CRI-O).  
   - **How it works**:  
     - Implements the **Container Runtime Interface (CRI)** – Kubernetes' API for runtimes.  
     - **Key Tasks**: Pull images, start/stop containers, stream logs.  
     - **Common Runtimes**:  
       - **containerd** (default in kubeadm): Lightweight, used by Docker.  
       - **CRI-O**: Kubernetes-native, optimized for OCI compliance.  
       - **Docker** (deprecated since v1.20): Requires `dockershim` (removed in v1.24).  
   - **Why it matters**:  
     - **Security**: `containerd`/`CRI-O` reduce attack surface vs. Docker.  
     - **CRI is mandatory**: Without a CRI-compatible runtime, Kubelet won't start Pods.

---

#### **III. Add-ons**  
*Essential extensions (deployed as Pods in `kube-system` namespace).*

1. **CoreDNS**  
   - **What it is**: The **default DNS server** for Kubernetes (replaced `kube-dns` in v1.11).  
   - **How it works**:  
     - Assigns DNS names to Services: `<service>.<namespace>.svc.cluster.local`  
       (e.g., `nginx.default.svc.cluster.local`).  
     - Configured via `ConfigMap` (e.g., add custom upstream DNS).  
   - **Why it matters**:  
     - **Critical for Service discovery**: Pods resolve Services via DNS.  
     - **Autoscaling**: Scales with cluster size (via `CoreDNS` Deployment).

2. **Kubernetes Dashboard**  
   - **What it is**: **Web-based UI** for managing cluster resources.  
   - **Key Notes**:  
     - Not installed by default (use `kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/master/aio/deploy/recommended.yaml`).  
     - **Requires RBAC**: Needs a `ClusterRoleBinding` for access.  
     - **Security Risk**: Never expose publicly without auth (use `kubectl proxy`).

3. **Ingress Controllers**  
   - **What it is**: **Not a core component** – a *separate controller* that implements the `Ingress` API.  
   - **How it works**:  
     - Watches for `Ingress` resources.  
     - Configures a **load balancer** (e.g., Nginx, Traefik, AWS ALB) to route HTTP(S) traffic to Services.  
   - **Why it matters**:  
     - **Replaces Service: type=LoadBalancer** for HTTP routing (saves cost).  
     - **Critical for production**: Handles TLS termination, path-based routing (`/api` → backend, `/` → frontend).

---

### **Chapter 5: Kubernetes API & Declarative Model**  
*How you interact with Kubernetes and define your desired state.*

---

#### **I. Understanding the Kubernetes API**  
- **RESTful Design**:  
  - Resources are **nouns** (Pod, Service, Deployment).  
  - Verbs are **HTTP methods**: `GET`, `POST`, `PUT`, `PATCH`, `DELETE`.  
- **API Groups**:  
  - **Core Group** (`/api/v1`): Pods, Services, Nodes.  
  - **Named Groups** (`/apis/<group>/<version>`):  
    - `apps/v1`: Deployments, StatefulSets  
    - `networking.k8s.io/v1`: Ingress, NetworkPolicies  
- **Resource Structure**:  
  ```yaml
  apiVersion: apps/v1     # Which API group/version
  kind: Deployment        # Resource type
  metadata:
    name: nginx           # Unique name in namespace
    namespace: default    # Optional (default=namespace)
    labels:               # User-defined key/value
      app: nginx
  spec:                   # Desired state (MUST)
    replicas: 3
    template:             # Pod template
      spec:
        containers:
        - name: nginx
          image: nginx:1.25
  status:                 # Current state (set by system)
    replicas: 3
    conditions: [...]     # e.g., "Available=True"
  ```

---

#### **II. Declarative vs Imperative Configuration**  
| **Declarative**                          | **Imperative**                          |
|------------------------------------------|-----------------------------------------|
| **Define "what"** (desired state)        | **Define "how"** (commands to execute)  |
| Use `kubectl apply -f manifest.yaml`     | Use `kubectl create/delete/run`         |
| **Preserves updates** (3-way merge)      | **Overwrites** all changes              |
| **Idempotent**: Safe to run repeatedly   | Not idempotent (e.g., `create` fails if exists) |
| **Best for GitOps/CI/CD**                | Good for quick debugging/testing        |

- **Why Declarative Wins**:  
  - Tracks ownership via `kubectl.kubernetes.io/last-applied-configuration` annotation.  
  - Handles **field-level conflicts** (e.g., you change `replicas`, someone else changes `image`).  
  - Enables **infrastructure as code** (store YAML in Git).

---

#### **III. YAML and JSON Manifests**  
- **YAML is preferred** (human-readable, supports comments).  
- **Critical Sections**:  
  - `apiVersion`: Must match the resource's API group (e.g., `batch/v1` for CronJobs).  
  - `kind`: Resource type (case-sensitive: `Deployment`, not `deployment`).  
  - `metadata.name`: Unique per namespace (max 63 chars, DNS-subdomain rules).  
  - `spec`: **Mandatory** – defines desired state (structure varies by resource).  
- **Common Pitfalls**:  
  - Indentation errors (YAML is space-sensitive).  
  - Missing `apiVersion` (e.g., using `extensions/v1beta1` for Ingress – now deprecated).  
  - Forgetting `namespace` in metadata (defaults to `default`).

---

#### **IV. kubectl CLI Basics**  
*The Swiss Army knife for Kubernetes.*

| **Command**               | **Description**                                                                 | **Example**                                  |
|---------------------------|-------------------------------------------------------------------------------|----------------------------------------------|
| `kubectl get`             | List resources (short names: `po`=Pod, `svc`=Service, `no`=Node)              | `kubectl get pods -n kube-system`            |
| `kubectl describe`        | Show detailed resource info (events, mounts, conditions)                      | `kubectl describe pod nginx`                 |
| `kubectl create`          | **Imperative**: Create resource from manifest                                 | `kubectl create -f pod.yaml`                 |
| `kubectl apply`           | **Declarative**: Create/update resource (preserves manual changes)            | `kubectl apply -f deployment.yaml`           |
| `kubectl delete`          | Delete resource by name/file/label                                            | `kubectl delete svc nginx`                   |
| `kubectl logs`            | Print container logs (add `-f` for follow, `-c` for multi-container Pods)     | `kubectl logs nginx -f`                      |
| `kubectl exec`            | Run command in container (add `-it` for interactive shell)                    | `kubectl exec -it nginx -- sh`               |
| **Pro Tips**              |                                                                               |                                              |
| `--dry-run=client`        | Validate manifest without creating (add `-o yaml` to see generated YAML)      | `kubectl create deploy nginx --image=nginx --dry-run=client -o yaml` |
| `-l app=nginx`            | Filter by label                                                               | `kubectl get pods -l app=nginx`              |
| `--context` / `--namespace` | Switch contexts/namespaces                                                  | `kubectl get pods --context=prod --namespace=staging` |

---
