# Terraform on AWS: EKS, VPC, and Remote State Backend

This directory contains the Terraform implementation for the AWS infrastructure used by this project.

It provisions:
- A custom VPC with public and private subnets across 3 AZs
- Internet Gateway, NAT Gateways, route tables, and subnet associations
- An Amazon EKS cluster and managed node group
- A remote Terraform backend (S3 + DynamoDB) for state storage and locking

## Architecture Context

![Project Architecture](../Images/Ultimate%20Project%20Architecture.gif)

## What This Terraform Stack Builds

### Networking (VPC module)
- VPC CIDR: `10.0.0.0/16`
- 3 public subnets and 3 private subnets across `us-west-2a`, `us-west-2b`, `us-west-2c`
- 1 Internet Gateway
- 3 NAT Gateways (one per public subnet / AZ)
- Public and private route tables + associations

### Kubernetes Platform (EKS module)
- EKS cluster: `my-eks-cluster`
- Kubernetes version: `1.30`
- Managed node group: `general`
- Node instance type: `t3.medium`
- Default node scaling in Terraform:
  - `min_size = 1`
  - `desired_size = 2`
  - `max_size = 4`

### Remote State (backend)
- S3 bucket for Terraform state: `demo-terraform-eks-state-184353012435`
- DynamoDB table for state locking: `terraform-eks-state-locks`
- S3 encryption enabled and bucket versioning enabled

## Implementation Evidence (Your Screenshots)

### Terraform Planning

![Terraform Plan](../Images/Terraform%20%20Plan.png)

### AWS Networking

![VPC Overview](../Images/VPC%20Overview.png)

![NAT Gateways](../Images/NAT%20Gateways.png)

### EKS Resources

![EKS Cluster](../Images/EKS%20Cluster.png)

![EKS Node Group](../Images/EKS%20Node%20group.png)

![EKS Nodes](../Images/EKS%20Nodes.png)

![EC2 Instances](../Images/EC2%20Instances%20(EKS%20Worker%20Nodes).png)

## Directory Structure

```text
ultimate-devops-project-terraform/
├── README.md
└── Terraform/
    ├── main.tf
    ├── variables.tf
    ├── outputs.tf
    ├── backend/
    │   ├── main.tf
    │   └── outputs.tf
    └── modules/
        ├── vpc/
        └── eks/
```

## Prerequisites

- AWS account and IAM credentials with permissions for VPC, EKS, IAM, S3, DynamoDB, EC2
- Terraform v1.x
- AWS CLI configured locally
- `kubectl` (for cluster verification)

## Deployment Workflow

### 1. Configure AWS CLI

```bash
aws configure
aws sts get-caller-identity
```

### 2. Create Backend Resources (S3 + DynamoDB)

Run this once before provisioning the main stack:

```bash
cd Terraform/backend
terraform init
terraform plan
terraform apply
```

### 3. Provision VPC + EKS

```bash
cd ../
terraform init
terraform plan -out=.tfplan
terraform apply .tfplan
```

### 4. Verify Outputs

```bash
terraform output
```

Expected outputs include:
- `cluster_name`
- `cluster_endpoint`
- `vpc_id`

### 5. Connect kubectl to EKS

```bash
aws eks update-kubeconfig --region us-west-2 --name my-eks-cluster
kubectl get nodes
```

## Operational Notes

- If pod scheduling pressure increases, scale node group from AWS CLI or update Terraform variable `node_groups`.
- This stack intentionally uses private subnets for worker nodes and NAT for outbound access.
- Keep Terraform as source of truth for infra changes; avoid manual AWS console edits to prevent drift.

## Cleanup

To avoid ongoing cost:

```bash
cd Terraform
terraform destroy
```

If backend resources are no longer needed:

```bash
cd backend
terraform destroy
```

## Why This Matters

This Terraform implementation demonstrates production-relevant platform engineering practices:
- Reproducible infrastructure
- Team-safe state management (locking + remote state)
- Secure network segmentation (public/private design)
- Managed Kubernetes foundation for GitOps and CI/CD delivery




