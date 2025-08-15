# 📘 **Complete Kubernetes Course: From Beginner to Expert**  
**Table of Contents**

---

## 📗 **Part 1: Introduction & Foundations**

### Chapter 1: Introduction to Containerization
- What is Containerization?
- Docker Overview (Images, Containers, Dockerfile, Docker Compose)
- Why Containers?
- Limitations of Managing Containers Manually

### Chapter 2: Introduction to Kubernetes
- What is Kubernetes? (K8s)
- History and Evolution (Google Borg → Kubernetes)
- Key Features and Benefits
- Use Cases: Microservices, Cloud-Native Apps, DevOps
- Kubernetes vs Docker Swarm vs Nomad

### Chapter 3: Kubernetes Ecosystem
- CNCF (Cloud Native Computing Foundation)
- Kubernetes Distributions (OpenShift, EKS, GKE, AKS, Rancher, k3s)
- Related Tools: Helm, Istio, Prometheus, Fluentd, etc.

---

## 📗 **Part 2: Kubernetes Architecture & Components**

### Chapter 4: Kubernetes Architecture
- Master Node (Control Plane) Components:
  - API Server
  - etcd
  - Scheduler
  - Controller Manager
  - Cloud Controller Manager
- Worker Node Components:
  - Kubelet
  - Kube-proxy
  - Container Runtime (Docker, containerd, CRI-O)
- Add-ons: DNS (CoreDNS), Dashboard, Ingress Controllers

### Chapter 5: Kubernetes API & Declarative Model
- Understanding the Kubernetes API
- Declarative vs Imperative Configuration
- YAML and JSON Manifests
- `kubectl` CLI Basics (`get`, `describe`, `create`, `apply`, `delete`, `logs`, `exec`)

---

## 📗 **Part 3: Core Kubernetes Objects**

### Chapter 6: Pods
- What is a Pod?
- Pod Lifecycle
- Multi-Container Pods (Sidecar, Ambassador, Adapter patterns)
- Pod Networking and Storage
- Init Containers

### Chapter 7: Labels, Selectors & Annotations
- Labeling Resources
- Using Selectors (Label & Field Selectors)
- Annotations: Metadata vs Labels

### Chapter 8: Deployments
- What is a Deployment?
- Rolling Updates and Rollbacks
- Deployment Strategies (Rolling, Recreate, Blue/Green, Canary)
- Scaling Deployments
- Pausing and Resuming Deployments

### Chapter 9: ReplicaSets
- Purpose and Function
- Relationship with Deployments
- Manual ReplicaSet Management

### Chapter 10: Services
- Service Types:
  - ClusterIP
  - NodePort
  - LoadBalancer
  - ExternalName
- Service Discovery
- Endpoints and EndpointSlices
- Headless Services

### Chapter 11: Ingress
- What is Ingress?
- Ingress Controller (Nginx, Traefik, AWS ALB, etc.)
- Ingress Rules, Paths, Hosts
- TLS/SSL Termination
- Path-based and Name-based Routing
- Advanced Ingress (Rewrites, Rate Limiting, Authentication)

### Chapter 12: ConfigMaps & Secrets
- Storing Configuration with ConfigMaps
- Using Secrets (Opaque, TLS, Docker Registry)
- Security Best Practices for Secrets
- Secret Encryption at Rest
- Mounting ConfigMaps and Secrets as Volumes or Environment Variables

---

## 📗 **Part 4: Storage & Volumes**

### Chapter 13: Volumes
- Types of Volumes:
  - emptyDir
  - hostPath
  - configMap
  - secret
  - persistentVolumeClaim
  - gitRepo
  - downwardAPI
- Volume Mounts and SubPaths

### Chapter 14: Persistent Storage
- Persistent Volumes (PV)
- Persistent Volume Claims (PVC)
- Static vs Dynamic Provisioning
- Storage Classes
- Access Modes (ReadWriteOnce, ReadOnlyMany, ReadWriteMany)
- Reclaim Policies (Retain, Recycle, Delete)

### Chapter 15: Storage in Practice
- Using NFS, iSCSI, AWS EBS, GCP PD, Azure Disk
- CSI (Container Storage Interface) Drivers
- Local Persistent Volumes
- Stateful Applications (Databases) on Kubernetes

---

## 📗 **Part 5: Stateful Workloads**

### Chapter 16: StatefulSets
- When to Use StatefulSets
- Ordered Deployment and Scaling
- Stable Network Identifiers (Pod DNS)
- Stable Persistent Storage
- Use Cases: Databases (MySQL, MongoDB, Kafka, ZooKeeper)

### Chapter 17: DaemonSets
- Running Pods on Every Node (or Subset)
- Use Cases: Logging Agents, Monitoring, Networking
- Node Affinity and Tolerations

### Chapter 18: Jobs & CronJobs
- One-off Tasks with Jobs
- Parallelism and Completions
- CronJobs for Scheduled Tasks
- Job Patterns: Finite Workloads, Batch Processing

---

## 📗 **Part 6: Networking in Kubernetes**

