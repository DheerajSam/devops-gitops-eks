# 🚀 GitOps with ArgoCD on AWS EKS

End-to-end GitOps implementation — Terraform-provisioned AWS EKS cluster with ArgoCD continuously syncing application state from this Git repository.

## 🏗️ Architecture

```
Git Repo (k8s/ folder = desired state)
            ↓
      ArgoCD (watches repo every ~3 min)
            ↓
   Auto-syncs to AWS EKS Cluster
            ↓
  Self-heals on any manual drift
```

## 🔧 Tech Stack

| Tool | Purpose |
|---|---|
| **Terraform** | Provisions VPC, EKS Cluster, Node Group, IAM Roles |
| **AWS EKS** | Managed Kubernetes control plane |
| **AWS Load Balancer** | Public-facing NLB via Kubernetes Service |
| **ArgoCD** | GitOps continuous delivery — auto-sync + self-heal |
| **kubectl** | Cluster interaction and verification |

## 📁 Repository Structure

```
devops-gitops-eks/
├── terraform/
│   ├── main.tf          # VPC, EKS Cluster, Node Group, IAM
│   ├── variables.tf      # Region, cluster name, instance type
│   ├── outputs.tf         # Cluster endpoint, kubectl config command
│   └── .gitignore         # Excludes .tfstate and .terraform/ for security
├── k8s/
│   ├── deployment.yaml    # App deployment — 2 replicas, rolling updates, health probes
│   └── service.yaml       # LoadBalancer service — provisions AWS NLB
└── README.md
```

## 🌐 Infrastructure Provisioned (via Terraform)

- VPC with 2 public + 2 private subnets across 2 Availability Zones
- Internet Gateway + NAT Gateway for private subnet internet access
- EKS Cluster (Kubernetes v1.30)
- Managed Node Group — 2x t3.medium nodes
- IAM Roles for EKS Cluster and Worker Nodes

**22 AWS resources created entirely as code — fully reproducible with `terraform apply` / `terraform destroy`.**

## ☸️ Kubernetes Deployment

- 2 replicas with **RollingUpdate** strategy (maxSurge: 1, maxUnavailable: 0) — zero downtime deployments
- **Liveness** and **Readiness** probes hitting `/health` endpoint
- **LoadBalancer** service type — automatically provisions a real AWS Network Load Balancer with public DNS

## 🔄 GitOps Workflow with ArgoCD

ArgoCD Application configured with:
- **Automatic Sync Policy** — no manual `kubectl apply` required
- **Self Heal** enabled — reverts any manual cluster changes back to match Git
- **Prune Resources** enabled — removes resources deleted from Git

### ✅ Self-Healing Demonstrated

Manually scaled the deployment to 5 replicas:
```bash
kubectl scale deployment gitops-app --replicas=5
```

**Result:** ArgoCD detected the drift from the Git-defined state (2 replicas) and automatically terminated the extra 3 pods within ~20 seconds — reverting the cluster back to match the repository, with zero manual intervention.

## 📸 Screenshots

![EKS Nodes Ready](screenshots/eks-nodes.png)
![LoadBalancer with Public DNS](screenshots/loadbalancer.png)
![ArgoCD Application — Healthy & Synced](screenshots/argocd-synced.png)
![Self-Healing in Action](screenshots/self-healing.png)

## 🚀 How to Reproduce

```bash
# 1. Provision infrastructure
cd terraform
terraform init
terraform apply

# 2. Connect kubectl to EKS
aws eks update-kubeconfig --region ap-south-1 --name devops-gitops-eks

# 3. Install ArgoCD
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml --server-side

# 4. Get ArgoCD admin password
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d

# 5. Access ArgoCD UI
kubectl port-forward svc/argocd-server -n argocd 8080:443
# Open https://localhost:8080

# 6. Create Application in ArgoCD UI pointing to this repo's k8s/ folder

# 7. Destroy when done
terraform destroy
```

## 💡 Key Learnings

- **EKS vs Minikube** — Real managed control plane, AWS-provisioned worker nodes, and native LoadBalancer integration vs local single-node clusters
- **GitOps vs traditional CI/CD** — Git becomes the single source of truth; the cluster is constantly reconciled to match it, rather than being pushed to imperatively
- **Security practice** — `.tfstate` files excluded from version control via `.gitignore`; in a team environment, remote state in S3 with DynamoDB locking would be the next step

## 👤 Author

**Dheeraj Samudrala** — DevOps Engineer
- LinkedIn: linkedin.com/in/dheeraj-samudrala-b99b9540
- GitHub: github.com/DheerajSam
