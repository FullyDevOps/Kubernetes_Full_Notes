### **Chapter 19: Kubernetes Networking Model**

#### **1. IP-per-Pod Model: The Core Foundation**
*   **What it is:** **Every Pod gets a unique, routable IP address** within the cluster. This IP is assigned to the Pod's *entire network namespace*, shared by all containers within that Pod.
*   **Why it exists (Key Rationale):**
    *   **Simplicity & Compatibility:** Eliminates NAT complexities between Pods. Applications see a standard network interface (like a VM), making legacy apps work without modification.
    *   **Pod Identity:** The Pod IP is the *true network identity* of the workload. Services, Network Policies, and logs reference this IP.
    *   **Stateless Scaling:** Pods are ephemeral; their IP is their identity *while running*. Re-scheduling creates a new Pod with a new IP.
    *   **No Port Conflicts:** Containers in a Pod share ports (since they use `localhost` to communicate). Different Pods can use the same port numbers (e.g., multiple Pods all running on port 8080).
*   **Critical Implications:**
    *   **Flat Network:** All Pods can communicate with all other Pods *across nodes* **without NAT**. The network "looks" like a single subnet (logically, even if physically segmented).
    *   **Pod Lifecycle = IP Lifecycle:** When a Pod dies, its IP is released. A replacement Pod gets a *new* IP. **Services (Chapter 19.3) solve this discovery problem.**
    *   **Node Responsibility:** The *Node* (via CNI - Chapter 20) is responsible for ensuring Pod IPs are routable *within the cluster*. The Node's own IP is the gateway for Pod traffic leaving the node.
    *   **No Host Ports Needed:** Unlike Docker's default `bridge` network, you generally **do NOT** need to publish (`-p`/`hostPort`) container ports to the Node's IP for inter-Pod communication. Communication happens directly Pod IP -> Pod IP.
*   **How it Works Under the Hood:**
    1.  **Pod Creation:** Kubelet requests a network namespace for the Pod.
    2.  **CNI Invocation:** Kubelet calls the configured CNI plugin (Chapter 20).
    3.  **IP Assignment:** CNI plugin allocates an IP from the cluster's Pod CIDR range (e.g., `10.244.0.0/16`) to the Pod's network namespace.
    4.  **Network Setup:** CNI plugin configures virtual Ethernet pairs (veth), routes, and firewall rules on the Node to make the Pod IP reachable cluster-wide.
    5.  **Result:** All containers in the Pod share this IP and network namespace. `eth0` inside the container = the Pod's network interface.

#### **2. Container-to-Container (Same Pod) Communication**
*   **Mechanism:** Uses the **`localhost`** interface (`127.0.0.1`) within the **shared network namespace** of the Pod.
*   **Why it Works:**
    *   All containers in a Pod share the same Linux network namespace, IPC namespace, and often the same UTS namespace.
    *   Processes in different containers see each other as if they were on the same OS (same `netstat`, same `ip addr` output for `lo`).
*   **Key Characteristics:**
    *   **Zero Network Overhead:** Communication happens entirely within the kernel's loopback device (`lo`). No physical/virtual network traversal.
    *   **Port Sharing:** Containers *must* use different ports within the Pod (e.g., Container A: `:8080`, Container B: `:9090`). They communicate via `localhost:<port>`.
    *   **Use Cases:**
        *   **Sidecar Pattern:** Logging agent (scrapes logs from main app via `localhost`), proxy (Envoy/Istio), helper utilities.
        *   **Adapter Pattern:** Transforming data before it reaches the main app.
        *   **Batch Co-Processors:** Auxiliary services tightly coupled to the main workload.
*   **Critical Consideration:** Since it's `localhost`, **Network Policies (Chapter 21) DO NOT APPLY** to traffic within the same Pod. Security must be handled at the application/container level.