### Chapter 19: Kubernetes Networking Model
- Pod-to-Pod Communication
- Container-to-Container (Same Pod)
- Service Networking
- IP-per-Pod Model

### Chapter 20: CNI (Container Network Interface)
- What is CNI?
- Popular CNI Plugins:
  - Calico
  - Cilium
  - Flannel
  - Weave Net
- Choosing a CNI Plugin
- Network Policies with CNI

### Chapter 21: Network Policies
- Enforcing Pod Communication Rules
- Ingress and Egress Rules
- Policy Types (Allow/Deny)
- Use Cases: Zero-Trust Networking, Multi-Tenancy

---

## 📗 **Part 7: Scheduling & Resource Management**

### Chapter 22: Scheduling Basics
- How the Scheduler Works
- Default Scheduling Behavior
- Node Affinity & Pod Affinity/Anti-Affinity
- Taints and Tolerations
- Node Selectors

### Chapter 23: Resource Management
- CPU and Memory Requests & Limits
- Quality of Service (Guaranteed, Burstable, BestEffort)
- Vertical Pod Autoscaler (VPA)
- Resource Quotas
- LimitRanges

### Chapter 24: Horizontal Pod Autoscaler (HPA)
- Scaling Based on CPU/Memory
- Custom and External Metrics (Prometheus, Datadog)
- Metrics Server Setup
- HPA with Custom Metrics

### Chapter 25: Cluster Autoscaler
- Auto-Scaling Nodes in the Cluster
- Integration with Cloud Providers (AWS, GCP, Azure)
- Cost Optimization

---

## 📗 **Part 8: Security & Identity**

### Chapter 26: Kubernetes Security Model
- Defense in Depth
- Principle of Least Privilege
- Secure Defaults

### Chapter 27: Authentication
- Client Certificates
- Static Token Files
- Bootstrap Tokens
- OpenID Connect (OIDC)
- Webhook Token Authentication

### Chapter 28: Authorization
- ABAC (Attribute-Based Access Control)
- RBAC (Role-Based Access Control)
  - Roles & ClusterRoles
  - RoleBindings & ClusterRoleBindings
- Best Practices for RBAC

### Chapter 29: Admission Control
- What are Admission Controllers?
- Built-in Controllers (AlwaysPullImages, ResourceQuota, etc.)
- Dynamic Admission Control (Webhooks)
  - ValidatingAdmissionWebhook
  - MutatingAdmissionWebhook

### Chapter 30: Pod Security
- Pod Security Standards (Baseline, Restricted, Privileged)
- Pod Security Admission (PSA) - Replacing PodSecurityPolicy
- Pod Security Policies (Deprecated)
- Seccomp, AppArmor, SELinux Profiles
- Preventing Privileged Containers

### Chapter 31: Network Security
- Securing API Server
- Securing etcd
- TLS Bootstrapping
- Audit Logging
- Firewalling and Network Segmentation

---

## 📗 **Part 9: Observability & Monitoring**

### Chapter 32: Logging
- Centralized Logging Architecture
- Sidecar Logging Patterns
- Tools: Fluentd, Fluent Bit, Logstash
- Shipping Logs to: Elasticsearch, Loki, Cloud Providers

### Chapter 33: Monitoring
- Metrics in Kubernetes
- Metrics Server
- Prometheus Architecture
- Exporters (Node Exporter, cAdvisor, kube-state-metrics)
- ServiceMonitors and PodMonitors (Prometheus Operator)
- Grafana Dashboards

### Chapter 34: Tracing
- Distributed Tracing Overview
- Jaeger, Zipkin
- OpenTelemetry Integration

### Chapter 35: Debugging & Troubleshooting
- Common Issues and Fixes
- Using `kubectl describe`, `logs`, `exec`
- Diagnosing CrashLoopBackOff, ImagePullBackOff, etc.
- Debugging Networking and DNS
- Using ephemeral containers

---

## 📗 **Part 10: Configuration & Templating**

### Chapter 36: Helm
- Introduction to Helm (Package Manager for K8s)
- Helm Charts (Structure, Templates, Values)
- Helm Repositories
- Installing, Upgrading, Rolling Back Releases
- Helm Hooks
- Creating Custom Charts
- Helmfile for Multi-Environment Management

### Chapter 37: Kustomize
- Declarative Configuration Management
- Patches, Bases, Overlays
- Generating Resources (ConfigMaps, Secrets)
- Comparing Kustomize vs Helm

### Chapter 38: Configuration Best Practices
- Immutable Infrastructure
- GitOps Principles
- Environment-Specific Configurations
- Using Argo CD, Flux

---

## 📗 **Part 11: Advanced Workloads & Operators**

### Chapter 39: Custom Resources & CRDs
- Extending the Kubernetes API
- Creating and Using CRDs
- Validation with OpenAPI Schema
- Versioning CRDs

### Chapter 40: Operators
- What is an Operator?
- Operator Pattern (Controller + CRD)
- Operator SDK / Kubebuilder
- Building an Operator (Example: Database Operator)
- Community Operators (etcd, Prometheus, etc.)

---

## 📗 **Part 12: Cluster Operations & Administration**

