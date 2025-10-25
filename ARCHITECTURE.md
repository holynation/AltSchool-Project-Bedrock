# Project Bedrock — EKS Architecture (2 pages)

> Audience: assessor, devs. Tone: simple, practical. Purpose: explain **what we built**, **why**, and **how to operate** it.

## 1) Goals & Scope (what this does)
- Provision AWS **VPC + EKS** with Terraform.
- Run the **Retail Store Sample App** in Kubernetes using **in‑cluster** databases/queues for the assessment.
- Provide a **read‑only** IAM developer user with Kubernetes **view‑only** RBAC.
- Add **CI/CD** (GitHub Actions) to run `terraform plan` on Pull Requests and `terraform apply` on merges to `main` (via GitHub OIDC).
- Keep costs low and clean up fast.

---

## 2) High‑level Design (how things fit together)

```mermaid
flowchart TB
  subgraph GitHub
    A[Repo
 code + infra]
    W[Actions CI/CD]
  end

  subgraph AWS[VPC 10.0.0.0/16]
    IGW[Internet Gateway]
    NAT[NAT GW]
    subgraph Pub[Public subnets]
      ALB[(Optional ALB/ELB)]
      NGW[NodeGroup (public - demo mode)]
    end
    subgraph Priv[Private subnets]
      EKSN[NodeGroup (private - prod)]
    end
    CP[(EKS Control Plane)]
  end

  Dev[Dev Laptop/Ubuntu VM]-- kubectl/helm/awscli -->CP
  A-- push/PR -->W
  W-- OIDC AssumeRole -->AWS
  ALB-- HTTP(S) --> Users
  CP-- manages -->EKSN
  CP-- manages -->NGW
```

### Components
- **VPC**: /16 CIDR with 2 public + 2 private subnets (multi‑AZ).
- **EKS**: 1 cluster, 1 managed node group (t3.medium x 2).
- **IAM**:
  - Cluster role + node role (AWS managed policies).
  - `dev-readonly` user with `ReadOnlyAccess` + K8s **view** clusterrole via aws‑auth mapUsers.
  - **OIDC** role for GitHub Actions (no long‑lived secrets).
- **Kubernetes**: namespace `retail`, deployments/services for microservices; service type can be `LoadBalancer` or exposed via ALB Ingress (optional).

### Networking & Security
- **Public subnets**: ELB/ALB and (optionally) nodes for demo to avoid NAT cost.
- **Private subnets**: production node group behind **NAT** (safer, but costs money).
- **Security groups**: EKS control plane <-> nodes; NodePort/ALB rules if exposing services.

---

## 3) CI/CD (GitHub Actions with OIDC)
- **Pull Request** → `terraform fmt/validate/plan` in `infra/` and publish plan artifact.
- **Merge to main** → `terraform apply -auto-approve` (billable). You can protect with required reviewers/environments.
- OIDC trust policy restricts role assumption to this repo only.

**Why OIDC?** No plaintext AWS keys in GitHub. Short‑lived tokens per job.

---

## 4) Cost Controls (important)
- **EKS control plane**: billed hourly.
- **NAT Gateway**: billed hourly + data (can be most expensive in labs).
- **EC2 nodes**: billed per instance hour. Use **t3.medium**, 2 nodes.
- **ELB/ALB**: billed hourly + LCUs.
- **Note**: For lab, run nodes in **public subnets** (no NAT), scale down node count, **destroy** after grading.

---

## 5) Operations (day‑to‑day)
- **Access cluster**:
  ```bash
  aws eks update-kubeconfig --name <cluster> --region <region>
  kubectl get nodes
  kubectl get pods -A
  ```
- **App deploy** (namespace once, then manifests):
  ```bash
  kubectl create ns retail
  kubectl -n retail apply -f deploy/kubernetes/
  ```
- **Health**:
  ```bash
  kubectl -n retail get pods,svc
  kubectl -n retail logs deploy/ui | tail -n 50
  ```
- **Expose** (quick test): change UI svc to `NodePort` or `LoadBalancer`.
- **Tear down**: `terraform destroy` (destructive; confirm first).

---

## 6) Local ⇄ Cloud Parity (how we tested safely)
- **Local**: Docker + Minikube on Ubuntu VM (4GB RAM, 2 vCPU).
- **Same manifests** work on **EKS**.
- Local exposure via `minikube service ui --url`; cloud via `LoadBalancer` or ALB.
- Troubleshooting on local first reduces AWS billable trial‑and‑error.

---

## 7) Risks & Mitigations
| Risk | Impact | Mitigation |
|---|---|---|
| NAT Gateway cost balloons | High | Use public‑subnet node group for lab; destroy fast |
| Pods CrashLoop | Medium | `kubectl logs/describe`, ensure env vars and images resolve |
| OIDC misconfig | Medium | Verify role trust policy matches repo; test `configure-aws-credentials` step |
| aws‑auth drift | Low | Manage via Terraform `kubernetes_config_map` |
| Unintended apply on main | Medium | Use PR plan + required reviewers or manual approval |

---

## 8) Hand‑off (who can do what)
- **Maintainer**: full admin on AWS + repo.
- **Read‑only developer (`dev-readonly`)**: AWS ReadOnlyAccess; K8s **view** only.
  - Setup:
    ```bash
    aws configure   # their keys
    aws eks update-kubeconfig --name <cluster> --region <region>
    kubectl get pods -A   # should work; apply/delete should fail
    ```

---

## 9) Appendix (minimal alternates)
- **No NAT** lab: put node group in **public** subnets; skip NAT resources.
- **Ingress**: install AWS Load Balancer Controller, create `Ingress` for UI, add ACM cert + Route 53 (bonus section).
- **IRSA**: attach fine‑grained policies to service accounts if app needs AWS APIs.
