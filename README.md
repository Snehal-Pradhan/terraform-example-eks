<div align="center">

# terraform-eks-infrastructure

<br />

<img src="/image.png" alt="architecture" width="65%" />

<br />
<br />
<br />
Production-grade AWS EKS infrastructure provisioned with Terraform — one codebase, no hardcoded values, IRSA security, dual compute (EC2 node group + Fargate), and an S3-backed remote state.

</div>

## Architecture

```
                        ┌─────────────────────────────────────────────────┐
                        │              Developer / Terraform CLI          │
                        └───────────────┬─────────────────────────────────┘
                                        │ terraform apply
          ┌─────────────────────────────┼──────────────────────────────┐
          │                             ▼                              │
          │  ┌──────────────┐   ┌────────────────┐   ┌──────────────┐  │
          │  │  S3 Backend  │   │  Terraform     │   │ Prebuilt     │  │
          │  │  (state +    │◄──│  Codebase      │──►│ Modules:     │  │
          │  │  locking)    │   │  (11 files)    │   │ VPC, EKS, IAM│  │
          │  └──────────────┘   └────────────────┘   └──────────────┘  │
          └─────────────────────────────┬──────────────────────────────┘
                                        ▼
   ┌──────────────────────── AWS Account (eu-west-1) ─────────────────────────┐
   │                                                                          │
   │  ┌──────────────────────── VPC 10.2.0.0/16 ──────────────────────────┐   │
   │  │  Public subnets ×2  ──  NAT Gateway + Internet Gateway            │   │
   │  │  Private subnets ×2 ──  EKS control plane ENIs + worker nodes     │   │
   │  └───────────────────────────────────────────────────────────────────┘   │
   │                                                                          │
   │  ┌── EKS Cluster (k8s 1.31, KMS-encrypted secrets) ─────────────────┐    │
   │  │  ├── Managed node group: 2× c7i-flex.large (private subnets)     │    │
   │  │  ├── Fargate profile: "serverless" namespace (serverless pods)   │    │
   │  │  ├── IRSA OIDC provider ──► IAM roles for controllers            │    │
   │  │  ├── Add-ons: vpc-cni, kube-proxy, coredns, EBS CSI driver       │    │
   │  │  └── AWS Load Balancer Controller (Helm) ──► provisions ALBs     │    │
   │  └──────────────────────────────────────────────────────────────────┘    │
   │                                                                          │
   │  ECR (scan-on-push, KMS)  ·  KMS CMK (auto-rotation)  ·  gp3 StorageClass│
   └──────────────────────────────────────────────────────────────────────────┘

   Access path:  kubectl / helm  ──►  EKS API  (no SSH, no bastion)
```

## What gets created

| Component | Details |
|-----------|---------|
| VPC | 10.2.0.0/16, 2 public + 2 private subnets, NAT, IGW, EKS subnet tags |
| EKS cluster | Kubernetes 1.31, secrets encrypted with CMK, public+private API endpoint |
| Managed node group | 2× `c7i-flex.large`, autoscaling 1–3, private subnets |
| Fargate profile | Pods in the `serverless` namespace run serverless (no EC2) |
| IRSA | OIDC provider + IAM roles for EBS CSI and LB controller |
| EKS add-ons | vpc-cni, kube-proxy, coredns, aws-ebs-csi-driver |
| LB Controller | Installed via Helm, auto-creates ALBs for Ingress resources |
| ECR | Container registry, scan-on-push, KMS-encrypted |
| StorageClass | `gp3` (default), encrypted, WaitForFirstConsumer |

## Project structure

```
.
├── versions.tf      # Terraform + AWS/Helm/Kubernetes providers
├── variables.tf     # Inputs (cluster name, CIDRs, node sizing)
├── providers.tf     # AWS + kubernetes/helm exec auth to EKS
├── backend.tf       # Partial S3 backend config
├── backend.hcl      # Backend values (gitignored)
├── data.tf          # AZs, account, region discovery
├── network.tf       # VPC with EKS subnet tags
├── eks.tf           # EKS cluster, node group, Fargate, KMS
├── ecr.tf           # Container image repository
├── addons.tf        # IRSA roles, add-ons, LB controller, StorageClass
└── outputs.tf       # Cluster name/endpoint, ECR URL, kubectl command
```

## Prerequisites

- Terraform >= 1.9.0
- AWS CLI configured (`aws configure get region` must match `aws_region`!)
- `kubectl` and `helm` installed
- An existing S3 bucket for Terraform state (bootstrap once)

## Usage

### 1. Initialize

```bash
terraform init -backend-config=backend.hcl
```

### 2. Deploy (~15–20 minutes; control plane provisioning is the slow part)

```bash
terraform fmt
terraform validate
terraform plan -out=eks.tfplan
terraform apply eks.tfplan
```

### 3. Connect kubectl

```bash
aws eks update-kubeconfig --region eu-west-1 --name iac-eks-cluster
kubectl get nodes    # 2 nodes, Ready
```

### 4. Demo workloads

```bash
# EC2 node group workload + Ingress → AWS provisions an ALB
kubectl create deployment web --image=nginx
kubectl expose deployment web --port=80 --type=ClusterIP
kubectl apply -f - <<'EOF'
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web
spec:
  ingressClassName: alb
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web
            port:
              number: 80
EOF
kubectl get ingress   # ADDRESS fills with the ALB hostname

# Serverless workload on Fargate
kubectl create namespace serverless
kubectl create deployment nginx --image=nginx -n serverless
kubectl get pods -n serverless -o wide   # node shows fargate-ip-...
```

### 5. Push container images

```bash
ECR_URL=$(terraform output -raw ecr_repository_url)
aws ecr get-login-password | docker login --username AWS --password-stdin "$ECR_URL"
docker tag my-app:latest "$ECR_URL":latest
docker push "$ECR_URL":latest
```

### 6. Destroy

```bash
terraform destroy
```

> ⚠️ **Costs are real:** EKS control plane (~$73/mo) + 2× c7i-flex.large + NAT gateway (~$32/mo). Destroy when you're done practicing.

## Design principles

- **No hardcoding** — region, CIDRs, cluster name, node sizes all flow from `variables.tf`
- **Terraform = what exists** — cluster, network, IAM, storage. Workloads and configuration stay with kubectl/Ansible later
- **Private by default** — nodes and control-plane ENIs in private subnets; only the API endpoint is public
- **IRSA, not node IAM** — each controller gets its own least-privilege IAM role via OIDC; no broad node permissions
- **Encryption everywhere** — KMS for k8s secrets, ECR, and EBS volumes
- **Version pinning** — providers, modules, Helm chart, and Kubernetes version all pinned

## Troubleshooting (lessons learned)

| Symptom | Cause | Fix |
|---------|-------|-----|
| `Unauthorized` from helm/kubernetes providers | Cluster creator has no RBAC access | `enable_cluster_creator_admin_permissions = true` in `eks.tf` |
| kubectl: `connection refused localhost:8080` | kubeconfig missing | `aws eks update-kubeconfig --region eu-west-1 --name <cluster>` |
| SSM/EC2 "not found" in wrong region | CLI default region mismatch | `aws configure set region <region>` |
| ALB ingress: `CreateTargetGroupInput.Port` error | Known controller v2.8.1 bug with port resolution | Upgrade Helm chart version in `addons.tf` |
| `InvalidInstanceID.NotFound` | Querying a destroyed instance | Get live IDs from `terraform output` |

## Cleanup checklist

```bash
terraform workspace show            # not used here — single cluster
terraform state list | wc -l        # before destroy
terraform destroy
terraform state list                # should be empty
```
