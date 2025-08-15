### **Chapter 56: Production-Ready Kubernetes**
*The foundation for secure, manageable, and observable clusters.*

#### **1. Hardening the Cluster**
*Why:* Kubernetes defaults prioritize usability over security. Hardening mitigates attack surfaces.
*Key Actions:*
- **API Server:**
  - Restrict access via `--advertise-address` & firewall rules.
  - Enable RBAC (`--authorization-mode=RBAC`).
  - Disable anonymous access (`--anonymous-auth=false`).
  - Use TLS for all communications (etcd, kubelets).
  - Audit logging (`--audit-policy-file`, log to SIEM).
- **etcd:**
  - Run on dedicated nodes (isolated from workloads).
  - Encrypt data at rest (`--encryption-provider-config`).
  - Use client TLS certificates for API server communication.
  - Firewall: Only allow API server access.
- **kubelet:**
  - Enable read-only port *only if needed* (`--read-only-port=0` by default in v1.20+).
  - Restrict `--authorization-mode=Webhook` and `--authentication-token-webhook`.
  - Set `--streaming-connection-idle-timeout` (e.g., 30m).
  - Disable `--enable-debugging-handlers` in prod.
- **Network Policies:**
  - **Mandatory:** Default-deny all ingress/egress (`kubectl create -f default-deny.yaml`).
  - Explicitly allow required traffic (e.g., app → DB).
  - Tools: Calico, Cilium, or Kube-router for enforcement.
- **Pod Security:**
  - **Critical:** Use **Pod Security Admission (PSA)** (v1.23+). *Replace deprecated PodSecurityPolicy.*
    - `enforce` level: `restricted` (no root, no hostPath, etc.).
    - `audit` level: `baseline` for warnings.
    - Label namespaces: `pod-security.kubernetes.io/enforce: restricted`
  - **OR** use OPA/Gatekeeper for custom policies (e.g., "no privileged pods").
- **Control Plane Nodes:**
  - Isolate physically/logically from worker nodes.
  - Harden OS (CIS benchmarks), disable swap, use non-root kubelet.
- **Secrets:**
  - **Never** store in Git/plain YAML. Use:
    - External Secrets Operator (ESO) + Vault/Secrets Manager
    - Sealed Secrets (for GitOps)
  - Enable encryption at rest for Secrets (`--encryption-provider-config`).

*Best Practice:* Run **kube-bench** (CIS Kubernetes Benchmark) regularly. Automate checks with **kube-score**.

---

#### **2. Backup and Restore (Velero)**
*Why:* Cluster/node failures, accidental `kubectl delete`, or ransomware require reliable backups.
*How Velero Works:*
1. **Backup:** 
   - Takes cluster *state* (resources) via Kubernetes API.
   - Takes *volume snapshots* via cloud provider (AWS EBS, Azure Disk, GCP PD) or CSI driver.
   - Stores metadata in object storage (S3, GCS, MinIO).
2. **Restore:** 
   - Replays API resources from backup.
   - Restores volumes from snapshots.

*Key Commands:*
```bash
# Backup all namespaces (excluding kube-system)
velero backup create full-backup --include-cluster-resources --default-volumes-to-restic

# Backup with schedule (daily at 2am)
velero schedule create daily-backup --schedule="0 2 * * *" --include-cluster-resources

# Restore from backup
velero restore create --from-backup full-backup-20231005120000
```

*Critical Best Practices:*
- **Test Restores Monthly:** Validate backups *work* (common failure point!).
- **Exclude Transient Resources:** Skip `events`, `customresourcedefinitions` (handled by Velero automatically).
- **Volume Snapshots:** Essential for stateful apps (DBs). Use `--default-volumes-to-restic` for non-cloud volumes.
- **Object Storage:** Use versioning + lifecycle policies (e.g., S3). Store backups *off-cluster*.
- **Encryption:** Enable server-side encryption (SSE) for object storage buckets.
- **RBAC:** Run Velero with least-privilege ServiceAccount.

*Why Velero > `etcd` snapshots alone:* 
- `etcd` snapshots only restore *one cluster* to *exact point-in-time*. 
- Velero enables **cross-cluster migration**, selective restores, and handles cloud volumes.

