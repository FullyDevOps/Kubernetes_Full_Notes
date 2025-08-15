### **Chapter 6: Pods**  
#### **What is a Pod?**  
- **Definition**: The smallest *scheduling unit* in Kubernetes. A Pod is a logical group of **one or more tightly coupled containers** sharing:  
  - **Network namespace** (same IP/port space, `localhost` communication).  
  - **Storage volumes** (shared filesystem via `emptyDir`, `configMap`, etc.).  
  - **cgroups/IPC namespace** (process visibility).  
- **Why Pods?**: Containers in a Pod are co-located, co-scheduled, and designed to run as a single cohesive unit (e.g., app + logging sidecar).  
- **Key Insight**: Kubernetes *never* schedules individual containers—only Pods.  

#### **Pod Lifecycle**  
1. **Pending**:  
   - Scheduler assigns Pod to a Node.  
   - Container images pulled (delays if large images or registry issues).  
2. **Running**:  
   - All containers started.  
   - **Readiness Probe**: Determines if Pod is ready to serve traffic (`/healthz` endpoint). *Failing = traffic not routed*.  
   - **Liveness Probe**: Determines if container is alive (restarts container if fails).  
3. **Succeeded/Terminated**:  
   - All containers exited *successfully* (exit code 0).  
4. **Failed**:  
   - Any container exited *abnormally* (non-0 exit code).  
5. **Unknown**:  
   - Node unresponsive (e.g., network partition).  

> 💡 **Critical Nuance**: `initContainers` run *sequentially* **before** app containers start. If one fails, Pod restarts (based on `restartPolicy`).  

#### **Multi-Container Pod Patterns**  
| **Pattern**      | **Use Case**                          | **Example**                                  | **How It Works**                                  |  
|------------------|---------------------------------------|----------------------------------------------|---------------------------------------------------|  
| **Sidecar**      | Augment main app with helper tasks    | Log shipper (Fluentd), monitoring agent      | Sidecar shares volume with app; reads logs/metrics |  
| **Ambassador**   | Abstract networking complexity        | Database proxy (e.g., connect to sharded DB) | Main app talks to `localhost:5432` → Ambassador routes to correct DB instance |  
| **Adapter**      | Normalize output for monitoring       | Convert app logs to standard format          | Adapter processes app output → emits uniform metrics/logs |  

> ⚠️ **Pitfall**: Sidecars can starve main app of resources. *Always set resource limits!*  

#### **Pod Networking & Storage**  
- **Networking**:  
  - Each Pod gets a **unique IP** (assigned by CNI plugin like Calico/Flannel).  
  - Containers communicate via `localhost` (no NAT).  
  - DNS resolution via `CoreDNS` (e.g., `my-svc.default.svc.cluster.local`).  
- **Storage**:  
  - **Volumes**: Defined at Pod level (not container). Types:  
    - `emptyDir`: Ephemeral storage (deleted with Pod).  
    - `persistentVolumeClaim`: Binds to persistent storage (e.g., AWS EBS).  
    - `configMap`/`secret`: Inject config/secrets as files.  
  - **Key Point**: Volumes survive container restarts within the Pod.  

#### **Init Containers**  
- Run **before** app containers.  
- **Use Cases**:  
  - Wait for database to be ready (`until curl db:5432; do sleep 1; done`).  
  - Download config files from remote source.  
  - Set up filesystem permissions.  
- **Behavior**:  
  - Execute *sequentially* (one must finish before next starts).  
  - If any fail, Pod restarts (based on `restartPolicy: Always`/`OnFailure`).  
- **YAML**:  
  ```yaml
  initContainers:
  - name: init-db
    image: busybox
    command: ['sh', '-c', 'until nslookup mydb; do echo waiting for DB; sleep 2; done']
  ```

---

### **Chapter 7: Labels, Selectors & Annotations**  
#### **Labeling Resources**  
- **Labels**: Key/value pairs attached to objects (e.g., `env=prod`, `tier=frontend`).  
  - **Purpose**: Identify, organize, and select objects.  
  - **Constraints**: Max 63 chars per key/value, `a-z0-9A-Z_-.` only.  
