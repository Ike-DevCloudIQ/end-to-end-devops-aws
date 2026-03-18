# Kubernetes on EKS — Deploying the Astronomy Shop

This guide covers connecting to your EKS cluster, deploying the full 16-service platform, and verifying the deployment.

---

## Prerequisites

- EKS cluster provisioned via Terraform (see [Terraform README](../../ultimate-devops-project-terraform/README.md))
- AWS CLI configured (`aws configure`)
- `kubectl` installed locally

---

## Steps to Deploy

**Step 1: Update kubeconfig to connect to EKS**

```bash
aws eks --region us-west-2 update-kubeconfig --name my-eks-cluster
```

**Step 2: Verify cluster and nodes**

```bash
kubectl config current-context
kubectl get nodes
```

![EKS Nodes](../../Images/EKS%20Nodes.png)

**Step 3: Clone the repository and navigate to the kubernetes directory**

```bash
git clone https://github.com/Ike-DevCloudIQ/end-to-end-devops-aws.git
cd end-to-end-devops-aws/ultimate-devops-project-demo/kubernetes
```

**Step 4: Create the service account**

```bash
kubectl apply -f serviceaccount.yaml
```

**Step 5: Deploy all services at once**

```bash
kubectl apply -f complete-deploy.yaml
```

**Step 6: Verify all pods are running**

```bash
kubectl get pods -A
```

![kubectl get pods](../../Images/kubectl%20get%20pods.png)

**Step 7: Check deployments**

```bash
kubectl get deployments -A
```

![kubectl get deployment](../../Images/kubectl%20get%20deployment.png)

**Step 8: Verify the LoadBalancer service**

The frontend-proxy service is exposed via an AWS ELB (Elastic Load Balancer), created automatically by Kubernetes when service type is `LoadBalancer`.

```bash
kubectl get svc -A
```

![Load Balancer details](../../Images/Load%20Balancer%20details.png)

**Step 9: Access the live application**

Once the ELB is provisioned (check AWS Console → EC2 → Load Balancers), use the DNS name to access the app in your browser.

![Live App](../../Images/Live%20App.png)

---

## EC2 Instances (EKS Worker Nodes)

![EC2 Instances](../../Images/EC2%20Instances%20(EKS%20Worker%20Nodes).png)

---

## GitOps Status (ArgoCD)

Once ArgoCD is installed and configured, it continuously syncs the manifests from Git to the cluster:

![ArgoCD Synced and Healthy](../../Images/ArgoCD%20app%20Synced%20and%20Healthy.png)

---

## Useful Troubleshooting Commands

```bash
# Get events for a failing pod
kubectl describe pod <pod-name> -n <namespace>

# View logs
kubectl logs <pod-name> -n <namespace>

# Check resource usage
kubectl top nodes
kubectl top pods -A

# Watch pod status live
kubectl get pods -A --watch
```

---

## Notes

- This project uses an AWS ELB (Classic Load Balancer) for service exposure, not an Ingress controller.
- The load balancer DNS name is the public endpoint for the Astronomy Shop frontend.
- Worker nodes are deployed in private subnets; the ELB routes traffic to them via node ports.
- For ArgoCD-managed services, manifests are synced automatically from Git — no manual `kubectl apply` needed after initial setup.

