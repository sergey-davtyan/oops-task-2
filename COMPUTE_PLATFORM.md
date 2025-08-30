# Innovate Inc. Compute Platform (AWS EKS)

This README describes how Innovate Inc. will deploy and operate the web application (React SPA frontend, Python/Flask REST API) on Amazon EKS. It covers cluster design, node groups, scaling, resource allocation, containerization, image registry, and deployment processes tailored for a startup that expects rapid growth.

## Goals

- Minimize operational overhead while following AWS/EKS best practices
- Strong security posture for sensitive user data
- Smooth path from low traffic to millions of users without major rework
- Clear separation of concerns between environments and system vs app workloads

## Summary

- Use Amazon EKS in each workload account (dev, staging, prod) with private clusters across three AZs.
- Start with Managed Node Groups and Cluster Autoscaler; adopt Karpenter later for cost-efficient, flexible scaling.
- Use AWS Load Balancer Controller to expose the API via an internet-facing ALB. Serve the SPA via S3 + CloudFront (recommended).
- Store images in Amazon ECR. Build, scan, sign, and deploy via a centralized CI/CD account assuming roles into workload accounts.
- Enforce security with IRSA, Secrets Store CSI Driver + AWS Secrets Manager, Pod Security Standards, and NetworkPolicies.

---

## EKS Cluster Design

- One EKS cluster per environment in its respective account:
  - eks-dev, eks-staging, eks-prod
- Regions and AZs:
  - Primary region (e.g., us-east-1), spread nodes across three AZs for HA.
- Endpoint and networking:
  - Private cluster endpoint enabled; public endpoint disabled (or allow-listed temporarily for bootstrap).
  - Nodes in private subnets without public IPs.
  - VPC CNI with prefix delegation enabled for higher pod density.
- Control plane logging:
  - Enable API, audit, authenticator, controllerManager, scheduler logs to CloudWatch; forward to centralized log archive.

Core add-ons (managed or via Helm):
- AWS VPC CNI (managed)
- CoreDNS and kube-proxy (managed)
- EBS CSI Driver (gp3 default StorageClass)
- AWS Load Balancer Controller (for ALB/NLB)
- Metrics Server (for HPA)
- Cluster Autoscaler (or Karpenter later)
- Secrets Store CSI Driver + AWS provider (for Secrets Manager/Parameter Store)
- ExternalDNS (optional) to manage Route 53 records for Services/Ingress
- Prometheus stack and OpenTelemetry Collector (observability)
- Kyverno or Gatekeeper (policy), optional but recommended
- Goldilocks (VPA advisor) in nonprod for right-sizing recommendations

Namespaces (baseline):
- app-frontend (if served from EKS; see SPA note below)
- app-backend
- ingress-system (ALB controller)
- ops (monitoring/telemetry)
- kube-system (managed)

RBAC and access:
- Federate via AWS IAM Identity Center; bind Kubernetes RBAC to SSO groups.
- Separate roles for admin, operator, developer (least privilege).

---

## Node Groups, Instances, and OS

Start simple, grow as needed.

Managed Node Groups (MNG):
- mng-system:
  - Purpose: core controllers, telemetry, ingress controller
  - Instance type: t3.medium or t3.large initially
  - Size: min 2, max 4
  - Taint: node-role.kubernetes.io/system=true:NoSchedule
- mng-app-general:
  - Purpose: backend API and general workloads
  - Instance type: m6i.large (general purpose, good baseline) or c6i.large for compute-leaning workloads
  - Size: min 3 (one per AZ), max 12
  - No taint; use tolerations on system pods to avoid landing here
- mng-spot-optional (nonprod first, prod when comfortable):
  - Purpose: cost-optimized for non-critical jobs/queues/batch
  - Instance types: diversified (m6i.large,c6i.large,c6a.large,c5a.large, etc.)
  - Capacity type: Spot
  - Taint: spot=true:NoSchedule; schedule tolerant workloads only

OS and security:
- Prefer Bottlerocket for container-only, auto-updating, hardened nodes.
- Enforce IMDSv2 and limited hop count on nodes.
- Node root volumes: gp3, suitable size (e.g., 50–100 GiB) with proper throughput.

Scheduling and isolation:
- Use taints/tolerations and node selectors to separate system and app pods.
- TopologySpreadConstraints to distribute replicas across AZs and nodes.

Storage:
- Default StorageClass: gp3 via EBS CSI with encryption by default (CMK/KMS).
- If shared files needed (rare), add EFS CSI and a dedicated EFS.

