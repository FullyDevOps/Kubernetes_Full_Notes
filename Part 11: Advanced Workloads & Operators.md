### **Chapter 39: Custom Resources & CRDs**

#### **1. Extending the Kubernetes API: The *Why* and *How***
*   **The Problem**: Kubernetes core resources (`Pod`, `Service`, `Deployment`) are generic. Real-world applications (databases, ML pipelines, custom networking) need domain-specific logic.
*   **The Solution**: Kubernetes is designed as a **platform for building platforms**. Its API is **extensible** via:
    *   **Aggregated API Servers**: Full custom API servers (complex, e.g., `kube-aggregator`).
    *   **Custom Resource Definitions (CRDs)**: **The primary, simpler method** to add *declarative* resources without modifying Kubernetes core code.
*   **How CRDs Work**:
    *   You define a **CRD** (a Kubernetes resource itself) describing your new resource type (e.g., `Database`, `IngressRoute`).
    *   The **kube-apiserver** dynamically adds this new resource type to the API.
    *   Users interact with your new resource (`kubectl get databases`, `kubectl create -f mydb.yaml`) **exactly like built-in resources**.
    *   **CRDs are *declarative*:** They define the *desired state*. *Controllers* (covered in Operators) make reality match that state.

#### **2. Creating and Using CRDs: Anatomy & Lifecycle**
*   **CRD Definition (YAML Structure)**:
    ```yaml
    apiVersion: apiextensions.k8s.io/v1
    kind: CustomResourceDefinition
    metadata:
      name: databases.example.com # Format: <plural>.<group>
    spec:
      group: example.com          # API group (e.g., databases.example.com)
      names:
        plural: databases         # Plural name (used in URL: /apis/example.com/v1/databases)
        singular: database        # Singular name (kubectl get database)
        kind: Database            # Resource type (kubectl get databases -> list of Database objects)
        shortNames: [db]          # kubectl aliases (kubectl get db)
      scope: Namespaced           # Or "Cluster" (Cluster-scoped CRD)
      versions:
        - name: v1alpha1          # Version (semver-like)
          served: true            # API server serves this version
          storage: true           # This version is the *storage version* (single true per CRD)
          schema:
            openAPIV3Schema:      # CRITICAL: Validation schema (see next section)
              type: object
              properties:
                spec:
                  type: object
                  properties:
                    engine:
                      type: string
                      enum: [mysql, postgres]
          subresources:
            status: {}            # Enables /status subresource (MUST for Operators)
            scale: {}             # Enables /scale subresource (for HPA-like scaling)
    ```
*   **Key CRD Fields Explained**:
    *   **`group`**: Namespace for your resource (e.g., `databases.example.com`). Avoids naming conflicts.
    *   **`versions`**: List of API versions for this CRD (e.g., `v1alpha1`, `v1beta1`, `v1`). Crucial for evolution.
    *   **`storage: true`**: **Exactly one version** must be the storage version. Data is persisted in etcd in this format.
    *   **`subresources.status`**: **MANDATORY for Operators**. Allows controllers to update `.status` independently of `.spec` (avoids write conflicts).
    *   **`scope`**: `Namespaced` (default, exists within a namespace) or `Cluster` (single instance cluster-wide, e.g., `ClusterIngress`).
*   **Creating a CRD**:
    ```bash
    kubectl apply -f database-crd.yaml
    ```
    *   `kubectl get crd` shows all CRDs.
    *   `kubectl describe crd databases.example.com` shows details/schema.
*   **Using a Custom Resource (CR)**:
    ```yaml
    # my-postgres.yaml
    apiVersion: example.com/v1alpha1
    kind: Database
    metadata:
      name: my-postgres
      namespace: prod
    spec: # Desired state (defined by YOU)
      engine: postgres
      version: "14.5"
      storageGB: 100
      replicas: 3
    ```
    ```bash
    kubectl apply -f my-postgres.yaml
    kubectl get databases -n prod
    kubectl describe database my-postgres -n prod
    ```
    *   **CRs are inert**: Creating a CR *does nothing* by itself. It's just data. **Controllers (Operators) react to CRs**.

#### **3. Validation with OpenAPI Schema: Ensuring Correctness**
*   **Why?**: Prevent invalid CRs from being stored (e.g., `engine: "oracle"` when only `mysql/postgres` allowed). Critical for reliability.
*   **Where?**: Defined in `spec.versions[].schema.openAPIV3Schema` within the CRD.
*   **Core Validation Concepts**:
    *   **`type`**: `object`, `string`, `integer`, `number`, `boolean`, `array`, `null`.
    *   **`properties`**: Schema for child fields (for `type: object`).
    *   **`required`**: Mandatory fields *within this object*.
    *   **`enum`**: Allowed string values.
    *   **`minimum`/`maximum`**: For numbers.
    *   **`pattern`**: Regex validation for strings.
    *   **`items`**: Schema for array elements.
    *   **`anyOf`/`allOf`/`oneOf`**: Complex logical combinations.