---

#### **3. Cost Monitoring (Goldilocks, Kubecost)**
*Why:* Unchecked resource requests lead to 50-70% wasted cloud spend.

**A. Goldilocks**
- **Purpose:** Visualize **over-provisioned** CPU/memory requests.
- **How:** 
  1. Deploys VPA (Vertical Pod Autoscaler) in *recommendation mode*.
  2. Analyzes actual usage vs. requests.
  3. Shows dashboard: "Your pod requests 1000m CPU but uses 200m → reduce to 300m".
- **Key Output:** 
  ```yaml
  recommendations:
    containerRecommendations:
    - containerName: nginx
      lowerBound: {cpu: 100m, memory: 50Mi}   # Minimum safe
      target: {cpu: 200m, memory: 100Mi}       # Optimal
      uncappedTarget: {cpu: 250m, memory: 120Mi} # Max observed
      upperBound: {cpu: 500m, memory: 200Mi}   # Current request
  ```
- **Best Practice:** 
  - Run in *dry-run* mode first. 
  - **Never** use VPA in *auto* mode with HPA (conflicts). 
  - Target = 70-80% of *upperBound*.

**B. Kubecost**
- **Purpose:** Full cost allocation, optimization, and forecasting.
- **How:**
  1. Scrapes metrics (CPU, RAM, network, PVs) from Prometheus/cadvisor.
  2. Applies cloud pricing (AWS/Azure/GCP APIs or custom rates).
  3. Allocates costs to namespaces, deployments, labels.
- **Key Features:**
  - **Cost Allocation:** "Team X spent $1,200 this month on namespace `prod`."
  - **Savings Recommendations:**
    - "Reduce `app-frontend` CPU request by 40% → save $220/mo."
    - "Use Spot Instances for `batch-jobs` → save 70%."
  - **Idle Resource Detection:** "Namespace `dev` has 12 idle pods → save $800/mo."
  - **Multi-Cluster Views:** Aggregate costs across clusters.
- **Best Practice:** 
  - Integrate with CI/CD: Block PRs if cost increase >5%. 
  - Set budgets with alerts (e.g., "Alert if `prod` exceeds $5k/week").

*Pro Tip:* Combine both: Goldilocks for *technical* right-sizing, Kubecost for *financial* impact.

---

#### **4. Naming Conventions and Labels**
*Why:* Chaos without standards → impossible to manage at scale.

**A. Naming Conventions (DNS-compliant: `a-z0-9-`):**
| Resource          | Convention Example                     | Why                                      |
|-------------------|----------------------------------------|------------------------------------------|
| **Namespace**     | `prod`, `staging`, `team-frontend`     | Clear environment/team ownership         |
| **Deployment**    | `api-prod`, `frontend-staging`         | App + env (avoid `app-v1` → mutable)     |
| **Service**       | `api-internal`, `frontend-external`    | Purpose (internal/external)              |
| **StatefulSet**   | `postgres-cluster`, `es-data`          | Cluster role (avoid `db`)                |
| **ConfigMap**     | `app-config-prod`, `nginx-ingress`     | App + env or component                   |

**B. Mandatory Labels (for operations & cost tracking):**
```yaml
metadata:
  labels:
    app.kubernetes.io/name: "frontend"          # [REQUIRED] App name
    app.kubernetes.io/instance: "frontend-prod" # [REQUIRED] Unique instance
    app.kubernetes.io/version: "1.5.0"          # App version
    app.kubernetes.io/environment: "prod"       # env (prod/staging/dev)
    team: "sre"                               # Owning team
    cost-center: "cc-12345"                   # For cost allocation (Kubecost)
```
*Why Labels Matter:*
- `kubectl get pods -l app.kubernetes.io/environment=prod`
- Kubecost: `cost-center` tags for billing.
- Network Policies: `podSelector: matchLabels: app: mysql`
- **Golden Rule:** Labels are *queryable metadata*. Names are *unique identifiers*.

---

#### **5. Namespace Management**
*Why:* Isolation, security, quotas, and billing boundaries.

**Best Practices:**
1. **Structure by:**
   - **Environment:** `prod`, `staging`, `dev` (most common)
   - **Team:** `team-a`, `team-b` (for multi-tenant clusters)
   - **Criticality:** `critical`, `standard` (for priority scheduling)
