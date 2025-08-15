### **Chapter 63: Mini Projects**

#### **1. Deploy a Full-Stack App (React + Node.js + MongoDB)**
*   **What it is:** Deploying a complete application where:
    *   **React:** Frontend (static assets served via Nginx/Node)
    *   **Node.js:** Backend API (REST/GraphQL)
    *   **MongoDB:** Database (Stateful storage)
*   **Kubernetes Components & Workflow:**
    1.  **Containerize:**
        *   `Dockerfile` for React (multi-stage build: `npm build` -> Nginx)
        *   `Dockerfile` for Node.js (copy code, `npm install`, `CMD ["node", "server.js"]`)
        *   Use official `mongo` image (or custom if needed).
    2.  **K8s Manifests:**
        *   **Frontend (React):** `Deployment` (1+ replicas) + `Service` (ClusterIP or NodePort for dev). *Uses ConfigMap for environment variables (e.g., `REACT_APP_API_URL`).*
        *   **Backend (Node.js):** `Deployment` (1+ replicas) + `Service` (ClusterIP). *Uses Secret for DB credentials, ConfigMap for other configs.*
        *   **Database (MongoDB):** **`StatefulSet`** (Critical! Ensures stable network identity & persistent storage per pod) + **Headless `Service`** (for DNS resolution like `mongo-0.mongo-svc`) + `PersistentVolumeClaim` (PVC) template. *Never use a Deployment for stateful DBs!*
        *   **Ingress:** `Ingress` resource (e.g., Nginx Ingress Controller) to route `/` to Frontend Service and `/api` to Backend Service.
    3.  **Networking:** Services enable pod-to-pod communication (Backend talks to MongoDB via `mongo-svc:27017`).
    4.  **Storage:** PVCs bound to PVs (e.g., AWS EBS, GCP PD, local storage) for MongoDB data persistence.
    5.  **Configuration:** ConfigMaps (non-sensitive settings), Secrets (DB passwords, API keys - *base64 encoded, but use KMS/external secrets for prod*).
*   **Why Kubernetes?** Isolation, scaling (frontend/backend independently), self-healing, declarative infrastructure.
*   **Critical Gotchas:**
    *   MongoDB StatefulSet requires proper `volumeClaimTemplates` and anti-affinity rules.
    *   Node.js backend needs liveness/readiness probes (`/health` endpoint).
    *   React build must be configured with the correct API base URL (often via env vars injected at build *or* runtime via `.env` in Nginx).
    *   **Never** store MongoDB data in container filesystem (ephemeral).

#### **2. CI/CD Pipeline with GitHub Actions + Argo CD**
*   **What it is:** **GitOps** implementation. Code changes trigger automated build/test (CI), then declarative sync to K8s cluster (CD).
*   **Workflow & Components:**
    1.  **GitHub Actions (CI):**
        *   Trigger: `push` to `main`/`dev` branch or PR.
        *   Steps:
            *   Checkout code.
            *   Build Docker images (React, Node.js) - tag with commit SHA (`v1.2.3-$GITHUB_SHA`).
            *   Push images to container registry (Docker Hub, GHCR, ECR).
            *   Run unit/integration tests.
            *   **Crucially:** Update the *application manifest repository* (e.g., Helm chart values.yaml or Kustomize overlay) with the **new image tag**. *This is the GitOps trigger.*
    2.  **Argo CD (CD - GitOps Engine):**
        *   **Application Manifest Repo:** Separate repo (e.g., `k8s-manifests`) containing K8s manifests (Helm charts, Kustomize, plain YAML) defining the *desired state* of the cluster.
        *   **Argo CD Installation:** Deployed *in the target K8s cluster* as a controller.
        *   **Argo CD Application CRD:** Defines:
            *   `source`: Repo URL & path (e.g., `k8s-manifests/prod`).
            *   `destination`: Target cluster & namespace.
            *   `syncPolicy`: Auto-sync (with pruning) or manual.
        *   **The GitOps Loop:**
            1.  GitHub Actions updates the manifest repo (new image tag).
            2.  Argo CD (running in-cluster) **constantly polls** the manifest repo.
            3.  Argo CD detects the change (new image tag).
            4.  Argo CD **compares** the manifest repo state (desired) with the live cluster state (current).
            5.  Argo CD **automatically applies** the delta to the cluster (e.g., updates Deployment image).
            6.  Argo CD reports sync status (Healthy, Progressing, Degraded) in UI/CLI.
