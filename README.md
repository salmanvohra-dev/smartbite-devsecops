🚀 SmartBite Enterprise Cloud & DevSecOps Platform on AWS EKSA production-grade, self-healing, zero-trust Kubernetes platform built on AWS EKS.Engineered with Infrastructure-as-Code (Terraform), keyless OIDC CI/CD pipelines, automated Trivy security gates, GitOps continuous delivery (ArgoCD), Karpenter just-in-time node autoscaling, Istio mTLS Canary traffic routing, full-stack observability, and Velero disaster recovery.🏗️ System Architecture+---------------------------------------------------------------------------------------------------+
|                                   DEVELOPER & CI/CD (DEVSECOPS)                                   |
|                                                                                                   |
|  [Git Push] ---> [GitHub Actions] ---> [Trivy CVE Gate] ---> [AWS OIDC Auth] ---> [Amazon ECR]    |
|                         |                                                                         |
|                         +---> (Updates Image Tag & Commits [skip ci]) ---> [GitOps Manifests]     |
+---------------------------------------------------------------------------------------------------+
                                                                                   |
                                                                                   v (Auto-Sync)
+---------------------------------------------------------------------------------------------------+
|                                    AWS EKS PRODUCTION CLUSTER                                     |
|                                                                                                   |
|   [Internet] ---> [AWS ALB Ingress + cert-manager (HTTPS)] ---> [Istio Ingress Gateway]           |
|                                                                       |                           |
|                                                         +-------------+-------------+             |
|                                                         | Istio mTLS (STRICT)       |             |
|                                                         | Canary Split: 90% / 10%   |             |
|                                                         v                           v             |
|                                                  [Backend v1]                [Backend v2]         |
|                                                         |                           |             |
|                                                         +-------------+-------------+             |
|                                                                       v                           |
|                                                                  [MongoDB]                        |
|                                                                                                   |
|   +------------------------------------+  +---------------------------------------------------+   |
|   |        AUTOSCALING & COST          |  |             OBSERVABILITY & RECOVERY              |   |
|   | - HPA (Pod-level CPU/Memory)       |  | - Prometheus & Grafana (Metrics & Dashboards)     |   |
|   | - Karpenter (Just-in-Time EC2)     |  | - Velero + AWS S3 (Automated Cluster Backups)     |   |
|   +------------------------------------+  +---------------------------------------------------+   |
+---------------------------------------------------------------------------------------------------+
✨ Key Production Capabilities100% Modular Terraform Infrastructure: Automated provisioning of Multi-AZ VPC, public/private subnets, NAT Gateways, EKS Control Plane, Managed Node Groups, and IRSA (IAM Roles for Service Accounts).Keyless DevSecOps CI/CD Pipeline:Authenticates with AWS using GitHub OIDC (sts:AssumeRoleWithWebIdentity)—zero hardcoded AWS access keys.Enforces Trivy Vulnerability Scanning (exit-code: '1') to block any container image containing HIGH or CRITICAL CVEs before registry push.Automates GitOps manifest updates with [skip ci] loop prevention.GitOps App-of-Apps Pattern (ArgoCD): Automated declarative deployment and drift self-healing (prune: true, selfHeal: true) across microservices, Istio security policies, observability, and backup stacks.Zero-Trust Networking & Progressive Delivery (Istio):Enforces cluster-wide PeerAuthentication (STRICT mTLS) for encrypted service-to-service communication.Implements Canary Traffic Splitting (90% Stable / 10% Canary) via VirtualService and DestinationRule subsets.Dual-Layer Autoscaling (HPA + Karpenter): Horizontal Pod Autoscaler scales pods on load, while Karpenter dynamically provisions right-sized EC2 nodes in seconds when unschedulable pods appear, and consolidates them when idle.Observability & Disaster Recovery: Full cluster telemetry via kube-prometheus-stack (Prometheus + Grafana) and scheduled namespace/PV backups to Amazon S3 via Velero.🛠️ Tech StackLayerTechnologies UsedCloud & IaCAWS (EKS, ECR, VPC, ALB, IAM, S3), TerraformCI/CD & SecurityGitHub Actions, AWS OIDC, Trivy Vulnerability Scanner, Docker Multi-Stage BuildsGitOps & PackagingArgoCD (App-of-Apps), Helm, KustomizeService Mesh & IngressIstio (mTLS, VirtualService, DestinationRule), AWS Load Balancer Controller, cert-managerAutoscalingKubernetes HPA, Karpenter (NodePool, EC2NodeClass)Monitoring & DRPrometheus, Grafana, Velero🔥 Production Troubleshooting & Chaos Engineering War RoomReal production engineering isn't just building green pipelines—it's diagnosing and resolving failures under pressure.Below is a documented log of real CI/CD, DevSecOps, and Kubernetes runtime failures intentionally injected and resolved on this platform:#Incident / Failure ModeRoot Cause & Log SignatureDiagnostic & Resolution1AWS OIDC Auth FailureAWS STS rejected GitHub Actions token (Could not assume role with OIDC: Not authorized to perform sts:AssumeRoleWithWebIdentity).Added id-token: write permission in workflow, aligned IAM Trust Policy sub claim with exact repo, and scoped least-privilege ECR push policies.2Trivy CVE Security Gate BlockOutdated base image (node:18.0.0-alpine3.15) triggered CRITICAL OS/library CVEs, halting build with exit code 1.Inspected Trivy vulnerability table (Installed vs Fixed), upgraded base image to node:20-alpine, and ran apk update && apk upgrade --no-cache.3Base Image vs Package Manager MismatchSwitching from Alpine to Debian while retaining apk caused /bin/sh: 1: apk: not found (exit code 127).Matched OS package managers to distribution (apk for Alpine vs apt-get for Debian/Ubuntu).4Docker Build Context FailureMonorepo COPY failed ("/frontend": not found) because build context was restricted to ./backend.Expanded build context to repository root (context: .) and updated multi-stage Dockerfile paths accordingly.5GitOps Push 403 & CI LoopCI bot push rejected (Permission denied to github-actions[bot] - 403) and caused recursive pipeline triggers.Granted permissions: contents: write to GitOps job, added [skip ci] to commit messages, and configured paths-ignore on k8s/**.6EKS ImagePullBackOffDeployment referenced :latest tag while ECR held commit-SHA tags (rpc error: code = NotFound).Diagnosed via kubectl describe pod Events; synced exact immutable ECR image tag into GitOps manifest and verified node AmazonEC2ContainerRegistryReadOnly policy.7Pod CrashLoopBackOff (Exit Code 1)Corrupted MONGO_URI secret caused application crash on startup (MongooseServerSelectionError).Captured stack trace via kubectl logs --previous, patched Kubernetes Secret with valid connection URI, and executed kubectl rollout restart.8Istio 503 UC & 502 Readiness FailuresMissing DestinationRule subset labels (v1/v2) and misconfigured HTTP health probe paths dropped ingress traffic.Aligned pod labels with Istio subsets, verified PeerAuthentication mTLS mode, and corrected readinessProbe endpoints.🚀 Quick Start & Deployment Guide1. Provision AWS EKS Infrastructurecd terraform
terraform init
terraform plan -out=tfplan
terraform apply tfplan
2. Connect kubectl to EKS Clusteraws eks update-kubeconfig --region ap-south-1 --name smartbite-eks-cluster
kubectl get nodes
3. Bootstrap GitOps (ArgoCD App-of-Apps)kubectl apply -f k8s/argocd/root-bootstrap-app.yaml
kubectl get applications -n argocd
4. Clean Teardown (Zero Orphaned AWS Resources)# 1. Remove ArgoCD apps, namespaces & Karpenter nodes to release ALBs/ENIs
kubectl delete application --all -n argocd --ignore-not-found
kubectl delete ns prod istio-system --ignore-not-found
kubectl delete nodeclaims --all --ignore-not-found

# 2. Destroy underlying AWS infrastructure
cd terraform
terraform destroy -auto-approve
👨‍💻 AuthorSalman Vohra — Cloud & DevOps EngineerSpecializing in AWS Infrastructure Automation, Kubernetes (EKS), GitOps, and DevSecOps.
