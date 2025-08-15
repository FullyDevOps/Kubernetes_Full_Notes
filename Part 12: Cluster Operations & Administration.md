### **Chapter 41: Cluster Setup & Bootstrapping**

#### **1. Using kubeadm to Create Clusters**
*   **What it is:** `kubeadm` is the **official, supported tool** for bootstrapping production-ready Kubernetes clusters. It automates the complex setup of the control plane and worker nodes.
*   **Core Workflow:**
    *   **`kubeadm init`:** Initializes the **first control plane node**.
        *   Generates CA certificates & keys (or uses provided ones).
        *   Writes kubeconfig files for admin (`/etc/kubernetes/admin.conf`) and system components (`/etc/kubernetes/controller-manager.conf`, etc.).
        *   Starts static Pod manifests for `kube-apiserver`, `kube-controller-manager`, `kube-scheduler`, and `etcd` (if local) in `/etc/kubernetes/manifests`.
        *   Generates the `kubeadm join` command token for adding nodes.
    *   **`kubeadm join`:** Adds **worker nodes** or **additional control plane nodes** to the cluster.
        *   Uses the token/bootstrap token to securely connect to the API server.
        *   Fetches certificates and configuration.
        *   Starts kubelet and kube-proxy.
*   **Critical Details:**
    *   **Pre-requisites:** Docker/containerd, `kubeadm`, `kubelet`, `kubectl` installed; swap disabled; unique hostnames; network connectivity; ports open (6443, 2379-2380, 10250, etc.).
    *   **Configuration:** Use `kubeadm init --config=config.yaml` for precise control (API server flags, networking pod CIDR, etcd endpoints, image repositories). *Never skip config for production.*
    *   **Networking:** `kubeadm` **does NOT install a CNI plugin**. You **MUST** install one (Calico, Flannel, Cilium) immediately after `kubeadm init` (before `join`) or cluster networking fails.
    *   **Token Management:** Tokens expire (default 24h). Use `kubeadm token create --print-join-command` for new tokens. Bootstrap tokens are stored as Secrets in the `kube-system` namespace.
    *   **Reset:** `kubeadm reset` cleans up node state (deletes manifests, network config, certs) – **essential** before re-joining a node.
*   **Why it Matters:** Provides a **standardized, repeatable, and supported** method to create clusters, avoiding manual error-prone setup. Foundation for HA and upgrades.

#### **2. High Availability (HA) Clusters**
*   **What it is:** A cluster designed to **eliminate single points of failure (SPOF)**, specifically for the **control plane** (API server, etcd, scheduler, controller manager). Worker nodes are inherently scalable/fault-tolerant.
*   **Core Architecture:**
    *   **Multiple Control Plane Nodes:** Typically 3 or 5 nodes (odd number for quorum).
    *   **Load Balancer (Frontend):** Essential. Distributes traffic from `kubectl`, nodes, and components to *all* healthy API servers (e.g., HAProxy, Nginx, cloud LB). **Must be Layer 4 (TCP)**. Health checks on API server port (6443).
    *   **External etcd Cluster (Recommended):** Dedicated 3/5 node etcd cluster *separate* from control plane nodes. Provides resilience and avoids resource contention. *kubeadm can also set up stacked etcd (etcd collocated on CP nodes), but external is preferred for production HA.*
*   **kubeadm HA Setup Flow:**
    1.  Setup **external etcd cluster** (3 nodes) OR configure stacked etcd during init.
    2.  Initialize **first CP node** with `kubeadm init --control-plane-endpoint="LOAD_BALANCER_DNS:6443" --upload-certs` (upload-certs enables secure cert sharing for subsequent CP nodes).
    3.  Install CNI.
    4.  Join **additional CP nodes** using the special `kubeadm join ... --control-plane` command (uses certs uploaded in step 2).
    5.  Configure **Load Balancer** to point to all CP node IPs on port 6443.
    6.  Join **worker nodes** normally (`kubeadm join` without `--control-plane`).
*   **Critical Details:**
    *   **Quorum:** etcd requires a majority (N/2 + 1) of members to be healthy to operate. 3-node etcd tolerates 1 failure; 5-node tolerates 2.
    *   **Certificate Sharing:** `--upload-certs` uses a one-time token to securely share PKI assets (encrypted) between CP nodes during join. *Without this, you must manually copy certs.*
    *   **Load Balancer Persistence:** Session affinity (sticky sessions) is **NOT** required for API servers (stateless).
    *   **Worker Node LB:** Worker nodes only talk to the Load Balancer's VIP (Virtual IP), **not** individual CP nodes.
    *   **Failure Scenarios:** If LB fails, cluster control plane is inaccessible. If 1 CP node fails in a 3-node setup, cluster remains operational (quorum intact). If etcd loses quorum, cluster becomes read-only.
*   **Why it Matters:** Ensures cluster control plane remains available during node failures, maintenance, or network partitions – **critical for production uptime**.