#### **3. Pod-to-Pod Communication**
*   **The Goal:** Enable any Pod to communicate with any other Pod *anywhere in the cluster* using their Pod IPs, without NAT.
*   **How it Works (The Packet Journey):**
    1.  **Source Pod:** App sends packet to `Destination Pod IP:Port`.
    2.  **Source Node Routing:** The Pod's network namespace routes the packet to the Node's root network namespace via the veth pair.
    3.  **Node Routing Table:** The Node checks its kernel routing table:
        *   If Destination Pod IP is on *this same Node*: Packet is routed directly to the destination Pod's veth interface (within the Node).
        *   If Destination Pod IP is on *another Node*: Packet is routed to the **CNI Overlay Network Interface** (e.g., `flannel.1`, `caliXXXX`, `cilium_host`).
    4.  **Overlay Network Encapsulation (Critical Step):** The CNI plugin encapsulates the original Pod-to-Pod packet inside a new packet:
        *   **Source IP:** *Source Node's Physical IP*
        *   **Destination IP:** *Destination Node's Physical IP*
        *   **Protocol:** Often UDP (VXLAN, GENEVE, Flannel UDP) or IP (IPIP, Calico IPIP/BGP). Cilium uses eBPF for direct routing or encapsulation.
        *   **Payload:** The original Pod-to-Pod packet (Source Pod IP -> Dest Pod IP).
    5.  **Physical Network Transit:** The encapsulated packet travels over the underlying physical/datacenter network (L3) to the destination Node. **Standard network routing rules apply here (firewalls, routers).**
    6.  **Destination Node Decapsulation:** The CNI plugin on the destination Node receives the encapsulated packet, strips off the outer header, and extracts the original Pod-to-Pod packet.
    7.  **Destination Node Routing:** The Node routes the original packet (Dest Pod IP) to the correct destination Pod's veth interface via its internal routing table.
    8.  **Destination Pod:** Packet arrives at the target container's network stack.
*   **Key Concepts & Nuances:**
    *   **Source IP Preservation:** The *original Pod IP* is preserved end-to-end. The destination Pod sees traffic coming directly from the source Pod IP, **NOT** the Node IP. (Crucial for logging, authz).
    *   **No NAT:** Kubernetes networking **does not use NAT** for Pod-to-Pod traffic *within the cluster*. (NAT is used for traffic *leaving* the cluster - egress).
    *   **CNI Dependency:** The *specific* encapsulation method (VXLAN, BGP, etc.) and routing is **100% determined by the CNI plugin** (Chapter 20).
    *   **Performance Impact:** Encapsulation adds overhead (CPU, packet size/MTU). Direct routing (BGP, eBPF) minimizes this.
    *   **Firewall Rules:** Physical network firewalls *must* allow the encapsulation protocol/ports (e.g., UDP 4789 for VXLAN) between *all Kubernetes Nodes*.