- **Best Practices**:  
  - Use standardized prefixes (e.g., `app.kubernetes.io/name=nginx`).  
  - Avoid high-cardinality labels (e.g., `timestamp=12345`).  

#### **Using Selectors**  
1. **Label Selectors**:  
   - **Equality-based**: `env=prod` (exact match).  
   - **Set-based**: `env in (prod, staging)`, `tier notin (test)`.  
   - **Used by**: Deployments, Services, `kubectl get pods -l "env=prod"`.  
2. **Field Selectors**:  
   - Filter by resource *fields* (not metadata).  
   - Example: `kubectl get pods --field-selector status.phase=Running`.  
   - **Limitations**: Not all fields are selectable (e.g., `metadata.labels` isn’t a field selector).  

#### **Annotations vs. Labels**  
| **Labels**                            | **Annotations**                              |  
|---------------------------------------|----------------------------------------------|  
| For *identifying/organizing* objects  | For *non-identifying* metadata               |  
| Used in selectors                     | **Not** used in selectors                    |  
| Low cardinality (few values)          | High cardinality (e.g., build timestamps)    |  
| `app=web`, `env=prod`                 | `build-date=2023-10-05`, `contact-email=dev@company.com` |  

> 💡 **Annotation Use Cases**:  
> - Tooling integrations (e.g., `cluster-autoscaler.kubernetes.io/safe-to-evict=true`).  
> - Documentation (e.g., `description=Main frontend service`).  

---

### **Chapter 8: Deployments**  
#### **What is a Deployment?**  
- **Definition**: Manages **ReplicaSets** to declaratively update Pods (rollouts, rollbacks, scaling).  
- **Purpose**: Ensure desired state (e.g., "always 3 replicas of nginx:v1").  
- **Key Feature**: Tracks Pod template changes via **revision history**.  

#### **Rolling Updates & Rollbacks**  
- **Rolling Update** (Default strategy):  
  - Gradually replaces old Pods with new ones (minimizes downtime).  
  - Controlled by:  
    ```yaml
    strategy:
      type: RollingUpdate
      rollingUpdate:
        maxSurge: 25%       # Max % new Pods above desired count
        maxUnavailable: 25% # Max % unavailable Pods during update
    ```  
- **Rollback**:  
  - `kubectl rollout undo deployment/my-dep --to-revision=2`  
  - Uses stored ReplicaSet revisions (check with `kubectl rollout history deployment/my-dep`).  

#### **Deployment Strategies**  
| **Strategy**   | **How It Works**                                  | **Use Case**                     | **YAML Config**                     |  
|----------------|---------------------------------------------------|----------------------------------|-------------------------------------|  
| **Rolling**    | Incrementally replace old Pods with new           | Zero-downtime updates            | `strategy.type: RollingUpdate`      |  
| **Recreate**   | Kill all old Pods → Start new ones (downtime)     | DB schema changes                | `strategy.type: Recreate`           |  
| **Blue/Green** | Deploy new version → Switch traffic instantly     | Critical apps; instant rollback  | Use 2 Services + Ingress host rules |  
| **Canary**     | Route % of traffic to new version (e.g., 5%)      | Risk mitigation; A/B testing     | Service mesh (Istio) or Ingress     |  

#### **Scaling & Pausing**  
- **Scaling**:  
  - `kubectl scale deployment/my-dep --replicas=10`  
  - **Horizontal Pod Autoscaler (HPA)**: Scales based on CPU/memory/custom metrics.  
- **Pausing/Resuming**:  
  - `kubectl rollout pause deployment/my-dep` → Make multiple changes → `kubectl rollout resume`  
  - *Prevents intermediate states from triggering rollouts*.  

---

