### **Chapter 22: Scheduling Basics**

#### **1. How the Scheduler Works**
*   **Core Function**: Assigns pending Pods to available Nodes.
*   **Decision Process (2-Phase)**:
    1.  **Filtering (Predicates)**:
        *   Eliminates Nodes that *cannot* run the Pod.
        *   Checks: Resource availability (CPU/memory), Node selectors, Taints/Tolerations, Pod/Node Affinity rules, Volume constraints, Host ports.
        *   *Example*: A Pod needing 2GB RAM won't be scheduled on a Node with only 1GB free.
    2.  **Scoring (Priorities)**:
        *   Ranks *feasible* Nodes from Filtering phase.
        *   Uses priority functions (e.g., `LeastRequestedPriority`, `BalancedResourceAllocation`, `NodeAffinityPriority`).
        *   *Example*: `LeastRequestedPriority` favors Nodes with lower resource utilization.
*   **Binding**: Once a Node is chosen, the Scheduler sends a `Binding` object to the API Server, instructing the kubelet on that Node to start the Pod.
*   **Key Nuance**: Scheduler is **stateless**. It reacts to events (new Pod, Node status change) but doesn't track cluster state itself. Relies on API Server.

#### **2. Default Scheduling Behavior**
*   **Resource-Based**: Primarily uses CPU/Memory **requests** (not limits) for scheduling decisions.
*   **Bin Packing**: Favors Nodes with the *least available resources* (to leave room for larger Pods on other Nodes) – driven by `LeastRequestedPriority`.
*   **No Affinity/Taints**: Without explicit rules, Pods can land on *any* Node meeting resource requirements.
*   **Critical Implication**: Default behavior **ignores pod density** on a Node. Two CPU-heavy Pods *could* end up on the same Node, causing contention. *Always* define resource requests!