---

## Scaling Strategy

Application-level:
- Horizontal Pod Autoscaler (HPA) on deployments:
  - Backend API: scale by CPU and memory; add custom metrics (p95 latency, RPS) via Prometheus Adapter.
  - Frontend (if containerized): usually static; scale to maintain availability.
- PodDisruptionBudgets (PDB) for critical deployments to maintain minimum availability during upgrades.
- Readiness/liveness/startup probes on all pods.

Cluster-level:
- Cluster Autoscaler with Managed Node Groups initially:
  - Scale up when pending pods cannot be scheduled; scale down gracefully.
  - Separate autoscaling groups per node group with distinct labels.
- Karpenter (future enhancement):
  - Replace/augment Cluster Autoscaler for faster, cost-efficient, right-sized provisioning across many instance types and purchase options.
  - Especially useful as traffic grows unpredictably.

Right-sizing:
- Require resource requests/limits for all pods; use LimitRange and ResourceQuota per namespace.
- Use Goldilocks in nonprod to recommend CPU/memory from historical usage.
- Consider VPA in “recommendation” mode only (avoid auto-updating limits in prod).

Availability:
- Min replicas per critical service: 3 (one per AZ).
- Use RollingUpdate with maxUnavailable=0 and maxSurge=1 for zero-downtime updates on critical paths.
- Graceful termination and preStop hooks to drain connections (especially for Flask/Gunicorn).

---

## Service Exposure and Networking

- Ingress:
  - AWS Load Balancer Controller provisions an internet-facing ALB for the API.
  - Use Ingress annotations for WAF association, idle timeouts, target group health checks.
  - TLS via ACM certificates; redirect HTTP to HTTPS.
- Internal services remain ClusterIP with Kubernetes NetworkPolicies to restrict traffic.
- Optionally enable Security Groups for Pods for fine-grained controls.

SPA note:
- Recommended: Build React static assets and host on S3 with CloudFront + OAC; API behind ALB.
- Alternative: Serve SPA via NGINX container in EKS (then expose via the same ALB). Prefer S3/CloudFront for cache and cost.

---

## Containerization Strategy

Backend (Python/Flask):
- Use a multi-stage Dockerfile:
  - Builder: python:3.x-slim with build tools to compile wheels; pin dependencies (pip-tools/requirements.txt); run tests.
  - Runtime: distroless/python or python:3.x-slim with production deps only; non-root user; read-only root filesystem; drop Linux capabilities.
- Web server: Gunicorn with uvicorn workers (for async) or standard workers tuned by CPU (e.g., 2–4 per vCPU) and graceful timeouts.
- Health endpoints: /healthz (liveness), /readyz (readiness) with lightweight checks.

Frontend (React):
- Build stage: node:18-alpine to produce optimized static assets.
- Delivery:
  - Preferred: upload to S3 bucket (private) with CloudFront OAC, immutable file names (content hashed), long cache TTL.
  - If containerized: NGINX distroless serving /usr/share/nginx/html; non-root; read-only FS; security headers set.

Security baselines:
- Use .dockerignore to minimize build context.
- Avoid root user; set USER to non-privileged.
- Pin base images and verify signatures where possible.
- Generate SBOM (Syft) and run Trivy/Grype in CI.

---

## Image Registry and Supply Chain Security

Registry:
- Amazon ECR repositories per app and environment (or per account with env-specific tags).
- Enable image scanning (Amazon Inspector) on push.

Tagging and immutability:
- Tags: app-backend:<git-sha>, app-backend:<semver>, latest only in nonprod.
- Disallow tag mutability in prod repos; use SHA for deployments.

Signing and provenance:
- Use Sigstore Cosign to sign images in CI; store signatures in OCI registry (ECR).
- Record build provenance (SLSA level 2+) using GitHub Actions OIDC or CodeBuild attestation.

Cross-account flows:
- CI/CD account assumes an IAM role in workload accounts to push to ECR and deploy.
- ECR pull from within each account using IRSA-bound service accounts (no imagePullSecrets needed for ECR).

Retention:
- ECR lifecycle policies to retain recent N versions and purge old tags/SHA.

---

## Deployment Process

Manifests:
- Use Helm charts with environment-specific values or Kustomize overlays (dev/staging/prod).
- Keep manifests in a separate “deploy” repository/folder; enforce code review.