#### **3. etcd Clustering and Backup**
*   **What it is:** etcd is the **distributed, consistent key-value store** acting as Kubernetes' **single source of truth** (cluster state, configs, secrets). Clustering provides HA; backup provides recovery.
*   **Clustering Mechanics:**
    *   **Raft Consensus Algorithm:** Ensures all members agree on the state. Requires odd number of members (3,5,7) for quorum.
    *   **Member Roles:** Leader (handles client requests & log replication), Followers (replicate logs, vote in elections).
    *   **Quorum:** `(N/2) + 1` members must be available for writes (e.g., 2 out of 3).
    *   **Peer Communication:** Uses ports 2380 (inter-member) and 2379 (client API).
*   **Backup Strategies (CRITICAL):**
    *   **Snapshot Backup (`etcdctl snapshot save`):**
        *   **Command:** `ETCDCTL_API=3 etcdctl --endpoints=<ETCD_IPS> snapshot save snapshot.db --cacert=/etc/kubernetes/pki/etcd/ca.crt --cert=/etc/kubernetes/pki/etcd/server.crt --key=/etc/kubernetes/pki/etcd/server.key`
        *   **What it does:** Takes an **online, consistent snapshot** of the entire etcd database *at a specific point in time*. **Does NOT lock etcd.**
        *   **Frequency:** **Daily minimum**; often hourly for critical clusters. Store *offsite* (different storage, different cloud region).
        *   **Verification:** `etcdctl snapshot status snapshot.db` (check size, hash, revision).
    *   **Filesystem Backup (Less Reliable):**
        *   Stop etcd process -> Copy `/var/lib/etcd` -> Restart etcd.
        *   **Risky:** Requires stopping etcd (downtime), potential for corruption if not done perfectly. **Snapshot is strongly preferred.**
*   **Critical Details:**
    *   **Backup Location:** **NEVER** store backups on the same disk/volume as etcd data (`/var/lib/etcd`). Use separate storage (S3, NFS, different disk).
    *   **Encryption:** Backups contain **ALL SECRETS**. Encrypt them (e.g., `gpg`, `sops`, cloud KMS) before storage.
    *   **Retention:** Follow organizational policy (e.g., 7 daily, 4 weekly).
    *   **Test Restores:** **MANDATORY.** Regularly test restoring from backup to a *disposable* cluster. A backup you haven't tested is useless.
    *   **Size Matters:** etcd DB size directly impacts restore time. Prune resources (e.g., old events, completed jobs) and set `--auto-compaction-retention` (e.g., `--auto-compaction-retention=1h`).
*   **Why it Matters:** etcd failure = cluster death without a valid backup. This is the **most critical operational procedure** for cluster survival.

#### **4. Upgrading Control Plane and Nodes**
*   **What it is:** The process of moving the cluster from one Kubernetes version to the next (e.g., v1.26.x -> v1.27.x), following the **strict sequential order** defined by Kubernetes.
*   **Upgrade Order (NON-NEGOTIABLE):**
    1.  **Upgrade `kubeadm`** on all control plane nodes.
    2.  **Drain** the first control plane node (see Ch44).
    3.  **Upgrade the control plane** on that node (`kubeadm upgrade apply v1.XX.Y`).
    4.  **Uncordon** the node.
    5.  **Repeat steps 2-4** for remaining control plane nodes (one at a time!).
    6.  **Upgrade `kubeadm`, `kubelet`, `kubectl`** on **all worker nodes**.
    7.  **Drain, upgrade kubelet, uncordon** each worker node **one at a time**.
*   **kubeadm Upgrade Mechanics:**
    *   `kubeadm upgrade plan`: Shows available versions and current state.
    *   `kubeadm upgrade apply v1.XX.Y`: Does the heavy lifting:
        *   Checks version skew compatibility.
        *   Downloads new control plane images.
        *   Updates static Pod manifests (`/etc/kubernetes/manifests`).
        *   Updates kube-config files.
        *   Upgrades Cluster API objects (e.g., `kubeadm-config` ConfigMap).
        *   **Does NOT touch `kubelet` or `kubectl`** (must be upgraded separately on nodes).
*   **Critical Details:**
    *   **Version Skew Policy:** Only one minor version difference allowed between components (e.g., v1.27 CP can talk to v1.26 nodes). **Never skip minor versions** (e.g., v1.25 -> v1.27). Go step-by-step (v1.25->v1.26->v1.27).
    *   **Draining:** **Essential** before upgrading a node (CP or worker) to safely evict pods (see Ch44).
    *   **Rolling Upgrades:** Upgrading CP nodes one-by-one maintains HA. Upgrading workers one-by-one minimizes application impact.
    *   **Addons:** Check addon compatibility (CNI, CoreDNS, Metrics Server) *before* upgrade. CoreDNS often needs manual version update via `kubeadm upgrade apply`.
    *   **Backup First:** **ALWAYS** take an etcd snapshot **immediately before** starting the upgrade process.
    *   **Phased Rollouts:** Test upgrades on a non-production cluster first. Use canary upgrades for worker nodes if possible.
*   **Why it Matters:** Keeps the cluster secure, stable, and compatible with new features. Skipping steps or order risks cluster instability or data loss.

---

### **Chapter 42: Managed Kubernetes (EKS, GKE, AKS)**