*   **Critical Nuances**:
    *   **`x-kubernetes-preserve-unknown-fields: true`**: (Deprecated in v1 CRDs) Allows unknown fields. **Avoid if possible**; breaks compatibility. Prefer `x-kubernetes-embedded-resource: true` for nested objects like `PodSpec`.
    *   **`nullable: true`**: Explicitly allows `null` values (default: `false`).
    *   **`x-kubernetes-int-or-string: true`**: Allows a field to be either an integer or a string (e.g., `replicas`).
    *   **`x-kubernetes-validations`**: (v1.16+) **Powerful CEL-based validations** for complex rules:
        ```yaml
        x-kubernetes-validations:
          - rule: "self.storageGB > 10"
            message: "Storage must be at least 10GB"
          - rule: "self.engine == 'postgres' ? self.version.startsWith('14.') : true"
            message: "Postgres requires version 14.x"
        ```
*   **Example Schema Snippet**:
    ```yaml
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            required: [engine, storageGB] # MUST have these
            properties:
              engine:
                type: string
                enum: [mysql, postgres]    # Only these values
              storageGB:
                type: integer
                minimum: 10                # Min 10
              version:
                type: string
                pattern: '^\d+\.\d+\.\d+$' # Semantic version
              resources:
                x-kubernetes-embedded-resource: true
                x-kubernetes-preserve-unknown-fields: true
                type: object               # Allows nested Pod/Container resource requests
        required: [spec]                   # Top-level spec is mandatory
    ```
*   **Testing Validation**:
    *   `kubectl apply -f invalid-cr.yaml` will fail *before* reaching etcd if validation fails.
    *   `kubectl explain database --recursive` shows the schema structure.

#### **4. Versioning CRDs: Evolving Your API Safely**
*   **Why Version?**: Your resource definition *will* change (new fields, deprecated fields, structural changes). Versioning ensures backward compatibility.
*   **CRD Versioning vs. Resource Versioning**:
    *   **CRD Versioning**: Managed by the `versions` array in the CRD itself. Defines *API surface* for the CRD.
    *   **Resource Versioning**: How individual CR *instances* are stored/converted (handled via `storageVersion` and conversion).
*   **Key Strategies**:
    *   **Additive Changes (Safe)**:
        *   Add new optional fields to `v1beta1` schema.
        *   Set `served: true` for `v1beta1`, keep `v1alpha1` served but deprecated.
        *   Old clients still use `v1alpha1`; new clients use `v1beta1`.
    *   **Breaking Changes (Require Conversion)**:
        *   Field renamed (`oldField` -> `newField`), type changed (`string` -> `integer`).
        *   Requires **Webhook Conversion** (v1 CRDs).
*   **Webhook Conversion (v1 CRDs - Essential for Breaking Changes)**:
    1.  Define multiple versions in CRD (`v1alpha1`, `v1beta1`).
    2.  Mark one as `storage: true` (e.g., `v1beta1`).
    3.  Deploy a **Conversion Webhook** (a Kubernetes Service + Deployment).
    4.  When a client requests `v1alpha1`:
        *   kube-apiserver calls your webhook: "Convert this `v1beta1` object to `v1alpha1`".
        *   Webhook returns converted object.
        *   kube-apiserver sends `v1alpha1` to client.
    5.  When a client submits `v1alpha1`:
        *   kube-apiserver calls webhook: "Convert this `v1alpha1` object to `v1beta1`".
        *   Webhook returns converted object.
        *   kube-apiserver stores `v1beta1` in etcd.
    *   **CRD Definition Snippet**:
        ```yaml
        versions:
          - name: v1alpha1
            served: true
            storage: false
            schema: ... # Old schema
          - name: v1beta1
            served: true
            storage: true
            schema: ... # New schema
        conversion:
          strategy: Webhook
          webhook:
            clientConfig:
              service:
                namespace: operators
                name: my-conversion-webhook
                path: /convert
            conversionReviewVersions: ["v1"]
        ```