*   **Why GitOps?** Auditability (all changes via Git), consistency (cluster state = repo state), rollback (git revert), security (no direct cluster access needed for deploy).
*   **Critical Gotchas:**
    *   **Separation of Concerns:** Code repo ≠ Manifest repo. Manifest repo is the *single source of truth* for cluster state.
    *   Argo CD needs read-only access to the manifest repo and cluster admin access (via ServiceAccount).
    *   Handle secrets carefully (use Sealed Secrets, SOPS, or external secrets manager - **never commit plaintext secrets**).
    *   Understand sync windows, auto-sync policies, and health checks.

#### **3. Secure Cluster with RBAC and Network Policies**
*   **What it is:** Implementing least-privilege access control (RBAC) and pod-level network segmentation (Network Policies).
*   **RBAC (Role-Based Access Control):**
    *   **Core Concepts:**
        *   `User`/`Group`/`ServiceAccount` (Subjects)
        *   `Role`/`ClusterRole` (Permissions - *what* can be done: `get`, `list`, `create`, `delete` on `pods`, `secrets`, etc.)
        *   `RoleBinding`/`ClusterRoleBinding` (Links Subjects to Roles - *who* gets permissions)
    *   **Implementation:**
        *   **ServiceAccounts:** Create dedicated SA for each app (`kubectl create serviceaccount frontend-sa`). Assign minimal permissions via `RoleBinding`.
        *   **Example:** Grant `frontend-sa` `get`, `list` on `configmaps` in `frontend` namespace.
            ```yaml
            apiVersion: rbac.authorization.k8s.io/v1
            kind: Role
            metadata:
              namespace: frontend
              name: config-reader
            rules:
            - apiGroups: [""]
              resources: ["configmaps"]
              verbs: ["get", "list"]
            ---
            apiVersion: rbac.authorization.k8s.io/v1
            kind: RoleBinding
            metadata:
              name: read-configs
              namespace: frontend
            subjects:
            - kind: ServiceAccount
              name: frontend-sa
              namespace: frontend
            roleRef:
              kind: Role
              name: config-reader
              apiGroup: rbac.authorization.k8s.io
            ```
        *   **Cluster-Wide:** Use `ClusterRole` (e.g., `view`, `edit`, `admin`) + `ClusterRoleBinding` sparingly (e.g., for cluster operators).
    *   **Why?** Prevents compromised pods/apps from accessing sensitive resources (secrets, other namespaces).