#### **1. Differences Between Managed Services**
*   **Core Concept:** Cloud providers host and manage the **Kubernetes control plane** (API server, scheduler, controller manager, etcd). You manage **worker nodes** and **workloads**.
*   **Key Differences Breakdown:**

    | Feature               | Amazon EKS                                     | Google GKE                                     | Azure AKS                                      |
    | :-------------------- | :--------------------------------------------- | :--------------------------------------------- | :--------------------------------------------- |
    | **Control Plane HA**  | Automatic (3 AZs), billed per cluster          | Automatic (regional cluster = 3+ zones), free  | Automatic (zonal or regional), billed per cluster |
    | **Node Management**   | Self-managed (ASG), EKS Managed Node Groups (MNG) | GKE Autopilot (serverless), Standard (self-managed) | AKS Managed Clusters (MC), VMSS (self-managed) |
    | **Networking**        | Uses VPC CNI (IP-per-pod), Calico optional     | Uses VPC-native (alias IPs), seamless GCP integration | Uses Azure CNI or Kubenet, seamless Azure integration |
    | **Upgrades**          | Control plane: Manual (1 click). Nodes: Manual | Control plane: Auto/manual. Nodes: Auto/manual (with maintenance windows) | Control plane: Auto/manual. Nodes: Manual (with node pools) |
    | **Pricing Model**     | $0.10/hr per cluster + Node costs              | Free control plane + Node costs                | Free control plane + Node costs                |
    | **Unique Strength**   | Deep AWS integration (IAM, ALB, RDS)           | Best-in-class GCP integration (Cloud SQL, BigQuery), Autopilot serverless | Deep Azure integration (AD, SQL DB, Event Hubs), Hybrid (Arc) |
    | **Unique Weakness**   | Complex networking setup, slower upgrades      | Less flexible node customization (Autopilot)   | Historically slower feature adoption           |
    | **CLI Tool**          | `eksctl` (community), AWS Console/CLI          | `gcloud` (native)                              | `az aks` (native)                              |

*   **Critical Insight:** **"Managed" only applies to the control plane.** You are **always responsible** for:
    *   Worker node OS/security/patches
    *   Cluster networking configuration (CNI, ingress)
    *   Application deployments & health
    *   Monitoring & logging setup
    *   Cost optimization (right-sizing nodes)
*   **Why it Matters:** Choosing the right service depends on cloud ecosystem lock-in, specific feature needs, pricing sensitivity, and operational preferences.

#### **2. IAM Integration**
*   **What it is:** Mapping cloud provider **Identity and Access Management (IAM)** identities (users, groups, service accounts) to **Kubernetes RBAC** roles for secure access.
*   **Mechanics by Provider:**
    *   **EKS:**
        *   Uses `aws-auth` ConfigMap in `kube-system`.
        *   Maps AWS IAM ARNs (Users, Roles) to Kubernetes usernames/groups.
        *   Example: `mapRoles: - rolearn: arn:aws:iam::123456789012:role/eks-node-role username: system:node:{{EC2PrivateDNSName}} groups: - system:bootstrappers - system:nodes` (Nodes). For users: `- userarn: arn:aws:iam::123456789012:user/admin username: admin groups: - system:masters`.
        *   **IRSA (IAM Roles for Service Accounts):** **CRITICAL.** Allows pods to assume IAM roles via Kubernetes ServiceAccount annotations (`eks.amazonaws.com/role-arn`). Uses OIDC federation. *Avoids using node IAM roles for pods.*
    *   **GKE:**
        *   Uses Google Identity (Cloud Identity, Workspace) directly.
        *   IAM permissions granted via Cloud Console/`gcloud` map to Kubernetes RBAC roles *automatically* (e.g., `roles/container.admin` -> `cluster-admin`).
        *   **Workload Identity:** **RECOMMENDED.** Federates GCP service accounts (GSA) to Kubernetes service accounts (KSA). Pod uses KSA, which is bound to GSA. KSA annotated with `iam.gke.io/gcp-service-account=gsa-name@project.iam.gserviceaccount.com`. *Replaces legacy metadata server access.*
    *   **AKS:**
        *   Uses Azure AD (Entra ID) integration.
        *   **Option 1 (Legacy):** Cluster admin/service principal. Less secure.
        *   **Option 2 (Recommended - AAD Integration):** Cluster binds to Azure AD. Users/groups authenticated via `kubectl` using Azure AD tokens. RBAC defined in Kubernetes using Azure AD object IDs. Requires `az aks get-credentials --admin` for initial setup, then standard `kubectl` uses AD auth.
        *   **Managed Identity:** AKS cluster uses system/user-assigned managed identity for Azure resource access (e.g., pulling images from ACR). Pods use **Pod Identity** (e.g., `aad-pod-identity`, now largely superseded by **Workload Identity Federation**) or **Azure AD Workload Identity** (similar to GKE/EKS) to access Azure resources.
*   **Critical Details:**
    *   **Least Privilege:** Always grant the *minimum* required IAM *and* RBAC permissions.
    *   **IRSA/Workload Identity:** **Mandatory best practice** for secure pod access to cloud resources. Avoids compromising node IAM roles.
    *   **Audit Logs:** Cloud provider IAM logs + Kubernetes audit logs are essential for security forensics.
    *   **Service Accounts:** Cloud provider-managed clusters often create default service accounts (e.g., `default` in `default` namespace) with *excessive* permissions. **Restrict or delete them.**
*   **Why it Matters:** Securely connects cloud infrastructure permissions to Kubernetes access – **fundamental for production security and least privilege.**

