### **Chapter 32: Logging - The "What Happened?" Layer**

**Why Logging Matters in Kubernetes:**
*   Containers are ephemeral – logs vanish when pods die.
*   Microservices generate logs across *hundreds* of pods/nodes.
*   Debugging requires correlating events across services.
*   **Goal:** Capture, aggregate, search, analyze, and alert on log data *centrally*.

#### **1. Centralized Logging Architecture**
*   **The Problem:** Logs scattered across nodes/pods → impossible to search or correlate.
*   **The Solution:** Ship logs from *every* container/node to a central system.
*   **Core Components:**
    *   **Log Sources:** Application containers (stdout/stderr), Node system logs (journald, syslog), Kubernetes components (kubelet, apiserver).
    *   **Log Agents (Shippers):** Lightweight processes running *on each node* or *as sidecars* that collect logs. *(Fluent Bit, Filebeat, Vector)*.
    *   **Log Router/Processor:** Optional layer for filtering, parsing, enriching, buffering logs *before* storage. *(Fluentd, Logstash)*.
    *   **Log Storage & Query Engine:** Backend optimized for high-volume log ingestion and fast search. *(Elasticsearch, Loki, Cloud Logging)*.
    *   **Visualization/Alerting:** UI for searching, dashboards, and setting alerts. *(Kibana, Grafana, Cloud Console)*.
*   **Key Design Principles:**
    *   **Decoupling:** Apps write to stdout/stderr; infrastructure handles shipping.
    *   **Resilience:** Buffering (in-memory/disk) to survive network/backend outages.
    *   **Scalability:** Agents scale per-node; storage scales independently.
    *   **Security:** TLS for shipping, RBAC for access control.

#### **2. Sidecar Logging Patterns**
*   **Why Sidecars?** When:
    *   App *can't* write to stdout/stderr (legacy apps writing to files).
    *   You need *per-pod* log processing (e.g., different parsers per app).
    *   You want isolation (one app's logging failure doesn't affect others).
*   **Patterns:**
    *   **a) Streaming Sidecar (Most Common):**
        *   App writes logs to a *volume* (emptyDir) shared with the sidecar.
        *   Sidecar (e.g., Fluent Bit) tails the log file and ships to central store.
        *   *Pros:* Simple, isolates app from logging infra. *Cons:* Extra resource overhead per pod.
    *   **b) Adapter Sidecar:**
        *   App writes logs to a *custom format* or *non-standard location*.
        *   Sidecar *transforms* logs into a standard format (e.g., JSON) *before* shipping.
        *   *Use Case:* Legacy apps with non-structured logs.
    *   **c) Syslog Sidecar:**
        *   App configured to send logs via Syslog protocol.
        *   Sidecar runs a lightweight Syslog server, receives logs, and forwards them.
        *   *Use Case:* Apps tightly coupled to Syslog.
*   **Critical Consideration:** Sidecars consume CPU/memory *per pod*. Avoid for simple stdout apps (use Node Agent pattern instead).

#### **3. Tools: Fluentd, Fluent Bit, Logstash**
*   **Fluentd (Cloud Native Computing Foundation Project):**
    *   **Role:** Primarily a *log router/processor*. (Can also collect, but heavier).
    *   **Strengths:** Massive plugin ecosystem (1500+), powerful filtering/parsing (Ruby), flexible routing.
    *   **Weaknesses:** Higher resource usage (Ruby VM), complex config for beginners.
    *   **K8s Use:** Often runs as a *DaemonSet* (Node Agent) or *Deployment* (central aggregator). Configured via `ConfigMap`.
*   **Fluent Bit (CNCF Project, *Sibling* of Fluentd):**
    *   **Role:** Primarily a *lightweight log shipper/collector* (Node Agent or Sidecar).
    *   **Strengths:** Extremely low resource footprint (C), faster startup, simpler config, built-in Kubernetes metadata filter. *Ideal for edge/node-side*.
    *   **Weaknesses:** Fewer plugins than Fluentd, less complex processing capabilities.
    *   **K8s Use:** **The de facto standard for Node Agent pattern.** Runs as a `DaemonSet`. Often feeds Fluentd for heavy processing.