*   **Network Policies:**
    *   **Core Concept:** `NetworkPolicy` resource defining *ingress* (inbound) and *egress* (outbound) traffic rules for pods using label selectors. **Default: Allow All.**
    *   **Implementation (Default-Deny is Key!):**
        1.  **Default Deny Namespace:** Block *all* ingress/egress by default.
            ```yaml
            apiVersion: networking.k8s.io/v1
            kind: NetworkPolicy
            metadata:
              name: default-deny
              namespace: my-app
            spec:
              podSelector: {} # Selects ALL pods in namespace
              policyTypes: ["Ingress", "Egress"]
            ```
        2.  **Allow Specific Traffic:** Add policies *on top* of default deny.
            *   **Frontend -> Backend:**
                ```yaml
                apiVersion: networking.k8s.io/v1
                kind: NetworkPolicy
                metadata:
                  name: allow-frontend-to-backend
                  namespace: my-app
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
                    - protocol: TCP
                      port: 3000
                ```
            *   **Backend -> MongoDB:**
                ```yaml
                apiVersion: networking.k8s.io/v1
                kind: NetworkPolicy
                metadata:
                  name: allow-backend-to-mongo
                  namespace: my-app
                spec:
                  podSelector:
                    matchLabels:
                      app: mongo
                  policyTypes:
                  - Ingress
                  ingress:
                  - from:
                    - podSelector:
                        matchLabels:
                          app: backend
                    ports:
                    - protocol: TCP
                      port: 27017
                ```
    *   **Why?** Limits blast radius of breaches, enforces microservice boundaries, meets compliance (PCI DSS, HIPAA).
    *   **Critical Gotchas:**
        *   Requires a **CNI plugin that supports Network Policies** (Calico, Cilium, Weave Net - *not all do!*).
        *   Default-deny is essential; policies are additive.
        *   Egress policies are often overlooked but critical (prevent data exfiltration).
        *   Test policies thoroughly (use `kubectl run debug-pod --rm -it --image=busybox`).

#### **4. Monitor with Prometheus + Grafana**
*   **What it is:** Metrics collection (Prometheus) + Visualization & Alerting (Grafana).
*   **Components & Workflow:**
    1.  **Prometheus Server:**
        *   **Scraping:** Pulls metrics from HTTP endpoints (`/metrics`) at configured intervals.
        *   **Targets:** Defined via `ServiceMonitor` (CRD from Prometheus Operator) or static config.
        *   **ServiceMonitor Example (for Node.js app):**
            ```yaml
            apiVersion: monitoring.coreos.com/v1
            kind: ServiceMonitor
            metadata:
              name: nodejs-monitor
              labels:
                release: prometheus-stack # Matches Prometheus instance label
            spec:
              selector:
                matchLabels:
                  app: nodejs
              endpoints:
              - port: http # Port name in Service
                interval: 15s
                path: /metrics
              namespaceSelector:
                matchNames:
                - default
            ```
        *   **Storage:** Time-series database (TSDB) on local disk or remote write (e.g., Thanos, Cortex).
    2.  **Prometheus Operator (Crucial for K8s):**
        *   Manages Prometheus, Alertmanager, Grafana instances *declaratively* via CRDs (`Prometheus`, `Alertmanager`, `ServiceMonitor`, `PodMonitor`, `PrometheusRule`).
        *   Automates configuration, scaling, and updates.
    3.  **Node Exporter:** DaemonSet collecting node-level metrics (CPU, memory, disk, network).
    4.  **cAdvisor:** Built into Kubelet; collects container metrics (CPU, memory, network, filesystem). Scraped by Prometheus.
    5.  **Grafana:**
        *   Dashboarding: Visualize Prometheus (and other data sources) metrics.
        *   Alerting: Create alerts based on queries (e.g., `node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes < 0.1`).
        *   **Datasource:** Configured to point to Prometheus server URL.
    6.  **Alertmanager:** Handles alerts sent by Prometheus (deduplication, grouping, routing to Slack/Email/etc.).
*   **Key Metrics to Monitor:**
    *   **Cluster:** Node CPU/Mem/Disk, API Server Latency, etcd Health, Scheduler/Controller Manager Health.
    *   **Workloads:** Pod CPU/Mem Limits/Requests, Restarts, HTTP Error Rates (5xx), Request Duration (P95/P99), Queue Lengths.
*   **Why?** Proactive issue detection, performance optimization, capacity planning, SLO/SLI tracking.
*   **Critical Gotchas:**
    *   **Resource Requests/Limits:** Prometheus/Grafana need sufficient resources (OOM kills are common).
    *   **Scrape Interval:** Too low = high load; too high = missed spikes. Start with 15s.
    *   **Cardinality:** High-cardinality labels (e.g., `user_id`) explode storage. Design metrics carefully.
    *   **Alert Fatigue:** Start with critical alerts only (e.g., "Service Down", "High Error Rate"). Use `for:` clause.
    *   **Use Managed Stacks:** `kube-prometheus-stack` Helm chart (Prometheus Operator + Grafana + Alertmanager pre-configured).

