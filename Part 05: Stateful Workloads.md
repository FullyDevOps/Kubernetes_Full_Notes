## 🔒 Chapter 16: StatefulSets – Managing Stateful Applications

### ❓ **When to Use StatefulSets (The Critical "Why")**
StatefulSets solve problems **Deployments cannot**:
- **Stateful Applications**: Databases (MySQL, PostgreSQL), message brokers (Kafka, RabbitMQ), consensus systems (ZooKeeper, etcd).
- **Identity Matters**: Pods need stable, unique network identifiers (DNS names) and persistent storage that **survives pod recreation**.
- **Order Matters**: Applications requiring **strict startup/shutdown sequences** (e.g., primary must start before replicas).
- **No Stateless Assumptions**: Deployments assume pods are interchangeable. StatefulSets acknowledge pods have **identity and dependencies**.

> ⚠️ **Never use a Deployment for stateful apps!**  
> *Example:* Scaling a MySQL replica set with a Deployment could cause data corruption because new pods get random names/IPs and mount the *wrong* persistent volume.

---

### ⚙️ **Core Mechanics of StatefulSets**

#### 1. **Stable Network Identifiers (Pod DNS)**
- **Headless Service Requirement**: StatefulSets **require** a headless Service (`clusterIP: None`). This prevents load balancing and enables direct pod addressing.
- **DNS Naming Pattern**:  
  `<pod-name>.<service-name>.<namespace>.svc.cluster.local`  
  *Example:* `mysql-0.mysql.default.svc.cluster.local`
- **Predictable Hostnames**:  
  Pods are named `<statefulset-name>-<ordinal-index>` (e.g., `kafka-0`, `kafka-1`).  
  **Ordinal index starts at 0** and is **fixed for the pod's lifetime**.
- **DNS Resolution**:  
  The headless Service creates:
  - A DNS A record for each pod: `kafka-0.kafka.default.svc` → Pod IP
  - A DNS SRV record listing all pods: `_port._tcp.kafka.default.svc` → `kafka-0:9092, kafka-1:9092, ...`

#### 2. **Stable Persistent Storage**
- **Volume Claim Templates**: StatefulSets define a `volumeClaimTemplate` in their spec.  
  ```yaml
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: [ "ReadWriteOnce" ]
        storageClassName: "fast"
        resources:
          requests:
            storage: 10Gi
  ```
- **How It Works**:
  1. When `kafka-0` is created, Kubernetes auto-creates a PVC named `data-kafka-0`.
  2. This PVC binds to a PV (dynamically provisioned or pre-existing).
  3. If `kafka-0` is **deleted**, its PVC **remains intact**. When recreated, it reattaches to the *same* PVC/PV.
  4. **Scaling**: `kafka-1` gets `data-kafka-1` – **each pod has dedicated storage**.
- **Critical Behavior**:  
  - PVCs **are NOT deleted** when the StatefulSet is scaled down (only when the StatefulSet is *deleted*).  
  - `volumeClaimTemplate` changes **only affect new pods** (existing pods retain original storage).

#### 3. **Ordered Deployment & Scaling**
- **Rolling Out**:  
  Pods are created **sequentially** (0 → 1 → 2). `kafka-1` starts **only after** `kafka-0` is `Ready`.
- **Scaling Up**:  
  New pods created **one at a time** in ordinal order (`kafka-2` before `kafka-3`).
- **Scaling Down**:  
  Pods terminated **in reverse order** (`kafka-3` → `kafka-2` → `kafka-1`).  
  **Crucially**: Kubernetes waits for a pod to be **fully terminated** before deleting the next.
- **Rolling Updates**:  
  - `strategy.type: RollingUpdate` (default): Updates pods **in reverse ordinal order** (highest index first).  
    *Why?* To avoid losing quorum (e.g., in a 3-node ZooKeeper, update node 2 first, then 1, then 0).  
  - `partition` field: Allows partial updates (e.g., update only pods with index ≥ 2).

#### 4. **Ordered Termination**
- During deletion/scale-down, pods terminate **in reverse ordinal order** (N → N-1 → ... → 0).  
  *Example:* `kafka-2` must fully terminate before `kafka-1` is deleted.  
  **Why?** To maintain application consistency (e.g., Kafka brokers gracefully hand off leadership).

---

### 🌐 **Key Use Cases & Real-World Examples**