*   **Best Practices**:
    *   Start with `v1alpha1` -> `v1beta1` -> `v1`.
    *   **Never** change a field's meaning in a minor version (e.g., `v1beta1` -> `v1beta2`).
    *   Deprecate fields *before* removal (add warnings in docs/schema).
    *   Use conversion webhooks for *any* structural change.
    *   **Always** have `status` subresource enabled from day one.

---

### **Chapter 40: Operators**

#### **1. What is an Operator?**
*   **Definition**: An **Operator is a method of packaging, deploying, and managing a Kubernetes application**. It's a **Kubernetes controller** that **encapsulates human operational knowledge** for a specific application into code.
*   **Core Idea**: Automate complex, stateful application management tasks that are typically done manually by SREs/DBAs (e.g., backups, scaling, failover, version upgrades).
*   **Key Characteristics**:
    *   **Domain-Specific**: Built for *one specific application* (e.g., PostgreSQL, Cassandra, Prometheus).
    *   **Controller Pattern**: Implements the reconcile loop (see below).
    *   **CRD-Driven**: Uses Custom Resources as the user-facing API (e.g., `kind: PostgreSQL`).
    *   **Idempotent**: Can be run repeatedly; only acts if reality != desired state.
    *   **Self-Healing**: Detects and corrects deviations (e.g., restarts failed pods).
*   **Why Operators?**:
    *   **Complex Stateful Apps**: Databases, message queues, AI/ML platforms need nuanced management.
    *   **Consistency**: Enforce best practices across environments.
    *   **Reduce Toil**: Automate repetitive, error-prone tasks.
    *   **GitOps Friendly**: Desired state defined in CRs (YAML), fits CI/CD pipelines.

#### **2. Operator Pattern: Controller + CRD (The Engine)**
*   **The Pattern**: An Operator = **CRD (API)** + **Controller (Logic)**.
    *   **CRD**: Defines the *desired state* of the application (e.g., `spec.replicas: 5`, `spec.version: "14.5"`).
    *   **Controller**: Watches the CRD and **reconciles** the actual cluster state to match the desired state.
*   **The Reconcile Loop (Heart of the Operator)**:
    ```go
    func (r *Reconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
        // 1. FETCH: Get the current CR (e.g., Database)
        db := &examplev1.Database{}
        if err := r.Get(ctx, req.NamespacedName, db); err != nil {
            return ctrl.Result{}, client.IgnoreNotFound(err)
        }

        // 2. COMPARE: Desired State (db.Spec) vs. Actual State (Cluster)
        //   - Check if StatefulSet exists? Correct size?
        //   - Check if Secrets exist? Correct credentials?
        //   - Check if Service exists? Correct ports?
        //   - Check if database pods are healthy?

        // 3. ACT: Make reality match desired state
        //   IF desired replicas != actual replicas -> Scale StatefulSet
        //   IF desired version != actual version -> Perform upgrade
        //   IF backup needed -> Trigger backup job
        //   IF pod unhealthy -> Replace pod

        // 4. UPDATE STATUS: Report actual state (CRITICAL!)
        db.Status.ReadyReplicas = 3
        db.Status.Phase = "Running"
        r.Status().Update(ctx, db)

        // 5. SCHEDULE NEXT RECONCILE (Optional)
        return ctrl.Result{RequeueAfter: 5 * time.Minute}, nil
    }
    ```
*   **Key Reconciler Principles**:
    *   **Idempotency**: Must handle being called repeatedly safely.
    *   **Declarative**: Focus on *what*, not *how*. Don't track state between runs.
    *   **Graceful Failure**: Return errors to retry later; don't crash.
    *   **Status Updates**: **ALWAYS update `.status`** to reflect observed state (visible via `kubectl get`).
    *   **Finalizers**: (For cleanup) Add `metadata.finalizers` to CR to block deletion until external resources (e.g., cloud DB) are cleaned up.
*   **Watched Resources**: The controller watches:
    *   Its primary CR (e.g., `Database`)
    *   **Secondary Resources** it creates (e.g., `StatefulSet`, `Service`, `Secret`). If these change, it triggers reconciliation.

#### **3. Operator SDK / Kubebuilder: Building Operators Made Easier**
*   **The Problem**: Writing controllers from scratch (using `client-go`) is complex and boilerplate-heavy.
*   **The Solution**: Frameworks generating scaffolding and handling boilerplate.
*   **Kubebuilder (CNCF Standard)**:
    *   **Focus**: Pure Kubernetes, Go-based, opinionated structure.
    *   **Core Tools**:
        *   `kubebuilder init`: Sets up project.
        *   `kubebuilder create api`: Generates CRD + Controller stubs.
        *   Uses **Controller Runtime** library (lower-level than Operator SDK).
    *   **Pros**: Lightweight, direct control, strong community (used by core Kubernetes projects).
    *   **Cons**: Steeper learning curve, Go-only.