#### **3. Networking Setup**
*   **Core Challenge:** Integrating Kubernetes pod/node networking with the cloud provider's VPC/VNet while enabling service exposure.
*   **Key Components & Provider Differences:**
    *   **Pod Networking (CNI):**
        *   **EKS:** Default = **AWS VPC CNI** (IPs from VPC subnet assigned directly to pods). Requires careful IP planning. Calico (for policy) or Cilium often layered on top. *Alternative:* `aws-k8s-cni-chaining` mode.
        *   **GKE:** **VPC-native (Alias IPs)** is default/only option for Standard clusters. Pods get IPs *within* the VPC subnet range without NAT. Seamless integration with GCP networking (firewalls, routes). Autopilot uses a managed CNI.
        *   **AKS:** **Azure CNI** (pods get IPs directly from VNet subnet) or **kubenet** (simpler, uses NAT, pods get IPs from cluster-defined range). Azure CNI is recommended for production. Calico/Cilium for policy.
    *   **Service Exposure:**
        *   **`LoadBalancer` Services:** Cloud provider creates a **native load balancer** (ELB/ALB on AWS, Cloud Load Balancing on GCP, Azure Load Balancer on Azure). Automatically configures health checks, listeners, target groups.
        *   **Ingress Controllers:** Typically deployed (Nginx, Traefik, ALB Ingress Controller for EKS) to handle HTTP(S) routing. Integrates with cloud LBs.
        *   **Private Clusters:** All providers support control plane endpoint only accessible from within VPC/VNet (no public IP). EKS/GKE require a proxy/NAT; AKS uses private link.
    *   **DNS:**
        *   **CoreDNS:** Deployed by default. Integrates with cloud DNS (Route53, Cloud DNS, Azure DNS) for service discovery *within* the cluster.
        *   **External DNS:** Often added (e.g., `external-dns` controller) to automatically create public DNS records for `Ingress`/`LoadBalancer` resources in cloud DNS.
*   **Critical Details:**
    *   **IP Exhaustion:** VPC CNI (EKS) and Azure CNI consume significant IPs (1 per pod + overhead). Plan subnets carefully! GKE VPC-native is more efficient.
    *   **Security Groups / Firewall Rules:** Cloud network security groups/firewalls **MUST** allow traffic between nodes (ports 10250, 30000-32767), to the control plane (port 443), and to the LB health check IPs.
    *   **Network Policies:** Enable CNI plugins that support `NetworkPolicy` (Calico, Cilium, Azure Network Policies) and **enforce them rigorously**. Cloud security groups *alone* are insufficient for pod-to-pod security.
    *   **Egress Control:** Configure NAT gateways, egress proxies, or service endpoints to control outbound pod traffic securely.
*   **Why it Matters:** Networking is the most common source of cluster failures and security gaps. Proper setup is essential for performance, security, and application connectivity.

#### **4. Upgrades and Maintenance**
*   **What it is:** How managed services handle control plane and node upgrades, including maintenance windows and rollback.
*   **Provider Mechanics:**
    *   **Control Plane Upgrades:**
        *   **EKS:** Manual via Console/CLI (`aws eks update-cluster-config --resources-vpc-config ...`). Takes 1-2 hours. **Rollback not supported** – you must recreate the cluster from backup if needed. *Requires etcd backup before upgrade!*
        *   **GKE:** Automatic (within maintenance window) or manual. Regional clusters upgrade CP with minimal downtime (rolling across zones). Rollback possible for a short window after upgrade.
        *   **AKS:** Automatic (within maintenance window) or manual. Upgrades CP nodes sequentially. Rollback possible for a short window.
    *   **Node Upgrades:**
        *   **EKS:** Manual. Requires replacing nodes (using ASG rolling updates, `eksctl`, or `kured`). Drain nodes first!
        *   **GKE:** Automatic (with node auto-upgrade) or manual. Autopilot nodes are fully managed/transparent. Standard nodes upgrade via node pools (rolling update).
        *   **AKS:** Manual. Drain nodes and recreate via node pool upgrade (`az aks nodepool upgrade`). Supports surge upgrades for faster rollout.
    *   **Maintenance Windows:** All providers allow scheduling upgrades during off-peak hours.
    *   **Patch Management:** **YOU are responsible** for OS/kernel patches on worker nodes (except GKE Autopilot). Use tools like `kured`, `daemonsets`, or cloud-native patching (SSM, OS Config).
*   **Critical Details:**
    *   **Backup Before CP Upgrade:** **ABSOLUTELY MANDATORY** for EKS (no rollback). Highly recommended for GKE/AKS even with rollback.
    *   **Node Replacement vs. In-Place:** Managed services typically **replace nodes** (new VM + drain old) rather than in-place OS upgrades. This is safer.
    *   **Addons:** Managed services often auto-upgrade critical addons (CoreDNS, Metrics Server). Verify compatibility.
    *   **Version Support:** Providers only support specific minor versions (e.g., N, N-1, N-2). Stay within supported range.
    *   **Draining:** Cloud providers usually handle draining during node replacement, but verify behavior (e.g., `pod-eviction-timeout`).
*   **Why it Matters:** Minimizes downtime during upgrades, ensures security patching, and leverages provider automation while understanding your responsibilities (especially node patching).

