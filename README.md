# 🚀 SmartBite Enterprise Cloud & DevSecOps Platform (AWS EKS)

## 📌 Project Overview

SmartBite is a production-grade, self-healing microservices platform deployed on **Amazon EKS** using **Terraform** and **GitOps (ArgoCD)**. The platform automates end-to-end container builds, enforces **Trivy** vulnerability security gates before promoting images to **AWS ECR** via passwordless **IAM OIDC**, and manages zero-trust **Istio mTLS** traffic with **90/10 Canary rollouts**.

Designed to mirror real-world enterprise cloud environments, the infrastructure dynamically scales at both the pod (**HPA**) and EC2 node (**Karpenter**) levels. It is backed by full-stack **Prometheus & Grafana** observability, automated **Velero** S3 disaster recovery backups, and a battle-tested **Production Troubleshooting Playbook** covering live CI/CD and Kubernetes chaos recoveries.

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

* **Zero-Trust GitOps Delivery:** Automated CI/CD pipeline builds, scans, and pushes container images to AWS ECR using passwordless **IAM OIDC** authentication, then updates Kubernetes manifests (`[skip ci]`) for **ArgoCD** to auto-sync with drift self-healing.
* **Progressive Canary Releases:** Configured **Istio VirtualService & DestinationRule** to route **90%** production traffic to stable `v1` pods and **10%** to `v2` canary pods under strict mutual TLS (`mTLS`) encryption.
* **Dual-Layer Autoscaling:** Combined pod-level **HPA** with node-level **Karpenter** to dynamically provision right-sized EC2 instances during memory/CPU spikes and consolidate idle nodes to optimize AWS costs.
* **Disaster Recovery & Observability:** Integrated **Prometheus & Grafana** for real-time cluster telemetry and **Velero** for scheduled namespace/volume backups to Amazon S3.

---

## 🔥 Production Troubleshooting & Chaos Engineering Playbook

To validate end-to-end resilience, real-world **CI/CD Supply Chain** and **Kubernetes / Service Mesh** failure scenarios were intentionally injected, diagnosed via live logs/events, and resolved:

### Part 1: CI/CD & DevSecOps Pipeline Troubleshooting (6 Core + 1 Live Bonus Error)

| # | Incident / Failure Scenario | Root Cause | Diagnostic Signal | Resolution |
| :--- | :--- | :--- | :--- | :--- |
| **1** | **AWS ECR Auth / IAM OIDC Failure** | Missing `id-token: write` permission, wrong `AWS_ROLE_ARN`, or mismatched repo `sub` claim in IAM Trust Policy. | `Could not assume role with OIDC: Not authorized to perform sts:AssumeRoleWithWebIdentity` / `AccessDeniedException` | Added `permissions: id-token: write`, matched exact GitHub repo (`repo:salmanvohra-dev/smartbite-devsecops:*`) in IAM Trust Policy, and attached least-privilege ECR push policies. |
| **2** | **Trivy Security Gate Block** | Unpatched base image (`node:18.0.0-alpine3.15`) or outdated `npm` packages introduced `HIGH`/`CRITICAL` CVEs. | `Total: X (HIGH: Y, CRITICAL: Z)` / `Process completed with exit code 1` | Upgraded base image to `node:20-alpine`, ran `apk update && apk upgrade --no-cache`, and patched vulnerable `npm` libraries via `npm audit fix`. |
| **2B** | **Live Bonus: Base Image vs Package Manager Mismatch** | Base image switched from Alpine to Debian while retaining Alpine's `apk` package manager in Dockerfile. | `/bin/sh: 1: apk: not found` (`exit code: 127`) | Standardized Alpine base images with `apk` (or Debian/Ubuntu base images with `apt-get`). |
| **3** | **Docker Monorepo Build Context Failure** | Build `context` set to `./backend` instead of root (`.`), hiding sibling `frontend/` and `images/` directories. | `ERROR: failed to calculate checksum of ref ... "/frontend": not found` | Set `context: .` in `docker/build-push-action` and updated Dockerfile `COPY` paths relative to repo root. |
| **4** | **GitOps Push Denied & Infinite CI Loop** | Default `GITHUB_TOKEN` was read-only (`contents: read`), concurrent pushes caused conflicts, and bot commits re-triggered CI endlessly. | `Permission denied to github-actions[bot] (403)` / `! [rejected] main -> main (non-fast-forward)` | Added `permissions: contents: write`, appended `[skip ci]` to bot commits, configured `paths-ignore: ['k8s/**']`, and ran `git pull --rebase`. |
| **5** | **Kubernetes `ImagePullBackOff` / `ErrImagePull`** | Manifest (`app.yaml`) referenced `:latest` while CI pushed immutable SHA tags to ECR, or EKS node role lacked ECR pull access. | `Failed to pull image ... rpc error: code = NotFound: :latest: not found` / `403 Forbidden` | Updated `app.yaml` with exact ECR image tag and verified `AmazonEC2ContainerRegistryReadOnly` policy on EKS Node Group IAM role. |
| **6** | **Kubernetes `CrashLoopBackOff` (Broken Secret / Env)** | Missing or corrupted `MONGO_URI` / `JWT_SECRET` in Kubernetes Secret caused app container to exit immediately on boot. | `State: Waiting (CrashLoopBackOff)` (`Exit Code: 1`) / `MongooseServerSelectionError` in `kubectl logs --previous` | Inspected `kubectl logs --previous` and decoded Secrets, patched missing/corrupted keys, and ran `kubectl rollout restart`. |

