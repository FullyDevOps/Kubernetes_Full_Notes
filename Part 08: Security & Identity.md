### **Chapter 26: Kubernetes Security Model**
The foundation of all K8s security. It's not a single feature, but a **layered philosophy**.

1.  **Defense in Depth (DiD):**
    *   **What it is:** Implementing *multiple, independent* security controls across different layers (network, node, pod, application). If one layer fails, others provide protection.
    *   **Why it matters:** K8s is complex. Relying on a single control (e.g., just RBAC) is insufficient. An attacker bypassing network policies might still be stopped by pod security standards or admission control.
    *   **K8s Implementation Layers:**
        *   **Infrastructure:** Cloud/network firewalls, host OS hardening (SELinux/AppArmor), etcd encryption.
        *   **Cluster:** API server hardening (TLS, authn/authz), secure kubelet config, etcd security.
        *   **Workload:** Pod Security Standards (PSA), network policies, secure image practices, runtime security (gVisor).
        *   **Application:** Secure coding, secrets management *within* the app.
    *   **Best Practice:** Audit *all* layers. Don't assume security at one level (e.g., network segmentation) negates the need for others (e.g., PSA).

2.  **Principle of Least Privilege (PoLP):**
    *   **What it is:** Granting entities (users, service accounts, processes) *only* the absolute minimum permissions required to perform their specific task, for the shortest time necessary.
    *   **Why it matters:** Limits the "blast radius" of a compromised account or pod. A pod with only `list` access to its own namespace can't delete cluster resources.
    *   **K8s Implementation:**
        *   **RBAC:** The primary mechanism (Chapter 28). Define Roles/ClusterRoles with minimal verbs (`get`, `list`, `watch` vs `*`).
        *   **Service Accounts:** *Always* use dedicated SAs for workloads (never `default` SA). Bind minimal Roles.
        *   **Pod Security:** Restrict capabilities (e.g., `NET_ADMIN`), disallow privileged mode, drop Linux capabilities (Chapter 30).
        *   **Node Access:** Restrict `kubectl` access via RBAC; use SSH key rotation; minimize `sudo` access on nodes.
    *   **Best Practice:** Start with *no* permissions. Add *only* what's demonstrably needed. Regularly review permissions (e.g., `kubectl auth can-i --list`).

3.  **Secure Defaults:**
    *   **What it is:** The system should be secure *out-of-the-box* without requiring complex configuration changes. Default settings prioritize security over convenience.
    *   **Why it matters:** Most clusters are deployed with defaults. Insecure defaults lead to widespread vulnerabilities (e.g., early K8s versions allowing anonymous API access).
    *   **K8s Evolution & Current State:**
        *   **Historical Weakness:** Early versions had insecure defaults (e.g., no authn/authz enabled by default).
        *   **Modern Improvements:**
            *   `kubeadm` enables RBAC by default.
            *   Pod Security Admission (PSA) *enforces* Baseline/Restricted profiles by default in new clusters (v1.25+).
            *   API server requires TLS; anonymous auth disabled by default.
            *   etcd requires client TLS authentication.
        *   **Remaining Gaps:** `default` ServiceAccount *still* has no explicit RBAC bindings (but has limited implicit permissions). Network Policies are *not* enabled by default (require CNI plugin support).
    *   **Best Practice:** *Never* assume defaults are sufficient. Explicitly configure security (RBAC, PSA, Network Policies). Audit cluster against CIS Benchmarks.

---

### **Chapter 27: Authentication**
Verifying **"Who are you?"** for API server requests. *Multiple* methods can be enabled simultaneously; the first successful method authenticates the request.

1.  **Client Certificates:**
    *   **What it is:** X.509 certificates presented by the client (e.g., `kubectl`, kubelet, kube-proxy). The API server validates the certificate against its configured CA (`--client-ca-file`).
    *   **How it Works:**
        *   Admin generates a CSR, signs it with the cluster CA.
        *   Client config (e.g., `~/.kube/config`) uses the cert/key.
        *   API server verifies signature chain back to its CA.
    *   **Pros:** Strong, standard TLS mechanism. Works offline.
    *   **Cons:** Certificate management (rotation, revocation) is complex. Primarily for *components* (kubelets, nodes, admin users), less ideal for end-users.
    *   **Best Practice:** Use for cluster components (kubelet, scheduler, controller-manager). Rotate certificates proactively (e.g., `kubeadm certs renew`). Use short-lived certs where possible.