#### **4. Service Networking: Abstracting Pod Volatility**
*   **The Problem:** Pods are ephemeral (crash, scale, update). Their IPs change constantly. Applications need a stable endpoint to connect to.
*   **The Solution: Services** - An abstract way to expose a set of Pods as a single network endpoint.
*   **Core Types & How They Work:**
    *   **ClusterIP (Default):**
        *   **What:** Exposes the Service on a cluster-internal IP. Only reachable from within the cluster.
        *   **Mechanism:**
            1.  A stable, virtual IP (ClusterIP) is assigned from a dedicated Service CIDR (e.g., `10.96.0.0/12`).
            2.  **kube-proxy** (runs on every Node) watches the API server for Service/Endpoint changes.
            3.  For each Service, kube-proxy sets up **iptables** (default) or **IPVS** rules on *every Node*.
            4.  **Rule Action:** When traffic hits the *ClusterIP:Port* on *any Node*, the rules **DNAT** (Destination NAT) the traffic to one of the *current healthy Pod IPs* backing the Service (selected via SessionAffinity or random).
            5.  **Source IP:** The *original client Pod IP* is preserved (unless `externalTrafficPolicy: Local` is set - see below).
        *   **Key Nuance:** The ClusterIP *only exists as iptables/IPVS rules*. It's not assigned to any network interface. `ping`ing it fails; only TCP/UDP to the specific port works.
    *   **NodePort:**
        *   **What:** Exposes the Service on a static port (`nodePort`, e.g., 30000-32767) on *each Node's IP*.
        *   **Mechanism:**
            1.  Allocates a ClusterIP (as above).
            2.  kube-proxy sets up rules so that traffic to *any Node's IP:nodePort* is DNAT'ed to the ClusterIP:ServicePort, then to a Pod IP (as in ClusterIP).
        *   **Use Case:** Direct access from outside the cluster (if Nodes are publicly reachable). **Not recommended for production external access** (use LoadBalancer/Ingress instead).
    *   **LoadBalancer:**
        *   **What:** Exposes the Service externally using a cloud provider's load balancer (AWS NLB, GCP CLB, Azure LB).
        *   **Mechanism:**
            1.  Allocates a ClusterIP and NodePort (as above).
            2.  The **Cloud Controller Manager** (CCM) detects the Service type.
            3.  CCM instructs the cloud provider to create a load balancer.
            4.  The LB is configured to forward traffic to *all healthy Nodes* on the Service's `nodePort`.
            5.  Traffic flows: Client -> Cloud LB -> Node (nodePort) -> ClusterIP -> Pod.
        *   **Critical Nuance - Source IP:**
            *   `externalTrafficPolicy: Cluster` (Default): Traffic goes through *any* Node. **Source IP is SNAT'ed to the Node IP.** LB health checks use NodePort.
            *   `externalTrafficPolicy: Local`: Traffic only goes to Nodes *running the Pod*. **Source IP preserved.** LB health checks only hit Nodes with Pods. Avoids a "hairpin" hop but can cause uneven load if Pods aren't evenly distributed.
    *   **ExternalName:**
        *   **What:** Maps the Service to a DNS name (e.g., `my.database.example.com`).
        *   **Mechanism:** kube-proxy sets up a DNS CNAME record in the cluster DNS (CoreDNS) for the Service name pointing to the external name. No proxying.
        *   **Use Case:** Accessing external services within the cluster using a consistent name.
*   **Core Service Components:**
    *   **Endpoints/EndpointSlice:** The *actual list* of Pod IPs and ports backing a Service. Managed automatically by the control plane based on the Service's `selector`. **Endpoints are the bridge between the stable Service IP and the volatile Pod IPs.**
    *   **kube-proxy:** The critical agent on every Node that implements the Service virtual IP via iptables/IPVS. **Without kube-proxy, Services DO NOT WORK.**
    *   **CoreDNS:** Provides DNS resolution for Service names (`<service>.<namespace>.svc.cluster.local` -> ClusterIP). **Essential for Service discovery.**
*   **Why Services Matter:** They provide **stable networking identity**, **load balancing**, **service discovery**, and **decoupling** from Pod lifecycle. They are the primary way applications talk to each other.

---

### **Chapter 20: CNI (Container Network Interface)**

#### **1. What is CNI?**
*   **Definition:** A **specification and set of tools** for configuring network interfaces in Linux containers. It's **NOT Kubernetes-specific**, but Kubernetes adopted it as its standard networking plugin mechanism.
*   **Core Purpose:** Define a *simple, pluggable interface* between container runtimes (like containerd, CRI-O) and networking solutions. When a container runtime needs to set up networking for a container (Pod), it calls the configured CNI plugin(s).
*   **How Kubernetes Uses CNI:**
    1.  **kubelet** is configured with a CNI config directory (e.g., `/etc/cni/net.d`).
    2.  When a Pod is scheduled to a Node, **kubelet** invokes the container runtime (CRI).
    3.  The **container runtime** (via CRI) calls the **CNI plugin** specified in the config.
    4.  The CNI plugin:
        *   **ADD:** Allocates an IP, sets up veth pairs, routes, firewall rules. Returns the IP/netmask to the runtime.
        *   **DEL:** Cleans up networking when the Pod is deleted.
    5.  The runtime injects the network config (IP, routes) into the Pod's network namespace.