2. **Resource Quotas (Mandatory):**
   ```yaml
   apiVersion: v1
   kind: ResourceQuota
   metadata:
     name: prod-quota
   spec:
     hard:
       requests.cpu: "20"
       requests.memory: 50Gi
       limits.cpu: "40"
       limits.memory: 100Gi
       persistentvolumeclaims: "10"
   ```
   - Prevents one namespace from starving others.
   - Set via `LimitRange` for default requests/limits.
3. **Network Policies:** Default-deny per namespace.
4. **RBAC:** Bind roles *at namespace level* (e.g., `dev-team` can only access `dev` NS).
5. **Lifecycle:** 
   - Auto-delete stale namespaces (e.g., `ci-*` after 7 days) with tools like **Namespace Auto-Deleter**.
   - Tag namespaces with `owner: jane@company.com`, `expiry: 2024-01-01`.

*Critical:* **Never** run production workloads in `default` namespace.

---

### **Chapter 57: Disaster Recovery & Backup**
*Beyond backups: ensuring business continuity.*

#### **1. Velero: Backup, Restore, Migration**
*Deep Dive from Ch56:*
- **Disaster Scenarios Velero Solves:**
  | Scenario                | Velero Solution                     |
  |-------------------------|-------------------------------------|
  | Accidental `kubectl delete -f .` | Restore specific resources/namespaces |
  | Cluster destroyed (AZ outage) | Restore to new cluster in another region |
  | Data corruption (e.g., DB) | Restore volumes + app state to pre-corruption point |
  | Migration to new cluster | `velero backup` old cluster → `velero restore` to new |

- **Advanced Features:**
  - **Hooks:** Run pre/post-restore commands (e.g., `pre-restore: pg_dump`).
  - **Selective Restore:** `--include-resources=deployments,services`
  - **Backup Location:** Multi-cloud (S3 + GCS) for redundancy.
  - **Backup TTL:** `--ttl 168h0m0s` (auto-delete after 7 days).

*DR Drill Checklist:*
1. Simulate region outage → spin up cluster in new region.
2. Restore Velero backups to new cluster.
3. Validate DNS failover, data consistency, and app functionality.
4. **Time Target:** RTO (Recovery Time Objective) < 1 hour, RPO (Recovery Point Objective) < 15 mins.

---

#### **2. Snapshotting Volumes**
*Why:* Velero's API backup *doesn't* capture live DB data. Volume snapshots do.