### **Chapter 9: ReplicaSets**  
#### **Purpose & Function**  
- **Definition**: Ensures a **stable set of identical Pods** are running (e.g., "always 5 nginx Pods").  
- **How It Works**:  
  - Watches Pods matching its `selector`.  
  - Creates/deletes Pods to match `replicas` count.  
- **Key Limitation**: Cannot update Pods (only scale/create/delete). *Use Deployments for updates*.  

#### **Relationship with Deployments**  
- **Deployment** → Creates **ReplicaSet** → Creates **Pods**.  
- When Deployment updates:  
  1. New ReplicaSet created with new Pod template.  
  2. Old ReplicaSet scaled down to 0 (during rolling update).  
  3. Old ReplicaSets retained for rollback (configurable via `revisionHistoryLimit`).  

#### **Manual ReplicaSet Management**  
- **Rarely used directly** (Deployments are preferred).  
- **When to use**:  
  - Simple scaling needs (no rollouts).  
  - Debugging (e.g., inspect ReplicaSet status).  
- **Critical Command**:  
  ```bash
  kubectl describe replicaset/my-rs  # Check events for scaling issues
  ```

---

### **Chapter 10: Services**  
#### **Service Types**  
| **Type**         | **How It Works**                                  | **Use Case**                          | **Access Method**                     |  
|------------------|---------------------------------------------------|---------------------------------------|---------------------------------------|  
| **ClusterIP**    | Exposes service on internal cluster IP            | Internal communication (default)      | `http://<service-name>.<namespace>`   |  
| **NodePort**     | Opens port on **all Nodes** (30000-32767) → ClusterIP | Dev/test; no cloud LB                | `<Node-IP>:<NodePort>`                |  
| **LoadBalancer** | Cloud provider creates external LB → NodePort     | Public internet access (AWS/GCP/Azure)| External LB IP                        |  
| **ExternalName** | Maps service to external DNS (CNAME)              | Access external services (e.g., SaaS)| `my-db.default.svc.cluster.local → prod-db.example.com` |  

#### **Service Discovery**  
- **DNS-Based**:  
  - `my-svc.default.svc.cluster.local` → Resolves to ClusterIP.  
  - Pods in same namespace: `http://my-svc`.  
- **Environment Variables**:  
  - `MY_SVC_SERVICE_HOST`, `MY_SVC_SERVICE_PORT` (deprecated; DNS preferred).  

#### **Endpoints & EndpointSlices**  
- **Endpoints**:  
  - Automatically created object listing **Pod IPs** backing a Service.  
  - *If no Pods match selector → empty Endpoints → 503 errors*.  
- **EndpointSlices** (v1.21+):  
  - Scales better than Endpoints (handles >1000 Pods).  
  - Managed by `endpoint-slice-controller`.  

#### **Headless Services** (`clusterIP: None`)  
- **Purpose**: Direct Pod-to-Pod communication (bypasses kube-proxy/LB).  
- **How**:  
  - Returns **Pod IPs directly** (not ClusterIP) via DNS.  
  - DNS returns **multiple A records** (one per Pod).  
- **Use Cases**:  
  - StatefulSets (e.g., MongoDB replicas need direct Pod addresses).  
  - Custom load balancing.  

---

### **Chapter 11: Ingress**  
#### **What is Ingress?**  
- **Definition**: **API object** managing *external HTTP/HTTPS access* to services (L7 routing).  
- **Not a Service type**! Requires an **Ingress Controller** (separate deployment).  

#### **Ingress Controller**  
- **Role**: Implements Ingress rules (listens for Ingress resources, configures LB).  
- **Popular Controllers**:  
  - **Nginx**: Most common (uses ConfigMap for tuning).  
  - **Traefik**: Dynamic config, supports Let's Encrypt.  
  - **AWS ALB**: Cloud-native (uses ALB instead of EC2 instances).  
- **Installation**:  
  ```bash
  helm install nginx-ingress ingress-nginx/ingress-nginx  # Typical Helm install
  ```

