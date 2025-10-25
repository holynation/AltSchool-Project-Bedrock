# Project Bedrock — EKS Assessment

Simple, safe guide to run the Retail Store Sample App locally (Minikube) and in AWS EKS (Terraform).

## Repo layout
```
.
├─ app/                        
├─ infra/                       # Terraform IaC
│  ├─ versions.tf
│  ├─ providers.tf
│  ├─ variables.tf
│  ├─ outputs.tf
│  ├─ vpc.tf
│  ├─ eks.tf
│  ├─ iam.tf
│  ├─ aws_auth_configmap.yaml   # (managed via TF kubernetes provider equivalents)
│  └─ rbac_readonly.yaml
└─ .github/workflows/terraform.yml
```

## 1) Local run (Minikube) — free
```bash
# start cluster
minikube start --driver=docker --cpus=2 --memory=4096

# get app
git clone https://github.com/aws-containers/retail-store-sample-app.git
cd retail-store-sample-app

# create namespace + deploy
kubectl create namespace retail || true
kubectl -n retail apply -f deploy/kubernetes/

# checks
kubectl -n retail get pods -w
kubectl -n retail get svc
minikube -n retail service ui --url   # open printed URL
```

---

## 2) AWS EKS (Terraform)
> Keep node count small; avoid NAT (or destroy ASAP).

```bash
aws configure  # set keys, region, json
cd infra
terraform init
terraform validate
terraform plan -out tfplan
```


```bash
terraform apply "tfplan"
```

After apply:
```bash
aws eks update-kubeconfig --name innovatemart-eks --region us-east-1
kubectl get nodes
kubectl create namespace retail || true
kubectl -n retail apply -f ../retail-store-sample-app/deploy/kubernetes/
kubectl -n retail get pods -w
```

**Expose service** (quick test):
```bash
# Convert UI service to LoadBalancer (or edit YAML and re-apply)
kubectl -n retail patch svc ui -p '{"spec":{"type":"LoadBalancer"}}'
kubectl -n retail get svc ui -w  # wait for EXTERNAL-IP
```

**Cleanup (stop costs)**
```bash
cd infra
terraform destroy
```

---

## 3) Read‑only developer account
Created by Terraform:
- IAM user: `dev-readonly` with `ReadOnlyAccess`.
- aws‑auth maps this user to K8s group `dev-readonly` bound to `view` role.

Usage (on developer machine):
```bash
aws configure
aws eks update-kubeconfig --name innovatemart-eks --region us-east-1
kubectl get pods -A        # works
# kubectl apply -f something # should be forbidden
```

---

## 4) CI/CD with GitHub Actions + OIDC
- PR → `fmt/validate/plan` (no apply). Plan artifact uploaded.
- Merge to `main` → `terraform apply -auto-approve` (billable).
- Store `AWS_TERRAFORM_ROLE_ARN` as a GitHub Actions secret (the IAM role for OIDC).

Workflow: `.github/workflows/terraform.yml` (excerpt)
```yaml
permissions: { id-token: write, contents: read }
steps:
  - uses: actions/checkout@v4
  - uses: aws-actions/configure-aws-credentials@v4
    with:
      role-to-assume: ${{ secrets.AWS_TERRAFORM_ROLE_ARN }}
      aws-region: us-east-1
  - uses: hashicorp/setup-terraform@v3
    with: { terraform_version: 1.9.8 }
  - run: terraform fmt -check -recursive
  - run: terraform init
  - run: terraform validate
  - if: github.event_name == 'pull_request'
    run: terraform plan -no-color -out tfplan
  - if: github.event_name == 'push'
    run: terraform apply -auto-approve
```

---

## 5) Branching strategy
- `main` protected; changes via PR.
- Feature branches: `feat/*`, `fix/*`.
- PR: review + `terraform plan`.
- Merge to `main`: auto‑apply (or require manual approval using environments).W

---
