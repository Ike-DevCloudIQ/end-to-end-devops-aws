# Your Complete DevOps Learning Journey - From Zero to Production

This guide is your end-to-end roadmap to learn, build, and deploy the complete DevOps project from beginner to expert level.

## What You Are Building

You are building the Astronomy Shop: a production-grade e-commerce app with 16 microservices written in 9 languages, deployed on AWS EKS using Terraform, with GitHub Actions CI/CD, ArgoCD GitOps, and a full observability stack.

## 8-Phase Learning Path

| Phase | Topic | Outcome |
|---|---|---|
| 0 | Setup and Prerequisites | Local machine and accounts ready |
| 1 | Run Locally with Docker Compose | Understand architecture and service flow |
| 2 | Containerization Deep Dive | Build and push production images |
| 3 | AWS IaC with Terraform | Provision VPC + EKS from code |
| 4 | Kubernetes on EKS | Deploy and operate microservices |
| 5 | CI/CD with GitHub Actions | Automate build, test, image push, manifest updates |
| 6 | GitOps with ArgoCD | Auto-sync deployments from Git |
| 7 | Observability | Monitor metrics, traces, logs |
| 8 | Production Hardening | TLS, autoscaling, security and reliability |

---

## Phase 0 - Setup and Prerequisites

### Core Concepts You Should Understand First

- What DevOps means: collaboration + automation across development and operations.
- Microservices architecture: many small services instead of one monolith.
- CI/CD: continuous integration and continuous delivery/deployment.
- Infrastructure as Code (IaC): define infrastructure in files, not manual console clicks.
- Kubernetes basics: Pods, Deployments, Services, Ingress.
- Observability basics: metrics, traces, logs.

### Install Required Tools

1. Git

```bash
git --version
git config --global user.name "Your Name"
git config --global user.email "you@example.com"
```

2. Docker Desktop

```bash
docker --version
docker compose version
```

3. AWS CLI

```bash
brew install awscli
aws --version
```

4. Terraform

```bash
brew tap hashicorp/tap
brew install hashicorp/tap/terraform
terraform --version
```

5. kubectl

```bash
brew install kubectl
kubectl version --client
```

6. eksctl

```bash
brew tap weaveworks/tap
brew install weaveworks/tap/eksctl
eksctl version
```

7. Helm

```bash
brew install helm
helm version
```

### Accounts You Need

- GitHub
- Docker Hub
- AWS account (note: EKS has ongoing cost)

---

## Phase 1 - Run the Project Locally with Docker Compose

### Goal

Run all microservices locally and understand service communication.

### Steps

1. Fork and clone your repo.

```bash
git clone https://github.com/YOUR_USERNAME/end-to-end-devops-aws.git
cd end-to-end-devops-aws/ultimate-devops-project-demo
```

2. Start the stack.

```bash
set -a
source ./.env
source ./.env.override
source ./.env.arm64
set +a

docker compose up --no-build
# or
docker compose build
docker compose up
```

Important: On this project, several services rely on pass-through environment variables. Loading all env files into your shell before running Docker Compose prevents startup issues in frontend-proxy and related routes.

3. Explore key URLs.

- Storefront: `http://localhost:8080`
- Jaeger (via proxy path): `http://localhost:8080/jaeger/ui`
- Grafana (via proxy path): `http://localhost:8080/grafana`
- Prometheus (via proxy path): `http://localhost:8080/prometheus`

4. Useful commands.

```bash
docker ps
docker compose logs frontend
docker compose logs checkout
docker stats
docker compose down
```

### Learning Checkpoint

- You can trace a user request across services.
- You understand how service names are used for internal networking.

---

## Phase 2 - Containerization Deep Dive

### Goal

Understand how each service is packaged and optimized for runtime.

### Key Topics

- Dockerfile layers
- Multi-stage builds
- Image size optimization
- Container networking and runtime config

### Example Workflow

```bash
cd ultimate-devops-project-demo/src/product-catalog
docker build -t your-dockerhub-username/product-catalog:v1.0 .
docker run -p 8080:8080 your-dockerhub-username/product-catalog:v1.0
docker login
docker push your-dockerhub-username/product-catalog:v1.0
```

### Learning Checkpoint

- You can build, run, and publish a service image.
- You understand why multi-stage builds matter.

---

## Phase 3 - Provision AWS Infrastructure with Terraform

### Goal

Create cloud infrastructure from code: VPC, networking, and EKS cluster.

### Architecture Summary

- Region: `us-west-2`
- VPC CIDR: `10.0.0.0/16`
- 3 public + 3 private subnets
- EKS Kubernetes cluster
- Node group on `t3.medium`
- S3 backend for Terraform state
- DynamoDB table for state locking

### Steps

1. Configure AWS credentials.

```bash
aws configure
aws sts get-caller-identity
```

2. Create Terraform backend first.

```bash
cd ultimate-devops-project-terraform/Terraform/backend
terraform init
terraform apply
```

3. Create infrastructure.

```bash
cd ../
terraform init
terraform plan
terraform apply
```

4. Connect kubectl to EKS.

```bash
aws eks update-kubeconfig --name my-eks-cluster --region us-west-2
kubectl get nodes
```

### Learning Checkpoint

