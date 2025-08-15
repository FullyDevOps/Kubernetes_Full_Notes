### **Chapter 45: CI/CD with Kubernetes**

**Core Philosophy:** Automate building, testing, and deploying applications to Kubernetes *reliably and frequently*. Replace manual `kubectl` commands with pipelines triggered by code changes.

#### **1. Integrating CI/CD Tools (Jenkins, GitHub Actions, GitLab CI)**
*   **Why Integrate?** Kubernetes requires precise YAML manifests and image management. Manual deployments are error-prone and slow. CI/CD automates the pipeline from code commit to production.
*   **How Integration Works:**
    *   **Trigger:** Code push/PR to Git repo → Triggers CI/CD pipeline.
    *   **Pipeline Stages:**
        1.  **Checkout:** Fetch code from Git.
        2.  **Build:** Compile code, run unit tests.
        3.  **Build Image:** Create Docker container image (using `Dockerfile`).
        4.  **Scan Image:** (Critical!) Run security/vulnerability scans (e.g., Trivy, Clair).
        5.  **Push Image:** Push scanned image to Container Registry (Docker Hub, ECR, GCR, ACR).
        6.  **Deploy:** Update Kubernetes manifests to use the *new* image tag & apply changes.
        7.  **Test:** Run integration/end-to-end tests *against the deployed environment*.
        8.  **Promote:** (Optional) Move artifact through environments (dev → staging → prod).
    *   **Kubernetes Access:** CI/CD tools need **secure credentials** to interact with the cluster:
        *   **Service Account Tokens:** Best practice. Create a dedicated SA in a namespace (e.g., `ci-cd`) with *least privilege* RBAC (e.g., `role: edit` for dev, `role: view` for prod read-only). Store token securely (Secret Manager, Vault, CI/CD native secrets).
        *   **kubeconfig File:** Less secure. Contains cluster endpoint & token. *Must* be encrypted/secrets management.
        *   **Cloud IAM:** (EKS, GKE, AKS) Use workload identity federation (e.g., GitHub Actions OIDC + IAM Roles).

*   **Tool Comparison:**
    | **Tool**         | **Key Characteristics**                                                                 | **K8s Integration Strengths**                                                                 | **Weaknesses for K8s**                                   |
    | :--------------- | :------------------------------------------------------------------------------------ | :------------------------------------------------------------------------------------------ | :------------------------------------------------------- |
    | **Jenkins**      | - Mature, extensible (plugins)<br>- Master/Agent architecture<br>- Groovy DSL (`Jenkinsfile`) | - Huge plugin ecosystem (Kubernetes, Docker, Helm)<br>- Agents can run *in* K8s pods (dynamic)<br>- Great for complex pipelines | - Requires significant运维 (server management, scaling)<br>- Steeper learning curve<br>- Slower startup |
    | **GitHub Actions** | - Native to GitHub<br>- YAML-based workflows<br>- Free for public repos, usage-based pricing | - Seamless Git integration (triggers)<br>- Built-in secrets, OIDC for cloud IAM<br>- Huge marketplace (K8s, Helm, EKS)<br>- Runners can be self-hosted on K8s | - Vendor lock-in (GitHub)<br>- Limited runner control vs. Jenkins<br>- Complex workflows get messy |
    | **GitLab CI**    | - Deeply integrated with GitLab<br>- YAML-based (`.gitlab-ci.yml`)<br>- Built-in registry, security | - Single platform (code, CI, registry, security)<br>- Auto DevOps (pre-built K8s pipelines)<br>- Native K8s cluster integration<br>- Merge Request pipelines | - GitLab platform lock-in<br>- Resource-intensive (needs GitLab runner)<br>- Can be complex to scale |

*   **Critical Best Practices:**
    *   **Immutable Artifacts:** Build image *once*, push *once*, deploy *multiple times* (dev/staging/prod). Tag with Git SHA (`v1.2.3-abc123`).
    *   **Infrastructure as Code (IaC):** Store *all* Kubernetes manifests (Deployments, Services, ConfigMaps) in Git alongside app code.
    *   **Secrets Management:** **NEVER** hardcode secrets in manifests or pipeline config. Use K8s Secrets (backed by Vault/ECS SSM) or CI/CD-native secrets.
    *   **Idempotency:** Pipeline steps must be safe to run multiple times (e.g., `kubectl apply` vs `kubectl create`).