*   **Logstash (Elastic Stack):**
    *   **Role:** Powerful *log processor/router* (part of ELK stack).
    *   **Strengths:** Rich filtering/parsing (grok), excellent for complex log transformations, strong Elasticsearch integration.
    *   **Weaknesses:** **High resource consumption** (JVM), slower startup, overkill for simple shipping.
    *   **K8s Use:** Typically runs as a *central processing layer* (Deployment) *after* Fluent Bit/Fluentd agents. Rarely used directly on nodes due to weight.
*   **Comparison Summary:**
    | Feature          | Fluent Bit          | Fluentd             | Logstash            |
    | :--------------- | :------------------ | :------------------ | :------------------ |
    | **Primary Role** | Shipper/Collector   | Router/Processor    | Processor/Router    |
    | **Language**     | C                   | Ruby                | Java (JVM)          |
    | **Resource Use** | **Very Low**        | Medium              | **High**            |
    | **Best For**     | Node Agent/Sidecar  | Central Processing  | Complex Processing  |
    | **K8s Popularity**| **⭐⭐⭐⭐⭐**          | ⭐⭐⭐⭐               | ⭐⭐                  |

#### **4. Shipping Logs To: Elasticsearch, Loki, Cloud Providers**
*   **Elasticsearch (ELK Stack - Elasticsearch, Logstash, Kibana):**
    *   **How:** Fluentd/Bit ship logs → Elasticsearch index.
    *   **Pros:** Extremely powerful full-text search, Kibana visualization, mature ecosystem.
    *   **Cons:** Complex to operate at scale, resource-intensive (ES needs significant RAM/CPU), expensive, schema/index management overhead.
    *   **K8s Tip:** Use `fluent-plugin-elasticsearch`. Consider managed services (AWS OpenSearch, Elastic Cloud).