| Application | Why StatefulSet? | Critical StatefulSet Feature |
|-------------|------------------|------------------------------|
| **MySQL (Primary-Replica)** | Replicas need stable DNS to connect to primary | Stable DNS (`mysql-0` = primary), Ordered startup (primary first) |
| **MongoDB Replica Set** | Members must know each other's stable addresses | DNS SRV records for automatic discovery |
| **Kafka** | Brokers have fixed IDs tied to pod index | `kafka-0` = broker 0, PVC retains logs |
| **ZooKeeper** | Quorum requires ordered startup/shutdown | Sequential pod creation, stable storage for snapshots |
| **etcd (K8s control plane)** | Strict leader election & data consistency | Ordered operations, persistent storage |

> 💡 **Kafka Deep Dive**:  
> - Broker ID = Pod ordinal index (set via `KAFKA_BROKER_ID=$(hostname | awk -F'-' '{print $NF}')`).  
> - If `kafka-1` dies and restarts, it **reuses broker ID 1** because DNS/storage is stable.  
> - Scaling requires updating `KAFKA_ADVERTISED_LISTENERS` – StatefulSets make this predictable.

---

### ⚠️ **Critical Pitfalls & Best Practices**
- **Never delete PVCs manually** during scaling – StatefulSets rely on them for identity.
- **Always use a headless Service** – Without it, pods can't resolve each other's DNS.
- **Set resource limits** – Stateful apps (like DBs) are resource hogs; prevent node starvation.
- **Use PodDisruptionBudgets (PDBs)** – Prevent concurrent pod evictions during maintenance:  
  ```yaml
  minAvailable: 2  # For a 3-node DB cluster
  selector:
    matchLabels:
      app: mysql
  ```
- **Backup PVCs** – Use tools like Velero; StatefulSets don't manage data backups.

---

## 🦶 Chapter 17: DaemonSets – Run Pods on Every Node

### 📌 **Core Concept**
DaemonSets ensure **exactly one pod runs on every node** (or a subset of nodes). Ideal for **node-level infrastructure**.

---

### ⚙️ **How DaemonSets Work**
1. **Node Assignment**:  
   - When a **new node joins** the cluster, the DaemonSet controller **immediately** creates a pod on it.  
   - When a **node is deleted**, its DaemonSet pod is **cleaned up**.
2. **No Scaling**: You don’t specify `replicas`. The number of pods = number of eligible nodes.
3. **Update Strategy**:  
   - `RollingUpdate` (default): Replaces pods one node at a time.  
   - `OnDelete`: Old pods stay until manually deleted (rarely used).

---

### 🎯 **Key Use Cases**
| Use Case | Example Tools | Why DaemonSet? |
|----------|---------------|----------------|
| **Logging Agents** | Fluentd, Filebeat, Logstash | Need to tail logs from *every* node's `/var/log` |
| **Node Monitoring** | Prometheus Node Exporter, Datadog Agent | Collect CPU/memory/disk metrics per-node |
| **Network Plugins** | Calico, Flannel, Cilium | Must run on every node to handle pod networking |
| **Storage Daemons** | Ceph OSD, Portworx | Manage storage directly on the host |
| **Security Agents** | Falco, Sysdig | Monitor kernel events on the host |

---

### 🔍 **Node Selection: Affinity & Tolerations**
DaemonSets target **specific nodes** using:

#### 1. **Node Affinity**
```yaml
spec:
  template:
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: node-role.kubernetes.io/worker
                operator: Exists
```
- **Example**: Run only on worker nodes (not control plane).

#### 2. **Tolerations**
```yaml
spec:
  template:
    spec:
      tolerations:
      - key: node-role.kubernetes.io/control-plane
        operator: Exists
        effect: NoSchedule
```
- **Example**: Run logging agents on control plane nodes (which normally block workloads).

#### 3. **nodeSelector** (Simpler alternative)
```yaml
spec:
  template:
    spec:
      nodeSelector:
        disktype: ssd  # Only run on nodes with SSDs
```

---

### ⚠️ **Critical Considerations**
- **Resource Pressure**: DaemonSet pods run on **every node**. If they leak memory, *all* nodes suffer.
- **Control Plane Access**: To run on master nodes:  
  - Tolerate `node-role.kubernetes.io/master:NoSchedule`  
  - Set `nodeAffinity` for control-plane nodes.