#### **2. Building and Pushing Images**
*   **The Process:**
    1.  **Dockerfile:** Defines image build (base OS, dependencies, app code, entrypoint).
    2.  **Multi-Stage Builds (ESSENTIAL):** Minimize final image size & attack surface.
        ```dockerfile
        # Stage 1: Build app
        FROM golang:1.22 AS builder
        WORKDIR /app
        COPY . .
        RUN go build -o myapp .

        # Stage 2: Runtime image
        FROM alpine:3.19
        WORKDIR /root/
        COPY --from=builder /app/myapp .
        CMD ["./myapp"]
        ```
    3.  **Build in CI/CD:** Pipeline runs `docker build -t my-registry/my-app:$GIT_SHA .`
    4.  **Scan:** `trivy image my-registry/my-app:$GIT_SHA` (fail pipeline on critical vulns).
    5.  **Push:** `docker push my-registry/my-app:$GIT_SHA`
*   **Registry Choices:**
    *   **Public:** Docker Hub (less secure, rate limits)
    *   **Cloud-Managed:** ECR (AWS), GCR (GCP), ACR (Azure) - Tight IAM integration, private by default, optimized for cloud K8s.
    *   **On-Prem:** Harbor (adds vulnerability scanning, replication, chart museum).
*   **Critical Gotchas:**
    *   **Tagging Strategy:** Avoid `:latest` in prod! Use commit SHA (`:abc123`) or semantic version (`:v1.2.3`). Enables traceability & rollbacks.
    *   **Registry Auth:** CI/CD needs `docker login` credentials (stored as secrets).
    *   **Image Size:** Smaller = faster pull, less attack surface. Alpine base images are common.

#### **3. Deploying to Kubernetes (Helm, Kustomize, kubectl)**
*   **The Goal:** Update the cluster state to match the desired state defined by manifests using the *new* image tag.
*   **Methods Compared:**
    | **Method**       | **How it Works**                                                                 | **Pros**                                                                 | **Cons**                                                                 | **Best For**                                  |
    | :--------------- | :------------------------------------------------------------------------------- | :----------------------------------------------------------------------- | :------------------------------------------------------------------------ | :-------------------------------------------- |
    | **`kubectl apply`** | Directly apply YAML manifests (`kubectl apply -f deployment.yaml`)               | - Simple<br>- Native K8s command<br>- Immediate feedback                 | - Hard to manage many files<br>- No templating<br>- No versioning<br>- Risk of manual drift | Simple apps, quick testing, learning          |
    | **Helm**         | Package manager. Charts = templates + values. `helm upgrade --install myapp ./chart -f values-prod.yaml` | - Rich templating (Go templates)<br>- Versioned releases (`helm history`)<br>- Reusable charts<br>- Huge public repo (Artifact Hub) | - Learning curve<br>- "Tiller" legacy (v2)<br>- Complex charts can be opaque | Complex apps, reusable components, enterprises |
    | **Kustomize**    | Overlay-based. Base manifests + Patches (dev/staging/prod). `kustomize build overlays/prod | kubectl apply -f -` | - No templating (YAML patching)<br>- Git-friendly (pure YAML)<br>- Native to `kubectl` (`kubectl apply -k`)<br>- Simple for env variants | Teams wanting GitOps simplicity, env variants | 

*   **Deployment Workflow in CI/CD:**
    1.  **Update Manifest Source:**
        *   *Helm:* Update `values.yaml` (or env-specific values file) with new image tag. Commit to Git (or pass directly to `helm upgrade`).
        *   *Kustomize:* Update `kustomization.yaml` `images:` field or a patch file with new tag. Commit to Git (or generate on fly in pipeline).
        *   *kubectl:* Generate updated `deployment.yaml` with new tag (e.g., `sed`), commit to Git.
    2.  **Apply Changes:** Pipeline runs the deploy command (`helm upgrade`, `kustomize build | kubectl apply`, `kubectl apply -f`).
    3.  **Verify:** Pipeline checks rollout status (`kubectl rollout status`, Helm tests).