*   **Loki (Grafana Labs):**
    *   **How:** Fluent Bit (via `loki` output plugin) or Promtail (Loki's dedicated shipper) ship logs → Loki.
    *   **Core Idea:** "Logs are logs, not data." Stores logs *compressed* with minimal metadata (labels like `pod`, `namespace`, `container`). **No full-text indexing!**
    *   **Pros:** **Extremely cheap/storage efficient**, integrates *seamlessly* with Grafana (same labels as metrics!), simple to operate, logQL query language.
    *   **Cons:** Slower for complex full-text searches (vs ES), less mature for pure log analysis (vs Kibana).
    *   **K8s Tip:** **Highly recommended for cost-effective, Grafana-centric logging.** Use `loki-distributed` Helm chart. Query with LogQL in Grafana.
*   **Cloud Providers (GCP Cloud Logging, AWS CloudWatch Logs, Azure Monitor Logs):**
    *   **How:** Native agents (e.g., Google Cloud Logging Agent, CloudWatch Agent) or Fluentd/Bit plugins ship logs.
    *   **Pros:** Fully managed, integrated with cloud IAM/billing, often includes basic dashboards/alerting, scales automatically.
    *   **Cons:** Vendor lock-in, pricing can escalate quickly (especially per-log-volume), less flexible/customizable than open-source, query languages vary.
    *   **K8s Tip:** Use cloud-specific operators or config (e.g., GKE's `gke-metadata-server`). Understand pricing model (e.g., CWL charges per ingested GB *and* per queried GB).

---

### **Chapter 33: Monitoring - The "What's Happening?" Layer (Metrics)**

**Why Monitoring Matters:**
*   Understand system health *in real-time*.
*   Detect performance bottlenecks (CPU, memory, disk, network).
*   Set alerts for SLO/SLI breaches (e.g., high error rates, latency).
*   Capacity planning.

#### **1. Metrics in Kubernetes**
*   **What are Metrics?** Numerical measurements over time (e.g., `container_cpu_usage_seconds_total`, `http_requests_total`).
*   **Key Metric Types:**
    *   **Counters:** Monotonically increasing (e.g., total requests). *Reset on restart.*
    *   **Gauges:** Can go up/down (e.g., current memory usage).
    *   **Histograms:** Track distributions (e.g., request durations - buckets).
    *   **Summaries:** Similar to histograms, but quantile-based (less common now).
*   **Kubernetes Metric Sources:**
    *   **Application Metrics:** Exposed by your app (e.g., `/metrics` endpoint in Go/Java apps).
    *   **cAdvisor:** Built into kubelet. Exposes *container-level* resource usage (CPU, memory, network, disk I/O).
    *   **kube-state-metrics:** *Crucial!* Translates Kubernetes object state (Deployments, Pods, Nodes) into metrics (e.g., `kube_deployment_status_replicas_available`).
    *   **Node Metrics:** System-level metrics (via Node Exporter).
    *   **API Server Metrics:** Request rates, latencies, errors.

#### **2. Metrics Server**
*   **What it is:** A *cluster-wide* agent that collects basic resource metrics (CPU/memory) from kubelets via the Summary API.
*   **Purpose:** **Solely to power `kubectl top node/pod` and the Horizontal Pod Autoscaler (HPA).**
*   **Key Limitations:**
    *   **NO HISTORICAL DATA:** Only provides *current* (last 1-5 min) metrics.
    *   **NO PERSISTENCE:** Data is ephemeral.
    *   **NO ALERTING/DASHBOARDS:** Not a monitoring solution!
*   **Why it's Essential:** HPA *requires* Metrics Server to function. Lightweight and core to K8s autoscaling.

#### **3. Prometheus Architecture (The Gold Standard)**
*   **Core Components:**
    *   **Prometheus Server:** Pulls/scrapes metrics from targets, stores time-series data in local TSDB, runs alerting rules, provides query API.
    *   **Exporters:** Agents that expose metrics *in Prometheus format* from systems that don't natively support it (e.g., Node Exporter, MySQL Exporter). **Crucial for K8s!**
    *   **Pushgateway:** *Optional.* Accepts *pushed* metrics (for short-lived jobs). Use sparingly.
    *   **Alertmanager:** Handles alerts sent by Prometheus, deduplicates, groups, routes (email, Slack, PagerDuty).
    *   **Client Libraries:** For apps to expose custom metrics (e.g., `prom-client` for Node.js).
*   **How it Works in K8s:**
    1.  Prometheus server discovers targets (Pods, Services, Nodes) via K8s API.
    2.  It *scrapes* (HTTP GETs) the `/metrics` endpoint on each target at configured intervals.
    3.  Metrics are stored with labels (e.g., `pod="my-app-7d5f8b6c4d-2xq9k", namespace="prod"`).
    4.  Alertmanager processes alerts based on rules.
*   **Strengths:** Powerful query language (PromQL), pull-based (simpler security), robust alerting, massive ecosystem.
*   **Weaknesses:** Single-server storage (sharding/replication complex), no built-in auth (relies on network/security), local storage limits scalability (use Thanos/VictoriaMetrics for scale).

#### **4. Exporters (The Data Collectors)**
*   **Node Exporter:**
    *   **Runs:** As a `DaemonSet` (one per node).
    *   **Collects:** Host-level metrics (CPU, memory, disk, network, filesystem, systemd).
    *   **Endpoint:** `http://<node-ip>:9100/metrics`
*   **cAdvisor:**
    *   **Runs:** Built into the `kubelet` (on every node).
    *   **Collects:** *Container-level* resource metrics (CPU, memory, network, disk I/O per container).
    *   **Endpoint:** `http://<node-ip>:10250/metrics/cadvisor` (secured by kubelet auth).
    *   **Note:** Prometheus usually scrapes cAdvisor *via* the kubelet proxy, not directly.
*   **kube-state-metrics (KSM):**
    *   **Runs:** As a `Deployment`.
    *   **Collects:** *Kubernetes object state* as metrics (e.g., `kube_pod_status_phase`, `kube_deployment_spec_replicas`, `kube_node_status_condition`).
    *   **Why Critical?** Translates K8s API state into queryable metrics. **Essential for SLO monitoring!**
    *   **Endpoint:** `http://<ksm-service>:8080/metrics`
*   **Other Key Exporters:** `kube-prometheus-stack` bundles many (CoreDNS, etcd, kube-apiserver, kube-scheduler, kube-controller-manager).

#### **5. ServiceMonitors & PodMonitors (Prometheus Operator)**
*   **The Problem:** Manually configuring Prometheus scrape targets for dynamic K8s is impossible.
*   **The Solution:** **Prometheus Operator** (a Kubernetes controller).
*   **How it Works:**
    1.  Install the Operator (usually via Helm chart `prometheus-community/kube-prometheus-stack`).
    2.  Define `ServiceMonitor` or `PodMonitor` *Custom Resources (CRs)*.
    3.  Operator watches these CRs and **automatically generates** Prometheus scrape configuration.
*   **ServiceMonitor:**
    *   Targets *Services*. Scrapes *all pods* backing that Service.
    *   **Use When:** You want to monitor a logical service (e.g., all `frontend` pods).
    *   **Example:**
        ```yaml
        apiVersion: monitoring.coreos.com/v1
        kind: ServiceMonitor
        metadata:
          name: my-app-monitor
          labels:
            release: prometheus-stack # Must match Prometheus CR's 'spec.serviceMonitorSelector'
        spec:
          selector:
            matchLabels:
              app: my-app # Selects Service with this label
          endpoints:
          - port: web # Port name in Service definition
            interval: 15s
        ```
*   **PodMonitor:**
    *   Targets *Pods* directly (using label selectors), bypassing Services.
    *   **Use When:** You need to scrape pods not exposed by a Service (e.g., StatefulSets, sidecars).
    *   **Example:**
        ```yaml
        apiVersion: monitoring.coreos.com/v1
        kind: PodMonitor
        metadata:
          name: my-sidecar-monitor
          labels:
            release: prometheus-stack
        spec:
          podMetricsEndpoints:
          - port: metrics
            interval: 10s
          selector:
            matchLabels:
              app: my-app
              sidecar: logging
        ```
*   **Key Benefit:** Declarative, GitOps-friendly monitoring setup. Scales with your cluster.

#### **6. Grafana Dashboards**
*   **Role:** The *visualization and alerting frontend* for Prometheus (and Loki, Tempo, others).
*   **Why Essential:**
    *   **Dashboards:** Build rich, interactive visualizations (graphs, tables, heatmaps) from PromQL queries.
    *   **Alerting:** Create visual alert rules (beyond Prometheus Alertmanager).
    *   **Correlation:** View metrics (Prometheus), logs (Loki), and traces (Tempo/Jaeger) *in the same pane*.
    *   **Templating:** Use variables (e.g., `$namespace`, `$pod`) for dynamic dashboards.
*   **K8s Integration:**
    *   Often installed alongside Prometheus via `kube-prometheus-stack`.
    *   Pre-built dashboards: Use community dashboards (e.g., [Kubernetes Mixins](https://github.com/kubernetes-monitoring/kubernetes-mixin)) or vendor templates (e.g., for Node Exporter, KSM).
    *   **Example PromQL in Grafana:** `sum(rate(http_requests_total{job="my-app", status=~"5.."}[5m])) by (pod) / sum(rate(http_requests_total{job="my-app"}[5m])) by (pod) > 0.01` (Error rate >1%).
*   **Best Practice:** Store dashboards as JSON in Git and deploy via ConfigMaps/Helm.

---

### **Chapter 34: Tracing - The "Why is it Slow?" Layer (Distributed Tracing)**

**Why Tracing Matters:**
*   Microservices mean a single user request flows through *many* services.
*   Performance issues are often *distributed* (e.g., slow DB call in Service B causing timeout in Service A).
*   **Goal:** Track a single request as it propagates through the system, measuring latency at each hop.

#### **1. Distributed Tracing Overview**
*   **Core Concepts:**
    *   **Trace:** Represents the *entire* journey of a single request through the system. Has a unique `trace_id`.
    *   **Span:** A single unit of work within a trace (e.g., an HTTP request, a DB call). Has a `span_id`, `parent_span_id`, operation name, timestamps, tags, logs.
    *   **Span Context:** Propagated between services (via HTTP headers like `b3`, `traceparent`) to link spans into a trace. Contains `trace_id`, `span_id`, `parent_span_id`.
*   **How it Works:**
    1.  Ingress receives a request. Creates a **Root Span** (new `trace_id`).
    2.  App makes a call to Service B. Creates a **Child Span**, sets `parent_span_id = Root Span's id`, propagates context.
    3.  Service B receives request, extracts context, creates its own span as child of the received span.
    4.  Spans are reported to a tracing backend (Jaeger/Zipkin).
    5.  Backend reconstructs the full trace timeline.
*   **Key Benefits:** Identify bottlenecks, understand service dependencies, debug latency issues, measure end-to-end latency.

#### **2. Jaeger & Zipkin (Tracing Backends)**
*   **Jaeger (CNCF Project):**
    *   **Strengths:** Native OpenTelemetry support, scalable architecture (collector, query, agent), strong Kubernetes integration, rich UI (timeline view, dependencies), supports multiple storage backends (Cassandra, ES, memory).
    *   **K8s Deployment:** Typically Helm chart (`jaegertracing/jaeger-operator`). Operator manages CRs (`Jaeger`).
    *   **UI:** `/search` (find traces), `/dependencies` (auto-generated service map).
*   **Zipkin:**
    *   **Strengths:** Simpler architecture, lightweight, mature, good UI.
    *   **Weaknesses:** Less scalable than Jaeger for very high volume, fewer features.
    *   **K8s Deployment:** Helm chart (`openzipkin/zipkin`) or Deployment + Service.
*   **Comparison:** Jaeger is generally preferred for production K8s due to scalability, OTel integration, and CNCF backing. Zipkin is simpler for small setups.

#### **3. OpenTelemetry (OTel) Integration - The Future Standard**
*   **The Problem:** Proprietary SDKs (Jaeger, Zipkin clients) caused vendor lock-in and inconsistent instrumentation.
*   **The Solution: OpenTelemetry (CNCF Project)**
    *   **Unified API/SDK:** Single set of libraries (`opentelemetry-api`, `opentelemetry-sdk`) for *all* languages to generate telemetry (Traces, Metrics, Logs).
    *   **Vendor-Neutral:** Instrument *once*, send data to *any* backend (Jaeger, Zipkin, Prometheus, Dynatrace, Datadog, etc.).
    *   **Components:**
        *   **Instrumentation Libraries:** Auto-instrument common frameworks (HTTP servers, DB clients) OR manual instrumentation via API.
        *   **OTel Collector:** **Critical Component.** Receives, processes (filter, transform, enrich), and exports telemetry data. Runs as `DaemonSet` (Agent mode) or `Deployment` (Gateway mode).
        *   **Exporters:** Configured in Collector to send data to backends (e.g., `jaeger`, `zipkin`, `prometheus`, `logging`).
*   **Why OTel is Essential for K8s:**
    *   **Standardization:** Avoids SDK lock-in.
    *   **Flexibility:** Collector decouples apps from backends. Change backends without re-instrumenting apps.
    *   **Resource Efficiency:** Collector handles batching, retries, protocol translation.
    *   **Unified Pipeline:** Process traces, metrics, *and logs* through the same Collector.
*   **Getting Started:**
    1.  Instrument your apps with OTel SDK (auto-instrumentation is easiest).
    2.  Deploy OTel Collector (`opentelemetry-operator` Helm chart).
    3.  Configure Collector to receive data (e.g., OTLP over gRPC) and export to Jaeger/Zipkin/Prometheus.
    4.  (Optional) Use Service Mesh (Istio, Linkerd) for *automatic* tracing instrumentation without app code changes.

---

### **Chapter 35: Debugging & Troubleshooting - The "Fix It Now!" Layer**

**Mindset:** Start broad (`kubectl get`), narrow down (`describe`, `logs`), then dive deep (`exec`, network checks). **Always check logs first!**

#### **1. Common Issues & Fixes (The "Big 5")**
*   **CrashLoopBackOff:**
    *   **Meaning:** Container starts → crashes → restarts → crashes... backoff delay increases.
    *   **Diagnosis:**
        *   `kubectl logs <pod> --previous` (Logs from *crashed* instance!)
        *   `kubectl describe pod <pod>` → Check `Events` (Last State: Exit Code) and `Containers.<name>.State.Waiting.Reason`.
        *   Check resource limits (OOMKilled?).
        *   Check startup dependencies (DB down? Config missing?).
    *   **Fix:** Fix app code/config, increase resources, fix dependencies, add readiness probes.
*   **ImagePullBackOff:**
    *   **Meaning:** K8s can't pull the container image.
    *   **Diagnosis:**
        *   `kubectl describe pod <pod>` → Check `Events` for exact error (e.g., `Failed to pull image "x": rpc error: code = Unknown desc = Error response from daemon: pull access denied`).
        *   Common Causes:
            *   Typo in image name/tag.
            *   Private registry requires `imagePullSecret`.
            *   Registry auth expired/invalid.
            *   Tag doesn't exist.
            *   Network policy blocking egress to registry.
    *   **Fix:** Correct image name, create/configure `imagePullSecret`, verify registry credentials, check network policies.
*   **ErrImagePull:** Similar to `ImagePullBackOff`, but *temporary* network/auth issue. Often resolves itself or requires registry fix.
*   **Pending:**
    *   **Meaning:** Pod created but not assigned to a node.
    *   **Diagnosis:**
        *   `kubectl describe pod <pod>` → Check `Events` (e.g., `Insufficient cpu`, `node(s) had taints that the pod didn't tolerate`, `no nodes available to schedule pods`).
        *   Check resource requests vs cluster capacity (`kubectl describe node`).
        *   Check taints/tolerations (`kubectl describe node`, pod spec).
        *   Check node selectors/affinity rules.
    *   **Fix:** Scale cluster, adjust resource requests, add tolerations, fix affinity rules.
*   **ContainerCreating / Error:**
    *   **Meaning:** Pod scheduled to node, but container failed to start.
    *   **Diagnosis:**
        *   `kubectl describe pod <pod>` → `Events` is **KEY** (e.g., `FailedMount: MountVolume.SetUp failed for volume...`, `Error: cannot find volume "xyz"`).
        *   Common Causes: Volume mount issues (PV/PVC not bound, wrong storage class, permissions), init container failure, runtime error.
    *   **Fix:** Check PV/PVC status, storage class config, volume definitions, init container logs.

#### **2. Core Debugging Commands (`kubectl` Lifelines)**
*   **`kubectl describe <resource> <name> -n <namespace>`:**
    *   **The #1 Command.** Shows detailed object configuration, **Events** (chronological timeline of what K8s did), status, conditions, mounted volumes, resource requests/limits.
    *   **Always check Events first!** (`kubectl describe pod my-pod | grep -A 10 Events`)
*   **`kubectl logs <pod> [-c <container>] [-n <namespace>] [--previous] [--tail=100]`**:
    *   Get stdout/stderr logs. **`--previous` is critical for CrashLoopBackOff.**
    *   For multi-container pods, **specify `-c <container>`**.
*   **`kubectl exec -it <pod> [-c <container>] [-n <namespace>] -- /bin/sh`**:
    *   Get a shell *inside* a running container. **Essential for debugging network/filesystem.**
    *   Common tools: `curl`, `wget`, `nslookup`, `dig`, `ping`, `netstat`, `tcpdump`, `strace`, `lsof`, `cat /etc/resolv.conf`.
    *   **If `/bin/sh` fails:** Try `/bin/bash`, `/busybox/sh`, or specific binaries (`kubectl exec ... -- curl localhost:8080`).

#### **3. Diagnosing Networking & DNS**
*   **Symptom: App can't reach Service/External URL**
    *   **Step 1: Inside the Pod**
        *   `kubectl exec -it <pod> -- nslookup <service-name>` (Should resolve to ClusterIP)
        *   `kubectl exec -it <pod> -- nslookup <service-name>.<namespace>.svc.cluster.local` (Full DNS name)
        *   `kubectl exec -it <pod> -- cat /etc/resolv.conf` (Check nameserver - should be `10.96.0.10` (CoreDNS))
    *   **Step 2: Check CoreDNS**
        *   `kubectl get pods -n kube-system -l k8s-app=kube-dns` (Check status)
        *   `kubectl logs -n kube-system <coredns-pod>` (Look for errors)
        *   `kubectl run -i --tty debug --image=busybox:1.28 --restart=Never -- sh` (Test DNS from *new* pod)
    *   **Step 3: Check Connectivity**
        *   `kubectl exec -it <pod> -- curl -v http://<service-ip>:<port>` (Is the service *endpoint* reachable?)
        *   `kubectl get endpoints <service-name>` (Does the service have healthy endpoints?)
        *   Check Network Policies (`kubectl get networkpolicies -A`)
        *   Check Service type (ClusterIP, NodePort, LoadBalancer) and status.
*   **Symptom: High Latency/Intermittent Failures**
    *   Use `tcpdump` inside a pod (requires `net-tools` or `tcpdump` installed): `kubectl exec -it <pod> -- tcpdump -i eth0 port 80 -w /tmp/capture.pcap` (then `kubectl cp` to get pcap file).
    *   Check kube-proxy logs (`kubectl logs -n kube-system <kube-proxy-pod>`).
    *   Check CNI plugin status/logs (Calico, Flannel, Cilium).

#### **4. Using Ephemeral Containers (K8s 1.16+)**
*   **The Problem:** `kubectl exec` fails if:
    *   Container has no shell (`/bin/sh` missing).
    *   Container is *crashed* (no running process to exec into).
    *   Container is *not starting* (init containers failing).
*   **The Solution: Ephemeral Containers**
    *   **What:** Temporary containers *added* to an *existing* pod for debugging. **Do not restart the pod.**
    *   **Why Revolutionary:** Debug pods that are broken *without* restarting them (which might clear state/crash logs).
*   **How to Use:**
    1.  **Prerequisites:** Feature gate `EphemeralContainers` enabled (default in 1.23+), `kubectl debug` plugin (built-in since 1.18).
    2.  **Basic Debug Session:**
        ```bash
        kubectl debug -it <pod-name> --image=busybox:1.28 --target=<container-name> -- sh
        ```
        *   `--target`: Attaches to the process namespace of the target container (critical for seeing its network/filesystem).
        *   `--image`: Specifies the debug container image (must have debugging tools).
    3.  **Common Debug Images:** `nicolaka/netshoot` (best!), `busybox`, `praqma/network-multitool`.
    4.  **Advanced:**
        *   **Capture Process List:** `kubectl debug --copy-to=my-debugger <pod> --image=busybox --target=app-container -- ps aux`
        *   **Capture Network Traffic:** `kubectl debug -it <pod> --image=nicolaka/netshoot --target=app-container -- tcpdump -i eth0 -w /tmp/capture.pcap`
        *   **Debug Init Containers:** `kubectl debug <pod> --image=busybox --container=debugger --share-processes --copy-to=debug-init`
*   **Key Advantages over `exec`:**
    *   Works on *crashed* pods (no running main container process needed).
    *   Works on pods with no shell in the main container.
    *   `--target` provides accurate view of the target container's context.
    *   Less intrusive than modifying the pod spec (no restart).

---