#### **5. Build a Custom Helm Chart**
*   **What it is:** Creating a reusable, parameterized template for deploying applications to Kubernetes.
*   **Helm Structure:**
    ```
    mychart/
    ├── Chart.yaml      # Metadata (name, version, apiVersion, dependencies)
    ├── values.yaml     # Default configuration values
    ├── charts/         # (Optional) Subcharts (dependencies)
    └── templates/      # Template files (rendered with values)
        ├── NOTES.txt   # Post-install instructions
        ├── _helpers.tpl # Named templates (labels, common logic)
        ├── deployment.yaml
        ├── service.yaml
        ├── ingress.yaml
        └── ...
    ```
*   **Key Concepts:**
    *   **Templating:** Uses Go's `text/template` syntax. Values injected via `{{ .Values.key }}`.
        *   Example (`deployment.yaml` snippet):
            ```yaml
            spec:
              replicas: {{ .Values.replicaCount }}
              containers:
                - name: {{ .Chart.Name }}
                  image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
                  ports:
                    - containerPort: {{ .Values.service.port }}
            ```
    *   **Values:** Override defaults via `values.yaml`, CLI (`--set key=value`), or custom YAML file (`-f custom-values.yaml`).
    *   **Subcharts:** Declare dependencies in `Chart.yaml` (`dependencies:`). Values can be nested (`subchart.key`).
    *   **Hooks:** Special annotations (`helm.sh/hook: pre-install`) for jobs (e.g., DB migrations).
*   **Building a Chart:**
    1.  `helm create mychart` (scaffold).
    2.  Edit `Chart.yaml` (name, version, description).
    3.  Define parameters in `values.yaml` (replicas, image, resources, config).
    4.  Create templates in `templates/` using `.Values`.
    5.  Add `NOTES.txt` for usage instructions.
    6.  Test: `helm install test-release . --dry-run --debug` (template check), `helm install test-release .`.
    7.  Package: `helm package .` -> `mychart-0.1.0.tgz`.
*   **Why Helm?** Reusability, versioning, parameterization, dependency management, rollback (`helm rollback`).
*   **Critical Gotchas:**
    *   **Use `--dry-run --debug` religiously** before installing.
    *   **Avoid hardcoding:** *Everything* should be configurable via `values.yaml`.
    *   **Labels & Selectors:** Must match exactly between Deployment, Service, Ingress. Use `_helpers.tpl` for consistency.
    *   **Resource Requests/Limits:** Define sensible defaults in `values.yaml`.
    *   **Security:** Never put secrets in `values.yaml`! Use `--set` with encrypted files or external secrets.

#### **6. Create a Simple Operator**
*   **What it is:** A Kubernetes controller that extends the API to manage complex stateful applications *declaratively* (beyond basic Deployments/StatefulSets).
*   **Core Concepts:**
    *   **Custom Resource Definition (CRD):** Defines a new Kubernetes resource type (e.g., `CronTab`).
    *   **Custom Resource (CR):** An instance of the CRD (e.g., `my-cron` of type `CronTab`).
    *   **Controller:** Watches CRs and other resources. Reconciles desired state (CR) with actual cluster state.
    *   **Reconciliation Loop:** The core logic (`Reconcile` function) that runs whenever a relevant event occurs (CR created/updated/deleted, or watched resource changes).