*   **Critical Considerations:**
    *   **Rolling Updates:** Default K8s Deployment strategy (`strategy.type: RollingUpdate`). Controlled by `maxSurge`/`maxUnavailable`.
    *   **Readiness Probes:** **MUST** be configured! Ensures traffic only goes to healthy pods *after* startup.
    *   **Liveness Probes:** Kills unhealthy pods. Prevents "zombie" containers.
    *   **Resource Requests/Limits:** Prevent resource starvation. Essential for cluster stability.

#### **4. Canary Deployments with Argo Rollouts**
*   **What is Canary?** Gradually shift traffic from old version (v1) to new version (v2) to a small subset of users. Monitor metrics (errors, latency). If good, shift 100% traffic. If bad, abort and rollback.
*   **Why Argo Rollouts?** Native K8s Deployments only support basic rolling updates. Argo Rollouts adds:
    *   Canary, Blue/Green, A/B Testing strategies.
    *   Progressive traffic shifting (Istio, SMI, ALB, Nginx Ingress).
    *   Automated analysis (metrics checks) for promotion/rollback.
    *   Manual gating (pause for QA sign-off).
*   **How it Works (Simplified):**
    1.  Define `Rollout` resource (replaces `Deployment`).
    2.  Configure canary steps:
        ```yaml
        strategy:
          canary:
            steps:
            - setWeight: 5   # 5% traffic to v2
            - pause: {}      # Pause for 5m (or manual approval)
            - setWeight: 20  # 20% traffic
            - pause: {duration: 10m} # Analyze metrics
            - setWeight: 100
        ```
    3.  **Traffic Shifting:** Requires a service mesh (Istio) or ingress controller (ALB, Nginx) that supports weighted routing. Argo Rollouts updates the routing rules.
    4.  **Analysis (Key Feature):** Define metrics (Prometheus, Datadog, Wavefront) to check *during pauses*:
        ```yaml
        analysis:
          templates:
          - templateName: success-rate
          args:
          - name: service-name
            value: myapp
        ```
        *   If metrics fail threshold → **Automatic rollback**.
        *   If metrics pass → **Automatic promotion** to next step.
    5.  **Abort & Rollback:** `kubectl argo rollouts abort myapp` instantly reverts traffic to stable version (v1).
*   **Benefits:** Drastically reduces risk of bad deployments. Enables safe experimentation.

#### **5. Blue/Green Deployments**
*   **What is Blue/Green?** Maintain two *identical* production environments ("Blue" = current live, "Green" = new version). Test Green *fully* (including smoke tests). Flip a switch (e.g., update Service selector) to route *all* traffic to Green instantly. Blue becomes the new standby.
*   **Implementation in K8s:**
    1.  Deploy new version (`v2`) alongside old version (`v1`). They have different labels (e.g., `version: v1`, `version: v2`).
    2.  Point a **testing Service** to `v2` for smoke/integration tests.
    3.  **The Flip:** Update the *production* Service's `selector` to point to `v2` labels.
        ```yaml
        apiVersion: v1
        kind: Service
        metadata:
          name: myapp-prod
        spec:
          selector:
            app: myapp
            # version: v1   # Old version (Blue)
            version: v2   # New version (Green) - CHANGE THIS
          ports:
            - port: 80
        ```
    4.  **Instant Cutover:** Traffic switches from Blue to Green in seconds (DNS TTL irrelevant).
    5.  **Rollback:** Change Service selector back to `version: v1`.
*   **Comparison vs Canary:**
    | **Feature**       | **Blue/Green**                          | **Canary**                                |
    | :---------------- | :-------------------------------------- | :---------------------------------------- |
    | **Traffic Shift** | Instant (100% flip)                     | Gradual (5% → 10% → ... → 100%)           |
    | **Risk**          | High per flip (all users affected)      | Low per step (only % affected)            |
    | **Testing**       | Full pre-production test *before* flip  | Real-user testing *during* rollout        |
    | **Complexity**    | Simpler networking (no weighted routing)| Requires service mesh/advanced ingress    |
    | **Rollback Time** | Very fast (seconds)                     | Fast (Argo Rollouts abort)                |
    | **Best For**      | Critical apps needing zero-downtime flip| Apps needing real-user validation; lower risk tolerance |
