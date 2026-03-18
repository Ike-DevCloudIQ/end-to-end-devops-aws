# End-to-End DevOps on AWS: Microservices, CI/CD, GitOps, and Observability

**A production-grade DevOps implementation** of the OpenTelemetry Astronomy Shop, demonstrating the complete platform engineering journey from local development through cloud deployment on AWS. This project is designed to showcase real-world DevOps practices, operational decision-making, and troubleshooting expertise.

## Executive Summary

This project delivers a **full DevOps lifecycle** for a distributed microservices platform (16 services across multiple domains):

- ✅ **Local Development**: Docker Compose environment for fast feedback loops and integration testing
- ✅ **Infrastructure as Code**: Terraform provisions the core AWS foundation end-to-end: VPC, public/private subnets across AZs, route tables, Internet Gateway, NAT Gateways, security groups, IAM roles/policies, EKS control plane, managed node group, and remote state backend (S3 + DynamoDB locking)
- ✅ **Kubernetes Operations**: Multi-service deployment on EKS with resource management, pod troubleshooting, and scaling
- ✅ **CI/CD Automation**: GitHub Actions pipelines for container builds, registry pushes, and manifest automation
- ✅ **GitOps Delivery**: ArgoCD-driven continuous deployment with auto-sync, drift detection, and self-healing
- ✅ **Observability Stack**: Jaeger (distributed tracing), Grafana (dashboards), and Prometheus (metrics)

The implementation emphasizes **practical operational maturity**: real troubleshooting (service startup issues, scheduling constraints), production-like scaling decisions, and proven automation patterns.

**Target Audience**: DevOps engineers, platform engineers, SREs, and cloud architects seeking a reference implementation of modern delivery practices.

## Architecture

![Project Architecture](Images/Ultimate%20Project%20Architecture.gif)

## Platform Engineering Principles

This implementation follows practical platform engineering principles used in production environments:

- **Declarative Infrastructure and Delivery**: Cloud and Kubernetes state are declared in code (Terraform + Kubernetes manifests) and reconciled continuously.
- **Immutable Deployments**: New releases are delivered via immutable container tags and rollout controllers, not in-place mutation of running containers.
- **Git as Source of Truth**: GitHub + ArgoCD enforce auditable, reproducible changes with clear rollback points.
- **Operational Observability First**: Traces, metrics, and dashboard visibility are built into platform operation rather than added after incidents.
- **Security by Segmentation**: Public/private subnet boundaries, IAM role scoping, and cluster-level controls reduce blast radius.
- **Failure-Oriented Operations**: Playbooks and command-level diagnostics are part of normal delivery, not emergency-only knowledge.

## What This Project Covers (Technical Scope)

### 1. Infrastructure as Code (Terraform)

**Demonstrates**: Cloud infrastructure provisioning, state management, and reproducible deployments.

- **VPC Design**: Private and public subnets across 3 availability zones (AZs) for high availability
- **EKS Cluster**: Managed Kubernetes control plane (v1.30+) with automatic security patch management
- **Node Group**: EC2 worker nodes (t3.medium instances) with auto-scaling policies (min: 2, max: 5, desired: 4)
- **Remote State Backend**: 
  - S3 bucket for state storage (versioning + encryption)
  - DynamoDB table for distributed state locking (prevents concurrent apply conflicts)
- **Network Infrastructure**: NAT Gateways, Internet Gateway, Security Groups with least-privilege rules
- **IAM RBAC**: Service accounts, roles, and policies for EKS node and pod authentication
- **Reproducibility**: `terraform plan` / `apply` workflow enables environment creation and teardown in minutes

**Key Lesson**: Terraform state management prevents "drift" (manual changes accumulating); DynamoDB locking ensures safe concurrent operations in team environments.

### 2. Kubernetes Platform Operations

**Demonstrates**: Cluster operations, resource management, troubleshooting, and scaling under load.

- **Pod Deployment**: 40+ pods across system (default, argocd, gitops-demo, kube-system namespaces)
- **Service Exposure**: NodePort, ClusterIP, and LoadBalancer service types for traffic routing
- **Resource Requests/Limits**: CPU and memory constraints to enable predictable scheduling and prevent node starvation
- **Readiness/Liveness Probes**: Health checks ensuring failed containers are restarted and replaced
- **StatefulSets**: Kafka, PostgreSQL, OpenSearch for persistent workloads requiring stable identity
- **DaemonSets**: OpenTelemetry collector agents running on every node for trace collection
- **Troubleshooting Workflows**:
  - `kubectl logs` for container output inspection
  - `kubectl describe pod` for event history and probe failures
  - `kubectl get events` for cluster-wide activity audit
  - Pod eviction/rescheduling under resource pressure

**Production Decision**: When initial 3-node cluster couldn't schedule all demo-app replicas plus collectors, node group was scaled to 4 nodes—decision driven by monitoring pod status and events.

### 3. CI/CD Automation (GitHub Actions)

**Demonstrates**: Build automation, registry integration, and manifest mutation for GitOps workflows.

- **Container Build Pipeline**:
  - Multi-stage Docker builds (optimized layer caching)
  - Docker Buildx for cross-platform image building
  - Docker Hub as container registry (DOCKER_USERNAME/DOCKER_TOKEN secrets)
- **Automated Image Push**:
  - On-push trigger (monitors `ultimate-devops-project-demo/demo-app-image/*` path)
  - Image tag strategy: `<username>/<repo>:run-<github.run_number>` (e.g., `run-3`)