*   **Operator SDK (Red Hat / CNCF)**:
    *   **Focus**: Higher-level abstractions, supports multiple languages (Go, Ansible, Helm).
    *   **Core Tools**:
        *   `operator-sdk init`: Sets up project.
        *   `operator-sdk create api`: Generates CRD + Controller.
        *   **Bundles**: Packaging for Operator Lifecycle Manager (OLM).
    *   **Language Options**:
        *   **Go**: Uses Kubebuilder/Controller Runtime under the hood. Most powerful.
        *   **Ansible**: Define reconciliation logic in Ansible playbooks. Great for config-driven tasks.
        *   **Helm**: Manage application via Helm charts. Simplest for existing Helm users.
    *   **Pros**: Easier onboarding (especially Ansible/Helm), OLM integration, multi-language.
    *   **Cons**: Go version less "bare metal" than Kubebuilder; Ansible/Helm less flexible for complex logic.
*   **Which to Choose?**
    *   **Complex Logic / Max Performance**: Kubebuilder (Go) or Operator SDK (Go).
    *   **Config Management / Existing Ansible**: Operator SDK (Ansible).
    *   **Simple Helm Charts**: Operator SDK (Helm).
    *   **Enterprise Deployment (OLM)**: Operator SDK (Go/Ansible/Helm bundles).

#### **4. Building an Operator: Database Operator Example (Step-by-Step)**
*   **Goal**: Automate PostgreSQL cluster creation, scaling, backups, and failover.
*   **CRD Design (`Database`)**:
    ```yaml
    apiVersion: db.example.com/v1beta1
    kind: Database
    metadata:
      name: my-cluster
    spec:
      engine: postgres
      version: "14.5"
      replicas: 3
      storageGB: 100
      backupSchedule: "0 2 * * *" # Daily at 2 AM
      monitoring: true
    status:
      phase: Creating # Creating, Running, Failed, Upgrading
      readyReplicas: 0
      lastBackup: "2023-10-27T02:00:00Z"
    ```
*   **Operator Steps**:
    1.  **Install CRD**: `kubectl apply -f database-crd.yaml` (with `status` subresource!).
    2.  **Deploy Operator**: `kubectl apply -f operator.yaml` (Deployment running controller logic).
    3.  **Create CR**: `kubectl apply -f my-cluster.yaml`.
    4.  **Reconcile Loop Execution**:
        *   **Phase 1: Initial Creation**:
            *   Check if CR exists -> Yes (`my-cluster`).
            *   Check if `StatefulSet` exists -> No.
            *   **Action**: Create `StatefulSet` (3 pods), `Service`, `ConfigMap` (config), `Secret` (passwords).
            *   Update `status.phase: "Creating"`.
        *   **Phase 2: Running & Monitoring**:
            *   Check `StatefulSet` replicas == `spec.replicas` -> Yes (3).
            *   Check pod health -> All healthy.
            *   Check `status.readyReplicas` -> 3. Update `status`.
            *   Check `backupSchedule` -> Time for backup? Yes.
            *   **Action**: Create `CronJob` for backup. Update `status.lastBackup`.
        *   **Phase 3: Scaling Event**:
            *   User updates `spec.replicas: 5`.
            *   CR changes -> Reconcile triggered.
            *   Check `StatefulSet` replicas == 5 -> No (3).
            *   **Action**: Patch `StatefulSet` replicas to 5. Update `status.phase: "Scaling"`.
            *   Wait for new pods to be ready -> Update `status.readyReplicas: 5`, `phase: "Running"`.
        *   **Phase 4: Failure Recovery**:
            *   One pod crashes (`kubectl delete pod my-cluster-2`).
            *   `StatefulSet` controller creates new pod -> Reconcile triggered (watched resource changed).
            *   Check pod health -> New pod initializing.
            *   **Action**: Wait for pod to join cluster (check DB replication). Update `status.readyReplicas`.
    5.  **Deletion**:
        *   User runs `kubectl delete database my-cluster`.
        *   Operator adds finalizer (`deletion-protection.db.example.com`).
        *   Reconcile loop triggered:
            *   Checks if cloud resources exist (e.g., AWS RDS snapshot) -> Yes.
            *   **Action**: Trigger cloud DB deletion/snapshot cleanup.
            *   Once cleanup done, remove finalizer -> CR deleted.