#### **3. Node Affinity & Pod Affinity/Anti-Affinity**
*   **Purpose**: Fine-grained control over *where* Pods run.
*   **Types**:
    *   **Node Affinity**: Schedule Pod based on **Node labels**.
        *   `requiredDuringSchedulingIgnoredDuringExecution`: Hard rule (like NodeSelector on steroids). *Pod won't schedule if no match*.
        *   `preferredDuringSchedulingIgnoredDuringExecution`: Soft rule (Scheduler tries but doesn't guarantee).
        *   *YAML Snippet*:
          ```yaml
          affinity:
            nodeAffinity:
              requiredDuringSchedulingIgnoredDuringExecution:
                nodeSelectorTerms:
                - matchExpressions:
                  - key: topology.kubernetes.io/zone
                    operator: In
                    values: [us-east-1a]
              preferredDuringSchedulingIgnoredDuringExecution:
              - weight: 80
                preference:
                  matchExpressions:
                  - key: disktype
                    operator: In
                    values: [ssd]
          ```
    *   **Pod Affinity/Anti-Affinity**: Schedule Pod based on **other Pods' labels** (co-location or separation).
        *   `podAffinity`: Attract Pods to Nodes running specific other Pods (e.g., frontend near cache).
        *   `podAntiAffinity`: Repel Pods from Nodes running specific other Pods (e.g., spread replicas).
        *   `requiredDuringScheduling...`: Hard rule (e.g., *must* be near a database Pod).
        *   `preferredDuringScheduling...`: Soft rule (e.g., *prefer* not to be with another replica).
        *   `topologyKey`: Defines the scope (e.g., `kubernetes.io/hostname` = same node, `topology.kubernetes.io/zone` = same zone).
        *   *Critical Use Case*: **High Availability (HA)**. Use `podAntiAffinity` with `topologyKey: kubernetes.io/hostname` to spread replicas across Nodes.
        *   *YAML Snippet (Anti-Affinity for HA)*:
          ```yaml
          affinity:
            podAntiAffinity:
              requiredDuringSchedulingIgnoredDuringExecution:
              - labelSelector:
                  matchExpressions:
                  - key: app
                    operator: In
                    values: [my-app]
                topologyKey: kubernetes.io/hostname
          ```

#### **4. Taints and Tolerations**
*   **Purpose**: **Restrict** which Pods can run on a Node (opposite of Affinity).
*   **Taint (Node Level)**: `key=value:effect`
    *   **Effects**:
        *   `NoSchedule`: New Pods without matching toleration *cannot* be scheduled.
        *   `PreferNoSchedule`: Scheduler *tries* to avoid, but not guaranteed.
        *   `NoExecute`: Evicts *existing* Pods without matching toleration (and prevents new ones).
    *   *Apply Taint*: `kubectl taint nodes node1 key=value:NoSchedule`
*   **Toleration (Pod Level)**: Allows a Pod to tolerate a Node's taint.
    *   *YAML Snippet*:
      ```yaml
      tolerations:
      - key: "key"
        operator: "Equal" # or "Exists"
        value: "value"
        effect: "NoSchedule"
        tolerationSeconds: 3600 # Only for NoExecute - time before eviction
      ```
*   **Key Use Cases**:
    *   **Dedicated Nodes**: Taint `dedicated=ml:NoSchedule`, add toleration to ML Pods.
    *   **Specialized Hardware**: Taint `gpu=true:NoSchedule`, add toleration to GPU workloads.
    *   **Node Maintenance**: Taint `dedicated=maintenance:NoExecute` to safely drain Nodes (evicts non-tolerating Pods).
*   **Critical Nuance**: Taints **only affect new scheduling decisions** (except `NoExecute`). Existing Pods stay unless they lack the toleration *and* effect is `NoExecute`.

#### **5. Node Selectors**
*   **Purpose**: Simplest way to constrain Pods to Nodes with specific labels (legacy, largely superseded by Node Affinity).
*   **How it Works**:
    *   Label Nodes: `kubectl label nodes node1 disktype=ssd`
    *   Pod Spec:
      ```yaml
      spec:
        nodeSelector:
          disktype: ssd
      ```
*   **Limitation vs. Affinity**: Only supports `key=value` matches (no operators like `In`, `NotIn`, `Exists`). Use **Node Affinity** for complex requirements.

---

### **Chapter 23: Resource Management**

#### **1. CPU and Memory Requests & Limits**
*   **Requests** (`spec.containers[].resources.requests`):
    *   **Guaranteed minimum** resources the container gets.
    *   **Used for Scheduling**: Scheduler ensures Node has *at least* this much free.
    *   *Units*: CPU = `millicores` (e.g., `500m` = 0.5 vCPU), Memory = `Mi` (Mebibytes), `Gi` (Gibibytes).
*   **Limits** (`spec.containers[].resources.limits`):
    *   **Hard ceiling** on resources the container can use.
    *   **Enforced by Kernel**: CPU throttled, Memory OOMKilled if exceeded.
    *   *Units*: Same as Requests.
*   **Critical Rules**:
    *   `limits >= requests` (Must be true!).
    *   **Never set limits without requests** (Scheduler ignores limits for placement).
    *   **Always set requests** (Otherwise, default = 0, leading to poor scheduling & QoS issues).
*   *Example YAML*:
  ```yaml
  resources:
    requests:
      memory: "64Mi"
      cpu: "250m"
    limits:
      memory: "128Mi"
      cpu: "500m"
  ```

#### **2. Quality of Service (QoS) Classes**
*   **Purpose**: Kubernetes uses QoS to decide **which Pods to evict first** under Node memory pressure.
*   **How it's Determined**:
    *   **Guaranteed**:
        *   *Condition*: Every container has `limits == requests` (and limits are set).
        *   *Eviction Priority*: **LOWEST** (Evicted only as last resort).
        *   *Use Case*: Critical system Pods (kube-proxy, coreDNS), latency-sensitive apps.
    *   **Burstable**:
        *   *Condition*: NOT `Guaranteed` AND at least one container has a `request` set.
        *   *Eviction Priority*: **MEDIUM** (Evicted before `BestEffort`).
        *   *Use Case*: Most stateless applications (web servers, APIs).
    *   **BestEffort**:
        *   *Condition*: NO requests or limits set for *any* container.
        *   *Eviction Priority*: **HIGHEST** (Evicted first under pressure).
        *   *Use Case*: **AVOID!** Only for testing/non-critical batch jobs. Highly unstable.
*   **Eviction Order**: `BestEffort` -> `Burstable` (lowest `request` first) -> `Guaranteed`.
*   **Critical Insight**: Setting `requests` significantly higher than actual usage wastes resources but improves stability. Tune based on monitoring!

#### **3. Vertical Pod Autoscaler (VPA)**
*   **Purpose**: **Automatically adjusts CPU/Memory `requests` and `limits`** for Pods *within a single Node* (vertical scaling).
*   **How it Works**:
    1.  **Recommender**: Analyzes historical resource usage (via Metrics Server).
    2.  **Updater**: Updates Pod specs (requires Pod restart) OR
    3.  **Admission Controller**: Injects new `requests/limits` during Pod creation (no restart needed).
*   **Modes**:
    *   `Off`: Only provides recommendations (safe for testing).
    *   `Initial`: Sets requests/limits only on Pod *creation*.
    *   `Recreate`: Updates requests/limits, **restarts** Pods (most common).
    *   `Auto`: Like `Recreate`, but VPA manages the update process (uses `vpa-updater`).
*   **YAML Snippet (VPA Object)**:
  ```yaml
  apiVersion: autoscaling.k8s.io/v1
  kind: VerticalPodAutoscaler
  metadata:
    name: my-app-vpa
  spec:
    targetRef:
      apiVersion: "apps/v1"
      kind: Deployment
      name: my-app
    updatePolicy:
      updateMode: "Auto" # or "Off", "Initial", "Recreate"
    resourcePolicy:
      containerPolicies:
      - containerName: "*"
        minAllowed:
          cpu: "100m"
          memory: "64Mi"
        maxAllowed:
          cpu: "2000m"
          memory: "2Gi"
  ```
*   **Limitations**:
    *   **Cannot be used with HPA** for the *same resource metric* (CPU/Memory). Choose one scaling strategy.
    *   Requires Pod restarts (except `Initial` mode).
    *   Less mature than HPA. Test thoroughly!
*   **Use Case**: Workloads with stable, predictable resource needs that change slowly over time (e.g., monolithic apps).

#### **4. Resource Quotas**
*   **Purpose**: **Limit aggregate resource consumption** *per Namespace* (prevent one team from hogging the cluster).
*   **Scope**: Namespace-level (must be created *within* the target namespace).
*   **What it Controls**:
    *   Total `requests`/`limits` for CPU, Memory, GPU, etc.
    *   Total count of objects (Pods, Services, PersistentVolumeClaims, etc.).
    *   Storage requests (`requests.storage`).
*   **YAML Snippet (ResourceQuota)**:
  ```yaml
  apiVersion: v1
  kind: ResourceQuota
  metadata:
    name: compute-resources
    namespace: dev-team
  spec:
    hard:
      pods: "20"
      requests.cpu: "10"
      requests.memory: "20Gi"
      limits.cpu: "20"
      limits.memory: "40Gi"
      requests.storage: "100Gi"
      persistentvolumeclaims: "10"
  ```
*   **Critical Behavior**:
    *   Prevents Pod creation if quota would be exceeded.
    *   **Does NOT** affect existing Pods (only future creations/updates).
    *   **Does NOT** enforce limits on individual Pods (use `LimitRange` for that).
*   **Use Case**: Multi-tenant clusters, enforcing cost budgets per team/project.

#### **5. LimitRanges**
*   **Purpose**: Set **defaults and constraints** for resource `requests`/`limits` *per Pod/Container* within a Namespace.
*   **Scope**: Namespace-level.
*   **What it Controls**:
    *   Default `requests`/`limits` if not specified in Pod spec.
    *   Minimum/Maximum `requests`/`limits` per container.
    *   Minimum/Maximum `requests`/`limits` per Pod.
    *   Min/Max `LimitRequestRatio` (limits/requests).
*   **YAML Snippet (LimitRange)**:
  ```yaml
  apiVersion: v1
  kind: LimitRange
  metadata:
    name: resource-limits
    namespace: dev-team
  spec:
    limits:
    - type: Container
      default: # Applied if Pod doesn't specify limits
        cpu: 500m
        memory: 512Mi
      defaultRequest: # Applied if Pod doesn't specify requests
        cpu: 100m
        memory: 128Mi
      min:
        cpu: 50m
        memory: 64Mi
      max:
        cpu: 2000m
        memory: 2Gi
      maxLimitRequestRatio: # max(limit/request)
        cpu: 5 # limit <= 5 * request
        memory: 3
  ```
*   **Critical Interaction**:
    *   `LimitRange` **defaults** fill in missing `requests`/`limits`.
    *   `LimitRange` **constraints** enforce min/max values *before* scheduling.
    *   Works *with* `ResourceQuota` (Quota uses the final values after LimitRange defaults).
*   **Use Case**: Enforcing sensible defaults, preventing users from requesting 0 resources (`BestEffort` QoS), or setting hard ceilings per container.

---

### **Chapter 24: Horizontal Pod Autoscaler (HPA)**

#### **1. Scaling Based on CPU/Memory**
*   **Purpose**: **Automatically scale the number of Pod replicas** based on observed metrics (horizontal scaling).
*   **Core Mechanism**:
    *   Monitors average metric value across *all* target Pods.
    *   Calculates: `desiredReplicas = ceil[currentReplicas * (currentMetricValue / desiredMetricValue)]`
    *   *Example*: 3 Pods averaging 60% CPU, target 50% -> `desiredReplicas = ceil[3 * (60 / 50)] = ceil[3.6] = 4`
*   **CPU-Based Scaling**:
    *   `targetAverageUtilization` (e.g., 50%) - Most common.
    *   `targetAverageValue` (e.g., 100m CPU) - Less common.
*   **Memory-Based Scaling**:
    *   **Only `targetAverageValue`** (e.g., 200Mi). *Avoid `targetAverageUtilization` for memory* (usage != pressure; high usage might be expected/cached).
*   **YAML Snippet (CPU HPA)**:
  ```yaml
  apiVersion: autoscaling/v2
  kind: HorizontalPodAutoscaler
  metadata:
    name: my-app-hpa
  spec:
    scaleTargetRef:
      apiVersion: apps/v1
      kind: Deployment
      name: my-app
    minReplicas: 2
    maxReplicas: 10
    metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 50 # Target 50% CPU usage
  ```

#### **2. Custom and External Metrics (Prometheus, Datadog)**
*   **Purpose**: Scale based on **application-specific metrics** (e.g., requests per second, queue depth).
*   **How it Works**:
    1.  **Metrics Adapter**: Required (e.g., Prometheus Adapter, Datadog Cluster Agent). Translates custom metrics into K8s API format.
    2.  **Custom Metrics API**: HPA queries this API (provided by the adapter).
*   **Key Concepts**:
    *   **Pod Metric**: Average value *per Pod* (e.g., `http_requests_per_second`). Scales to hit target *per Pod*.
    *   **Object Metric**: Single value for an external object (e.g., `kafka_topic_partition_lag`). Scales to hit target *for the whole object*.
    *   **External Metric**: Value from *outside* K8s (e.g., CloudWatch SQS queue size). Similar to Object metric.
*   **YAML Snippet (Custom Metric - Pod)**:
  ```yaml
  metrics:
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second
      target:
        type: AverageValue
        averageValue: 100 # Target 100 req/s per Pod
  ```
*   **YAML Snippet (External Metric - SQS Queue)**:
  ```yaml
  metrics:
  - type: External
    external:
      metric:
        name: aws_sqs_approximate_number_of_messages_visible
        selector:
          matchLabels:
            queue: my-queue
      target:
        type: AverageValue
        averageValue: 20 # Target 20 messages per replica
  ```
*   **Critical Setup**: Requires deploying a **custom metrics adapter** (non-trivial!). Test thoroughly.

#### **3. Metrics Server Setup**
*   **Purpose**: Provides **resource metrics** (CPU/Memory) for HPA (v2) and `kubectl top`.
*   **How it Works**:
    *   Runs as a Deployment in `kube-system`.
    *   Scrapes kubelets via Summary API (secure port 10250).
    *   Exposes metrics via Kubernetes Aggregated API (`metrics.k8s.io`).
*   **Installation (Typical)**:
    ```bash
    kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
    # For minikube: minikube addons enable metrics-server
    # For issues (certs): Add args to Deployment:
    #   - --kubelet-insecure-tls
    #   - --kubelet-preferred-address-types=InternalIP
    ```
*   **Verification**:
    ```bash
    kubectl top nodes
    kubectl top pods
    ```

#### **4. HPA with Custom Metrics**
*   **Process Flow**:
    1.  App emits custom metric (e.g., via Prometheus client library).
    2.  Prometheus scrapes app.
    3.  Prometheus Adapter (deployed in cluster) watches Prometheus and exposes metrics via `custom.metrics.k8s.io` API.
    4.  HPA queries `custom.metrics.k8s.io` for the metric.
    5.  HPA calculates desired replicas and updates Deployment/ReplicaSet.
*   **Critical Considerations**:
    *   **Metric Resolution**: Ensure metrics are scraped frequently enough (HPA checks every 15-30s by default).
    *   **Stabilization Window**: HPA ignores scaling recommendations that would cause rapid scaling up/down (`--horizontal-pod-autoscaler-downscale-stabilization` flag, default 5m).
    *   **Tolerance**: Small deviations from target are ignored (`--horizontal-pod-autoscaler-tolerance`, default 0.1 = 10%).
    *   **Cooldown Periods**: Prevent thrashing (configurable via HPA spec `behavior` field - advanced).
*   **Troubleshooting**:
    *   `kubectl describe hpa my-app-hpa` (Look for `MetricsNotAvailable`, `FailedGetExternalMetric`).
    *   Check Metrics Server/Adapter logs.
    *   Verify metric exists: `kubectl get --raw "/apis/custom.metrics.k8s.io/v1beta1" | jq .`

---

### **Chapter 25: Cluster Autoscaler**

#### **1. Auto-Scaling Nodes in the Cluster**
*   **Purpose**: **Automatically adjusts the size of the Node pool** (up *and* down) based on Pod scheduling needs.
*   **How it Works**:
    *   **Watches** for:
        *   **Pending Pods**: Pods unschedulable due to *insufficient resources* (CPU, Memory, GPU, etc.).
        *   **Underutilized Nodes**: Nodes where all Pods could fit on other Nodes (considering requests, affinity, taints).
    *   **Scaling Up**:
        1.  Detects unschedulable Pod.
        2.  Simulates adding a Node of each available type.
        3.  If simulation succeeds, requests cloud provider to add a Node.
    *   **Scaling Down**:
        1.  Identifies Nodes with utilization below threshold (default 50% of allocatable).
        2.  Checks if *all* Pods on Node can be safely scheduled elsewhere (respecting PDBs, affinity, etc.).
        3.  If safe, cordons Node, drains Pods, deletes Node.
*   **Critical Dependencies**:
    *   Must run **alongside HPA** (HPA scales Pods, CA scales Nodes for those Pods).
    *   Requires **resource requests** on *all* Pods (CA uses requests for simulation).
    *   Requires **Pod Disruption Budgets (PDBs)** to protect against excessive downtime during scale-down.

#### **2. Integration with Cloud Providers (AWS, GCP, Azure)**
*   **Mechanism**: CA uses **cloud provider SDKs** to manage Node pools (Instance Groups, ASGs, VMSS).
*   **Provider-Specific Setup**:
    *   **AWS (EKS)**:
        *   Uses Auto Scaling Groups (ASGs).
        *   Tag ASGs with `k8s.io/cluster-autoscaler/<cluster-name>: owned` and `k8s.io/cluster-autoscaler/enabled: true`.
        *   CA Args: `--node-group-auto-discovery=asg:tag=k8s.io/cluster-autoscaler/enabled,k8s.io/cluster-autoscaler/<cluster-name>`
    *   **GCP (GKE)**:
        *   Enable "Cluster Autoscaling" on Node Pools in GKE console.
        *   *No separate CA deployment needed* (managed by GKE).
    *   **Azure (AKS)**:
        *   Enable "Cluster Autoscaler" on Node Pools in AKS console.
        *   *No separate CA deployment needed* (managed by AKS).
*   **On-Prem/Bare Metal**: Requires custom cloud provider plugin (e.g., for vSphere, OpenStack) or cluster-api provider.

#### **3. Cost Optimization**
*   **Core Strategy**: Match Node count precisely to actual workload demand.
*   **Key Techniques**:
    *   **Right-Size Requests**: Accurate `requests` are critical for CA efficiency (use VPA recommendations or monitoring).
    *   **Bin Packing**: CA tries to pack Pods densely (using `LeastRequestedPriority` scoring) to free up Nodes for scale-down.
    *   **Multiple Node Pools**: Group workloads by resource profile (e.g., CPU-optimized, Memory-optimized pools). CA scales pools independently.
    *   **Spot/Preemptible Instances**: Use cheaper instances for fault-tolerant workloads (CA handles evictions gracefully).
    *   **Scale-Down Safety**:
        *   `--scale-down-unneeded-time`: Time a Node must be unneeded before scale-down (default 10m).
        *   `--scale-down-delay-after-add`: Wait after scale-up before scaling down (default 10m).
        *   `--scale-down-utilization-threshold`: Node utilization threshold for scale-down (default 0.5).
    *   **Exclude Critical Nodes**: Use `--skip-nodes-with-system-pods=false` cautiously; better to use taints for critical system Pods.
*   **Cost Pitfalls to Avoid**:
    *   **Over-Provisioning**: Setting `max` too high in HPA/CA.
    *   **Under-Provisioning Requests**: Causes CA to over-provision Nodes (Pods need more space than requested).
    *   **Ignoring Scale-Down Delays**: Too aggressive scale-down causes constant churn.
    *   **No PDBs**: CA might disrupt critical services during scale-down.

---