*   **Building a Simple Operator (e.g., `CronTab` Operator):**
    1.  **Define CRD (`crontab_crd.yaml`):**
        ```yaml
        apiVersion: apiextensions.k8s.io/v1
        kind: CustomResourceDefinition
        metadata:
          name: crontabs.stable.example.com
        spec:
          group: stable.example.com
          names:
            plural: crontabs
            singular: crontab
            kind: CronTab
            shortNames: [ct]
          scope: Namespaced
          versions:
            - name: v1
              schema:
                openAPIV3Schema:
                  properties:
                    spec:
                      properties:
                        cronSpec:
                          type: string
                        image:
                          type: string
                        replicas:
                          type: integer
              served: true
              storage: true
        ```
    2.  **Install CRD:** `kubectl apply -f crontab_crd.yaml`
    3.  **Write Controller Logic (Using Operator Framework - e.g., Kubebuilder):**
        *   Scaffold: `kubebuilder create api --group stable --version v1 --kind CronTab`
        *   Implement `Reconcile` function (in `controllers/crontab_controller.go`):
            ```go
            func (r *CronTabReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
                // 1. Fetch the CronTab CR
                crontab := &stablev1.CronTab{}
                if err := r.Get(ctx, req.NamespacedName, crontab); err != nil {
                    return ctrl.Result{}, client.IgnoreNotFound(err)
                }

                // 2. Define desired state (e.g., a Deployment)
                desiredDeployment := &appsv1.Deployment{
                    ObjectMeta: metav1.ObjectMeta{
                        Name:      crontab.Name + "-deployment",
                        Namespace: crontab.Namespace,
                    },
                    Spec: appsv1.DeploymentSpec{
                        Replicas: &crontab.Spec.Replicas,
                        Selector: &metav1.LabelSelector{
                            MatchLabels: map[string]string{"app": crontab.Name},
                        },
                        Template: corev1.PodTemplateSpec{
                            ObjectMeta: metav1.ObjectMeta{Labels: map[string]string{"app": crontab.Name}},
                            Spec: corev1.PodSpec{
                                Containers: []corev1.Container{{
                                    Name:  "cronjob",
                                    Image: crontab.Spec.Image,
                                    // ... (args based on cronSpec)
                                }},
                            },
                        },
                    },
                }

                // 3. Check if Deployment exists
                foundDeployment := &appsv1.Deployment{}
                err := r.Get(ctx, types.NamespacedName{Name: desiredDeployment.Name, Namespace: desiredDeployment.Namespace}, foundDeployment)
                if err != nil && errors.IsNotFound(err) {
                    // 4a. Create if not found
                    if err = r.Create(ctx, desiredDeployment); err != nil {
                        return ctrl.Result{}, err
                    }
                } else if err == nil {
                    // 4b. Update if found (if spec changed)
                    if !reflect.DeepEqual(foundDeployment.Spec, desiredDeployment.Spec) {
                        foundDeployment.Spec = desiredDeployment.Spec
                        if err = r.Update(ctx, foundDeployment); err != nil {
                            return ctrl.Result{}, err
                        }
                    }
                } else {
                    return ctrl.Result{}, err
                }

                // 5. Requeue periodically? (e.g., to check cron job status)
                return ctrl.Result{RequeueAfter: time.Minute}, nil
            }
            ```
    4.  **Build & Deploy Operator:**
        *   `make docker-build docker-push` (build/push operator image)
        *   `make deploy` (install RBAC, CRD, Operator Deployment)
*   **Why Operators?** Automate complex operational knowledge (e.g., DB backups, failover, scaling logic), provide domain-specific abstractions.
*   **Critical Gotchas:**
    *   **Reconciliation is NOT 1:1:** One CR event might trigger multiple reconciliations; multiple CR events might be batched. **Idempotency is mandatory.**
    *   **Watch Relevant Resources:** Controller must watch CR *and* resources it manages (e.g., Deployments) to react to external changes.
    *   **Error Handling:** Return errors to requeue; use `ctrl.Result` for timing.
    *   **RBAC:** Operator needs precise permissions (defined in `config/rbac/role.yaml`).
    *   **Start Simple:** Begin with basic CRUD before adding complex logic (backups, failover).

