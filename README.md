# 🚀 SmartBite Enterprise Cloud & DevSecOps Platform (AWS EKS)

## 📌 Project Overview

SmartBite is a production-grade, self-healing microservices platform deployed on **Amazon EKS** using **Terraform** and **GitOps (ArgoCD)**. The platform automates end-to-end container builds, enforces **Trivy** vulnerability security gates before promoting images to **AWS ECR**, and manages zero-trust **Istio mTLS** traffic with **90/10 Canary rollouts**.

Designed to mirror real-world enterprise cloud environments, the infrastructure dynamically scales at both the pod (**HPA**) and EC2 node (**Karpenter**) levels. It is backed by full-stack **Prometheus & Grafana** observability, automated **Velero** S3 disaster recovery backups, and a battle-tested **Production Troubleshooting Playbook** documenting live CI/CD and Kubernetes failure recoveries.

---

## 🏗️ System Architecture

<p align="center">
  <img width="1376" height="768" alt="SmartBite AWS EKS DevSecOps Architecture" src="https://github.com/user-attachments/assets/0e2c83f0-2230-4593-a10e-de61b08bf3db" />
</p>

---

## 🛠️ Tech Stack & Cloud Ecosystem

| Layer | Technologies Used |
| :--- | :--- |
| **Cloud & Infrastructure** | AWS (EKS, VPC, EC2, ALB, IAM OIDC, S3), Modular Terraform |
| **Container & Orchestration** | Docker (Multi-Stage Builds), Kubernetes (EKS), Helm |
| **CI/CD & GitOps** | GitHub Actions, ArgoCD (App-of-Apps Pattern, Self-Heal) |
| **DevSecOps & Policy** | Trivy Vulnerability Scanner, Istio mTLS (`STRICT`), Least-Privilege IAM |
| **Networking & Traffic** | AWS Load Balancer Controller, cert-manager (TLS), Istio Service Mesh (90/10 Canary) |
| **Autoscaling** | Horizontal Pod Autoscaler (HPA), Karpenter (Node Provisioning & Consolidation) |
| **Observability & Backup** | Prometheus, Grafana, Velero (Automated S3 Cluster Snapshots) |

---

## ⚡ Key Engineering Highlights

* **Zero-Trust GitOps Delivery:** Automated CI/CD pipeline builds, scans, and pushes container images to AWS ECR using passwordless **IAM OIDC** authentication, then updates Kubernetes manifests for **ArgoCD** to auto-sync with drift self-healing.
* **Progressive Canary Releases:** Configured **Istio VirtualService & DestinationRule** to route **90%** production traffic to stable `v1` pods and **10%** to `v2` canary pods under strict mutual TLS (`mTLS`) encryption.
* **Dual-Layer Autoscaling:** Combined pod-level **HPA** with node-level **Karpenter** to dynamically provision right-sized EC2 instances during memory/CPU spikes and consolidate idle nodes to optimize AWS costs.
* **Disaster Recovery & Observability:** Integrated **Prometheus & Grafana** for real-time cluster telemetry and **Velero** for scheduled namespace/volume backups to Amazon S3.

---

## 🔥 Production Troubleshooting & Chaos Engineering War Room

To validate system resilience, critical CI/CD, security, and Kubernetes failure scenarios were intentionally injected, diagnosed via live logs/events, and resolved:

| # | Incident / Failure Scenario | Root Cause | Diagnostic Signal | Resolution |
| :--- | :--- | :--- | :--- | :--- |
| **1** | **AWS ECR / IAM OIDC Auth Failure** | Missing `id-token: write` permission or mismatched GitHub repo `sub` claim in IAM Trust Policy. | `Could not assume role with OIDC: Not authorized to perform sts:AssumeRoleWithWebIdentity` | Configured `id-token: write`, matched exact repo condition in IAM Trust Policy, and scoped least-privilege ECR push policies. |
| **2** | **Trivy Security Gate Block** | Unpatched base image (`node:18.0.0-alpine3.15`) contained `HIGH`/`CRITICAL` OS & npm CVEs. | `Total: X (HIGH: Y, CRITICAL: Z)` / `Process completed with exit code 1` | Upgraded base image to `node:20-alpine`, ran `apk upgrade --no-cache`, and patched vulnerable `npm` dependencies. |
| **3** | **Base Image Package Manager Mismatch** | Switched base image from Alpine to Debian while retaining Alpine's `apk` command in Dockerfile. | `/bin/sh: 1: apk: not found` (`exit code: 127`) | Standardized hardened Alpine base image with `apk` (or Debian with `apt-get`) across multi-stage Dockerfiles. |
| **4** | **Docker Monorepo Build Context Failure** | Build `context` pointed to `./backend` instead of repo root (`.`), hiding sibling `frontend/` directories. | `failed to calculate checksum of ref ... "/frontend": not found` | Set `context: .` in GitHub Actions and updated Dockerfile `COPY` paths relative to root. |
| **5** | **GitOps Push Denied & Infinite CI Loop** | Default `GITHUB_TOKEN` had read-only access, and bot commits re-triggered CI workflows endlessly. | `Permission denied to github-actions[bot] (403)` / `non-fast-forward` | Granted `permissions: contents: write`, added `[skip ci]` to bot commits, configured `paths-ignore: ['k8s/**']`, and used `git pull --rebase`. |
| **6** | **Istio Mesh Strict mTLS `503` Error** | `PeerAuthentication` (`STRICT`) blocked plaintext pod-to-pod traffic before sidecar injection was enabled. | `upstream connect error or disconnect/reset before headers (503)` | Enabled `istio-injection=enabled` on `prod` namespace and rolled out pods with Envoy sidecars. |
| **7** | **Kubernetes `ImagePullBackOff`** | Deployment manifest referenced `:latest` tag while CI pushed immutable commit SHA tags to AWS ECR. | `Failed to pull image ... rpc error: code = NotFound: :latest: not found` | Synchronized exact ECR image tags in GitOps manifests and verified node `AmazonEC2ContainerRegistryReadOnly` IAM permissions. |
| **8** | **Kubernetes `CrashLoopBackOff`** | Corrupted/missing `MONGO_URI` in Kubernetes Secret caused backend boot failure on startup. | `CrashLoopBackOff` (`Exit Code: 1`) / `MongooseServerSelectionError` in `kubectl logs --previous` | Audited decoded Secret keys, restored valid database connection strings, and triggered `kubectl rollout restart`. |

---

## 🚀 Deployment & Safe Teardown Workflow

### 1. Provision AWS Infrastructure (Terraform)
```bash
cd terraform
terraform init
terraform apply -auto-approve
aws eks update-kubeconfig --region ap-south-1 --name smartbite-eks-cluster
