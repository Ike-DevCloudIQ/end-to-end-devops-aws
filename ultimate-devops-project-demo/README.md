# Ultimate DevOps Project Demo

This directory contains the application and delivery stack for a full end-to-end DevOps implementation on AWS.

It demonstrates:
- Local platform execution with Docker Compose
- Kubernetes deployment of multi-service workloads
- GitHub Actions CI/CD automation
- GitOps continuous delivery with ArgoCD
- Observability with OpenTelemetry, Jaeger, Prometheus, and Grafana

## Architecture

![Project Architecture](../Images/Ultimate%20Project%20Architecture.gif)

## Purpose and Scope

The goal of this project is to show practical DevOps and platform engineering capability beyond a basic app deploy.

Core capabilities implemented:
- Build and run a realistic microservices platform locally
- Promote workloads to Kubernetes with repeatable manifests
- Automate image build and manifest update flows in CI
- Use Git as source of truth for deployments (GitOps)
- Operate with observability-first debugging

## Platform Overview

### 1. Cloud and Infrastructure Foundation
Infrastructure is provisioned with Terraform from [../ultimate-devops-project-terraform/README.md](../ultimate-devops-project-terraform/README.md), including:
- VPC with public/private subnets across AZs
- NAT gateways and internet routing
- EKS cluster and managed node group
- S3 + DynamoDB Terraform state backend

### 2. Kubernetes Application Layer
- Multi-service workloads deployed with manifests in [kubernetes/](kubernetes/)
- Service exposure through services and ingress resources
- Namespace-based workload organization
- Resource and readiness controls for operational stability

### 3. CI/CD Automation (GitHub Actions)
Workflows automate image build and deployment updates.

Main automation files:
- [../.github/workflows/cd-demo-app-gitops.yaml](../.github/workflows/cd-demo-app-gitops.yaml)
- [.github/workflows/ci.yaml](.github/workflows/ci.yaml)

The demo-app GitOps pipeline updates Kubernetes image tags and pushes manifest changes to trigger ArgoCD reconciliation.

### 4. GitOps Delivery (ArgoCD)
- ArgoCD watches Git manifests and syncs cluster state
- Drift is detected and corrected automatically when auto-sync is enabled
- Application definitions and setup docs are in [ArgoCD/README.md](ArgoCD/README.md)

### 5. Observability
- Distributed tracing: Jaeger
- Metrics and dashboards: Prometheus + Grafana
- Telemetry pipeline: OpenTelemetry collector and instrumented services

## Evidence Screenshots

### Local and Application Access
![Live App](../Images/Live%20App.png)

### Kubernetes Runtime
![kubectl get deployment](../Images/kubectl%20get%20deployment.png)

![kubectl get pods](../Images/kubectl%20get%20pods.png)

### GitOps Status
![ArgoCD Synced and Healthy](../Images/ArgoCD%20app%20Synced%20and%20Healthy.png)

### AWS Runtime Context
![Load Balancer details](../Images/Load%20Balancer%20details.png)

## Directory Structure

```text
ultimate-devops-project-demo/
├── .github/workflows/
├── ArgoCD/
├── internal/
├── kubernetes/
├── pb/
├── src/
├── test/
├── demo-app-image/
├── docker-compose.yml
├── local-setup-readme.md
├── GitHub Actions Readme.md
└── README.md
```

## Important Links (Updated)

- Project root README: [../README.md](../README.md)
- Terraform documentation: [../ultimate-devops-project-terraform/README.md](../ultimate-devops-project-terraform/README.md)
- Local setup guide: [local-setup-readme.md](local-setup-readme.md)
- Kubernetes docs: [kubernetes/README.md](kubernetes/README.md)
- ArgoCD docs: [ArgoCD/README.md](ArgoCD/README.md)
- GitHub Actions notes: [GitHub Actions Readme.md](GitHub%20Actions%20Readme.md)
- Root CI/CD workflow: [../.github/workflows/cd-demo-app-gitops.yaml](../.github/workflows/cd-demo-app-gitops.yaml)

## Quick Start

### Local (Docker Compose)

```bash
docker compose up -d
docker compose ps
```

### Kubernetes (cluster context configured)

```bash
kubectl get pods -A
kubectl get svc -A
```

### CI/CD Trigger for demo-app
Changes inside [demo-app-image/](demo-app-image/) trigger the demo-app workflow and image rollout path.

## Author

Ikenna Ubah  
Cloud Platform and DevOps Engineer

Contact:
- LinkedIn: https://www.linkedin.com/in/ikenna2/
- Email: ikennaubah2@yahoo.com