### Part 2: EKS, Karpenter, ArgoCD & Istio Chaos Scenarios (7 Scenarios)

| # | Chaos Scenario | Root Cause | Key Signal / Command | Resolution |
| :--- | :--- | :--- | :--- | :--- |
| **1** | **Pod Stuck in `Pending` (Karpenter Rejection)** | Pod CPU/RAM requests (plus DaemonSet overhead) exceeded Karpenter `NodePool` allowed EC2 instance limits. | `FailedScheduling` / `incompatible with nodepool "default"` (`kubectl describe pod`) | Adjusted pod `resources.requests` within NodePool bounds or allowed larger EC2 instance types (`xlarge`/`2xlarge`). |
| **2** | **`OOMKilled` (`Exit Code 137`)** | Container exceeded `resources.limits.memory`, triggering Linux Kernel `OOMKiller` (`SIGKILL`). | `Reason: OOMKilled`, `Exit Code: 137` (`kubectl describe pod`) | Distinguished Exit Code 137 (RAM kill) from Exit Code 1 (app error) and tuned `resources.limits.memory` to realistic peak load. |
| **3** | **Service Selector Mismatch (ALB `503`)** | Kubernetes Service `spec.selector` labels did not match Deployment pod labels, leaving ALB Target Group empty. | `Endpoints: <none>` (`kubectl get endpoints`) + ALB `503 Service Temporarily Unavailable` | Synchronized Service `spec.selector` and Deployment `spec.template.metadata.labels` to register Pod IPs immediately. |
| **4** | **ArgoCD Configuration Drift & `selfHeal`** | Manual `kubectl scale` or `kubectl patch` created drift between GitHub (Desired State) and live EKS cluster. | Status `OutOfSync` (`kubectl get applications -n argocd`) | Paused auto-sync during emergency debugging, committed permanent fixes to Git, and re-enabled `selfHeal: true`. |
| **5** | **App / DB Crash (`Exit Code 1` / Zombie Pod)** | Missing/wrong DB URI or Secret caused app exit (`process.exit(1)`) or silent DB disconnect while pod stayed `1/1 Running`. | `Exit Code: 1` / `MongoServerSelectionError` (`kubectl logs --previous`) | Rolled back via `kubectl rollout undo`, fixed Secret config in Git, and enforced live DB ping checks on `/health`. |
| **6** | **Istio Mesh Strict mTLS `503` Error** | `PeerAuthentication` (`STRICT`) rejected plaintext traffic (`DISABLE`), or VirtualService subset labels mismatched pods. | `503 upstream connect error or disconnect/reset before headers` | Configured `tls.mode: ISTIO_MUTUAL` in `DestinationRule` and aligned VirtualService subset labels with pod labels. |
| **7** | **`502 Bad Gateway` During Rollout** | Kubernetes routed traffic to new pods before Node.js finished connecting to MongoDB and opening port 3000. | `502 Bad Gateway` in Ingress/Istio logs during deployment rollout | Added HTTP `readinessProbe` (`/health`, port `3000`) and zero-downtime `maxUnavailable: 0` + `preStop: sleep 5` hook. |

---

## 🚀 Deployment & Safe Teardown Workflow

### 1. Provision AWS Infrastructure (Terraform)
```bash
cd terraform
terraform init
terraform apply -auto-approve
aws eks update-kubeconfig --region ap-south-1 --name smartbite-eks-cluster