- **Startup Order**: DaemonSets start **before** Deployments/StatefulSets. Critical for network plugins (pods can't start without CNI).
- **Pod Overhead**: Each node runs the pod – impacts allocatable resources. Account for this in node sizing.
- **Update Safety**: Use `updateStrategy.rollingUpdate.maxUnavailable` (e.g., `10%`) to avoid cluster-wide outages.

---

## ⏱️ Chapter 18: Jobs & CronJobs – Running Finite Workloads

### 🎯 **Jobs: One-off Tasks**
For tasks that **must complete successfully** (not long-running services).

#### **Core Concepts**
- **`spec.template`**: Defines the pod (like a Deployment).
- **`spec.completions`**: Total successful pods needed to mark the Job "complete".  
  *Example:* `completions: 5` → Run 5 pods to completion.
- **`spec.parallelism`**: Max pods running *concurrently*.  
  *Example:* `parallelism: 2` → Only 2 pods run at once (even if `completions=5`).
- **`spec.backoffLimit`**: Retries before marking Job as `Failed` (default: 6).
- **`spec.activeDeadlineSeconds`**: Max runtime (e.g., kill if not done in 24h).

#### **Job Patterns & Execution Flow**
| Pattern | `completions` | `parallelism` | Use Case |
|---------|---------------|---------------|----------|
| **Single Task** | `1` (default) | `1` (default) | DB migration, image processing |
| **Fixed Count** | `5` | `1` | Process 5 files sequentially |
| **Parallel Processing** | `50` | `10` | Process 50 files with 10 workers |
| **Work Queue** | `1` | `N` | Workers pull tasks from RabbitMQ |

> 💡 **Work Queue Pattern**:  
> - Set `completions: 1` + `parallelism: 10` → 10 pods start *simultaneously*.  
> - Each pod pulls tasks from a queue (e.g., RabbitMQ) until the queue is empty.  
> - **No coordination needed** – Kubernetes handles pod lifecycle.

#### **Critical Behaviors**
- Pods are **not replaced** if they succeed (only failed pods are retried).
- Job status: `Active` (pods running), `Succeeded` (all completions met), `Failed` (backoff exceeded).
- **Never use `RestartPolicy: Always`** – Jobs **must** use `OnFailure` or `Never`.

---

### 🕰️ **CronJobs: Scheduled Jobs**
Like `cron` in Linux, but cluster-managed.

#### **Key Fields**
```yaml
spec:
  schedule: "0 0 * * *"  # Daily at midnight UTC
  startingDeadlineSeconds: 300  # Fail if missed by 5+ mins
  concurrencyPolicy: Forbid  # [Allow, Forbid, Replace]
  suspend: false  # Pause execution
  jobTemplate:  # Standard Job spec
    spec:
      template:
        spec:
          containers: [...]
      backoffLimit: 2
```

#### **Concurrency Policies**
| Policy | Behavior |
|--------|----------|
| **`Allow`** (default) | Run new Job even if previous is still running |
| **`Forbid`** | Skip new run if previous Job is active |
| **`Replace`** | Terminate previous Job and start new one |

> ⚠️ **Timezone Gotcha**:  
> Kubernetes **uses UTC by default**! To use local time:  
> ```yaml
> env:
>   - name: TZ
>     value: "America/New_York"
> ```

#### **Real-World Use Cases**
- **Nightly backups** (DB dump to S3)
- **Log rotation** (Compress/delete old logs)
- **ML model retraining** (Daily at 2 AM)
- **Report generation** (Weekly sales report)

---

### 🚨 **Jobs/CronJobs Pitfalls & Best Practices**
1. **Avoid `Never` for `RestartPolicy`**:  
   If a pod fails, it won’t restart. Use `OnFailure` to retry within the same pod.
2. **Set `backoffLimit`**: Prevent infinite retries on broken tasks.
3. **Use `activeDeadlineSeconds`**: Stop runaway jobs (e.g., `86400` for daily jobs).
4. **CronJob Timezones**: Always specify `TZ` env var – UTC causes missed runs.
5. **Clean Up Old Jobs**:  
   ```yaml
   spec:
     ttlSecondsAfterFinished: 86400  # Auto-delete Jobs 24h after completion
   ```
6. **Idempotency**: Jobs must handle being run multiple times (e.g., backups should overwrite safely).
7. **Resource Requests**: Critical for batch jobs – prevent node starvation during parallel runs.

---

## 🔑 **Key Differences Summary Table**

| Feature | StatefulSet | DaemonSet | Job | CronJob |
|---------|-------------|-----------|-----|---------|
| **Purpose** | Stateful apps (DBs, brokers) | Node-level agents | One-off task | Scheduled task |
| **Pod Identity** | Stable DNS/storage | None (ephemeral) | None | None |
| **Scaling** | Ordered (0→N) | 1 per node | `completions`/`parallelism` | N/A (scheduled) |
| **Storage** | PVC per pod | HostPath/emptyDir | Ephemeral | Ephemeral |
| **Networking** | Headless Service required | ClusterIP/None | None | None |
| **Termination** | Graceful (reverse order) | Node deletion | On success/failure | On success/failure |
| **Update Strategy** | RollingUpdate (ordered) | RollingUpdate | N/A | N/A |

---