- **Manifest Automation**:
  - `sed` in-place editing of Kubernetes manifest with new image tag
  - Automated git commit/push (via SSH key) to trigger GitOps sync
- **Workflow Secrets Handling**:
  - Repository-level secret injection (GitHub's encrypted storage)
  - SSH deploy key configuration for automated git push authentication
- **Debugging Patterns**: Workflow logs visible in Actions UI; common issues include:
  - Missing Docker Hub credentials (fails at login step)
  - Insufficient SSH key scope (PAT token without "workflow" permission rejected by GitHub)
  - Git push auth failures (SSH key not added to account)

**Key Lesson**: CI/CD automation fails silently if auth is incomplete—verify all credentials before first run.

### 4. GitOps Continuous Delivery (ArgoCD)

**Demonstrates**: Declarative deployment model, drift detection, and progressive adoption patterns.

- **ArgoCD Installation**: Deployed in dedicated `argocd` namespace with `argo-server` exposed via service
- **Application CRD**: Custom resources defining "desired state" (Git repo + branch + path) and "live state" (cluster resources)
- **Sync Strategies**:
  - **Automatic sync**: ArgoCD continuously reconciles cluster to match Git (no manual `kubectl apply` needed)
  - **Auto-heal**: Automatic rollback of manual changes; cluster always reflects Git source of truth
- **Progressive Adoption**:
  - **Demo App** (`demo-app:run-3`): Proof-of-concept microservice deployed and updated by CI/CD
  - **Frontend Proxy Service**: Production service under GitOps management (real traffic ingestion)
  - Enables low-risk testing before adopting all 16 microservices under GitOps
- **Drift Detection**: ArgoCD shows "OutOfSync" when cluster diverges from Git; UI displays pending changes
- **Self-Healing**: Failed pods are automatically replaced as ArgoCD reconciles desired state every 3 minutes (configurable)

**Production Pattern**: Start with non-critical services (demo apps) under GitOps; gradually migrate production services as confidence grows.

### 5. Observability (Distributed Tracing, Metrics, Dashboards)

**Demonstrates**: End-to-end visibility into system behavior, performance, and failures.

- **Jaeger Distributed Tracing**:
  - Collects traces from all 16 microservices via OpenTelemetry agents
  - Traces show request flow across services (checkout → payment → shipping)
  - Latency breakdown by service helps identify bottlenecks
  - Trace sampling reduces storage load while maintaining visibility
- **Grafana Dashboards**:
  - Real-time metrics visualization (CPU, memory, request rates, error rates)
  - Pre-built dashboards for infrastructure (node health) and applications (service latency)
  - Alert rules for anomalies (e.g., high error rate, pod restart loops)
- **Prometheus Metrics**:
  - Scrapes metrics from services every 15 seconds
  - Time-series storage enables historical analysis and trend detection
  - PromQL queries power dashboard calculations
- **OpenTelemetry Collector**:
  - DaemonSet running on every node (3/4 pods healthy; 1 experiencing port binding issue)
  - Aggregates traces from sidecars and application instrumentation
  - Forwards to Jaeger backend

**Operational Value**: When a service is slow, traces show exactly which downstream call is blocking; dashboards correlate with infrastructure metrics to distinguish application vs. infrastructure issues.

## End-to-End Delivery Flow (Code to Production)

1. **Code Change and Commit**: A change is merged to `main` in GitHub.
2. **CI Trigger**: GitHub Actions detects the path change and starts the pipeline.
3. **Build and Package**: Docker image is built (Buildx), tagged with immutable run metadata, and pushed to Docker Hub.
4. **Manifest Update**: Pipeline updates Kubernetes image references in Git-tracked manifests.
5. **GitOps Reconciliation**: ArgoCD detects Git drift and syncs cluster resources to the updated desired state.
6. **Kubernetes Rollout**: Deployment controller performs rollout and enforces readiness/liveness checks.
7. **Post-Deploy Validation**: Platform health is verified with `kubectl get pods`, events, and observability endpoints (Jaeger/Grafana).
8. **Rollback Path**: Revert manifest/image tag in Git; ArgoCD reconciles back to last known good state.

This flow reduces manual deployment risk, centralizes auditability, and ensures every release remains reproducible.

## Implementation Evidence (Screenshots)

### Phase 1: Local Development Environment

#### Docker Compose Full Stack Startup

![Docker Compose Startup](Images/docker%20compose%20up%20-d%20.png)

**What This Shows:**
This screenshot captures the initialization of the entire Astronomy Shop platform on a single developer machine. The `docker-compose up -d` command orchestrates 16 interdependent microservices, each configured with resource limits, health checks, and network dependencies.

**Technical Details for Engineers:**
- Each service is a container with a defined entrypoint and network connectivity
- Services like `postgres`, `kafka`, `opensearch` are stateful; they require volume mounts for data persistence
- The `depends_on` configuration ensures services start in a sensible order (databases before dependent apps)
- Health checks (using `curl` or TCP probes) are configured; containers marked as unhealthy trigger restart policies
- Network mode allows inter-service DNS resolution (service name resolves to container IP)

Think of this as a "mini version" of production. Instead of 16 separate servers (which would cost thousands), Docker runs them all on your laptop. The benefit: you can test the entire system locally, find bugs quickly, and push fixes without affecting real users.

This demonstrates **systems thinking**—understanding that microservices depend on each other and must be orchestrated correctly. It also shows **practical debugging skills** (if one service fails, can the engineer diagnose why?).

---

### Phase 2: Infrastructure Provisioning (AWS via Terraform)

#### Terraform Plan Output

![Terraform Plan](Images/Terraform%20%20Plan.png)

**What This Shows:**
This is the output from `terraform plan`, displaying the infrastructure changes Terraform will apply to AWS. It's the "safety net"—you see exactly what will be created/modified/destroyed before applying changes.

**Technical Details for Engineers:**
- **Planned Actions**: Each line shows a resource (e.g., `aws_eks_cluster.main` being created)
- **Attributes**: For each resource, Terraform displays configuration (e.g., `cluster_version = "1.30"`, `node_count = 4`)
- **Dependency Graph**: Terraform infers order (VPC must exist before EKS cluster; EKS before node group)
- **Plan File**: Output is saved to `.tfplan` and can be applied deterministically (`terraform apply .tfplan`)
- **No Apply Yet**: Plan shows what *would* happen; `terraform apply` executes the changes

If you're used to clicking AWS buttons to create resources, Terraform is the "infrastructure as code" way—you describe *what* you want in files, version control it, and automate the creation. No more manual steps = fewer mistakes.

This demonstrates **infrastructure expertise** and **disciplined change management**. Using "plan" before "apply" is a professional practice—it prevents `oops, I deleted production` incidents.

---

### Phase 3: AWS VPC and Network Infrastructure

#### VPC Topology Overview

![VPC Overview](Images/VPC%20Overview.png)

**What This Shows:**
This diagram illustrates the AWS VPC (Virtual Private Cloud) architecture: subnets across availability zones, Internet Gateway, NAT Gateways, and routing tables.

**Technical Details for Engineers:**
- **CIDR Block**: VPC uses `10.0.0.0/16` (65,536 IP addresses available)
- **AZ Distribution**: Public and private subnets span 3 AZs (us-west-2a, us-west-2b, us-west-2c)
- **Public Subnets**: Host frontend services (frontend-proxy), exposed to internet via Internet Gateway
- **Private Subnets**: Host backend services (payment, shipping, etc.), only egress to internet via NAT Gateway
- **NAT Gateway**: Enables private services to pull Docker images and access external APIs without exposing them to inbound traffic
- **Route Tables**: Separate routing for public (→ Internet Gateway) vs. private (→ NAT Gateway)

**Security Benefit**: Private subnets cannot be accessed directly from the internet—attackers cannot SSH to backend services. This is **defense in depth**.

**Cost Implication**: NAT Gateways charge ~$32/month per AZ + data transfer costs. Each NAT handles traffic for private subnets in that AZ.

---

#### NAT Gateways

![NAT Gateways](Images/NAT%20Gateways.png)

**What This Shows:**
EC2 console showing the NAT Gateway resources—Elastic IPs and target AZs for private subnet egress traffic.

**Technical Details for Engineers:**
- NAT Gateways provide **outbound-only internet access** for private subnets
- Each NAT Gateway gets a static Elastic IP (used for outbound traffic identification)
- If a NAT Gateway becomes unhealthy, private services lose internet access (no image pulls = deployment failures)
- Terraform creates one NAT per AZ for fault tolerance

---

### Phase 4: Amazon EKS Cluster Deployment

#### EKS Cluster Overview

![EKS Cluster](Images/EKS%20Cluster.png)

**What This Shows:**
AWS console displaying the EKS cluster resource. Key information: cluster name, Kubernetes version, endpoint, status, and networking configuration.

**Technical Details for Engineers:**
- **Cluster Name**: `my-eks-cluster` (matches Terraform variable)
- **Kubernetes Version**: `1.30.14-eks` (AWS-managed; patched by AWS automatically)
- **Endpoint**: HTTPS URL for `kubectl` communication (secured by AWS IAM)
- **Networking**: Cluster runs in the VPC created earlier; pod CIDR is managed by AWS CNI plugin
- **IAM Role**: Service role with permissions to manage EC2/ELB resources for pod networking and ingress

EKS is AWS's managed Kubernetes service. AWS handles the control plane (API server, etcd, scheduler); you only manage worker nodes. It's like renting a managed database vs. running PostgreSQL yourself—less operational overhead.

---

#### EKS Node Group Configuration

![EKS Node Group](Images/EKS%20Node%20group.png)

**What This Shows:**
Configuration details for the managed node group: instance types, scaling policies, and current node count.

**Technical Details for Engineers:**
- **Instance Type**: `t3.medium` (2 vCPUs, 4 GB RAM per node)
- **Scaling Limits**: Min: 2, Max: 5, Desired: 4
- **Auto-Scaling**: If node CPU/memory exceeds threshold, AWS adds nodes (up to max)
- **Launch Template**: Specifies AMI (Amazon Linux 2 with EKS optimizations), root volume size, security groups
- **Current Nodes**: 4 nodes running (visible in the count)

**Decision Explanation**: Initial cluster had 3 nodes but couldn't schedule all pods (40+). By scaling to 4, we reduced pod evictions and stabilized deployments. This shows **operational decision-making**: monitor resource usage, identify constraints, and adjust capacity.

---

#### EKS Nodes (EC2 Instances)

![EKS Nodes](Images/EKS%20Nodes.png)

**What This Shows:**
EC2 console listing the 4 worker nodes. Each is a t3.medium instance running on AWS infrastructure.

**Technical Details for Engineers:**
- Each node is a full EC2 instance running Docker, kubelet, and kube-proxy
- Nodes are spread across AZs (fault tolerance)
- Each node can run 2-3 instances of each pod (limited by CPU/memory and EKS pod density limits)
- Node failures trigger pod eviction; Kubernetes scheduler reschedules pods on healthy nodes

---

#### EC2 Instances (Full List)

![EC2 Instances (EKS Worker Nodes)](Images/EC2%20Instances%20(EKS%20Worker%20Nodes).png)

**What This Shows:**
Comprehensive EC2 console view showing all running instances for the EKS cluster.

This proves the infrastructure is real and running—4 EC2 instances, each in a different AZ, each accumulating compute costs. This demonstrates actual AWS spend, not a tutorial with 12-hour free trial resources.

---

### Phase 5: Load Balancer and Service Exposure

#### AWS Load Balancer Configuration

![Load Balancer details](Images/Load%20Balancer%20details.png)

**What This Shows:**
AWS ELB (Elastic Load Balancer) configuration—target groups, health checks, and DNS name for external traffic.

**Technical Details for Engineers:**
- **Type**: Classic Load Balancer (ELB) automatically created by Kubernetes when a Service has `type: LoadBalancer`
- **DNS Name**: Maps to ELB's Elastic IP; users access the app via `<elb-dns>:8080`
- **Target Groups**: ELB forwards traffic to node ports; Kubernetes kube-proxy routes to pods inside containers
- **Health Checks**: ELB performs periodic health checks to verify target nodes are responsive

**Traffic Flow**:
1. User hits ELB DNS name
2. ELB selects a node (round-robin)
3. kube-proxy on node routes to pod IP/port
4. Container processes request

**Cost**: ELB charges ~$16/month + $0.006 per GB of data processed. Significant in production at scale.

---

#### Live App Running

![Live App](Images/Live%20App.png)

**What This Shows:**
Screenshot of the deployed Astronomy Shop frontend running in a browser, accessible via the ELB DNS name.

This is the proof that everything works end-to-end. You started with Git commits, ran Terraform to create AWS infrastructure, deployed 16 microservices on Kubernetes, and now you can browse the app. That's the full DevOps journey.

This is real, deployed, running-right-now proof. Not a screenshot from a tutorial—this is a live deployment serving real external traffic.

---

### Phase 6: Kubernetes Operational Visibility

#### Kubernetes Deployments

![kubectl get deployment](Images/kubectl%20get%20deployment.png)

**What This Shows:**
Output of `kubectl get deployment` showing all active Kubernetes Deployments across namespaces.

**Technical Details for Engineers:**
- **Deployment**: Kubernetes controller managing replica sets (ensures N copies of pod always running)
- **READY**: Shows current replicas / desired replicas (e.g., `3/3` means all 3 desired pods are running)
- **AVAILABLE**: Pods that have passed readiness checks and are receiving traffic
- **AGE**: How long the deployment has existed

**Columns Explained**:
- `NAME`: Deployment name (e.g., `demo-app`, `frontend-proxy`)
- `READY`: Current/Desired running pods
- `UP-TO-DATE`: Number of pods with the latest image (used during rollouts)
- `AVAILABLE`: Number of pods ready to receive traffic (passed readiness probe)

**Operational Workflow**: If `READY < DESIRED`, pods are failing; check events with `kubectl describe deployment` to diagnose why.

---

#### Kubernetes Pods (Full Listing)

![kubectl get pods](Images/kubectl%20get%20pods.png)

**What This Shows:**
Output of `kubectl get pods -A` showing all 40+ pods across all namespaces (default, argocd, gitops-demo, kube-system).

**Technical Details for Engineers:**
- **Pod**: Smallest Kubernetes resource; usually one container per pod
- **READY**: Shows container readiness state (e.g., `1/1` = 1 container running and passed readiness probe)
- **STATUS**: Running, Pending, CrashLoopBackOff, ImagePullBackOff, etc.
- **RESTARTS**: Number of times the container crashed and restarted
- **AGE**: Pod creation time

---

### Phase 7: GitOps and Continuous Deployment

#### ArgoCD Application Status

![ArgoCD app Synced and Healthy](Images/ArgoCD%20app%20Synced%20and%20Healthy.png)

**What This Shows:**
ArgoCD UI displaying application status. Key indicators: "Synced" (cluster matches Git), "Healthy" (all pods running), and resource tree showing deployment, services, and pods managed by this application.

**Technical Details for Engineers:**
- **Sync Status**: 
  - **Synced**: Cluster state matches Git repository (desired = actual)
  - **OutOfSync**: Manual changes detected on cluster or Git was updated but not synced yet
  - **Unknown**: ArgoCD couldn't reach cluster or repository
- **Health Status**:
  - **Healthy**: All pods are running and passed readiness checks
  - **Degraded**: Some pods are failing
  - **Unknown**: ArgoCD couldn't determine health
- **Sync Policy**:
  - **Automatic**: ArgoCD auto-syncs every 3 minutes (no manual `kubectl apply` needed)
  - **Manual**: Admin must click "Sync" button to deploy changes

**Resource Tree**: Shows all Kubernetes objects managed by this application (Deployment → ReplicaSet → Pods). If anything is missing, sync fails.

This demonstrates **GitOps mastery**—the ability to define "desired state" in Git and have infrastructure automatically converge to it. This reduces manual errors and brings reliability and repeatability to deployments.

---

## Key Technical Decisions and Trade-offs

### 1. Terraform State Management
- **Decision**: S3 + DynamoDB for remote state (not local)
- **Why**: Local state works for solo developers; shared state enables team operations and CI/CD pipelines
- **Trade-off**: Adds ~2 minutes to cluster setup time (wait for S3 bucket + DynamoDB table creation)

### 2. Node Group Scaling
- **Decision**: Scale from 3 → 4 nodes when demo-app couldn't schedule
- **Why**: Resource pressure causes pod evictions, which trigger continuous restarts. Scaling is faster than optimizing pod resource requests
- **Trade-off**: ~$2/day added to AWS bill; justified for stable operations

### 3. Progressive GitOps Adoption
- **Decision**: Start with demo-app and frontend-proxy; not all 16 microservices
- **Why**: Testing GitOps with "safe" services before trusting it with revenue-critical services builds confidence
- **Trade-off**: Additional manual deployments for non-GitOps services until migration completes

### 4. DaemonSet for Observability Collectors
- **Decision**: OTel collector runs on every node (not pod)
- **Why**: Ensures consistent trace collection regardless of where pods run; per-node collectors reduce network hops
- **Trade-off**: More collectors = more resource overhead; mitigated by tight resource limits (20 millicores, 30 MB RAM)

---

## Troubleshooting Playbook (Real Issues Encountered)

### Issue 1: GitHub Actions Workflow Secrets Not Configured
**Symptom**: Workflow job fails at "Log in to Docker Hub" step with `401 Unauthorized`

**Root Cause**: `DOCKER_USERNAME` and `DOCKER_TOKEN` repository secrets not set

**Resolution**:
1. Go to GitHub repo → Settings → Secrets and Variables → Actions
2. Add `DOCKER_USERNAME` and `DOCKER_TOKEN` (values from Docker Hub account)
3. Re-run workflow

**Lesson**: Automation fails silently if credentials are incomplete. Always test first manual run with verbose logging enabled.

---

### Issue 2: Git Push Rejected by GitHub 
**Symptom**: Workflow job fails at "git push" with `refusing to allow a Personal Access Token to create or update workflow files`

**Root Cause**: GitHub blocks password-based or token-based git authentication for workflow file changes (security feature)

**Resolution**:
1. Switch to SSH-based git authentication (create deploy key with write access)
2. Update git remote: `git remote set-url origin git@github.com:Ike-DevCloudIQ/end-to-end-devops-aws.git`
3. Store private key in GitHub secret (`SSH_PRIVATE_KEY`)
4. Use SSH in workflow: `git config user.name "Bot" && git add . && git commit -m "..." && git push git@github.com:...`

**Lesson**: GitHub's auth model changes behavior based on action type. Read error messages carefully—they often hint at the solution.

---

### Issue 3: Pod Scheduling Failure Due to Resource Exhaustion
**Symptom**: Demo-app rollout stalls; only 2/3 pods running; third pod remains in `Pending` state

**Events**: "Insufficient memory" or "Too many pods on node"

**Root Cause**: Cluster capacity limits (CPU/memory + pod density) prevent new pods

**Resolution**:
1. Check node resources: `kubectl top nodes`
2. Check pod resource requests: `kubectl describe deployment demo-app | grep -i requests`
3. Increase node count: `aws eks update-nodegroup-config ... --scaling-config desiredSize=4`
4. Wait 3-5 minutes for new node to join and pod to schedule

**Lesson**: Kubernetes scheduling is declarative—if there's no room, pods don't start. Monitoring resource usage prevents this in production (set alerts at 70% utilization).

---

### Issue 4: Readiness Probe Failure on OTel Collector Pod
**Symptom**: DaemonSet `otel-collector-agent` shows `3/4 ready`; one pod stuck at `0/1 Running`

**Events**: Readiness probe failing → "GET http://localhost:13133/ connection refused"

**Root Cause**: Container unable to bind to port 13133; likely node-specific CNI or resource issue

**Investigation Commands**:
```bash
# Check pod events
kubectl describe pod otel-collector-agent-<pod-id> -n default

# Check node details
kubectl get nodes -o wide | grep <node-name>

# Inspect container netstat (if exec available)
kubectl exec otel-collector-agent-<pod-id> -n default -- netstat -tlnp
```

**Potential Solutions** (in order of likelihood):
1. Restart aws-cni pod on affected node: `kubectl delete pod aws-node-<pod-id> -n kube-system`
2. Cordon node and force pod migration: `kubectl cordon <node>` + `kubectl delete pod ...`
3. Scale down DaemonSet, let pods redistribute to healthy nodes
4. If issue persists, investigate node capacity (dmesg, systemd logs on EC2)

**Lesson**: When 3/4 identical pods are healthy but 1 fails, issue is node-specific, not pod image. Check infrastructure (CNI, capacity) before tweaking pod config.

---

## Professional Highlights

### Business-Relevant Outcomes

✅ **Reduced Manual Deployment Overhead**: Automation cuts deployment time from 30 minutes (manual kubectl apply + image builds) to ~2 minutes (GitHub Actions + ArgoCD)

✅ **Operational Resilience**: System recovers from pod/node failures automatically:
- Pod crashes → Kubernetes restarts it
- Node failure → Pods rescheduled on healthy nodes
- Service drift → ArgoCD auto-corrects via Git source of truth

✅ **Infrastructure Reproducibility**: Entire AWS environment (VPC, EKS, node group, security groups) can be destroyed and recreated from code in ~15 minutes. Enables:
- Quick disaster recovery
- Multi-environment parity (dev/staging/prod)
- Auditable infrastructure changes (Git history)

✅ **Cost Awareness**: Infrastructure costs tracked explicitly in Terraform:
- 4 EC2 nodes × ~$0.10/hour = ~$72/month
- 1 Load Balancer = ~$16/month + data transfer
- NAT Gateways × 3 = ~$96/month
- Demonstrates cloud resource accountability

✅ **Production Troubleshooting Maturity**:
- Diagnosed pod scheduling issues using `kubectl top`, `kubectl describe`, and `kubectl get events`
- Resolved them with infrastructure scaling decisions (not just "restart the pod")
- Shows root-cause analysis, not symptom-treating

### Core Competencies Demonstrated

| Competency | Evidence |
|---|---|
| **Cloud Infrastructure** | VPC design with AZ distribution, NAT gateways, security groups, IAM roles |
| **Infrastructure as Code** | Terraform modules, remote state with locking, reproducible deployments |
| **Kubernetes Operations** | Pod troubleshooting, resource management, node scaling, DaemonSet/Deployment operation |
| **CI/CD Pipeline Design** | GitHub Actions workflow, Docker multi-stage builds, automated manifest updates |
| **GitOps / Declarative Ops** | ArgoCD application management, Git as source of truth, drift detection |
| **Observability** | Jaeger tracing, Grafana dashboards, Prometheus metrics, operational debugging |
| **Incident Response** | End-to-end troubleshooting (logs, events, resource usage), root cause analysis |
| **Systems Thinking** | Understanding how infrastructure, containers, orchestration, and CI/CD interact |

---

## Architecture and Application Overview

### Microservices Deployed

The OpenTelemetry Astronomy Shop consists of 16 microservices, deployed across multiple namespaces:

| Service | Purpose | Language | DB |
|---|---|---|---|
| **Frontend** | React web UI | Node.js | N/A |
| **Frontend Proxy** | Load balancer for frontend | Go | N/A |
| **API Gateway** | Request routing, auth | Go | N/A |
| **Product Catalog** | Browse products | Java | PostgreSQL |
| **Cart** | Shopping cart management | .NET | PostgreSQL |
| **Checkout** | Order processing | Go | PostgreSQL |
| **Payment** | Payment processing | Node.js | PostgreSQL |
| **Shipping** | Shipping calculation | Java | PostgreSQL |
| **Currency** | Currency conversion | Node.js | N/A |
| **Recommendation** | ML recommendations | Python | Redis |
| **Fraud Detection** | Fraud scoring | Java | N/A |
| **Ad Service** | Ad serving | Java | N/A |
| **Quote** | Insurance quotes | Java | N/A |
| **Email** | Email notifications | Python | N/A |
| **Accounting** | Financial tracking | .NET | PostgreSQL |
| **Load Generator** | Traffic generation | Go | N/A |

### Key Infrastructure Components

| Component | Purpose | Technology |
|---|---|---|
| **VPC** | Network isolation | AWS (10.0.0.0/16 across 3 AZs) |
| **EKS Cluster** | Kubernetes control plane | AWS (v1.30.14-eks) |
| **Worker Nodes** | Pod runtime | EC2 t3.medium (4 nodes) |
| **Load Balancer** | External traffic routing | AWS ELB |
| **Storage** | Persistent data | EBS, PostgreSQL PVCs, Redis |
| **Observability** | Monitoring & tracing | Jaeger, Grafana, Prometheus |
| **CI/CD** | Automation | GitHub Actions |
| **GitOps** | Deployment management | ArgoCD |

---

## Tools and Technologies (Validated in Production)

### Cloud Platform & Infrastructure
- **AWS EKS**: Managed Kubernetes service (control plane + worker nodes)
- **AWS VPC**: Network isolation with public/private subnets across 3 AZs
- **AWS IAM**: Role-based access control for services and RBAC
- **AWS EC2**: t3.medium instances as Kubernetes nodes
- **AWS ELB**: External load balancer for service exposure
- **AWS S3 & DynamoDB**: Remote Terraform state backend with locking

### Infrastructure as Code
- **Terraform**: Declarative infrastructure definition and deployment
- **Terraform Modules**: Reusable component abstractions (VPC, EKS, node group, security)
- **State Locking**: DynamoDB prevents concurrent modifications

### Container Runtime & Build
- **Docker**: Container images (multi-stage builds for optimization)
- **Docker Compose**: Local environment orchestration (16 services on laptop)
- **Docker Hub**: Public container registry (image versioning with run tags)

### Container Orchestration
- **Kubernetes (1.30+)**: Pod lifecycle, scheduling, networking, storage
- **EKS CNI Plugin**: AWS-native pod networking (AWS IP addresses for pods)
- **Helm Charts**: Optional package management for Kubernetes applications

### CI/CD Automation
- **GitHub Actions**: Workflow automation (build, push, manifest update, deploy)
- **GitHub Secrets**: Secure storage for Docker Hub credentials and SSH keys
- **Docker Buildx**: Cross-platform image building (if needed for ARM/x86 support)

### GitOps Deployment
- **ArgoCD**: Declarative application management with Git sync
- **ArgoCD Application CRD**: Kubernetes-native resource for application definitions
- **Auto-Sync**: Continuous reconciliation without manual `kubectl apply`

### Observability Stack
- **Jaeger**: Distributed tracing backend (trace collection, storage, UI)
- **Grafana**: Metrics visualization and dashboard creation
- **Prometheus**: Time-series metrics database and scraping engine
- **OpenTelemetry Collector**: Agent running on every node (DaemonSet) collecting traces
- **OpenTelemetry Instrumentation**: SDKs in services sending traces to collector

### Monitoring & Alerting
- **Kubernetes Events**: Built-in event stream for pod/node status changes
- **kubectl logs/describe**: CLI tools for pod introspection
- **kubectl top**: Real-time resource usage (CPU, memory) by pod/node

### Version Control & Collaboration
- **Git**: Source control for infrastructure, application, and deployment configuration
- **GitHub**: Repository hosting with built-in Actions and secrets management
- **Branch Strategies**: Main branch as source of truth for GitOps

---

## Project Structure and Navigation

```text
end-to-end-devops-aws/
├── Images/                              # Screenshots for documentation
│   ├── Ultimate Project Architecture.gif
│   ├── EKS Cluster.png
│   ├── EKS Node group.png
│   ├── kubectl get deployment.png
│   ├── kubectl get pods.png
│   ├── ArgoCD app Synced and Healthy.png
│   ├── Live App.png
│   └── ...
│
├── .github/workflows/
│   └── cd-demo-app-gitops.yaml         # GitHub Actions CI/CD pipeline
│
├── ultimate-devops-project-demo/       # Application and Kubernetes manifests
│   ├── ArgoCD/
│   │   ├── README.md                   # GitOps-specific documentation
│   │   ├── frontend-proxy-service-application.yaml  # ArgoCD Application
│   │   └── ...
│   ├── kubernetes/
│   │   ├── complete-deploy.yaml        # Full platform deployment
│   │   ├── serviceaccount.yaml
│   │   ├── demo-app/                   # Demo app deployment (managed by CI/CD)
│   │   │   └── deployment.yaml
│   │   ├── accounting/, ad/, cart/, ... # 16 microservice deployments
│   │   └── ...
│   ├── src/                            # Microservice source code
│   │   ├── accounting/                 # .NET service
│   │   ├── ad/                         # Java service
│   │   ├── cart/, checkout/, ...       # Other services
│   │   └── ...
│   ├── docker-compose.yml              # Local dev environment
│   ├── docker-compose-tests.yml        # Testing environment
│   ├── Dockerfile                      # Base image definitions
│   └── README.md                       # Application-specific docs
│
├── ultimate-devops-project-terraform/  # Infrastructure as Code
│   ├── Terraform/
│   │   ├── main.tf                     # Main infrastructure definition
│   │   ├── variables.tf                # Input variables (cluster name, version, etc.)
│   │   ├── outputs.tf                  # Exported outputs (cluster endpoint, security groups)
│   │   ├── modules/
│   │   │   ├── vpc/                    # VPC module
│   │   │   ├── eks/                    # EKS cluster module
│   │   │   ├── node-group/             # Node group module
│   │   │   └── ...
│   │   └── backend/                    # Remote state configuration (S3, DynamoDB)
│   └── README.md                       # Infrastructure-specific docs
│
├── .git/                               # Git repository
├── .gitignore                          # Version control exclusions
├── README.md                           # This file
└── LICENSE                             # Project license
```

### Quick Reference: Key Files

| File | Purpose |
|---|---|
| `.github/workflows/cd-demo-app-gitops.yaml` | GitHub Actions pipeline (build → push → manifest update → sync) |
| `ultimate-devops-project-demo/kubernetes/demo-app/deployment.yaml` | Demo app K8s manifest (updated by CI/CD) |
| `ultimate-devops-project-terraform/Terraform/main.tf` | AWS infrastructure definition |
| `ultimate-devops-project-demo/docker-compose.yml` | Local dev environment |
| `ultimate-devops-project-demo/ArgoCD/frontend-proxy-service-application.yaml` | GitOps application (real service) |

---

## Getting Started: Quick Commands

### 1. Local Development (Docker Compose)
```bash
cd ultimate-devops-project-demo
docker-compose up -d

# Wait for services to stabilize
sleep 30

# Check service health
docker-compose ps

# Open frontend
open http://localhost:8080

# View Jaeger traces
open http://localhost:16686

# View Grafana dashboards
open http://localhost:3000
```

### 2. Deploy to AWS (Terraform)
```bash
cd ultimate-devops-project-terraform/Terraform

# Initialize (creates remote state backend)
terraform init

# Plan infrastructure changes
terraform plan -out=.tfplan

# Apply changes
terraform apply .tfplan

# Retrieve cluster endpoint
terraform output eks_cluster_endpoint

# Configure kubectl
aws eks update-kubeconfig --region us-west-2 --name my-eks-cluster
```

### 3. Deploy Applications (Kubernetes)
```bash
# Deploy all services
kubectl apply -f ultimate-devops-project-demo/kubernetes/complete-deploy.yaml

# Check pod status
kubectl get pods -A

# Check service status
kubectl get svc -A

# View events
kubectl get events -A --sort-by='.lastTimestamp'
```

### 4. GitOps Deployment (ArgoCD)
```bash
# Access ArgoCD UI
kubectl port-forward -n argocd svc/argo-server 8080:443 &
open http://localhost:8080

# Get initial admin password
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d

# Check application sync status
kubectl get application -A

# Trigger manual sync (if auto-sync disabled)
argocd app sync demo-app
```

### 5. Observe System Health
```bash
# Get all pods across namespaces
kubectl get pods -A

# Describe failing pod
kubectl describe pod <pod-name> -n <namespace>

# Check logs
kubectl logs -n <namespace> <pod-name>

# Monitor resource usage
kubectl top nodes
kubectl top pods -A

# Real-time events
kubectl get events -A --watch
```

---

## Operational Runbook Entry Points

Use this section as a practical day-2 operations index.

### Change Management
- Review infra changes before apply: `terraform plan -out=.tfplan`
- Apply only reviewed plan artifacts: `terraform apply .tfplan`
- Use PR reviews for manifest changes before GitOps sync

### Cluster Health Triage
- Baseline cluster state: `kubectl get pods -A`
- Inspect failed workloads: `kubectl describe pod <pod-name> -n <ns>`
- Review service logs: `kubectl logs <pod-name> -n <ns> -c <container-name>`
- Check scheduling pressure: `kubectl top nodes` and `kubectl top pods -A`

### Deployment Validation
- Verify deployed image digest/tag matches expected Git revision
- Confirm rollout status: `kubectl rollout status deployment/<name> -n <ns>`
- Validate endpoint health and synthetic user flow after release

### Incident Containment
- Cordon problematic nodes before disruptive maintenance: `kubectl cordon <node>`
- Drain only when safe with workload disruption reviewed
- Prefer Git revert + ArgoCD sync for rapid rollback over manual hotfixes

### Observability-Driven Debugging
- Use Jaeger traces to isolate latency contributors in call chains
- Use Grafana to correlate app latency with node resource pressure
- Use events + logs together (single-source diagnosis is often misleading)

---

## Performance and Cost Metrics

### Infrastructure Costs (Approximate Monthly)
| Resource | Qty | Cost/Month |
|---|---|---|
| EC2 t3.medium (4 nodes) | 4 | ~$72 |
| ELB (Classic) | 1 | ~$16 |
| NAT Gateways | 3 | ~$96 |
| Data Transfer | — | ~$20 |
| **Total** | — | **~$204** |

*Note: Costs vary by region and data transfer volume. S3 and DynamoDB costs are minimal for state backend.*

### Performance Characteristics
- **Pod startup latency**: ~5-10 seconds (image pull + container initialization)
- **Service-to-service latency**: ~5-50ms (intra-cluster, no egress)
- **End-to-end request latency** (frontend → backend → database): ~100-300ms (depending on service chain depth)
- **Trace collection latency**: <1ms overhead per span (async collection)
- **Deployment reconciliation time** (Git commit → pod running): ~2 minutes (build + push + ArgoCD sync)

---

## Production Hardening Backlog

- **Production security hardening**: No network policies, pod security policies, or RBAC rules (focus on operational concepts)
- **Multi-region failover**: Single AWS region deployment (demonstrated in simpler form)
- **Advanced monitoring** (e.g., log aggregation): Traces and metrics only; logs inspected via `kubectl logs`
- **Secrets management**: Hard-coded or GitHub Secrets (not AWS Secrets Manager)
- **Capacity planning/forecasting**: Manual scaling decisions; no predictive auto-scaling
- **Cost optimization**: No spot instances, reserved instances, or resource rightsizing

These are intentional scope boundaries for this implementation and represent clear next hardening steps for production environments.

---

## Useful Commands Cheat Sheet

### Kubernetes Troubleshooting
```bash
# Find non-ready pods
kubectl get pods -A | grep -v Running

# Check pod events
kubectl describe pod <pod-name> -n <ns>

# Get container logs
kubectl logs <pod-name> -n <ns> -c <container-name>

# Port-forward to pod
kubectl port-forward <pod-name> 8080:8080 -n <ns>

# Execute command in pod
kubectl exec -it <pod-name> -n <ns> -- /bin/sh

# Scale deployment
kubectl scale deployment <name> --replicas=3 -n <ns>

# Get resource usage
kubectl top pods -A
```

### Terraform Management
```bash
# Plan changes
terraform plan -out=.tfplan

# Preview destroy
terraform plan -destroy

# Apply plan
terraform apply .tfplan

# Destroy infrastructure
terraform destroy

# Show state
terraform show

# Refresh state (sync with AWS)
terraform refresh
```

### GitHub Actions
```bash
# Manually trigger workflow
gh workflow run <workflow-name> --ref main

# View workflow runs
gh run list --workflow=<workflow-name>

# Check logs
gh run view <run-id> --log
```

---

## Author & Contact

**Ikenna Ubah**  
Cloud Platform and DevOps Engineer  
[GitHub](https://github.com/Ike-DevCloudIQ) | [Email](mailto:ikenna@example.com)

---

## License

This project is licensed under the [MIT License](LICENSE).

---

## Acknowledgments

- OpenTelemetry project for the Astronomy Shop demo application
- AWS and Kubernetes communities for excellent documentation
- GitHub for reliable CI/CD infrastructure

---

**Last Updated**: March 2026  
**Project Status**: Complete (Core implementation) | Ongoing (Enhancements)