*   **Key Requirement:** Double the infrastructure capacity (for the duration of the deployment).

---

### **Chapter 46: GitOps**

**Core Philosophy:** **Git is the single source of truth for declarative infrastructure and applications.** *Automated* systems continuously reconcile the *actual* cluster state with the *desired* state defined in Git.

#### **1. What is GitOps?**
*   **Not Just CI/CD:** CI/CD *pushes* changes *to* the cluster (Pipeline → Cluster). GitOps *pulls* changes *from* Git (Cluster → Git).
*   **The GitOps Loop:**
    1.  **Desired State:** Defined in Git repo (manifests, Helm charts, Kustomize overlays).
    2.  **Reconciliation Agent:** (Argo CD/Flux) Runs *inside* the cluster.
    3.  **Agent Action:** Constantly compares *live cluster state* vs *Git state*.
    4.  **Auto-Apply:** If drift is detected, agent *automatically* applies Git state to cluster (using `kubectl apply`, Helm, Kustomize).
*   **Why GitOps?**
    *   **Auditability:** Every change is a Git commit (who, what, when, why).
    *   **Reproducibility:** `git checkout <commit>` = exact cluster state at that time.
    *   **Security:** No external systems need cluster credentials (agent runs inside with least-privilege SA).
    *   **Reliability:** Automated reconciliation fixes manual drift (e.g., someone `kubectl edit`).
    *   **Simplicity:** Standardize deployments across teams/environments via Git workflows.

#### **2. Principles: Declarative, Versioned, Automated**
*   **Declarative:** Define *what* the system should look like (YAML manifests), not *how* to achieve it (imperative commands like `kubectl scale`). K8s is inherently declarative; GitOps embraces this fully.
*   **Versioned:** All desired states are stored in Git. Every change is tracked, reviewed (PRs), and versioned. Enables:
    *   `git blame` for config changes.
    *   `git diff` to see environment differences.
    *   Rollback via `git revert` or `git checkout`.
*   **Automated:** The reconciliation loop (detect drift → apply fix) runs *automatically* without human intervention. Humans interact *only* via Git (PRs, commits).

#### **3. Tools: Argo CD vs Flux**
*   **Argo CD:**
    *   **Focus:** GitOps *Continuous Delivery* (CD) for K8s. Strong UI.
    *   **Core Concept:** **Application** CRD. Represents a deployed app (points to Git repo/path, target namespace, sync policy).
    *   **How it Works:**
        1.  Define `Application` manifest pointing to your Git repo (e.g., `https://github.com/your/repo, path: apps/prod/myapp`).
        2.  Argo CD controller polls Git repo.
        3.  Compares repo state vs cluster state.
        4.  **Sync:** Applies Git state to cluster (manually or auto).
        5.  **UI:** Shows sync status, health, diff view, rollback history.
    *   **Strengths:** Excellent UI/UX, strong RBAC, multi-cluster support, sync waves (order deployments), hooks (pre/post sync), drift detection visualization.
    *   **Weaknesses:** Slightly heavier resource usage, UI is primary interface (though CLI exists).
*   **Flux (v2 - GitOps Toolkit):**
    *   **Focus:** GitOps *operator* toolkit. Composable controllers. CLI-centric.
    *   **Core Concepts (CRDs):**
        *   `GitRepository`: Where is the source config?
        *   `Kustomization`: How to build/apply manifests (Kustomize, Helm, YAML)?
        *   `HelmRepository`/`HelmRelease`: For Helm charts.
        *   `ImageRepository`/`ImagePolicy`: For automated image updates.
    *   **How it Works:**
        1.  Install Flux controllers (`flux install`).
        2.  Create `GitRepository` pointing to your config repo.
        3.  Create `Kustomization` referencing the repo & path, defining sync interval.
        4.  Flux sources repo, builds manifests, applies to cluster.
    *   **Strengths:** Lightweight, composable (only install needed components), strong automation (image updates), Git-based automation, excellent for infrastructure teams.
    *   **Weaknesses:** Steeper learning curve (CRDs), less intuitive UI (uses `flux` CLI primarily), multi-cluster management less visual than Argo CD.
