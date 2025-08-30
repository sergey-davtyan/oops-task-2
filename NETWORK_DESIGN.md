# Innovate Inc. AWS Network Design

This README describes the AWS VPC architecture and network security controls for Innovate Inc.’s web application (React SPA + Python/Flask REST API on EKS; PostgreSQL on RDS/Aurora). It follows AWS best practices for isolation, scalability, security, and cost efficiency.

## Goals

- Strong isolation between environments and accounts
- Least-privilege network access with explicit allow-lists
- Highly available, multi-AZ design with clear growth path
- Minimize public exposure; prefer private connectivity to AWS services
- Centralized visibility and logs for detection and response

## High-Level Design

- Separate AWS accounts per environment (dev, staging, prod)
- One VPC per environment, three Availability Zones (AZs)
- Private subnets for compute and databases; public subnets only for load balancers and NAT
- CloudFront + AWS WAF at the edge; ALB in front of EKS; RDS in private DB subnets
- VPC endpoints for AWS APIs to reduce internet egress
- Optional centralized networking (Transit Gateway) for future multi-account/multi-VPC growth

### ASCII Diagram (prod example)
Internet
  -> CloudFront (TLS, caching)
     -> AWS WAF (managed + custom rules)
        -> ALB (public subnets, ACM cert)
           -> EKS (private subnets, no public IPs)
              - App Pods (NetworkPolicies, IRSA)
              - AWS LB Controller (manages ALB/TargetGroups)
           -> Egress via NAT Gateway(s) + Network Firewall (optional)
           -> VPC Endpoints (S3, ECR, CloudWatch, STS, SSM, KMS, Secrets Manager)
        -> RDS/Aurora PostgreSQL (DB subnets, no public access)
        -> Route 53 (public + private hosted zones)

---

## VPC Architecture

### VPCs and CIDR Strategy
- One VPC per environment in its workload account:
  - prod: 10.0.0.0/16
  - staging: 10.1.0.0/16
  - dev: 10.2.0.0/16
- Allocate with AWS VPC IPAM to avoid overlap and enable future expansion (add secondary CIDR blocks if needed).
- Reserve non-overlapping ranges for future VPCs (data platforms, analytics).

### Subnet Layout (per VPC, 3 AZs)
- Public subnets (for ALB, NAT Gateways)
- Private-app subnets (for EKS nodes/pods, ECS if used)
- Private-db subnets (for RDS/Aurora)
- Example (prod, us-east-1 with 3 AZs):
  - Public: 10.0.0.0/20, 10.0.16.0/20, 10.0.32.0/20
  - Private-app: 10.0.64.0/18, 10.0.128.0/18, 10.0.192.0/18
  - Private-db: 10.0.48.0/24, 10.0.49.0/24, 10.0.50.0/24
- Route tables:
  - Public: route 0.0.0.0/0 to Internet Gateway (IGW)
  - Private-app: route 0.0.0.0/0 to NAT Gateway in same AZ (zonal HA)
  - Private-db: no direct internet route; only VPC-local and service endpoints

### Ingress and Load Balancing
- CloudFront:
  - SPA static content (S3 origin) and/or ALB origin for API
  - TLS 1.2+, HSTS, origin access control to S3
  - Caching for SPA and safe API GETs
- AWS WAF:
  - Attach to CloudFront (preferred) or ALB
  - Managed Core/SQLi/XSS rules; custom IP allow/deny lists; bot control as needed
- ALB (public subnets):
  - Managed by AWS Load Balancer Controller in EKS for HTTP/HTTPS Ingress
  - ACM-managed certificates; redirect HTTP->HTTPS
  - NLB for TCP/gRPC if required by specific services

### EKS Networking (Workloads)
- Deploy EKS into private-app subnets; nodes have no public IPs
- Cluster endpoint: private access enabled; public access disabled or restricted to admin IPs/VPN/IAP
- VPC CNI with prefix delegation for higher pod density; consider Security Groups for Pods (advanced)
- Enforce Kubernetes NetworkPolicies (Calico/Cilium) for pod-to-pod and namespace isolation
- AWS IRSA (IAM Roles for Service Accounts) to remove node-wide credentials

### Database Networking
- Amazon RDS or Aurora PostgreSQL in private DB subnets across 3 AZs
- No public access; security group allows 5432 only from app-tier SG
- Use RDS Proxy for connection pooling (optional)
- Option groups/parameter groups tuned for TLS in-transit and performance

### Egress
- NAT Gateways:
  - Prod: one NAT per AZ for HA and zone-local routing
  - Dev/Staging: start with one NAT to reduce cost; scale to per-AZ later
- Optional AWS Network Firewall for egress allow-listing, TLS inspection (if mandated), and threat protections
- Route 53 Resolver DNS Firewall for domain-based egress controls

### Private Access to AWS Services (VPC Endpoints)
Create endpoints to keep traffic private and cut NAT cost:
- Gateway endpoints: S3, DynamoDB (if used)
- Interface endpoints (enable private DNS): ECR (api+dkr), CloudWatch Logs, CloudWatch, STS, SSM, SSMMessages, EC2, EC2 Messages, Secrets Manager, KMS, SNS/SQS (if used), CodeBuild/CodeArtifact (if used)
- Apply endpoint policies to restrict actions and namespaces as feasible