*   **Critical Distinction:** CNI handles **Pod-to-Pod networking within the cluster**. It does *not* handle:
    *   Service networking (kube-proxy/iptables)
    *   DNS (CoreDNS)
    *   Network Policies (Implemented *on top of* CNI by specific plugins)
    *   Ingress Controllers (Separate layer)
*   **Why CNI Exists:** Before CNI, container networking was a fragmented mess (Docker's built-in bridge, custom scripts). CNI provides vendor-neutral standardization.

#### **2. Popular CNI Plugins: Deep Comparison**

| Feature          | Flannel                                      | Calico                                      | Cilium                                      | Weave Net                                   |
| :--------------- | :------------------------------------------- | :------------------------------------------ | :------------------------------------------ | :------------------------------------------ |
| **Primary Tech** | **Overlay (VXLAN/UDP)**                      | **BGP (or IPIP Overlay)**                   | **eBPF (Native Routing or VXLAN)**          | **Overlay (FastDatapath - VXLAN)**          |
| **Routing**      | Simple overlay. Nodes form mesh.             | **Full BGP mesh** (or IPIP tunneling). Integrates with physical network. | **eBPF program** on veth. Direct routing or minimal encapsulation. | Proprietary "FastDatapath" (eBPF-like). Overlay by default. |
| **Performance**  | Good (VXLAN), Slower (UDP). MTU issues common. | **Excellent (BGP)**, Good (IPIP). Minimal overhead. | **Best-in-class (eBPF)**. Near-native speeds. Zero overhead for direct routing. | Very Good (FastDatapath). Better than basic VXLAN. |
| **Network Policy** | **NO native support** (Requires 3rd party like Calico) | **Yes (Full)**. Reference impl. Uses `felix` (Linux iptables) or eBPF dataplane. | **Yes (Full)**. Native eBPF enforcement. Highly efficient. | **Yes (Full)**. Uses `weave` router. |
| **Key Strength** | Simplicity, Ubiquity, Easy setup.            | **Scalability, Enterprise Features, BGP Integration**, Mature Policy. | **Performance, Security (L7), Observability**, Modern eBPF. | Simplicity + Policy, Good for small clusters. |
| **Key Weakness** | **No native Network Policies**, MTU headaches, Limited features. | Complexity (BGP), Larger footprint.         | Complexity (eBPF), Steeper learning curve.  | Less scalable than Calico/Cilium, Less mature than Calico. |
| **Best For**     | Getting started, Simple clusters, No Policy needed. | Large clusters, On-prem/Bare Metal, Need BGP, Strict Policy requirements. | High-performance needs, Deep security (L7), Cloud-native observability. | Small clusters, Quick setup with basic Policy. |

*   **Flannel Deep Dive:**
    *   **How:** Creates a `/24` subnet per Node from a cluster-wide Pod CIDR. Uses VXLAN (most common) to encapsulate Pod traffic between Nodes. Simple `flanneld` daemon manages subnet leases and routes.
    *   **Pros:** Dead simple, minimal config, works everywhere.
    *   **Cons:** **No Network Policies**, VXLAN adds overhead/MTU issues (`1450` vs standard `1500`), limited observability, single point of failure (`flanneld`).
*   **Calico Deep Dive:**
    *   **How:**
        *   **BGP Mode (Recommended):** `calico-node` pods run BGP (using `bird` daemon) to advertise Pod CIDR routes *directly* to the physical network (spine/leaf) or other Nodes. **No encapsulation!** Requires BGP-capable infrastructure.
        *   **IPIP Mode:** Encapsulates Pod traffic in IPIP tunnels between Nodes (like Flannel IPIP, but Calico manages routes via BGP internally).
    *   **Pros:** **Native Network Policies**, Excellent performance (BGP), Integrates with physical network, Mature, Highly scalable, Rich feature set (Egress Gateways, Host Endpoints).
    *   **Cons:** BGP configuration complexity, Larger resource footprint than Flannel.
*   **Cilium Deep Dive:**
    *   **How:** Uses **eBPF** loaded directly into the Linux kernel:
        *   **Direct Routing (Best):** Pod IPs are announced via BGP (using `bgp-agent`) or via `cilium` agent on Nodes. Traffic routed natively. eBPF handles policy, load balancing (replacing kube-proxy!), service translation.
        *   **VXLAN Mode:** Encapsulation for environments without direct routing support.
    *   **Pros:** **Unmatched performance**, **Native L3-L7 Network Policies**, **Built-in Service Mesh (Hubble)**, **Deep observability**, Replaces kube-proxy (Maglev, XDP), Zero-trust security model.
    *   **Cons:** Requires modern Linux kernel (v4.19.57+), Steeper learning curve, Complex debugging (though Hubble helps), Resource usage can be higher.
*   **Weave Net Deep Dive:**
    *   **How:** Creates an encrypted VXLAN overlay network. "FastDatapath" mode uses kernel modules (similar to eBPF) for faster packet processing within the overlay.
    *   **Pros:** Simple setup, Built-in encryption (optional), FastDatapath improves performance, Basic Network Policies.
    *   **Cons:** Overlay overhead, Less scalable than Calico/Cilium for large clusters, Policy implementation less mature/feature-rich.

#### **3. Choosing a CNI Plugin: Decision Framework**
*   **Critical Factors:**
    1.  **Network Policy Requirement?** If YES -> **Flannel is OUT**. Choose Calico, Cilium, or Weave Net.
    2.  **Environment:**
        *   *Cloud (AWS/GCP/Azure):* Cilium (performance/security), Calico (mature), Cloud-specific CNI (often limited).
        *   *On-Prem/Bare Metal:* **Calico (BGP)** is king for integration/performance. Cilium strong alternative.
        *   *Simple Test Cluster:* Flannel (if no Policy) or Weave Net (with basic Policy).
    3.  **Performance Needs:** Extreme perf -> **Cilium (eBPF)**. High perf -> **Calico (BGP)**. Moderate -> Others.
    4.  **Security Depth:** L7 Policies, Observability -> **Cilium**. Strong L3/L4 -> **Calico**.
    5.  **Operational Complexity:** Low -> Flannel/Weave Net. Medium -> Calico. High -> Cilium (but pays off).
    6.  **Existing Network:** BGP-capable fabric? -> **Calico BGP**. Limited L3? -> Overlay (Flannel, Calico IPIP, Cilium VXLAN).
    7.  **Future Proofing:** eBPF is the future -> **Cilium** has strongest commitment.
*   **Recommendation:**
    *   **Default Production Choice:** **Calico (BGP mode)**. Best balance of maturity, performance, policy, and on-prem integration.
    *   **Cutting-Edge/Cloud-Native:** **Cilium**. Ideal for high-scale, security-focused, observability-driven environments.
    *   **Avoid Flannel** if you need Network Policies (very common requirement).

#### **4. Network Policies with CNI**
*   **The Crucial Link:** The Kubernetes `NetworkPolicy` API resource **is just a specification**. **Enforcement is 100% dependent on the CNI plugin.**
*   **How it Works:**
    1.  You define a `NetworkPolicy` YAML (Chapter 21).
    2.  The Kubernetes API server stores it.
    3.  **The CNI plugin's policy controller** (e.g., Calico's `calico-policy-controller`, Cilium's `cilium-operator`) watches for `NetworkPolicy` changes.
    4.  The controller translates the high-level policy into **low-level enforcement rules** specific to its dataplane:
        *   *Calico (iptables):* Generates complex iptables rules on each Node.
        *   *Calico (BPF):* Loads eBPF programs.
        *   *Cilium:* Loads eBPF programs (highly efficient).
        *   *Weave Net:* Configures rules within the Weave router.
    5.  The dataplane (iptables, eBPF, router) **enforces the rules** on packet ingress/egress for Pods.
*   **Critical Implications:**
    *   **No CNI Policy Support = No Network Policies:** Flannel *cannot* enforce `NetworkPolicy` objects. You *must* choose a policy-capable CNI (Calico, Cilium, Weave Net).
    *   **Policy Implementation Varies:** Behavior (especially regarding default-deny, egress, L7) can differ slightly between plugins. **Always test policies!**
    *   **Performance Impact:** Enforcement has overhead. eBPF (Cilium, Calico BPF) has significantly lower overhead than iptables (Calico default).
    *   **Plugin Choice Dictates Policy Capabilities:** Only Cilium supports L7 (HTTP/gRPC) policies natively via eBPF. Calico requires an App Policy extension (not core).

---

### **Chapter 21: Network Policies**

#### **1. Enforcing Pod Communication Rules: The Zero Trust Principle**
*   **Core Concept:** `NetworkPolicy` is a Kubernetes API object that **specifies *what* network traffic is allowed *to* (ingress) and *from* (egress) a set of Pods.** It implements **micro-segmentation** at the Pod level.
*   **Default Behavior (The Problem):**
    *   **All Ingress Allowed:** Any Pod can be connected to by any other Pod (or external source).
    *   **All Egress Allowed:** Any Pod can connect to any other Pod, Service, or external destination.
    *   **This is INSECURE!** Violates Zero Trust ("never trust, always verify").
*   **NetworkPolicy Solution:**
    *   **Default-Deny is Key:** The *first* step to security is defining policies that **deny all traffic by default**, then *explicitly allow* only required communication.
    *   **Pod-Centric:** Policies select Pods (using labels) and define rules *for those Pods*.
    *   **Stateful:** Rules apply to connections. If ingress is allowed, the response traffic is automatically allowed (no need for explicit egress rule for responses).
*   **Why It Matters:** Prevents lateral movement during breaches, enforces least privilege, meets compliance requirements (PCI-DSS, HIPAA), enables multi-tenancy.

#### **2. Ingress and Egress Rules: The Building Blocks**
*   **Ingress Rules (`ingress` field):** Control **traffic *entering* the selected Pods.**
    *   **`from`:** Specifies *where* allowed traffic originates. Can be:
        *   `podSelector`: Other Pods (within same namespace).
        *   `namespaceSelector`: All Pods in a specific Namespace.
        *   `ipBlock`: Specific IP ranges (CIDRs - **use cautiously!** Pods IPs are dynamic).
        *   *Combination:* `podSelector` + `namespaceSelector` (AND logic).
    *   **`ports`:** Specifies *which* ports/protocols are allowed (`protocol: TCP/UDP/SCTP`, `port: <number/name>`, `endPort`).
*   **Egress Rules (`egress` field):** Control **traffic *leaving* the selected Pods.**
    *   **`to`:** Specifies *where* allowed traffic is destined. Same selectors as `from` (Pods, Namespaces, IP Blocks).
    *   **`ports`:** Specifies *which* ports/protocols are allowed for egress.
*   **Critical Nuances:**
    *   **No Rules = No Change:** A Policy with *only* `podSelector` but *no* `ingress`/`egress` rules **does nothing**. It selects Pods but allows all traffic.
    *   **Empty `from`/`to` = ALL:** An empty `from` in an ingress rule means "allow from *anywhere*". **This is often a critical mistake!** To deny all ingress, you need a policy *without* an `ingress` section (or with an empty `ingress` list `[]`).
    *   **IP Blocks are Fragile:** Using `ipBlock` for Pod IPs is **highly discouraged** because Pod IPs change. Use `podSelector`/`namespaceSelector` instead. IP Blocks are mainly for external traffic (egress to known API endpoints) or node IPs (use `nodeName` cautiously).
    *   **Egress is Often Overlooked:** Many focus only on ingress. Egress control is equally vital (e.g., preventing data exfiltration, limiting blast radius).

#### **3. Policy Types (Allow/Deny) & Default Behavior**
*   **There is NO explicit "Deny" Rule:** NetworkPolicy **only specifies *allowed* traffic.** Denial is **implicit**.
*   **The Golden Rules:**
    1.  **No Policies Exist:** **ALL ingress and egress allowed.** (Insecure default).
    2.  **At Least One Policy Exists for a Pod:**
        *   **Ingress:** Traffic is **DENIED** unless *at least one* `NetworkPolicy`'s `ingress` rules *explicitly allow it*.
        *   **Egress:** Traffic is **DENIED** unless *at least one* `NetworkPolicy`'s `egress` rules *explicitly allow it*.
    3.  **Default-Deny Policy:** A policy selecting Pods with **NO `ingress` rules** = **DENY ALL INGRESS**. A policy with **NO `egress` rules** = **DENY ALL EGRESS**.
*   **Policy Interaction (AND Logic):**
    *   **Multiple Policies on Same Pod:** Rules from *all* applicable policies are **combined (ANDed)**. Traffic must be allowed by *at least one rule* in *each* relevant policy type (ingress/egress).
    *   **Example:** Pod has Policy A (allows ingress on 80 from App) and Policy B (allows ingress on 443 from App). Ingress on 80 *and* 443 from App is allowed. Ingress on 8080 is denied.
*   **Namespace Isolation:** Policies *only* select Pods within their own Namespace using `podSelector`. To select Pods in *other* Namespaces, you **MUST** use `namespaceSelector` *in combination with* `podSelector`.

#### **4. Use Cases: Zero-Trust Networking & Multi-Tenancy**
*   **Zero-Trust Networking Implementation:**
    1.  **Default-Deny Namespace:** Apply a "default-deny" policy to *every* Namespace:
        ```yaml
        apiVersion: networking.k8s.io/v1
        kind: NetworkPolicy
        metadata:
          name: default-deny
        spec:
          podSelector: {} # Selects ALL Pods in the Namespace
          policyTypes:
          - Ingress
          - Egress
        ```
        *   **Effect:** Denies *all* ingress and egress for *every* Pod in the Namespace.
    2.  **Explicit Allow Policies:** For each application tier, create policies *only* allowing *necessary* traffic:
        *   *Frontend:* Allow ingress from LoadBalancer (via `ipBlock` cautiously or Service Mesh) + egress to `backend` Pods on port 8080.
        *   *Backend:* Allow ingress *only* from `frontend` Pods on port 8080 + egress to `database` Pods on port 5432.
        *   *Database:* Allow ingress *only* from `backend` Pods on port 5432 + **NO EGRESS** (or minimal egress for monitoring).
    3.  **L7 Security (Cilium):** Add HTTP rules to `backend` policy: Only allow `GET /api/*` from `frontend`, deny `DELETE`.
*   **Multi-Tenancy (Isolating Teams/Workloads):**
    *   **Scenario:** Team A (`team-a` namespace) and Team B (`team-b` namespace) share a cluster. They should not communicate.
    1.  **Default-Deny per Namespace:** Apply the `default-deny` policy (above) to *both* `team-a` and `team-b` namespaces.
    2.  **Allow Team A Internal Comms:**
        ```yaml
        apiVersion: networking.k8s.io/v1
        kind: NetworkPolicy
        metadata:
          name: allow-team-a
          namespace: team-a
        spec:
          podSelector: {} # All Pods in team-a
          ingress:
          - from:
            - namespaceSelector:
                matchLabels:
                  kubernetes.io/metadata.name: team-a # Selects team-a NS
            ports:
            - protocol: TCP
              port: 80
        ```
    3.  **Allow Team B Internal Comms:** (Similar policy in `team-b` namespace).
    4.  **Result:** Pods in `team-a` can talk *only* to other Pods in `team-a`. Pods in `team-b` can talk *only* to other Pods in `team-b`. **No cross-namespace communication.** (Unless explicitly allowed via additional policies).
    *   **Advanced:** Use `namespaceSelector` to allow specific communication between teams (e.g., `team-a` frontend to `team-b` API).

---