2.  **Static Token Files:**
    *   **What it is:** A file (`--token-auth-file`) on the API server containing pre-defined tokens (username, user UID, groups). Clients send the token via `Authorization: Bearer <token>` header.
    *   **How it Works:**
        *   File format: `token,user,uid,"group1,group2,group3"`
        *   API server checks token against file.
    *   **Pros:** Simple to set up.
    *   **Cons:** **Highly Insecure for production.** Tokens are long-lived, stored in plaintext, hard to rotate/revoke, no standard expiration. Compromised token = full access.
    *   **Best Practice:** **Avoid for user authentication.** Only suitable for *very* short-lived bootstrap scenarios (like initial cluster setup via `kubeadm init` using Bootstrap Tokens, which are a *specific, ephemeral* use case of static tokens). **Never** use for end-users or long-lived service accounts.

3.  **Bootstrap Tokens:**
    *   **What it is:** **A *specific, secure implementation* of short-lived static tokens for *node joining*.** Managed as `Secrets` (`bootstrap.kubernetes.io/token`) in the `kube-system` namespace.
    *   **How it Works:**
        *   Token format: `abcdef.0123456789abcdef` (ID + Secret).
        *   Token has a short TTL (default 24h), usage limits (e.g., only for `create`ing `Node` objects), and specific permissions (via RBAC bindings like `system:bootstrap-node-client`).
        *   Used by `kubeadm join` to authenticate to the API server and fetch certificates/config.
    *   **Pros:** Ephemeral, scoped permissions, managed via K8s API (rotation/deletion easy).
    *   **Cons:** Still fundamentally a token; if leaked *before* expiration, can be misused for node joining.
    *   **Best Practice:** Standard mechanism for secure node bootstrapping (`kubeadm`). Rotate tokens regularly. Delete expired tokens.