*   **Argo CD vs Flux Comparison:**
    | **Feature**           | **Argo CD**                                      | **Flux (v2)**                                     |
    | :-------------------- | :----------------------------------------------- | :------------------------------------------------ |
    | **Primary Interface** | Web UI (very strong)                             | CLI (`flux`) + Limited UI (Weave GitOps)          |
    | **Core Model**        | `Application` CRD                                | Composable CRDs (`GitRepository`, `Kustomization`) |
    | **Learning Curve**    | Easier for app teams (UI-driven)                 | Steeper (CRD configuration)                       |
    | **Image Automation**  | Requires Argo CD Image Updater (separate)        | Built-in (`ImageRepository`, `ImagePolicy`)       |
    | **Multi-Cluster**     | Native UI support                                | Requires Flux instance per cluster or management  |
    | **Best For**          | App teams, enterprises needing UI/RBAC           | Infrastructure teams, automation-focused, CLI fans |

#### **4. Sync Modes (Auto, Manual)**
*   **Manual Sync:**
    *   **How:** Argo CD detects drift → Shows "OutOfSync" in UI → User clicks "Sync" button (or runs `argocd app sync myapp`).
    *   **When to Use:**
        *   Production environments (requires human approval).
        *   Critical deployments needing final verification.
        *   Teams new to GitOps (more control).
    *   **Process:** PR merged → Argo CD detects change → Notification → Manual sync after checks.
*   **Automatic Sync:**
    *   **How:** Argo CD/Flux automatically applies changes from Git to the cluster *as soon as* they are detected (within sync interval).
    *   **When to Use:**
        *   Development/Staging environments.
        *   Low-risk applications.
        *   Teams with high confidence in testing/automation.
    *   **Critical Safeguards:**
        *   **Prune Resources:** Delete resources in cluster *not* in Git? (`Prune` option - **ENABLE IN PROD!**)
        *   **Self-Healing:** Auto-revert manual cluster changes (drift).
        *   **Sync Windows:** Only auto-sync during maintenance windows.
        *   **Automated Tests:** *Must* have robust pre-merge tests (CI pipeline) to prevent bad commits syncing.
*   **Sync Options (Argo CD Specific):**
    *   **Prune:** Delete resources present in cluster but absent from Git manifests. **Crucial for hygiene!**
    *   **Apply Out of Sync Only:** Only apply resources that are out of sync (faster).
    *   **Replace Resources:** Use `kubectl replace` semantics (vs `apply`).
    *   **Force:** Ignore resource conflicts (use cautiously).

#### **5. Rollback and Drift Detection**
*   **Drift Detection:**
    *   **What is Drift?** When the *live cluster state* differs from the *desired state in Git*.
    *   **Causes:** Manual `kubectl` commands, direct API calls, failed deployments, external operators modifying resources.
    *   **How GitOps Tools Detect Drift:**
        1.  **Reconciliation Loop:** Agent constantly compares cluster state (via K8s API) vs Git state (pulled repo).
        2.  **Hashing/Annotation:** Tools often store a hash of the applied manifest in the resource's annotation (`last-applied-configuration`). Changes to live resource break the hash.
        3.  **UI Indication:** Argo CD shows "OutOfSync" resources; Flux reports via `flux get kustomizations`.
*   **Rollback:**
    *   **GitOps Rollback = Git Revert:** The *only* action needed is to revert the Git commit that caused the bad state.
        1.  Identify bad commit (via Git history, Argo CD history, monitoring).
        2.  `git revert <bad-commit>` → Push to Git.
        3.  **Auto-Sync:** GitOps tool automatically detects the revert commit → Applies the *previous good state* from Git → Cluster rolls back.
        4.  **Manual-Sync:** Operator sees "OutOfSync" → Clicks "Sync" → Reverts cluster to last good Git state.
    *   **Why it's Superior:**
        *   **Speed:** Rollback time = Git commit time + reconciliation interval (seconds).
        *   **Reliability:** Uses the *exact same process* as deployment. No separate rollback scripts.
        *   **Auditability:** Rollback is a Git commit (`revert "Fix critical bug"`).
    *   **Argo CD Specific:** Has a built-in "History and Rollback" view showing all syncs. Can rollback to any previous sync revision with one click (which creates a Git revert commit under the hood if using auto-sync, or applies the old manifest directly).

---