### Chapter 41: Cluster Setup & Bootstrapping
- Using `kubeadm` to Create Clusters
- High Availability (HA) Clusters
- etcd Clustering and Backup
- Upgrading Control Plane and Nodes

### Chapter 42: Managed Kubernetes (EKS, GKE, AKS)
- Differences Between Managed Services
- IAM Integration
- Networking Setup
- Upgrades and Maintenance

### Chapter 43: Self-Hosted Clusters
- Bare Metal Setup
- Using k3s, MicroK8s, K0s
- Edge Computing with Lightweight K8s

### Chapter 44: Cluster Maintenance
- Node Drain and Cordon
- Certificate Rotation
- etcd Backup and Restore
- Disaster Recovery Strategies

---

## 📗 **Part 13: CI/CD & GitOps**

### Chapter 45: CI/CD with Kubernetes
- Integrating CI/CD (Jenkins, GitHub Actions, GitLab CI)
- Building and Pushing Images
- Deploying to Kubernetes (Helm, Kustomize, kubectl)
- Canary Deployments with Argo Rollouts
- Blue/Green Deployments

### Chapter 46: GitOps
- What is GitOps?
- Principles: Declarative, Versioned, Automated
- Tools: Argo CD, Flux
- Sync Modes (Auto, Manual)
- Rollback and Drift Detection

---

## 📗 **Part 14: Service Mesh & Advanced Networking**

### Chapter 47: Service Mesh Overview
- Problems Solved by Service Mesh
- Sidecar Proxy Pattern

### Chapter 48: Istio
- Architecture (Pilot, Citadel, Mixer, Envoy)
- Traffic Management (VirtualService, DestinationRule)
- Security (mTLS, AuthorizationPolicy)
- Observability (Telemetry, Tracing)
- Gateways and Ingress/Egress
- Canaries with Istio

### Chapter 49: Linkerd
- Lightweight Service Mesh
- Installation and Usage
- Comparison with Istio

### Chapter 50: Cilium & eBPF
- Modern Networking and Security with eBPF
- Cilium as CNI and Service Mesh
- Hubble for Observability
- Performance Benefits

---

## 📗 **Part 15: Multi-Cluster & Federation**

### Chapter 51: Multi-Cluster Management
- Reasons for Multi-Cluster (HA, Multi-Region, Tenancy)
- Cluster Federation (Kubernetes Cluster API)
- Tools: Karmada, Rancher, Anthos

### Chapter 52: Cluster API
- Declarative Cluster Lifecycle Management
- Machine, MachineSet, MachineDeployment
- Provisioning Clusters on AWS, GCP, etc.

---

## 📗 **Part 16: Serverless & Edge Computing**

### Chapter 53: Knative
- Serving: Serverless Workloads
- Eventing: Event-Driven Architectures
- Build (Deprecated): Source-to-Container

### Chapter 54: KEDA (Kubernetes Event-Driven Autoscaling)
- Scale to Zero
- Trigger-based Scaling (Kafka, RabbitMQ, Prometheus, etc.)

### Chapter 55: Edge Kubernetes
- Challenges at the Edge
- K3s, KubeEdge, OpenYurt
- Use Cases: IoT, Retail, 5G

---

## 📗 **Part 17: Real-World Patterns & Best Practices**

### Chapter 56: Production-Ready Kubernetes
- Hardening the Cluster
- Backup and Restore (Velero)
- Cost Monitoring (Goldilocks, Kubecost)
- Naming Conventions and Labels
- Namespace Management

### Chapter 57: Disaster Recovery & Backup
- Velero: Backup, Restore, Migration
- Snapshotting Volumes
- Cross-Cluster Recovery

### Chapter 58: Performance Tuning
- Optimizing etcd
- Kernel Tuning
- Network Optimization
- Reducing API Latency

### Chapter 59: Cost Optimization
- Right-Sizing Resources
- Spot Instances
- Autoscaling Strategies
- Monitoring with Kubecost

---

## 📗 **Part 18: Certification & Career**

### Chapter 60: CKAD (Certified Kubernetes Application Developer)
- Exam Topics
- Practice Labs
- Tips and Resources

### Chapter 61: CKA (Certified Kubernetes Administrator)
- Exam Topics
- Hands-on Practice
- Study Plan

### Chapter 62: Career in Kubernetes
- Roles: DevOps Engineer, SRE, Platform Engineer
- Salary Trends
- Building a Portfolio
- Open Source Contributions

---

## 📗 **Part 19: Projects & Hands-On Labs**

### Chapter 63: Mini Projects
- Deploy a Full-Stack App (React + Node.js + MongoDB)
- CI/CD Pipeline with GitHub Actions + Argo CD
- Secure Cluster with RBAC and Network Policies
- Monitor with Prometheus + Grafana
- Build a Custom Helm Chart
- Create a Simple Operator

### Chapter 64: Capstone Project
- Multi-Tier Application with:
  - Microservices
  - Database (StatefulSet)
  - Ingress + TLS
  - Monitoring + Logging
  - CI/CD + GitOps
  - Canary Deployment

---