4.  **OpenID Connect (OIDC):**
    *   **What it is:** Delegating authentication to an external Identity Provider (IdP) like Okta, Auth0, Keycloak, Azure AD, Google. Uses OAuth 2.0 flows.
    *   **How it Works:**
        *   API server configured (`--oidc-issuer-url`, `--oidc-client-id`, `--oidc-username-claim`, `--oidc-groups-claim`).
        *   User authenticates via browser to IdP, gets an ID Token (JWT).
        *   `kubectl` uses the token: `users: - name: oidc-user user: auth-provider: name: oidc ... token: <ID_TOKEN> ...`
        *   API server validates JWT signature (via IdP's JWKS endpoint), issuer, audience (`client-id`), and extracts username/groups from claims.
    *   **Pros:** Leverages enterprise identity, SSO, MFA, centralized user management. Strong standard.
    *   **Cons:** Complex setup (IdP configuration). Requires user interaction for token acquisition (mitigated by tools like `kubelogin`). Token expiration/refresh needs handling.
    *   **Best Practice:** **The gold standard for user authentication in production.** Configure strict claim validation. Use short-lived tokens. Integrate with your corporate IdP.

5.  **Webhook Token Authentication:**
    *   **What it is:** Delegating token validation to an external service via HTTP API call.
    *   **How it Works:**
        *   API server configured (`--authentication-token-webhook-config-file`).
        *   On receiving a token (Bearer token header), API server POSTs a `TokenReview` object to the webhook.
        *   Webhook service validates the token (e.g., checks against Vault, custom DB, another auth system) and returns a `TokenReview` response indicating success/failure and user info (username, UID, groups).
    *   **Pros:** Extreme flexibility. Integrate with *any* auth system (Vault, custom JWT, legacy systems).
    *   **Cons:** Adds latency (network call per auth). Complexity of building/maintaining the webhook service. Single point of failure.
    *   **Best Practice:** Use when OIDC isn't feasible or for integrating with highly custom auth systems. Ensure webhook is highly available and secure (TLS, mutual TLS). Implement caching cautiously.

---

### **Chapter 28: Authorization**
Verifying **"What are you allowed to do?"** *after* authentication. Determines if a request (verb, resource, namespace) is permitted for the authenticated user/group/SA.

1.  **ABAC (Attribute-Based Access Control - Deprecated):**
    *   **What it is:** Authorization based on *attributes* (user, group, resource, verb, namespace, API group) evaluated against policy files. **Deprecated since v1.20, removed in v1.25+.**
    *   **How it Worked (Historical):**
        *   Policy file (CSV/JSON) on API server (`--authorization-policy-file`).
        *   Rules like: `allow, user1, *, *, *, *` (user1 has all access).
        *   Rules evaluated top-down; first match wins.
    *   **Why Deprecated:** Static, file-based, hard to manage at scale, no native K8s API for management, no auditability within K8s. **RBAC is the replacement.**
    *   **Best Practice:** **Migrate to RBAC immediately if still using ABAC.** Do not deploy new clusters with ABAC.

2.  **RBAC (Role-Based Access Control):**
    *   **What it is:** The **standard, recommended** authorization mechanism. Grants permissions to *roles*, which are then bound to *users, groups, or service accounts*.
    *   **Core Components:**
        *   **Roles & ClusterRoles:**
            *   **Role:** Defines permissions *within a single namespace* (e.g., `get`, `list`, `create` on `pods` in `dev` namespace).
            *   **ClusterRole:** Defines permissions *across the entire cluster* (e.g., `get` on `nodes`, `list` on `clusterroles`). Can also be used *within* a namespace (for namespaced resources).
            *   **Structure:** Contains `rules` specifying `apiGroups`, `resources`, `verbs`, and optionally `resourceNames`.
            *   **Example (Role):**
                ```yaml
                apiVersion: rbac.authorization.k8s.io/v1
                kind: Role
                metadata:
                  namespace: dev
                  name: pod-reader
                rules:
                - apiGroups: [""] # Core API group
                  resources: ["pods"]
                  verbs: ["get", "list", "watch"]
                ```
        *   **RoleBindings & ClusterRoleBindings:**
            *   **RoleBinding:** Grants the permissions *defined in a Role* to subjects (users, groups, SAs) *within a specific namespace*.
            *   **ClusterRoleBinding:** Grants the permissions *defined in a ClusterRole* to subjects *across the entire cluster* (or within a specific namespace if the ClusterRole is used namespaced).
            *   **Structure:** Links a Role/ClusterRole to `subjects` (name, kind: `User`, `Group`, `ServiceAccount`, namespace for SAs).
            *   **Example (RoleBinding):**
                ```yaml
                apiVersion: rbac.authorization.k8s.io/v1
                kind: RoleBinding
                metadata:
                  name: read-pods
                  namespace: dev
                subjects:
                - kind: ServiceAccount
                  name: my-app-sa
                  namespace: dev
                roleRef:
                  kind: Role
                  name: pod-reader
                  apiGroup: rbac.authorization.k8s.io
                ```
            *   **Example (ClusterRoleBinding for SA):**
                ```yaml
                apiVersion: rbac.authorization.k8s.io/v1
                kind: ClusterRoleBinding
                metadata:
                  name: system:node-proxier
                subjects:
                - kind: ServiceAccount
                  name: kube-proxy
                  namespace: kube-system
                roleRef:
                  kind: ClusterRole
                  name: system:node-proxier
                  apiGroup: rbac.authorization.k8s.io
                ```

3.  **Best Practices for RBAC:**
    *   **Namespace Isolation:** Use Roles/RoleBindings for namespace-scoped access. Avoid ClusterRoles/Bindings unless absolutely necessary (e.g., accessing `nodes`, `persistentvolumes`).
    *   **Least Privilege:** Grant only specific verbs on specific resources. Avoid `*` unless truly needed (rare). Use `resourceNames` for granular object access.
    *   **Service Accounts:** **ALWAYS** create dedicated SAs for workloads. Bind minimal Roles to them. **NEVER** use the `default` SA for production workloads.
    *   **Avoid `cluster-admin`:** Reserve `cluster-admin` ClusterRoleBinding for cluster administrators only. Never bind it to application SAs.
    *   **Leverage Built-in Roles:** Use `view`, `edit`, `admin` cautiously (they are broad). Prefer custom Roles.
    *   **Review Permissions:** Regularly audit bindings (`kubectl get rolebindings,clusterrolebindings -A -o wide`). Use `kubectl auth can-i --list --as=<user>` for checks.
    *   **Bind to Groups:** Bind ClusterRoleBindings to LDAP/IdP groups (e.g., `oidc:engineering`) instead of individual users for easier management.
    *   **Immutable Roles:** Avoid modifying built-in ClusterRoles (`system:`). Create custom Roles/ClusterRoles.
    *   **SA Token Volume Projection:** Use projected service account tokens (v1.12+) for enhanced security (audience, expiration).

---

### **Chapter 29: Admission Control**
The **gatekeeper** for API requests *after* authn/authz but *before* object persistence. Modifies or rejects requests based on policies.

1.  **What are Admission Controllers?**
    *   **What it is:** Plugins compiled into the API server that intercept requests *to* and *from* the storage layer (etcd). They **modify** the object (`MutatingAdmissionWebhook`) or **validate** it (`ValidatingAdmissionWebhook`, built-in controllers) *before* it's persisted.
    *   **Why it matters:** Enforces cluster-wide policies that RBAC cannot (e.g., "all pods must have resource limits", "no privileged containers", "images must come from trusted registry"). Critical for security and operational consistency.
    *   **Lifecycle:** `Admission -> Authentication -> Authorization -> Admission Control (Mutation -> Validation) -> Persistence (etcd)`

2.  **Built-in Controllers (Enabled via `--enable-admission-plugins`):**
    *   **AlwaysPullImages:**
        *   **Action:** Mutates pod specs. Sets `imagePullPolicy: Always` for *all* containers.
        *   **Why:** Ensures the latest image is pulled from the registry on *every* node startup, preventing stale/malicious cached images from running. Crucial for security patching.
        *   **Trade-off:** Increases startup time; requires reliable registry access. Often used in highly secure environments.
    *   **ResourceQuota:**
        *   **Action:** Validates pod creation against namespace `ResourceQuota` objects (CPU, memory, storage, object counts).
        *   **Why:** Prevents namespace resource exhaustion (DoS). Essential for multi-tenant clusters.
    *   **LimitRanger:**
        *   **Action:** Validates/patches pod/container specs against namespace `LimitRange` objects (min/max CPU/memory, default requests/limits).
        *   **Why:** Ensures pods define reasonable resource requests/limits, improving cluster stability and scheduling.
    *   **NamespaceAutoProvision:**
        *   **Action:** Automatically creates a namespace if it doesn't exist when an object is created within it. (Usually enabled by default).
    *   **NamespaceExists:**
        *   **Action:** Rejects requests targeting a non-existent namespace. (Usually enabled by default).
    *   **ServiceAccount:**
        *   **Action:** Mutates pods to automatically inject the `default` ServiceAccount token *if none is specified*. (Crucial for component communication).
        *   **Security Note:** Reinforces why you should *always* use custom SAs - this controller injects the `default` SA token if you don't specify one!
    *   **PodSecurityPolicy (PSP - Deprecated):** *See Chapter 30 for details.* Historically the primary mechanism for pod security policies. **Replaced by PSA.**
    *   **Other Key Controllers:** `NodeRestriction` (kubelet restrictions), `EventRateLimit`, `DenyEscalatingExec`, `SecurityContextDeny` (deprecated PSP alternative).

3.  **Dynamic Admission Control (Webhooks):**
    *   **What it is:** External admission controllers implemented as HTTP APIs (webhooks), called by the API server. Allows custom, cluster-specific policies without recompiling K8s.
    *   **Types:**
        *   **MutatingAdmissionWebhook:**
            *   **Action:** *Modifies* the object *before* validation. Can add/modify fields (e.g., inject sidecar, set default labels, add security context).
            *   **When:** Called *after* built-in mutating controllers, *before* validation.
            *   **Critical:** Must be idempotent. Can cause failures if slow/unavailable (configurable timeout).
        *   **ValidatingAdmissionWebhook:**
            *   **Action:** *Validates* the object *after* mutation. Can *only* accept or reject the request (cannot modify).
            *   **When:** Called *after* all mutation (built-in and mutating webhooks).
            *   **Critical:** Failure usually rejects the request (configurable failure policy).
    *   **How it Works:**
        1.  Admin deploys webhook server (e.g., as a Deployment/Service).
        2.  Admin creates `MutatingWebhookConfiguration` or `ValidatingWebhookConfiguration` object.
        3.  Config object defines:
            *   Webhook name & URL (or service reference)
            *   Which resources/verbs/events to intercept (e.g., `pods` `create,update`)
            *   Rules for when to call it (namespaceSelector, objectSelector)
            *   Failure policy (`Fail` or `Ignore`)
            *   ClientConfig (CA bundle for TLS)
        4.  API server calls the webhook per configuration.
    *   **Use Cases:** Enforce custom security policies (e.g., "no root user", "must have network policy"), inject sidecars (Istio, OPA/Gatekeeper), enforce labeling standards, scan images.
    *   **Best Practices:**
        *   **Start with Validation:** Use ValidatingWebhooks first; MutatingWebhooks are more complex/risky.
        *   **Idempotency:** MutatingWebhooks *must* produce the same result on repeated calls.
        *   **Timeouts & Failure Policy:** Set reasonable timeouts (`timeoutSeconds`, default 10s). Prefer `Fail` for security-critical checks, `Ignore` for non-critical.
        *   **Namespace Selectors:** Use `namespaceSelector` to exempt system namespaces (`kube-system`, `kube-public`).
        *   **CA Bundles:** Rotate CA bundles carefully; webhook server certificate must be signed by the CA in the bundle.
        *   **Tools:** Use frameworks like **OPA/Gatekeeper** or **Kyverno** to simplify writing/maintaining policies.

---

### **Chapter 30: Pod Security**
Controlling **what a pod/container is allowed to do** at the kernel/process level. Critical for workload isolation.

1.  **Pod Security Standards (PSS):**
    *   **What it is:** **Official K8s-defined policy levels** (Baseline, Restricted, Privileged) specifying allowed pod configurations. *Replaces* the deprecated PodSecurityPolicy (PSP).
    *   **Why it matters:** Provides standardized, vendor-neutral security profiles. PSA (below) enforces these standards.
    *   **The Three Levels (Strictly Increasing Restrictions):**
        *   **Privileged:**
            *   **Goal:** *Unrestricted* pod configuration (equivalent to pre-security K8s). **NOT SECURE.**
            *   **Use Case:** *Only* for system pods requiring maximum flexibility (e.g., `kube-proxy`, `node-driver-registrar`). **Never for application workloads.**
            *   **Key Permissions:** `privileged: true`, arbitrary capabilities, host namespaces, hostPath volumes, unrestricted seccomp/AppArmor.
        *   **Baseline:**
            *   **Goal:** Minimally restricted pods. Prevents known privilege escalations but allows broad functionality. **Default for most workloads.**
            *   **Key Restrictions (vs Privileged):**
                *   `privileged: false`
                *   Capabilities: Only `NET_BIND_SERVICE` allowed by default (via `allowPrivilegeEscalation: false` and capability drops). No `SYS_ADMIN`, `DAC_OVERRIDE`, etc.
                *   Host Namespaces: `hostPID`, `hostIPC`, `hostNetwork: false`
                *   Host Path Volumes: Disallowed
                *   SELinux/AppArmor: Must be set to a known, non-root profile (e.g., `container_t`, `runtime/default`)
                *   Seccomp: Must use a defined profile (e.g., `RuntimeDefault`)
                *   `allowPrivilegeEscalation: false`
                *   `readOnlyRootFilesystem: false` (allowed, but not required)
        *   **Restricted:**
            *   **Goal:** Hardened pods adhering to strict security practices. **Recommended for untrusted workloads/multi-tenant.**
            *   **Key Restrictions (vs Baseline):**
                *   **Mandatory:** `runAsNonRoot: true` (No root user *at all*)
                *   **Mandatory:** `runAsUser` set to non-zero value (or `runAsGroup` set)
                *   **Mandatory:** `seccompProfile.type: RuntimeDefault` (or `Localhost`)
                *   **Mandatory:** `appArmor.security.beta.kubernetes.io`: `runtime/default` annotation (if AppArmor enabled)
                *   **Mandatory:** `capabilities.drop: ["ALL"]` (Only capabilities granted via `add` are allowed, very limited)
                *   `allowPrivilegeEscalation: false` (Enforced)
                *   `readOnlyRootFilesystem: true` (Required)
                *   No `hostPorts`
                *   No `procMount: Unmasked`
                *   `sysctls` restricted to safe profiles

2.  **Pod Security Admission (PSA):**
    *   **What it is:** **The built-in admission controller** (replacing PSP) that **enforces** the Pod Security Standards (PSS) at the namespace level. Enabled by default in v1.23+ (beta in v1.21+).
    *   **How it Works:**
        *   Admin sets **enforcement level** (`privileged`, `baseline`, `restricted`) and **audit**/**warn** levels on a namespace via **labels**:
            *   `pod-security.kubernetes.io/enforce: <level>` (e.g., `restricted`)
            *   `pod-security.kubernetes.io/enforce-version: v1.25` (Optional, pins policy version)
            *   `pod-security.kubernetes.io/audit: <level>` (Logs violations at specified level without blocking)
            *   `pod-security.kubernetes.io/warn: <level>` (Issues warnings via API for violations at specified level)
        *   PSA admission controller checks pod specs against the namespace's enforce level *during admission*.
        *   Violations at `enforce` level **block** pod creation/update.
        *   Violations at `audit`/`warn` levels generate logs/warnings.
    *   **Why it Replaced PSP:**
        *   **Simpler:** Labels instead of complex CRDs/ClusterRoles.
        *   **Namespaced:** Policies scoped to namespaces (easier management, multi-tenancy).
        *   **Standardized:** Based directly on PSS levels.
        *   **Built-in:** No external components needed (unlike PSP requiring RBAC setup).
        *   **Gradual Adoption:** `audit`/`warn` levels allow testing before enforcement.
    *   **Migration from PSP:** PSA is the direct successor. PSP is **completely removed** in v1.25+. Use PSA labels to enforce equivalent policies.
    *   **Best Practices:**
        *   **Start with `baseline`** in non-critical namespaces.
        *   Use `audit`/`warn` levels to identify violations *before* switching to `enforce`.
        *   **Aim for `restricted`** for new applications and untrusted workloads.
        *   Label *all* namespaces explicitly (PSA has default labels on `default` namespace, but custom namespaces inherit no policy).
        *   Use `enforce-version` to avoid unexpected policy changes during upgrades.
        *   **Combine with OPA/Gatekeeper/Kyverno:** PSA covers core PSS. Use these for *additional*, custom pod policies (e.g., image registries, specific capabilities).

3.  **Pod Security Policies (PSP - Deprecated):**
    *   **What it was:** **Cluster-scoped** resources defining pod security requirements (capabilities, volumes, users, SELinux, etc.). Required complex RBAC setup to bind policies to users/SAs.
    *   **Why Deprecated:** Complex, cluster-scoped (hard for multi-tenancy), required manual RBAC, difficult to audit. PSA provides a simpler, namespaced alternative based on standardized levels.
    *   **Status:** **Removed entirely in Kubernetes v1.25.** **Do not use.** Migrate to PSA immediately.
    *   **Legacy Note:** If maintaining an old cluster (<v1.21), understand PSP structure, but prioritize migration.

4.  **Seccomp, AppArmor, SELinux Profiles:**
    *   **What they are:** **Linux kernel security mechanisms** to restrict syscalls/process capabilities *within* the container.
    *   **How they Work in K8s:**
        *   **Seccomp (Secure Computing Mode):**
            *   Filters syscalls a process can make.
            *   **K8s:** Defined via `securityContext.seccompProfile` (v1.19+). Types: `RuntimeDefault` (uses container runtime's default profile, e.g., Docker's default drops ~40 syscalls), `Localhost` (reference profile on node), `Unconfined`.
            *   **Best Practice:** **Always set `seccompProfile.type: RuntimeDefault`** (enforced in Restricted PSS). Create custom profiles for high-security needs (stored on nodes).
        *   **AppArmor:**
            *   Mandatory Access Control (MAC) system. Profiles define file/network access per program.
            *   **K8s:** Defined via pod annotation: `container.apparmor.security.beta.kubernetes.io/<container_name>: runtime/default` or `localhost/<profile_name>`.
            *   **Best Practice:** Load profiles on nodes. Use `runtime/default` as a baseline. Requires node OS support (AppArmor enabled).
        *   **SELinux (Security-Enhanced Linux):**
            *   MAC system using labels (user, role, type, level) for fine-grained access control.
            *   **K8s:** Defined via `securityContext.seLinuxOptions` (`user`, `role`, `type`, `level`). Often managed by the container runtime (e.g., `container_t` type).
            *   **Best Practice:** Ensure nodes have SELinux enforcing. Runtimes usually handle labeling; avoid overriding unless necessary. Critical for multi-tenant isolation on RHEL/CentOS.

5.  **Preventing Privileged Containers:**
    *   **Why Critical:** `privileged: true` grants the container *almost all* host capabilities (bypassing seccomp/AppArmor/SELinux), equivalent to root on the host. **Major security risk.**
    *   **How to Prevent:**
        *   **PSA:** Both `baseline` and `restricted` PSS levels **require** `privileged: false`.
        *   **Admission Control:** PSA (enforce `baseline`/`restricted`), OPA/Gatekeeper, Kyverno policies (`container.privileged == false`).
        *   **RBAC:** Prevent users from creating pods with `privileged: true` (though PSA is more direct).
        *   **Image Scanning:** Detect privileged containers in CI/CD.
    *   **Best Practice:** **NEVER** set `privileged: true` for application workloads. Only for essential node-level components (and even then, scrutinize). PSA `baseline` is the absolute minimum.

---

### **Chapter 31: Network Security**
Securing communication **between components** and **within the cluster**.

1.  **Securing API Server:**
    *   **Critical Surface:** The central management endpoint. Compromise = cluster compromise.
    *   **Key Measures:**
        *   **TLS Everywhere:** API server *must* serve HTTPS (`--tls-cert-file`, `--tls-private-key-file`). Clients *must* use TLS and verify server cert (`--insecure-skip-tls-verify=false`).
        *   **Authentication:** Enable strong authn (OIDC, Certs). Disable anonymous access (`--anonymous-auth=false`).
        *   **Authorization:** Enable RBAC (`--authorization-mode=RBAC`).
        *   **Audit Logging:** Enable detailed audit logs (`--audit-policy-file`, `--audit-log-path`).
        *   **Firewalling:** Restrict API server port (6443/TCP) *only* to:
            *   Control plane components (kubelets, schedulers, controllers - usually on same subnet)
            *   Trusted admin networks (jump hosts)
            *   **Never** expose publicly without a strong auth proxy (e.g., OIDC-conformant reverse proxy).
        *   **Disable Unsafe Flags:** Avoid `--insecure-port`, `--insecure-bind-address`.
        *   **etcd Client TLS:** API server must use TLS client certs to connect to etcd (`--etcd-certfile`, `--etcd-keyfile`, `--etcd-cafile`).

2.  **Securing etcd:**
    *   **Critical Surface:** The cluster's "source of truth" database. Compromise = total cluster compromise.
    *   **Key Measures:**
        *   **Dedicated Nodes:** Run etcd on *dedicated, hardened* nodes (not shared with workloads).
        *   **TLS Everywhere:**
            *   **Peer TLS:** Encrypt communication *between* etcd members (`--peer-cert-file`, `--peer-key-file`, `--peer-trusted-ca-file`).
            *   **Client TLS:** API server *must* use client TLS to access etcd (as above). **Disable non-TLS client access (`--client-cert-auth=true`).**
        *   **Firewalling:** Restrict etcd ports (2379/TCP - client, 2380/TCP - peer) *only* to:
            *   Other etcd members (for peer port)
            *   API servers (for client port)
            *   **Strictly minimal access.**
        *   **Authentication:** Enable client certificate authentication (`--client-cert-auth=true`). Consider additional authn (v3.3+ supports simple auth, but certs are primary).
        *   **Encryption at Rest:** **MANDATORY** (`--encryption-provider-config`). Use KMS if possible. Rotate keys regularly.
        *   **Backups:** Regularly backup etcd data (encrypted).

3.  **TLS Bootstrapping:**
    *   **What it is:** Automated process for **kubelets** to obtain their *client certificates* from the API server *securely*, without manual CA signing.
    *   **Why it Matters:** Enables secure, scalable node joining. Prevents manual cert distribution.
    *   **How it Works (Simplified):**
        1.  Bootstrap Token (Chapter 27) is created (with limited permissions).
        2.  New node runs `kubelet --bootstrap-kubeconfig=...` pointing to bootstrap token.
        3.  Kubelet uses token to authenticate to API server and request a *certificate signing request (CSR)*.
        4.  CSR is auto-approved (by `csrsigning` controller) or manually approved (`kubectl certificate approve`).
        5.  Kubelet retrieves its signed client certificate.
        6.  Kubelet uses this cert for *all* future API communication (replacing the token).
    *   **Security:** Relies on secure bootstrap token distribution and short token lifetime. Critical for secure node onboarding.

4.  **Audit Logging:**
    *   **What it is:** Recording **who did what, when, and how** regarding API server requests.
    *   **Why Critical:** Essential for security monitoring, incident investigation, and compliance (detecting misuse, breaches, policy violations).
    *   **How it Works:**
        *   API server configured with audit policy file (`--audit-policy-file`) and log output (`--audit-log-path`, `--audit-log-maxage`, etc.).
        *   **Audit Policy Levels:** `None`, `Metadata`, `Request`, `RequestResponse`. **Use `Request` or `RequestResponse` for security.**
        *   **Policy Rules:** Define what to log based on `user`, `group`, `verb`, `resource`, `namespace`, `level`.
    *   **Best Practices:**
        *   **Log Everything Critical:** At minimum, log all `create`, `update`, `delete`, `patch` requests at `Request` level. Log `get` for sensitive resources (secrets, clusterroles).
        *   **Centralized Logging:** Ship audit logs to SIEM (Splunk, ELK, Datadog) for analysis/alerting. *Do not rely solely on local logs.*
        *   **Protect Logs:** Ensure audit logs are tamper-evident (write-only storage, integrity checks).
        *   **Review Regularly:** Actively monitor for suspicious activity (unusual users, access to secrets, clusterrole changes).

5.  **Firewalling and Network Segmentation:**
    *   **What it is:** **Network-level controls** (outside K8s) to restrict traffic flow between cluster components and the outside world.
    *   **Why Critical:** First line of defense. Limits attack surface. Complements K8s-native Network Policies.
    *   **Key Layers:**
        *   **Cloud/Infra Firewalls (Security Groups/NACLs):**
            *   Restrict **ingress** to cluster: Only necessary ports (e.g., 443 for Ingress Controllers, 6443 for API server *only from jump hosts*).
            *   Restrict **egress** from nodes: Only allow necessary outbound traffic (registries, monitoring, auth endpoints).
            *   **Isolate Control Plane:** Control plane nodes should *only* accept traffic from worker nodes (on kubelet port 10250) and trusted admin networks (on 6443). Block all other ingress.
            *   **Isolate etcd:** etcd nodes should *only* accept traffic from other etcd nodes (2380) and API servers (2379). Block all other ingress.
        *   **Host Firewalls (iptables/nftables):** On each node, restrict ports (e.g., block direct access to kubelet port 10250 from non-control-plane nodes).
        *   **Kubernetes Network Policies (Chapter 31 Context):**
            *   **Crucial Distinction:** Network Policies are *K8s-native*, *application-layer* controls *within* the cluster network (implemented by CNI plugins like Calico, Cilium). They **complement**, but **do NOT replace**, infrastructure firewalls.
            *   **Function:** Control pod-to-pod traffic based on labels (`ingress`/`egress` rules). Enforce zero-trust within the cluster.
            *   **Best Practice:** Implement a **default-deny** policy for namespaces (`ingress: {}` denies all ingress). Then explicitly allow necessary traffic. Start with non-critical namespaces. **Infrastructure firewalls are the outer wall; Network Policies are the internal doors.**

---