- You can read and explain a Terraform plan.
- You can provision and tear down infra safely.

---

## Phase 4 - Deploy and Operate on Kubernetes

### Goal

Deploy all services to EKS and learn Kubernetes operations.

### Steps

1. Create namespace.

```bash
kubectl create namespace opentelemetry-demo
```

2. Apply service account and manifests.

```bash
kubectl apply -f kubernetes/serviceaccount.yaml
kubectl apply -f kubernetes/complete-deploy.yaml
```

3. Verify workload health.

```bash
kubectl get pods -n opentelemetry-demo -w
kubectl get svc -n opentelemetry-demo
kubectl get ingress -n opentelemetry-demo
```

4. Troubleshoot if needed.

```bash
kubectl describe pod <pod-name> -n opentelemetry-demo
kubectl logs <pod-name> -n opentelemetry-demo
kubectl exec -it <pod-name> -n opentelemetry-demo -- sh
```

### Learning Checkpoint

- You understand Deployments, Services, and Ingress.
- You can debug pod startup and runtime issues.

---

## Phase 5 - CI/CD with GitHub Actions

### Goal

Automate build/test/lint/image-push and manifest updates.

### Pipeline Flow

- Trigger on push/PR
- Build and unit test
- Lint and quality checks
- Build and push Docker image
- Update Kubernetes deployment image tag in repo

### Required Repository Secrets

- `DOCKER_USERNAME`
- `DOCKER_TOKEN`
- `GITHUB_TOKEN` (provided by GitHub Actions)

### Learning Checkpoint

- Every code change triggers automated quality and delivery steps.
- You can read CI logs and fix failed jobs.

---

## Phase 6 - GitOps with ArgoCD

### Goal

Use Git as the source of truth and auto-deploy from manifest changes.

### Steps

1. Install ArgoCD.

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

2. Expose ArgoCD UI.

```bash
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'
kubectl get svc argocd-server -n argocd
```

3. Get admin password.

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 --decode; echo
```

4. Connect repo and create ArgoCD application.

- Repo URL: your GitHub fork
- Path: `ultimate-devops-project-demo/kubernetes`
- Target revision: `main`
- Sync policy: automated

### Learning Checkpoint

- A manifest commit in Git auto-syncs to EKS.
- You understand drift detection and self-healing.

---

## Phase 7 - Observability and Monitoring

### Goal

Observe and troubleshoot production behavior with metrics, traces, and logs.

### Stack

- OpenTelemetry Collector
- Jaeger (traces)
- Prometheus (metrics)
- Grafana (dashboards)
- OpenSearch (logs)

### Access Tools via Port Forwarding

```bash
kubectl port-forward svc/grafana 3000:3000 -n opentelemetry-demo
kubectl port-forward svc/prometheus 9090:9090 -n opentelemetry-demo
kubectl port-forward svc/jaeger-query 16686:16686 -n opentelemetry-demo
```

### Learning Checkpoint

- You can correlate high latency with specific services and traces.
- You can build dashboards and define actionable alerts.

---

## Phase 8 - Production Hardening

### Goal

Make the platform secure, resilient, and scalable.

### Focus Areas

- Route53 custom domain
- TLS/HTTPS with cert-manager
- Horizontal Pod Autoscaler (HPA)
- Cluster autoscaler
- Resource requests/limits tuning
- Pod disruption budgets and rollout strategies
- Secret management and least-privilege IAM

### Learning Checkpoint

- You can run this architecture with production-level guardrails.

---

## Full Project Architecture (High-Level)

1. Developer pushes code.
2. GitHub Actions runs CI pipeline.
3. New image is pushed to registry.
4. Kubernetes manifest image tag is updated in Git.
5. ArgoCD detects Git change and syncs to EKS.
6. App traffic flows through frontend proxy to microservices.
7. Telemetry goes to OTel Collector, then to Prometheus, Jaeger, and OpenSearch.

---

## Cost and Safety Controls (Important)

EKS and related networking are not free while running.

### Always clean up when finished

```bash
kubectl delete namespace opentelemetry-demo
kubectl delete namespace argocd
cd ultimate-devops-project-terraform/Terraform
terraform destroy
```

### Practical Habit

- Deploy for learning sessions.
- Destroy infra immediately when done.
- Review AWS billing dashboard regularly.

---

## Suggested 6-Week Study Plan

### Week 1

- Phase 0 and Phase 1

### Week 2

- Phase 2

### Week 3

- Phase 3

### Week 4

- Phase 4

### Week 5

- Phase 5 and Phase 6

### Week 6

- Phase 7 and Phase 8

---

## Final Beginner-to-Expert Checklist

- [ ] I can run and debug the app locally.
- [ ] I can build and optimize container images.
- [ ] I can provision AWS infrastructure with Terraform.
- [ ] I can deploy and operate workloads on Kubernetes.
- [ ] I can build CI/CD workflows in GitHub Actions.
- [ ] I can implement GitOps deployments with ArgoCD.
- [ ] I can observe system behavior with traces, metrics, and logs.
- [ ] I can harden production with scaling, TLS, and security practices.

If you want, the next step is to start Phase 0 together and execute every command in sequence on your machine with explanations for each output.