### DNS and Naming
- Route 53 public hosted zone for customer domains (e.g., app.example.com)
- Route 53 private hosted zones per VPC (e.g., svc.prod.local) for internal service discovery
- Route 53 Resolver:
  - Outbound endpoints + rules for on-prem/other VPCs (future)
  - Inbound endpoints if external resolvers must query private zones (rare)
- Split-horizon DNS for consistent names across internal/external as needed

### Inter-VPC and External Connectivity
- Transit Gateway (TGW) in Shared Services account for:
  - VPC-to-VPC connectivity at scale (prod, staging, dev, future data VPCs)
  - Centralized egress via inspection VPC (Network Firewall) if adopted
- Site-to-Site VPN or Direct Connect for future enterprise/on-prem connectivity
- Share TGW attachments and Route 53 rules via AWS RAM

---

## Network Security Controls

Layered controls from edge to workload:

1) Edge
- CloudFront + AWS WAF: OWASP managed rules, rate limiting, geo controls as needed
- AWS Shield (Standard by default; consider Shield Advanced for SLA/DDoS response)
- TLS:
  - ACM certificates for CloudFront/ALB
  - Enforce TLS 1.2+ and HSTS; mutual TLS for partner APIs if required

2) Load Balancing and L7 Policies
- ALB security group permits 80/443 from CloudFront/Internet as applicable; prefer 443-only
- ALB access logs to centralized S3 (Log Archive account)

3) VPC Boundaries
- Security Groups: primary enforcement mechanism
  - App SG -> DB SG only on 5432
  - ALB SG -> App SG on app ports (e.g., 80/8080), not wide-open
  - Principle of least privilege; use SG referencing rather than CIDRs where possible
- Network ACLs:
  - Keep stateless defaults simple (allow within VPC, block known bad CIDRs if required)
  - Rely on SGs for fine-grained control
- VPC Flow Logs:
  - Enabled at VPC/Subnet/ENI as appropriate
  - Send to CloudWatch Logs or S3 in Log Archive with lifecycle policies

4) Workload (EKS)
- Private nodes, no public IPs; IMDSv2 enforced on nodes
- Kubernetes NetworkPolicies (deny-by-default between namespaces)
- Pod Security Standards; restrict host networking/privileged pods
- Secrets via AWS Secrets Manager or Kubernetes Secrets encrypted with KMS; mount via CSI driver
- IRSA to scope AWS permissions per workload
- AWS Load Balancer Controller with least-privilege IAM

5) Data Layer
- RDS subnet group in private DB subnets
- Enforce SSL/TLS to database; rotate credentials; use IAM auth where feasible
- KMS CMKs for encryption at rest with strict key policies
- No security group ingress from 0.0.0.0/0

6) Egress Controls
- NAT egress monitored; prefer interface endpoints for AWS APIs
- Optional centralized egress VPC with Network Firewall (domain/IP allow-list, malware categories)
- Route 53 Resolver DNS Firewall to block known bad domains
- S3 access via Gateway Endpoint with bucket policies denying public access and enforcing aws:SourceVpce

7) Administrative Access
- No bastion hosts; use AWS Systems Manager Session Manager
- EKS API: private endpoint; kubectl via SSM/SSO jump (or VPC connectivity)
- Tight IAM boundaries; SCPs deny public IPs in prod and require encryption/tagging

8) Monitoring and Detection
- GuardDuty across all accounts/regions
- Security Hub with CIS/AWS Foundational Security Best Practices
- Inspector for ECR/ECS/EKS container scanning
- Centralized alerting to Slack/Email/PagerDuty

---

## Logging and Observability

- ALB, CloudFront, NAT, Network Firewall logs -> S3 in Log Archive account with S3 Object Lock and lifecycle retention
- VPC Flow Logs -> CloudWatch Logs/S3 (centralized)
- EKS:
  - Container logs to CloudWatch or OpenTelemetry collector -> observability backend
  - Metrics (Prometheus) and traces (OTel) exported to managed backends
- Route 53 Resolver query log and DNS Firewall logs for visibility

---

## Cost Considerations

- NAT Gateways: biggest variable; start with one in nonprod, scale to per-AZ in prod
- Prefer VPC endpoints (S3 Gateway, ECR, CloudWatch) to reduce NAT traffic
- Right-size flow log sampling and log retention (e.g., 30–90 days hot, archive to S3 Glacier)
- CloudFront caching to reduce origin load and egress cost

---

## Implementation Notes (Terraform-friendly)

- Create VPCs, subnets, route tables, IGW, NAT, EIPs
- Create VPC endpoints (S3 gateway + critical interface endpoints)
- Security groups: alb_sg, app_sg, db_sg; reference by ID
- EKS cluster: private endpoint, node groups in private-app subnets; attach LB Controller/IAM
- RDS/Aurora: subnet group in private-db subnets; SG from app_sg:5432 only; no public access
- CloudFront + WAF: OAC to S3, ALB origin, ACM certs
- Enable Flow Logs to centralized destinations
- Optional: TGW in Shared Services, attachments from workload VPCs, Network Firewall egress VPC, DNS sharing via RAM

---

## Future Expansion

- Multi-region DR: duplicate VPC topology, Route 53 health checks and failover, cross-region RDS read replica
- Cross-account service-to-service: Transit Gateway or AWS VPC Lattice for fine-grained, authenticated service connectivity
- Advanced egress controls: fully centralized inspection with Network Firewall and TLS decryption (if policy permits)

This network design provides secure defaults, high availability, and a clear path to scale without re-architecting core foundations.