---

### **Chapter 43: Self-Hosted Clusters**

#### **1. Bare Metal Setup**
*   **What it is:** Installing Kubernetes directly on physical servers (or dedicated VMs) without a cloud provider layer. Maximum control, performance, and cost efficiency for specific workloads.
*   **Key Considerations & Challenges:**
    *   **Hardware Provisioning:** Manual or via tools (Ironic, MAAS, Terraform + bare metal providers). Requires IPMI/iDRAC for automation.
    *   **OS Setup:** Standardized OS image (Ubuntu, RHEL, Flatcar) across nodes. Critical for consistency.
    *   **Networking:**
        *   **L2/L3 Configuration:** VLANs, BGP (using MetalLB, Calico BGP), or underlay networking. **No cloud LBs.**
        *   **CNI Choice:** Calico (BGP/IP-in-IP), Cilium (eBPF), Flannel (VXLAN) are common. Must handle pod networking *and* service exposure (`MetalLB` for `LoadBalancer` Services).
        *   **DNS:** CoreDNS + internal DNS server (e.g., CoreDNS forwarding, dnsmasq) for cluster and service discovery.
        *   **Load Balancing:** `MetalLB` (L2/BGP), HAProxy/Nginx ingress controllers, or cloud LBs if hybrid.
    *   **Storage:** **Major challenge.** Requires external solutions:
        *   **Shared File:** NFS, CephFS, GlusterFS.
        *   **Shared Block:** iSCSI, Ceph RBD, Portworx, Longhorn (distributed).
        *   **Local Storage:** `local-path-provisioner`, but no HA. Requires topology awareness.
    *   **Bootstrapping:** `kubeadm` is the primary tool (see Ch41). Alternatives: `kops` (can target bare metal), `rke`, manual.
    *   **HA Setup:** Requires manual setup of external etcd cluster (3+ nodes) and Load Balancer (HAProxy/Nginx) in front of CP nodes.
*   **Critical Details:**
    *   **Kernel Tuning:** Often required (e.g., `net.bridge.bridge-nf-call-iptables=1`, `vm.max_map_count` for Elasticsearch).
    *   **Clock Synchronization:** **Mandatory** (NTP/Chrony) across all nodes. etcd is time-sensitive.
    *   **Firewalling:** Strict `iptables`/`nftables` rules between nodes (control plane ports, node ports, etcd ports).
    *   **Bare Metal Switches:** Must allow MAC moves (for LB VIP failover) and potentially BGP sessions (for Calico/MetalLB).
    *   **Cost vs. Complexity:** Cheaper long-term for large scale, but significantly higher operational overhead than managed/cloud.
*   **Why it Matters:** Essential for air-gapped environments, extreme performance needs, strict data locality/compliance, or cost optimization at massive scale where cloud egress costs dominate.

#### **2. Using k3s, MicroK8s, K0s**
*   **What they are:** **Lightweight, opinionated Kubernetes distributions** designed for simplicity, lower resource usage, and edge/IoT scenarios. *Not drop-in replacements for full K8s in all cases.*
*   **Breakdown:**
    *   **k3s (Rancher Labs):**
        *   **Key Features:** Single binary (<100MB), embeds SQLite (default, replaces etcd), lightweight networking (Flannel + Service LB), simplified HA setup (`--server https://<IP>`), agent mode for workers.
        *   **Use Cases:** Edge, IoT, CI/CD runners, dev/test, resource-constrained environments. **Not recommended for large-scale stateful prod workloads** (SQLite limitation).
        *   **HA:** Uses embedded `etcd` *or* external DB (MySQL, PostgreSQL, etcd) for true HA. Simpler than `kubeadm` HA.
        *   **Management:** `k3s` CLI, `k3s-killall.sh` for reset. Integrates with Rancher for management.
    *   **MicroK8s (Canonical):**
        *   **Key Features:** `snap`-packaged, minimal dependencies, includes common addons (`dns`, `dashboard`, `ingress`, `storage`, `gpu`, `istio`), uses `dqlite` (distributed SQLite) for HA.
        *   **Use Cases:** Local development, single-node clusters, small edge deployments, Ubuntu-focused environments.
        *   **HA:** Multi-node HA via `microk8s add-node` (uses `dqlite` raft consensus). Simpler than `kubeadm`.
        *   **Management:** `microk8s` CLI (`microk8s.enable`, `microk8s.disable`, `microk8s.status`). Excellent for quick setup.
    *   **K0s (Mirantis):**
        *   **Key Features:** Single binary, **full etcd support** (external or embedded), designed for **production-grade simplicity**, strong HA focus, no dependencies. "Zero touch" operations.
        *   **Use Cases:** Production edge, private cloud, anywhere needing a simple, robust, full-featured K8s distribution. Closer to `kubeadm` capability than k3s/MicroK8s.
        *   **HA:** Built-in support for external etcd or embedded etcd cluster. Control plane nodes join via simple token.
        *   **Management:** `k0s` CLI (`k0s controller`, `k0s worker`, `k0s token`). Focuses on operational simplicity.