CI/CD pipeline (example):
- On pull request:
  - Run unit tests, lint, SAST
  - Build images (backend, optional frontend), SBOM, vulnerability scan
- On merge to main:
  - Build images and push to ECR
  - Sign with Cosign and attach provenance
  - Run integration tests (ephemeral namespace or dev cluster)
  - Apply database migrations as a Kubernetes Job (see below)
  - Deploy to dev via Helm/Kustomize
  - Promote to staging, then prod with manual approval
  - Post-deploy smoke tests and rollback on failure

Progressive delivery:
- Use Argo Rollouts for canary/blue-green on the API:
  - Stepwise traffic shifting with ALB weight annotations
  - Automated rollback based on metrics (error rate/latency)
- Alternatively, use standard Deployment rolling updates initially.

Ingress and DNS:
- ExternalDNS manages Route 53 DNS records for the ALB; or manage via Terraform.
- Associate AWS WAF WebACL with the ALB; Shield Standard is on by default.

Migrations:
- Build a migration container (Alembic or Flask-Migrate).
- Run as a pre-deployment Kubernetes Job with proper ordering and idempotency.
- Use RDS Proxy if connection churn is high during deploys.

Config and secrets:
- Non-sensitive config via ConfigMaps or SSM Parameter Store.
- Secrets via Secrets Manager mounted through Secrets Store CSI; rotate regularly.
- Avoid storing secrets in Git or plain Kubernetes Secrets.

---

## Resource Management and Quotas

Policies:
- Enforce LimitRange per namespace (default requests/limits).
- Set ResourceQuota per namespace to cap CPU/memory and object counts.
- Require requests/limits on every pod via admission policy (Kyverno).

Initial sizing (baseline guidance):
- Backend API:
  - requests: 200m CPU, 256Mi memory
  - limits: 500m CPU, 512Mi memory
  - replicas: 3 (min), HPA: 3–20 based on 60% CPU or p95 latency target
- Frontend (if containerized):
  - requests: 50m CPU, 64Mi memory
  - limits: 200m CPU, 256Mi memory
  - replicas: 2–4 for availability
- Tune with load testing and observability; adopt Goldilocks for right-sizing.

Topology and resilience:
- TopologySpreadConstraints ensure even pod distribution across AZs and nodes.
- PDBs to maintain at least N pods during updates.
- Use preStop and terminationGracePeriod for graceful drain.

---

## Security Controls in the Cluster

- IRSA (IAM Roles for Service Accounts) for AWS access from pods; no node-wide credentials.
- NetworkPolicies (Calico or Cilium) to implement deny-by-default between namespaces; allow only required traffic (ALB -> backend, backend -> RDS Proxy).
- Pod Security Standards (restricted):
  - runAsNonRoot, readOnlyRootFilesystem, seccompProfile=RuntimeDefault
  - drop all capabilities; add only needed ones
- Secrets encryption:
  - Kubernetes secrets envelope-encrypted with KMS
  - Prefer Secrets Manager via CSI for application secrets
- Admission control:
  - Kyverno policies to enforce non-root, resource limits, approved registries, image signatures (Cosign policy).
- Image scanning:
  - Amazon Inspector for ECR images; fail pipeline on critical vulns.
- Audit:
  - Enable EKS audit logs; ship to centralized logging and SIEM.
- Backups:
  - Backup manifests and state (e.g., Velero with S3/KMS) for cluster resources if needed.

---

## Local Development and Parity

- Docker Compose for local development with Flask API + Postgres + frontend.
- Use the same Dockerfiles as production to ensure parity.
- Feature branches deploy to dev cluster into isolated namespaces with temporary DNS.

---

## Roadmap Enhancements

- Adopt Karpenter for more efficient autoscaling and instance flexibility.
- Introduce service mesh (App Mesh or Istio) for mTLS and traffic policies if microservices multiply.
- Canary automation with metric-based gates (Prometheus, CloudWatch).
- Multi-region active/passive for DR; ECR cross-region replication and read-observability.

---

## Deliverables and Repos

- app/backend: Flask service, Dockerfile, tests
- app/frontend: React app, build scripts, tests
- deploy/helm or deploy/kustomize: Kubernetes manifests and environment overlays
- .github/workflows or buildspecs: CI/CD pipelines
- policies/: Kyverno/Gatekeeper policies
- scripts/: migration jobs, smoke tests

This compute platform design keeps operations lean for early-stage needs while laying a secure, scalable foundation capable of serving millions of users.