#### **Ingress Rules**  
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: example-ingress
spec:
  rules:
  - host: app.example.com          # Name-based routing
    http:
      paths:
      - path: /api                 # Path-based routing
        pathType: Prefix
        backend:
          service:
            name: api-service
            port: 
              number: 80
      - path: /                    # Default path
        backend: ...
  tls:                             # TLS termination
    - hosts: ["app.example.com"]
      secretName: tls-secret       # Must contain tls.crt/tls.key
```

#### **Advanced Features**  
| **Feature**          | **How to Implement**                                      |  
|----------------------|-----------------------------------------------------------|  
| **Path Rewrites**    | Nginx: `nginx.ingress.kubernetes.io/rewrite-target: /$1`  |  
| **Rate Limiting**    | Nginx: `nginx.ingress.kubernetes.io/limit-rps: "5"`       |  
| **Basic Auth**       | `nginx.ingress.kubernetes.io/auth-type: basic` + secret   |  
| **Canary Routing**   | `nginx.ingress.kubernetes.io/canary: "true"` + weight     |  

> ⚠️ **Critical Notes**:  
> - **TLS Termination**: SSL decrypted at Ingress → HTTP to services (use `backend-protocol: "HTTPS"` for end-to-end TLS).  
> - **PathType**: `Prefix` vs `Exact` vs `ImplementationSpecific` (Nginx treats `Prefix` as regex).  

---

### **Chapter 12: ConfigMaps & Secrets**  
#### **ConfigMaps**  
- **Purpose**: Inject **non-sensitive** configuration into Pods.  
- **Creation**:  
  ```bash
  kubectl create configmap app-config --from-file=config.properties
  kubectl create configmap app-env --from-literal=LOG_LEVEL=debug
  ```  
- **Usage in Pods**:  
  - **Env Variables**:  
    ```yaml
    env:
    - name: LOG_LEVEL
      valueFrom:
        configMapKeyRef:
          name: app-env
          key: LOG_LEVEL
    ```  
  - **Volume Mount**:  
    ```yaml
    volumes:
    - name: config-volume
      configMap:
        name: app-config
    containers:
    - volumeMounts:
      - name: config-volume
        mountPath: /etc/config
    ```  

#### **Secrets**  
- **Types**:  
  - `Opaque` (default): Generic secrets (e.g., passwords).  
  - `kubernetes.io/tls`: For TLS certificates (used in Ingress).  
  - `kubernetes.io/dockerconfigjson`: For private registry auth.  
- **Creation**:  
  ```bash
  # Base64 encode first (required!)
  echo -n 'password' | base64  # Output: cGFzc3dvcmQ=
  kubectl create secret generic db-secret --from-literal=password=cGFzc3dvcmQ=
  ```  
- **Security Best Practices**:  
  1. **Never store in Git** (use `.gitignore` + `.env` files).  
  2. **Use RBAC**: Restrict `secrets` access to specific ServiceAccounts.  
  3. **Avoid env variables**: Prefer volume mounts (less likely to leak in logs).  
  4. **Encrypt at Rest**:  
     - Enable **etcd encryption** in Kubernetes cluster:  
       ```yaml
       # /etc/kubernetes/manifests/kube-apiserver.yaml
       - --encryption-provider-config=/etc/kubernetes/enc.yaml
       ```  
       ```yaml
       # enc.yaml
       apiVersion: apiserver.config.k8s.io/v1
       kind: EncryptionConfiguration
       resources:
         - resources: ["secrets"]
           providers:
             - aescbc:
                 keys:
                   - name: key1
                     secret: <RANDOM-256-BIT-KEY-BASE64>
       ```  
  5. **Rotate secrets**: Update Secret → Restart Pods (no automatic reload).  

#### **Critical Secret Pitfalls**  
- **Base64 ≠ Encryption**: Secrets are *not* encrypted by default (only base64-encoded).  
- **Pod Access**: Any Pod on the Node can access secrets via `/var/run/secrets/kubernetes.io/serviceaccount`.  
- **etcd Risk**: Unencrypted etcd backups expose all secrets. *Always enable encryption at rest!*  

---