*   **Critical Comparison:**
    *   **Full K8s API:** All provide it, but **k3s disables some legacy APIs by default** (check compatibility!).
    *   **etcd:** k3s (SQLite default), MicroK8s (`dqlite`), K0s (full etcd). **For production HA stateful workloads, K0s (with etcd) or MicroK8s (with HA) are safer than k3s SQLite.**
    *   **Resource Footprint:** k3s < MicroK8s < K0s (but K0s still much lighter than vanilla K8s).
    *   **Management:** k3s/MicroK8s have simpler CLIs; K0s focuses on operational robustness.
    *   **Community/Support:** k3s (largest community), MicroK8s (Ubuntu ecosystem), K0s (growing, enterprise focus).
*   **Why it Matters:** Provides viable Kubernetes options for environments where `kubeadm`/vanilla K8s is too heavy or complex, especially at the edge.

#### **3. Edge Computing with Lightweight K8s**
*   **What it is:** Deploying Kubernetes clusters on resource-constrained devices (gateways, IoT hubs, factory floors, remote sites) to run containerized applications closer to data sources/users.
*   **Challenges Addressed by Lightweight Distros (k3s, MicroK8s, K0s):**
    *   **Limited Resources:** CPU, RAM, storage, power. Lightweight distros minimize overhead.
    *   **Intermittent Connectivity:** Must operate autonomously when disconnected from central cloud. Edge clusters often run standalone.
    *   **Remote Management:** Need secure, low-bandwidth remote administration (GitOps like Flux/Argo CD is ideal).
    *   **Physical Security:** Devices may be in unsecured locations. Hardening is critical.
    *   **Scalability:** Managing hundreds/thousands of small clusters.
*   **Key Patterns:**
    *   **Single-Node Clusters:** Common for smallest edge devices (k3s agent mode, MicroK8s single-node).
    *   **Small Multi-Node Clusters:** 2-3 nodes for HA at critical edge sites (using distro HA features).
    *   **GitOps:** **De facto standard.** Central repo (Git) defines desired state. Agents (Flux, Argo CD) on edge clusters sync and apply manifests. Works offline.
    *   **Edge Orchestrators:** Tools like **K3s with Fleet** (Rancher), **MicroK8s with Cluster API**, or **Open Horizon** manage *many* edge clusters from a central point.
    *   **Data Flow:** Edge apps process data locally; only relevant results/metadata sent upstream (cloud/on-prem).
*   **Critical Details:**
    *   **Security:** Harden OS, use minimal distros, disable unused services, rotate certs frequently, use strong network policies, secure GitOps channels (SSH keys, HTTPS with certs).
    *   **Updates:** Automated, phased rollouts via GitOps. Test updates on staging clusters first.
    *   **Monitoring:** Lightweight agents (Prometheus Node Exporter, Loki), ship logs/metrics only when connected or buffer locally.
    *   **Stateful Workloads:** Use local storage cautiously; prefer stateless designs. For critical state, use distributed DBs designed for edge (e.g., Dqlite in MicroK8s, embedded etcd in K0s/k3s).
    *   **Lifecycle:** Plan for remote wipe/reprovisioning if device is lost/stolen.
*   **Why it Matters:** Enables real-time processing, reduces latency/bandwidth costs, and improves resilience for IoT and distributed applications. Lightweight K8s distros make this feasible.

---

### **Chapter 44: Cluster Maintenance**

#### **1. Node Drain and Cordon**
*   **What they are:** Safe procedures for preparing a node for maintenance (reboot, upgrade, decommissioning).
*   **Mechanics:**
    *   **`kubectl cordon <node>`:**
        *   **Action:** Marks the node as **unschedulable**. **NO NEW PODS** will be scheduled on it.
        *   **Effect:** Existing pods **continue running**. Does **NOT** evict running pods.
        *   **Use Case:** First step before maintenance; prevents new workloads landing on the node.
    *   **`kubectl drain <node> [--ignore-daemonsets] [--delete-emptydir-data]`:**
        *   **Action:** **1.** `cordon` the node. **2.** **Gracefully evict** all non-daemonset, non-mirrored, non-critical pods.
        *   **Eviction Process:**
            *   Sends `SIGTERM` to pod containers (honors `terminationGracePeriodSeconds`).
            *   Waits for pods to terminate (or timeout).
            *   If `--force` is used (not recommended), skips graceful termination.
        *   **Flags:**
            *   `--ignore-daemonsets`: Required if node has DaemonSet pods (e.g., logging, network agents). DaemonSets are *not* evicted (they should run on all nodes).
            *   `--delete-emptydir-data`: **DANGER!** Deletes data in `emptyDir` volumes. Only use if you accept data loss.
        *   **Use Case:** **Essential step** before rebooting/upgrading a node. Ensures pods are safely rescheduled.
*   **Critical Details:**
    *   **PodDisruptionBudgets (PDBs):** `drain` respects PDBs. If evicting pods would violate a PDB (e.g., `< minAvailable`), `drain` fails. **PDBs are mandatory for stateful apps.**
    *   **Grace Period:** `drain` uses `--pod-eviction-timeout` (default 5m). Pods exceeding their `terminationGracePeriodSeconds` get force-killed after this timeout.
    *   **StatefulSets:** `drain` evicts pods in reverse ordinal order (n, n-1, ... 0), respecting `podManagementPolicy`. **Crucial for ordered shutdown of stateful apps.**
    *   **Uncordon:** `kubectl uncordon <node>` reverses `cordon`, making the node schedulable again.
    *   **Automation:** Tools like `kured` automate node reboots (e.g., for kernel patches) using `drain`/`reboot`/`uncordon`.