**How it Works:**
1. Velero triggers cloud provider's snapshot API (e.g., AWS EBS Snapshot).
2. For non-cloud volumes (local storage, NFS):
   - Use **Restic** (Velero's built-in backup tool): 
     ```bash
     velero backup create app-backup --default-volumes-to-restic
     ```
   - Restic does *file-level* backup (slower but universal).

**Critical Considerations:**
- **Consistency:** 
  - For DBs: Use pre-hooks to `FLUSH TABLES WITH READ LOCK` (MySQL) or `pg_start_backup()` (Postgres).
  - For stateful apps: Pause writes during snapshot (via Velero hooks).
- **Performance Impact:** 
  - Cloud snapshots are near-instant (copy-on-write). 
  - Restic = high I/O (run during off-peak).
- **Cost:** 
  - Snapshots cost 20-30% of volume price (AWS). 
  - Delete old snapshots aggressively (`velero backup delete old-backup --delete-namespace-snapshots`).

---

#### **3. Cross-Cluster Recovery**
*The ultimate DR test: restoring to a *different* cluster.*

**Steps:**
1. **Backup Source Cluster:**
   ```bash
   velero backup create cluster-backup --include-cluster-resources
   ```
2. **Prepare Target Cluster:**
   - Install Velero with same cloud provider plugin.
   - Configure identical backup storage location (S3 bucket).
3. **Restore:**
   ```bash
   velero restore create --from-backup cluster-backup --namespace-mappings old-ns:new-ns
   ```
   - `--namespace-mappings` renames namespaces (e.g., `prod` → `prod-dr`).

**Pitfalls to Avoid:**
- **Ingress/Load Balancers:** IPs/DNS will differ. Use `--restore-volumes=false` + recreate LBs manually.
- **Secrets:** Restore secrets *after* other resources (dependencies).
- **StatefulSets:** Ensure storage class names match in target cluster.
- **Cluster-Specific Resources:** Exclude `nodes`, `events`, `clusterroles` (use `--exclude-resources`).

*Pro Strategy:* Run **active-passive DR** with scheduled backups. For **active-active**, use tools like **Kasten K10** for app-consistent replication.

---

### **Chapter 58: Performance Tuning**
*Optimizing the Kubernetes control plane and worker nodes.*

#### **1. Optimizing etcd**
*Why:* etcd is the cluster's "brain." Slow etcd = slow everything.

**Tuning Parameters (`/etc/kubernetes/manifests/etcd.yaml`):**
```yaml
spec:
  containers:
  - command:
    - etcd
    - --max-request-bytes=1572864        # Increase from 1.5MB (for large Secrets)
    - --quota-backend-bytes=8589934592   # 8GB (default 2GB - avoid "DB space exceeded")
    - --heartbeat-interval=250           # ms (default 100ms - reduce traffic)
    - --election-timeout=1250            # ms (default 1000ms - faster leader election)
    - --max-snapshots=5                  # Retain snapshots
    - --max-wals=5                       # Retain WAL files
```

**Critical Practices:**
- **Dedicated Disks:** Use SSD/NVMe (latency < 1ms). **Never** share disk with workloads.
- **Size Correctly:** 
  - > 10k pods? → 8+ vCPUs, 32GB RAM per etcd node.
  - Monitor `etcd_disk_wal_fsync_duration_seconds` (P99 < 10ms).
- **Backup etcd:** `ETCDCTL_API=3 etcdctl --cacert=/etc/kubernetes/pki/etcd/ca.crt snapshot save snapshot.db`
- **Compaction:** Run hourly to reclaim space:
  ```bash
  REV=$(etcdctl endpoint status --write-out="json" | egrep -o '"revision":[0-9]*' | egrep -o '[0-9]*')
  etcdctl compact $REV
  ```

*Warning:* Misconfigured etcd can cause **split-brain**. Always run odd-numbered nodes (3,5,7).

---

#### **2. Kernel Tuning**
*Why:* Default Linux settings aren't optimized for high-density containers.

**Key `/etc/sysctl.d/99-k8s.conf` Settings:**
```conf
# Networking
net.core.somaxconn = 10000             # Max incoming connections
net.core.netdev_max_backlog = 5000     # Packet queue depth
net.ipv4.tcp_max_syn_backlog = 8192    # SYN queue size
net.ipv4.tcp_tw_reuse = 1              # Reuse TIME_WAIT sockets

# Memory
vm.swappiness = 10                     # Avoid swapping (0=never, 100=aggressive)
vm.overcommit_memory = 1               # Allow overcommit (needed for JVM apps)

# File Descriptors
fs.file-max = 2097152                  # System-wide max open files
fs.inotify.max_user_instances = 8192   # Inotify watches (for file watchers)
```
*Apply:* `sysctl -p /etc/sysctl.d/99-k8s.conf`

**Worker Node Tuning:**
- **HugePages:** For latency-sensitive apps (DBs):
  ```yaml
  spec:
    hugePages:
      - pageSize: "2Mi"
        memory: "1Gi"
  ```
- **CPU Manager:** `static` policy for guaranteed pods (avoid noisy neighbors):
  ```yaml
  apiVersion: kubelet.config.k8s.io/v1beta1
  kind: KubeletConfiguration
  cpuManagerPolicy: static
  ```
- **Disable Transparent HugePages (THP):** 
  ```bash
  echo never > /sys/kernel/mm/transparent_hugepage/enabled
  ```

---

#### **3. Network Optimization**
*Why:* Network latency = slow apps. Packet loss = failed connections.

**Critical Tunings:**
- **CNI Choice:** 
  - **Calico (BGP mode):** Lowest latency (no overlay), but needs BGP support.
  - **Cilium (eBPF):** Advanced features (L7 filtering), lower CPU than iptables.
  - **Avoid:** Flannel (slow iptables), Weave (high CPU).
- **Kernel Parameters:**
  ```conf
  net.ipv4.neigh.default.gc_thresh1 = 8192  # ARP cache min
  net.ipv4.neigh.default.gc_thresh2 = 32768 # ARP cache pressure point
  net.ipv4.neigh.default.gc_thresh3 = 65536 # ARP cache max
  net.ipv4.tcp_slow_start_after_idle = 0    # Disable slow start (for persistent connections)
  ```
- **MTU Optimization:** 
  - Set CNI MTU to 1450 (for VXLAN) or 1500 (BGP mode) to avoid fragmentation.
  - Test with `ping -s 1472 <pod-ip>` (1472 + 20 IP + 8 ICMP = 1500).
- **Service Mesh:** Avoid unless needed (Istio adds ~10ms latency). Use **Linkerd** for lower overhead.

**Diagnose:**
```bash
# Packet loss
ping <service-ip>

# Latency
kubectl run -it --rm debug --image=alpine --restart=Never -- ping <service-ip>

# TCP retransmits (bad!)
netstat -s | grep retrans
```

---

#### **4. Reducing API Latency**
*Why:* Slow API server → slow `kubectl`, deployments, autoscaling.

**Tuning Strategies:**
- **API Server Flags:**
  ```yaml
  # /etc/kubernetes/manifests/kube-apiserver.yaml
  - --max-mutating-requests-inflight=1000   # Default 200 (write-heavy clusters)
  - --max-requests-inflight=2000            # Default 400 (read-heavy clusters)
  - --watch-cache-sizes="*#2000"            # Increase watch cache (default 100)
  - --profiling=false                      # Disable profiling (security)
  ```
- **Etcd Proxy:** Run `etcd` on control plane nodes (reduces network hops).
- **Shard etcd (Advanced):** For >5k nodes, use [etcd sharding](https://kubernetes.io/docs/tasks/administer-cluster/configure-multiple-schedulers/).
- **Monitor:**
  - `apiserver_request_duration_seconds` (P99 < 1s)
  - `etcd_request_duration_seconds` (P99 < 100ms)
  - `workqueue_depth` (should be near 0)

*Critical:* **Never** expose API server publicly. Use private load balancer + firewall.

---

### **Chapter 59: Cost Optimization**
*Reducing cloud spend without sacrificing performance.*

#### **1. Right-Sizing Resources**
*Why:* Over-provisioning = wasted money. Under-provisioning = OOM kills.

**Process:**
1. **Collect Data:** 
   - Use Prometheus + [metrics-server](https://github.com/kubernetes-sigs/metrics-server) for CPU/memory usage.
   - Run for 2+ weeks (capture peak loads).
2. **Analyze:**
   - **Goldilocks** (Ch56) for quick recommendations.
   - **Kubecost** for cost impact: "This pod wastes $120/mo".
3. **Adjust:**
   - **CPU:** Target 50-70% avg utilization (burst to 100%).
   - **Memory:** Target 70-80% of request (avoid OOM).
   - **Formula:** 
     ```
     New Request = (Avg Usage * 1.5) + (Std Dev * 2)
     ```
4. **Validate:** 
   - Monitor `container_memory_working_set_bytes` (must < request).
   - Check OOM events: `kubectl get events -A | grep OOM`

*Pro Tip:* Start with non-critical workloads (batch jobs, dev envs).

---

#### **2. Spot Instances**
*Why:* 60-90% discount vs. on-demand. Ideal for fault-tolerant workloads.

**How to Implement:**
1. **Identify Suitable Workloads:**
   - Stateless apps (web servers, workers)
   - Batch jobs (CI/CD, data processing)
   - **Avoid:** Stateful apps (DBs), control plane, critical APIs.
2. **Cluster Setup:**
   - Use **Karpenter** (simpler) or **Cluster Autoscaler** with mixed instance policies.
   - Example Karpenter provisioner:
     ```yaml
     apiVersion: karpenter.sh/v1alpha5
     kind: Provisioner
     spec:
       requirements:
         - key: karpenter.sh/capacity-type
           operator: In
           values: ["spot", "on-demand"]
       limits:
         resources:
           cpu: 1000
     ```
3. **Handle Interruptions:**
   - **Spot Termination Handler:** 
     - Detects AWS `spot-itn` or Azure `Scheduled Events`.
     - Drains node gracefully (5-10 mins before termination).
   - **Pod Disruption Budgets (PDBs):** Ensure minimum availability during drain.
   - **Multi-AZ/Region:** Distribute spot instances across pools.

**Savings:** 70% typical for stateless workloads. **Risk:** 5-10% interruption rate (manageable with tooling).

---

#### **3. Autoscaling Strategies**
*Scale dynamically to match demand.*

**A. Horizontal Pod Autoscaler (HPA)**
- **Metrics:** CPU/memory (basic), custom metrics (Prometheus), external (SQS queue depth).
- **Best Practices:**
  - **Target Utilization:** 50-60% for CPU (avoids burst spikes).
  - **Stabilization Window:** `--horizontal-pod-autoscaler-downscale-stabilization=5m` (prevent flapping).
  - **Example (Custom Metric):**
    ```yaml
    metrics:
    - type: Pods
      pods:
        metric:
          name: http_requests_per_second
        target:
          type: AverageValue
          averageValue: 100
    ```

**B. Cluster Autoscaler (CA)**
- **How:** Adds/removes nodes based on pending pods.
- **Critical Settings:**
  - `--scale-down-unneeded-time=10m` (default 10m - reduce for cost savings)
  - `--expander=least-waste` (choose node type that minimizes unused resources)
  - **Scale-down disabled** for critical namespaces (e.g., `kube-system`).

**C. Vertical Pod Autoscaler (VPA)**
- **Use Case:** Right-size *individual pods* (complements HPA).
- **Limitations:** 
  - Requires pod restart (use with rolling updates).
  - **Conflict:** Never enable VPA *and* HPA on same metric (CPU).
- **Strategy:** 
  - Use VPA in *recommendation mode* → manually adjust requests → use HPA for scaling.

**Advanced:** **Keda** for event-driven scaling (Kafka, RabbitMQ, etc.).

---

#### **4. Monitoring with Kubecost**
*Why:* Cost visibility is the first step to optimization.

**Key Workflows:**
1. **Identify Waste:**
   - **Idle Resources:** "Namespace `dev` has 12 idle pods → save $800/mo."
   - **Over-Provisioning:** "Deployment `api` requests 2x needed CPU."
2. **Right-Sizing:**
   - Kubecost shows "Optimal Request" vs "Current Request".
   - Integrate with Goldilocks for technical validation.
3. **Spot Instance Savings:**
   - Track "On-Demand vs. Spot" spend per namespace.
   - "Using Spot for `batch-jobs` saved $1,200 this month."
4. **Budgets & Alerts:**
   - Set per-namespace budgets: "Alert if `prod` exceeds $5k/week".
   - Slack/email alerts for cost anomalies.
5. **Multi-Cluster Views:**
   - Compare cost per cluster (e.g., "Cluster-A costs 40% more than Cluster-B").

**Pro Tactics:**
- **Chargeback:** Export cost data to finance via CSV/API.
- **CI/CD Integration:** Fail PRs if cost increase >5%.
- **Forecasting:** Predict next month's spend based on trends.

---

### **Critical Cross-Chapter Insights**
1. **Backup ≠ DR:** 
   - Backup = "Can I restore data?" 
   - DR = "Can I restore *and run* the app in < RTO?" 
   - **Test DR quarterly** – backups are useless if restore fails.
2. **Cost vs. Performance Tradeoffs:**
   - Spot instances save money but increase complexity.
   - Right-sizing saves cost but requires monitoring (avoid OOM kills).
   - **Rule:** Optimize non-critical workloads first (dev/staging).
3. **Labels are Your Lifeline:**
   - Without `app.kubernetes.io/*` labels, cost allocation, monitoring, and network policies fail.
4. **Security is Non-Negotiable:**
   - Hardening (Ch56) prevents breaches that could destroy your DR plan.
5. **Monitor Everything:**
   - Cost (Kubecost), performance (Prometheus), backups (Velero logs). 
   - **Golden Signals:** Latency, Traffic, Errors, Saturation (for cost/performance).

---