---

### **Chapter 64: Capstone Project - Multi-Tier Application**

This integrates **ALL** concepts from Chapter 63 into a cohesive, production-grade system.

#### **Core Architecture**
```
[User] --> [Ingress (Nginx/Traefik)] --> [Frontend (React)]
                                      |
                                      --> [API Gateway (Optional)] --> [Backend Service 1 (Node.js)]
                                      |                              --> [Backend Service 2 (Python)]
                                      |
                                      --> [Database (MongoDB StatefulSet)]
                                      |
                                      --> [Monitoring Stack (Prometheus/Grafana)]
                                      |
                                      --> [Logging Stack (Loki/Fluentd)]
```

#### **Deep Dive on Each Component**

1.  **Microservices:**
    *   **Implementation:** Separate `Deployments`/`StatefulSets` for each service (Frontend, User Service, Order Service, Payment Service).
    *   **Communication:** Service-to-Service via Kubernetes `Services` (ClusterIP). **Use gRPC/REST with retries/timeouts.**
    *   **Discovery:** DNS-based (`service-name.namespace.svc.cluster.local`).
    *   **Resilience:** Circuit breakers (via Service Mesh like Istio/Linkerd - *advanced*), retries with backoff.
    *   **Why?** Independent scaling, tech diversity, fault isolation, faster deployments.

2.  **Database (StatefulSet):**
    *   **Why StatefulSet?** Guaranteed ordering, stable network identities (`mongo-0.mongo`, `mongo-1.mongo`), persistent storage per pod (PVCs).
    *   **Critical Config:**
        *   `serviceName: mongo` (Headless Service name)
        *   `volumeClaimTemplates` for data persistence.
        *   Anti-affinity rules (`podAntiAffinity`) to spread pods across nodes.
        *   Init Containers for bootstrap/config (e.g., MongoDB replica set init).
        *   **Backup:** Sidecar container running `mongodump` to object storage (S3/GCS) on schedule.
    *   **Connection:** Backend services connect via `mongo-0.mongo:27017` (primary) or replica set URI.

3.  **Ingress + TLS:**
    *   **Ingress Controller:** Nginx or Traefik deployed as `Deployment` + `Service` (LoadBalancer).
    *   **Ingress Resource:**
        ```yaml
        apiVersion: networking.k8s.io/v1
        kind: Ingress
        metadata:
          name: main-ingress
          annotations:
            nginx.ingress.kubernetes.io/ssl-redirect: "true"
            cert-manager.io/cluster-issuer: "letsencrypt-prod" # For TLS
        spec:
          tls:
          - hosts:
            - myapp.example.com
            secretName: myapp-tls
          rules:
          - host: myapp.example.com
            http:
              paths:
              - path: /
                pathType: Prefix
                backend:
                  service:
                    name: frontend
                    port:
                      number: 80
              - path: /api
                pathType: Prefix
                backend:
                  service:
                    name: backend-api
                    port:
                      number: 3000
        ```
    *   **TLS Automation:** **cert-manager** (Operator) + Let's Encrypt (or internal CA).
        *   Installs CRDs (`Certificate`, `Issuer`).
        *   `Certificate` resource references `Ingress` host, triggers ACME challenge.
        *   Automatically renews certs.
    *   **Why?** Secure external access, path-based routing, centralized TLS termination.

4.  **Monitoring + Logging:**
    *   **Monitoring (Prometheus/Grafana):**
        *   `ServiceMonitors` for *every* microservice (expose `/metrics` via Prometheus client lib).
        *   Pre-built dashboards (e.g., Kubernetes Cluster, Node Exporter, custom app dashboards).
        *   **Critical Alerts:** `HighErrorRate`, `ServiceDown`, `HighLatency`, `LowMemory`, `PodCrashLooping`.
    *   **Logging (Loki + Promtail / Fluentd):**
        *   **Agent (Promtail/Fluentd):** DaemonSet collecting container logs (`/var/log/containers/*.log`), adding K8s metadata (namespace, pod, labels), shipping to Loki.
        *   **Loki:** Log aggregation backend (stores logs as time-series, indexed by labels).
        *   **Grafana:** Query logs via Loki datasource (`{namespace="prod", app="backend"} |= "error"`).
        *   **Why?** Troubleshoot issues faster, correlate metrics with logs, audit trails.