*   **Why it Matters:** Prevents application downtime and data loss during node maintenance. **Skipping `drain` risks force-killing pods and violating PDBs.**

#### **2. Certificate Rotation**
*   **What it is:** Renewing the **TLS certificates** used for secure communication within the cluster (API server, kubelets, etcd, front-proxy). Certificates expire (default 1 year)!
*   **Why Rotation is Critical:** Expired certificates = **cluster outage**. Control plane components and kubelets fail to communicate. **This happens silently until expiration!**
*   **Rotation Methods:**
    *   **Manual Rotation (kubeadm):**
        *   **Check Expiry:** `kubeadm certs check-expiration`
        *   **Renew All:** `kubeadm certs renew all` (Does **NOT** restart components)
        *   **Renew Specific:** `kubeadm certs renew apiserver`
        *   **Restart Components:** **MANDATORY AFTER RENEWAL!** `sudo systemctl restart kubelet` (on all nodes) + restart control plane pods (they are static pods, restart by moving manifests: `mv /etc/kubernetes/manifests/* /tmp/ && sleep 20 && mv /tmp/* /etc/kubernetes/manifests/`).
        *   **Update kubeconfigs:** `kubeadm init phase kubeconfig all` (or specific: `admin`, `controller-manager`, etc.) - **Overwrites files!** Backup first.
    *   **Automatic Rotation (kubelet):**
        *   Kubelet automatically renews its **client certificate** (used to talk to API server) via the **Kubernetes CSR API**.
        *   Requires `--feature-gates=RotateKubeletClientCertificate=true` (default in recent versions).
        *   Admin must **APPROVE** CSRs: `kubectl get csr`, `kubectl certificate approve <csr-name>`. Automate with `kubelet-csr-approver` controller.
    *   **Managed Services:** GKE/AKS/EKS **fully automate** certificate rotation. **You don't manage it.** (A major benefit!).
*   **Critical Details:**
    *   **Expiry Monitoring:** **ALWAYS** monitor certificate expiry (Prometheus `kubeadm certs check-expiration` output, `openssl x509 -in /etc/kubernetes/pki/apiserver.crt -noout -enddate`). Set alerts 30+ days before expiry.
    *   **Static Pod Restart:** Renewing control plane certs **requires restarting** the static pods (API server, etc.). This causes brief API server unavailability (seconds). Schedule during maintenance window.
    *   **Front-Proxy Cert:** Often overlooked. Used by kube-aggregator (extension API servers). Renew with `kubeadm certs renew front-proxy-client`.
    *   **etcd Certs:** Renewed separately if external (`etcdctl` commands) or via `kubeadm certs renew etcd-*`.
    *   **Backup PKI:** **Before ANY rotation**, backup `/etc/kubernetes/pki` and `/etc/kubernetes/pki/etcd`! A mistake here can brick the cluster.
*   **Why it Matters:** Prevents catastrophic cluster outages due to certificate expiration. A core operational hygiene task for self-managed clusters.

#### **3. etcd Backup and Restore (Deep Dive)**
*   **Backup (Recap & Expansion):**
    *   **Command (External etcd):**
        ```bash
        ETCDCTL_API=3 etcdctl --endpoints=https://<ETCD_IP1>:2379,https://<ETCD_IP2>:2379,https://<ETCD_IP3>:2379 \
        --cacert=/etc/kubernetes/pki/etcd/ca.crt \
        --cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt \
        --key=/etc/kubernetes/pki/etcd/healthcheck-client.key \
        snapshot save /path/to/backup/snapshot-$(date +%Y%m%d).db
        ```
    *   **Verification:** `etcdctl --write-out=table snapshot status /path/to/backup/snapshot.db`
    *   **Encryption:** `gpg -c snapshot.db` (symmetric) or use cloud KMS envelope encryption.
    *   **Automation:** Script + cron/systemd timer. **Include error checking and alerting on failure.**
*   **Restore Scenarios & Procedures:**
    *   **Scenario 1: Restore to **Same Hardware** (e.g., after accidental deletion):**
        1.  Stop **all** kubelet and control plane services (`systemctl stop kubelet`, move `/etc/kubernetes/manifests`).
        2.  Stop etcd on **all** etcd members.
        3.  **Wipe etcd data dir** on **ALL** members: `rm -rf /var/lib/etcd/member/*`
        4.  Restore snapshot on **ONE** member: `etcdctl snapshot restore snapshot.db --data-dir /var/lib/etcd-restored`
        5.  Update etcd config on that member to point to `/var/lib/etcd-restored`.
        6.  Start etcd **ONLY** on that restored member.
        7.  Reconfigure cluster membership **from the restored member**:
            *   `etcdctl --endpoints=https://restored-ip:2379 member remove <old-member-id>`
            *   `etcdctl --endpoints=https://restored-ip:2379 member add new-member --peer-urls=https://new-ip:2380`
        8.  Start etcd on the **new** members (using `--initial-cluster-state existing`).
        9.  Once etcd cluster is healthy, restart control plane components and kubelet.
    *   **Scenario 2: Restore to **New Hardware** (Disaster Recovery):**
        1.  Provision **new** etcd cluster (same size, new IPs).
        2.  Restore snapshot to **one** new etcd node (as in Step 4 above).
        3.  Start etcd on that node.
        4.  Join other new etcd nodes to this restored member (as in Step 7/8 above).
        5.  Update **ALL** control plane nodes' kube-apiserver manifest to point to the **new etcd cluster endpoints**.
        6.  Restart kube-apiserver on all CP nodes (by moving manifests back).
        7.  Restart kube-controller-manager, kube-scheduler, kubelet.
    *   **Scenario 3: Point-in-Time Recovery (Using Backup + WAL):** Complex. Requires continuous WAL (Write-Ahead Log) backups. Rarely used; snapshot is primary method.