*   **Critical Operator Components**:
    *   **RBAC**: Precise permissions (e.g., `get/list/watch` CRs, `create/update/delete` StatefulSets, Secrets).
    *   **Leader Election**: For HA (multiple operator replicas).
    *   **Metrics**: Expose Prometheus metrics (reconcile duration, errors).
    *   **Logging**: Structured logs for debugging.
    *   **Testing**: Unit tests (reconciler logic), integration tests (real cluster).

#### **5. Community Operators (etcd, Prometheus, etc.): Leveraging the Ecosystem**
*   **Why Use Them?** Avoid reinventing the wheel. Battle-tested, maintained by experts.
*   **Key Examples**:
    *   **etcd Operator (CoreOS)**:
        *   **CRD**: `EtcdCluster`
        *   **Features**: Automated cluster creation, scaling, backup/restore, version upgrades, disaster recovery.
        *   **How it Works**: Manages `StatefulSet` of etcd pods, handles member addition/removal, configures TLS.
    *   **Prometheus Operator (CoreOS)**:
        *   **CRDs**: `Prometheus`, `Alertmanager`, `ServiceMonitor`, `PodMonitor`, `PrometheusRule`.
        *   **Features**:
            *   `ServiceMonitor`/`PodMonitor`: Auto-generate Prometheus scrape configs based on Kubernetes labels.
            *   `Prometheus`: Manages Prometheus server deployment/config.
            *   `Alertmanager`: Manages Alertmanager cluster.
            *   `PrometheusRule`: Define alerting/recording rules declaratively.
        *   **Revolutionized Monitoring**: Made Kubernetes-native monitoring trivial.
    *   **Other Notable Operators**:
        *   **Strimzi (Kafka)**: Full Apache Kafka cluster management.
        *   **RabbitMQ Cluster Operator**: Manages RabbitMQ clusters.
        *   **Kubeflow Operators**: For ML pipelines (TFJob, PyTorchJob).
        *   **Camel K**: Integrations on Kubernetes.
        *   **Elastic Cloud on Kubernetes (ECK)**: Elasticsearch, Kibana, APM.
*   **Finding Operators**:
    *   **OperatorHub.io**: Central catalog (works with OLM).
    *   **Kubernetes SIGs**: `sig-apps` often has references.
    *   **Vendor Sites**: (e.g., Red Hat OpenShift Container Platform includes many).
*   **Installing Community Operators**:
    1.  **Operator Lifecycle Manager (OLM)** (Preferred for production):
        *   Install OLM: `kubectl apply -f https://github.com/operator-framework/operator-lifecycle-manager/releases/...`
        *   Create `CatalogSource` (points to OperatorHub repo).
        *   Create `Subscription` (e.g., `name: prometheus-operator`).
        *   OLM handles install, updates, dependencies.
    2.  **Manual Installation**:
        *   `kubectl apply -f <operator-manifests-dir>` (CRDs + RBAC + Deployment).
        *   Simpler, but no automated updates.

---

### **Critical Best Practices & Pitfalls (Must-Know!)**

1.  **CRDs**:
    *   **ALWAYS enable `status` subresource** (`subresources: {status: {}}`). Without it, controllers *cannot* safely update status without conflicting with spec updates.
    *   **Validate rigorously**. Invalid CRs cause operator crashes. Use `x-kubernetes-validations` (CEL) for complex rules.
    *   **Design for versioning from day one**. Plan your `v1alpha1` -> `v1beta1` -> `v1` path. Use conversion webhooks early.
    *   **Avoid `preserveUnknownFields: true`** where possible. It breaks compatibility and tooling.

2.  **Operators**:
    *   **Idempotency is non-negotiable**. Your reconcile loop *must* be safe to run repeatedly.
    *   **Update `.status` religiously**. This is how users know what's happening (`kubectl get`).
    *   **Use Finalizers for external resources**. Prevent accidental deletion before cloud resources are cleaned up.
    *   **Implement proper RBAC**. Least privilege principle. Never use `cluster-admin`.
    *   **Test failure scenarios**: Simulate pod crashes, network partitions, etcd loss.
    *   **Don't over-engineer**: If a `Helm chart` or `Kustomize` suffices, use it. Operators add complexity.

3.  **General**:
    *   **CRDs are data, Operators are logic**. Never put business logic *only* in CRD validation.
    *   **Operators != Microservices**: They manage *applications*, not just deploy pods. Focus on domain expertise.
    *   **Start simple**: Begin with basic CRUD, then add complex features (backups, upgrades).
    *   **Monitor your Operator**: Track reconcile errors, queue depth, latency.

---