5.  **CI/CD + GitOps:**
    *   **Flow:**
        1.  Dev pushes code to feature branch -> PR -> GitHub Actions runs tests.
        2.  Merge to `main` -> GitHub Actions:
            *   Builds Docker images (tagged with SHA).
            *   Pushes images to registry.
            *   **Updates `k8s-manifests/prod/values.yaml`** (Helm) / `k8s-manifests/prod/backend/deployment.yaml` (Kustomize) with new image tag.
            *   Commits & pushes manifest change to `k8s-manifests` repo.
        3.  Argo CD detects manifest repo change.
        4.  Argo CD syncs cluster state to match manifest repo (deploying new image).
        5.  Argo CD reports sync status; health checks verify rollout.
    *   **Key Practices:**
        *   **Immutable Images:** Tag by SHA, not `latest`.
        *   **Signed Commits:** For manifest repo (provenance).
        *   **Canary Analysis:** Integrate metrics (see below) into Argo Rollouts.

6.  **Canary Deployment:**
    *   **What:** Gradually shift traffic from old version (Stable) to new version (Canary) while monitoring.
    *   **Implementation (Using Argo Rollouts):**
        *   **Install Argo Rollouts:** CRD-based controller (`rollout`, `analysis`).
        *   **Define Rollout (Instead of Deployment):**
            ```yaml
            apiVersion: argoproj.io/v1alpha1
            kind: Rollout
            metadata:
              name: backend
            spec:
              replicas: 5
              selector:
                matchLabels:
                  app: backend
              template:
                metadata:
                  labels:
                    app: backend
                spec:
                  containers:
                  - name: backend
                    image: backend:v1 # Stable version
              strategy:
                canary:
                  steps:
                  - setWeight: 20 # 20% traffic to canary
                  - pause: {} # Manual pause (or automated via analysis)
                  - setWeight: 50
                  - pause: {duration: 10m}
                  - setWeight: 100
                  trafficRouting:
                    nginx: # Or Istio, ALB, SMI
                      stableService: backend-stable
                      canaryService: backend-canary
                      ingress: main-ingress # Target Ingress
                  analysis:
                    templates:
                    - templateName: success-rate
                    args:
                    - name: service-name
                      value: backend
            ```
        *   **Analysis Template (Automated Promotion/Rollback):**
            ```yaml
            apiVersion: argoproj.io/v1alpha1
            kind: AnalysisTemplate
            metadata:
              name: success-rate
            spec:
              args:
              - name: service-name
              metrics:
              - name: success-rate
                interval: 5m
                # Query Prometheus for error rate
                successCondition: result[0] < 0.01 # <1% errors
                provider:
                  prometheus:
                    address: http://prometheus:9090
                    query: |
                      sum(rate(http_requests_total{service="{{args.service-name}}", code=~"5.."}[5m])) /
                      sum(rate(http_requests_total{service="{{args.service-name}}"}[5m]))
            ```
    *   **Why?** Reduce risk of bad releases, validate in production with minimal user impact.
    *   **Critical Gotchas:**
        *   Requires **traffic splitting** capability (Nginx Ingress w/ canary annotations, Istio, ALB).
        *   **Meaningful Metrics:** Success rate, latency, error rate are essential for analysis.
        *   **Rollback Automation:** Analysis must trigger rollback on failure (`measuredRollback`).
        *   **Session Affinity:** Breaks canary if sticky sessions are used (avoid if possible).

---