*   **Critical Details:**
    *   **Data Dir Consistency:** **Wiping `/var/lib/etcd/member` is mandatory** before restore. Mixing old and new data corrupts etcd.
    *   **Cluster Reconfiguration:** After restore, the etcd cluster membership **must be rebuilt** from the restored snapshot. Old member IDs are invalid.
    *   **Control Plane Config:** kube-apiserver `--etcd-servers` flag **MUST** reflect the *new* etcd cluster endpoints after restore (especially in Scenario 2).
    *   **Timing:** etcd restore is a **last resort**. Test restores regularly. A failed restore attempt can destroy the last good backup.
    *   **kubeadm Restore:** `kubeadm init --ignore-preflight-errors=All --experimental-upload-certs --etcd-upgrade` can sometimes help, but manual etcd restore is more reliable for major disasters.
*   **Why it Matters:** This is the **ultimate disaster recovery procedure**. Without a valid, tested etcd backup, cluster data loss is permanent.

#### **4. Disaster Recovery (DR) Strategies**
*   **What it is:** Comprehensive plans to recover cluster functionality and data after a catastrophic event (data center loss, major cloud outage, corrupted etcd).
*   **Key Strategies & Tiers:**
    *   **Tier 1: etcd Backup & Restore (Local):**
        *   **Goal:** Recover from etcd corruption/data loss *within the same infrastructure*.
        *   **Mechanism:** Regular etcd snapshots + tested restore procedure (as above).
        *   **RTO/RPO:** RTO: Hours. RPO: Snapshot interval (e.g., 24h). **Minimal viable DR.**
    *   **Tier 2: Cluster Replication (Active/Passive):**
        *   **Goal:** Failover to a secondary cluster in another location (AZ, Region, Cloud).
        *   **Mechanism:**
            *   **etcd Replication:** Tools like `etcd-backup-restore` (SAP), `etcd-druid` (Kyma), or custom scripts replicate snapshots to secondary location. **Not real-time.**
            *   **State Replication:** For stateful apps, use storage replication (e.g., Ceph, Portworx async DR) or application-level replication (DBs like PostgreSQL logical replication).
            *   **Manifest Sync:** GitOps (Flux/Argo CD) ensures app manifests are identical on primary and secondary clusters.
        *   **Failover Process:**
            1.  Stop primary cluster apps (if possible).
            2.  Promote secondary storage (if applicable).
            3.  Restore latest etcd snapshot to secondary etcd cluster.
            4.  Start secondary control plane pointing to restored etcd.
            5.  Update DNS/LB to point to secondary cluster.
        *   **RTO/RPO:** RTO: 30 mins - several hours. RPO: Snapshot interval + replication lag. **Most common production DR.**
    *   **Tier 3: Multi-Cluster Services (Active/Active):**
        *   **Goal:** Seamless failover with minimal downtime; clusters share load.
        *   **Mechanism:**
            *   **Global Load Balancing:** Cloud LB (GCP, AWS Global Accelerator) or DNS-based (Traffic Manager) directs traffic to healthy cluster.
            *   **State Synchronization:** Extremely challenging. Requires application-level active/active design (e.g., sharded DBs, conflict-free replicated data types - CRDTs). Often not feasible for all state.
            *   **Service Mesh:** Tools like Istio with multi-cluster support can route traffic, but state sync remains the hurdle.
        *   **RTO/RPO:** RTO: Seconds-minutes. RPO: Near-zero (if state sync possible). **Very complex, niche use cases.**
*   **Critical DR Components:**
    *   **Backup Everything:**
        *   etcd snapshots (offsite, encrypted, versioned)
        *   Kubernetes manifests (Git repo - **source of truth**)
        *   Cluster configuration (kubeadm config, CNI configs, cloud provider configs)
        *   Application data (via storage replication or app-native backup)
    *   **Documented Runbooks:** Step-by-step recovery procedures for each scenario. **Tested regularly!**
    *   **Regular DR Testing:** Schedule "fire drills". Restore etcd, failover to secondary cluster. **Untested DR is not DR.**
    *   **RTO/RPO Definition:** Know your business requirements. DR strategy must align with them.
    *   **Communication Plan:** Who does what during a DR event? How are stakeholders updated?
*   **Why it Matters:** Protects against existential threats to the cluster. **etcd backup is the foundation;** without it, higher-tier DR is impossible. DR readiness separates production-grade clusters from test environments.

---
