# Innovate Inc. AWS Cloud Environment Structure

This document recommends the optimal AWS account structure for Innovate Inc. to support isolation, security, scalability, and cost control as the company grows from early-stage to millions of users. It follows AWS best practices using AWS Organizations and Control Tower-style guardrails.

## Objectives

- Strong isolation boundaries (blast-radius reduction) between environments and foundational services
- Clear billing and cost attribution per environment and function
- Centralized security visibility and governance with minimal friction for developers
- Repeatable account vending and consistent guardrails from day one
- A path to scale (more teams, more apps) without major rework

## Summary Recommendation

Stand up an AWS Organization with 8–10 accounts to start, grouped by Organizational Units (OUs). This separates production from non-production, concentrates security controls, centralizes logging, and provides a clean runway for CI/CD and networking shared services.

Minimum viable set (8 accounts):
- Management (payer/root)
- Security
- Log Archive
- Shared Services (networking, IAM tools, directory, Transit Gateway)
- CI/CD and Tooling
- Development (nonprod)
- Staging (nonprod)
- Production

Optional (as you grow):
- Sandbox-<team> accounts under a Sandbox OU
- Data Platform accounts (e.g., data-nonprod, data-prod) if you want stricter data isolation

## Organizational Layout

Use AWS Organizations with OUs for policy and lifecycle control. If available, bootstrap via AWS Control Tower to get turn-key guardrails and Account Factory.

OUs:
- Foundational OU
  - Security Account
  - Log Archive Account
- Infrastructure OU
  - Shared Services Account
  - CI/CD and Tooling Account
- Workloads OU
  - Development Account
  - Staging Account
  - Production Account
- Sandbox OU (optional)
  - Sandbox-<team> (ephemeral/dev spikes)

Notes:
- The Management (payer) account sits at the root and should not host workloads.
- Keep Production in its own OU to apply stricter SCPs and guardrails.

## Account Purposes and Guardrails

1) Management (Payer)
- Purpose: Organization root, consolidated billing, organizations administration.
- Workloads: None. Never deploy resources here.
- Guardrails: Deny creation of IAM users and access keys; enforce MFA for root; restrict regions; allow only org management services.
- Access: Very limited. Break-glass only with hardware MFA.

2) Security
- Purpose: Central security services aggregation and control plane.
- Services: AWS Organizations SCP management (if delegated), IAM Identity Center configuration, GuardDuty (delegated admin), Security Hub (standards), Detective, Inspector, Macie (if used), AWS Config aggregator, centralized KMS keys (optionally), firewall policy management.
- Guardrails: No internet-facing workloads; deny creation of compute resources; restrict to security tools.
- Access: Security engineers only.

3) Log Archive
- Purpose: Immutable, centralized logging and evidence retention.
- Services: Organization CloudTrail (all regions, include management, data, and S3 object-level events as appropriate), AWS Config recorder storage, VPC Flow Logs, ALB/NLB logs, CloudTrail Lake (optional).
- Guardrails: Strict deny on deletion of log buckets/trails (SCPs); S3 object lock (compliance mode if required); no compute.
- Access: Read-only for security/audit; write-only from producer accounts.

4) Shared Services (Networking/Platform)
- Purpose: Shared VPC networking patterns (if using centralized networking), Transit Gateway, Private Hosted Zones, Directory/SSO integrations, shared bastion alternatives (prefer SSM Session Manager), central proxies, shared Container registry mirrors if needed.
- Services: AWS RAM shares (TGW, subnets, Route53 resolver rules), central DNS, maybe ECR mirrors, SSM.
- Guardrails: No business workloads; network change control; limit outbound internet exposure.
- Access: Platform/infra team.

5) CI/CD and Tooling
- Purpose: Build and release tooling isolated from workloads.
- Services: CodeCommit/CodeBuild/CodePipeline or GitHub Actions runners, artifact storage (S3/ECR), image signing with Sigstore/Cosign, vulnerability scanning, Terraform pipelines (with limited cross-account roles), SCA/SAST tools.
- Guardrails: CI roles assume into workload accounts via least-privilege roles; deny standing write access to production data.
- Access: Platform and release engineers. Developers can view logs/artifacts and trigger pipelines.

6) Development (Nonprod)
- Purpose: Day-to-day development and integration testing.
- Services: Application stacks (e.g., EKS clusters, RDS dev instances), feature testing resources.
- Guardrails: Cost and size guardrails; allow broad experimentation but restrict public data exposure; lower quota ceilings and strong budget alarms.
- Access: Developers with elevated rights; audit trails via CloudTrail and AWS Config.

7) Staging (Nonprod)
- Purpose: Pre-production environment for release validation, closer to prod parity.
- Services: Application stacks mirroring prod topology; integration with external systems via test credentials.
- Guardrails: Tighter than dev; limited internet exposure; performance and load test budgets; manual approval link to production release.
- Access: Developers (reduced permissions), SRE, QA.

8) Production
- Purpose: Run customer-facing workloads and sensitive data.
- Services: Production EKS/ECS workloads, RDS/Aurora, ALB/CloudFront, KMS keys with strict policies, WAF.
- Guardrails: Most restrictive SCP set; deny non-approved regions/services; deny creation of IAM users; enforce MFA; block public S3; require encryption (SSE-KMS) and TLS; change windows and deployment approvals; strong budget and anomaly detection.
- Access: SRE/operations with least privilege; deployments via CI/CD only; break-glass process with audit.

Optional) Sandbox OU and Accounts
- Purpose: Isolated playgrounds with strict budgets and automated teardown.
- Guardrails: Low quotas, budget stop actions, region restrictions, no access to org-shared resources by default.

Optional) Data Platform Accounts
- Purpose: If regulatory boundaries require, keep operational data services (e.g., production RDS, Redshift) in dedicated accounts.
- Trade-off: More isolation vs. more cross-account networking and IAM complexity.

## Why Multiple Accounts?

- Isolation and blast-radius reduction: A misconfiguration or compromise in dev cannot impact prod.
- Security boundaries: SCPs and IAM policies differ per OU and account to enforce least privilege.
- Billing clarity: Each account has its own spend and budget alarms; roll-up by OU and cost categories.
- Service quotas and limits: Avoid noisy-neighbor and quota contention across environments.
- Operational safety: Separate CI/CD from workloads; immutable log archive; central security visibility.

## Identity and Access Management Model

- Centralized SSO: Use AWS IAM Identity Center (AWS SSO) integrated with your IdP (e.g., Google Workspace, Okta). Assign permission sets per role and environment (e.g., Dev-PowerUser, Staging-ReadOnly, Prod-Operator, Security-Auditor).
- No IAM users or long-lived keys in workload accounts. Use federated roles and short-lived credentials.
- Cross-account roles: CI/CD account assumes tightly scoped roles in workload accounts for deploys. Human access to prod is read-only by default; privileged actions require Just-In-Time elevation with approvals and MFA (e.g., via IAM Identity Center + Access Portal).
- Break-glass: One emergency role per critical account, protected with MFA and logged; credentials in a secure vault.

## Guardrails with Service Control Policies (SCPs)

Apply SCPs at the OU level and inherit down to accounts. Examples:
- Deny use of non-approved regions (allow only needed, e.g., us-east-1, us-west-2).
- Deny IAM user and access key creation; allow only roles and STS.
- Require MFA for sensitive actions (e.g., UpdateAccessKey, DeleteBucketPolicy).
- Deny disabling or tampering with CloudTrail, AWS Config, GuardDuty, Security Hub.
- Deny S3 PutObject without encryption (SSE-S3 or SSE-KMS) and bucket policies that allow public read/write.
- Deny creation of resources without required tags (env, app, owner, data-classification).
- Restrict use of specific high-risk services (e.g., EC2 Classic—not applicable in new accounts—, or any not on the approved list).
- Enforce VPC usage: Deny creation of public IPs or internet gateways in prod unless via approved IaC paths.
- KMS controls: Restrict KMS key management to security and enforce key policies with grants for workloads.

Use AWS Control Tower “guardrails” where possible to codify these quickly.

## Billing and Cost Management

- Consolidated billing in the Management account.
- Budgets and Alerts:
  - Per account budget alerts (email/Slack) at 50/75/90/100% of forecast/actual.
  - OU-level and tag-based budgets (e.g., env:prod).
  - AWS Cost Anomaly Detection with action items.
- Cost and Usage Report (CUR) to S3 in the Log Archive (or a dedicated Billing/FinOps account if preferred).
- Cost Categories and mandatory tags:
  - env: dev|staging|prod|shared
  - app: innovate-web
  - owner: team-email
  - cost-center: e.g., ENG, PLATFORM
  - data-classification: public|internal|confidential|restricted
- Savings plans and RIs purchased centrally; allocation via cost categories.
- Control high-cost services in dev/staging via SCPs and quotas.

## Region Strategy

- Choose a primary region aligned to customers and latency (e.g., us-east-1).
- Enable critical global services: CloudTrail (all regions), GuardDuty (all regions), AWS Config (all regions).
- Restrict non-approved regions via SCP.
- For DR, add a secondary region (e.g., us-west-2) in the roadmap; replicate logs and critical data where appropriate.

## Naming and Tagging Conventions

- Account names: innovate-mgmt, innovate-security, innovate-log-archive, innovate-shared, innovate-cicd, innovate-dev, innovate-staging, innovate-prod, innovate-sandbox-<team>.
- Email aliases per account for root contact (use a plus-addressing pattern).
- Tag standards enforced via SCP and Config rules:
  - env, app, owner, cost-center, data-classification, confidentiality, compliance-scope (e.g., PCI, HIPAA).

## Provisioning and Lifecycle

- Bootstrap with AWS Control Tower if possible. Otherwise:
  - Create AWS Organization and OUs.
  - Apply baseline SCPs to OUs.
  - Create accounts via Account Factory or Organizations API.
  - Enable and centralize: CloudTrail, AWS Config, GuardDuty, Security Hub, Inspector.
  - Set up Organization-wide CloudTrail to Log Archive with S3 Object Lock; configure AWS Config aggregators.
  - Turn on cross-account Security Hub and GuardDuty with Security account as delegated admin.
  - Set budget alerts per account.
  - Establish CI/CD account roles to assume into workload accounts.
  - Enforce mandatory tags and region restrictions.
- Manage all baseline via IaC (Terraform or CloudFormation StackSets) to avoid drift.

## Access Patterns

- Developers:
  - Full access to Dev; limited to Staging; read-only in Prod.
  - No direct access to Management/Security/Log Archive.
- Platform/SRE:
  - Elevated access to Shared Services and CI/CD.
  - Operator privileges in Prod via JIT/MFA and change control.
- Security:
  - Audit and investigator roles across all accounts (read-only, with specific incident response roles).

## Growth Path

- New Workloads: Add accounts per product/team under the Workloads OU to prevent blast radius and clarify billing.
- Stricter Data Isolation: Introduce data-prod and data-nonprod accounts if regulatory or data exfiltration risks grow.
- Multi-Region: Mirror the structure for secondary regions with DR runbooks and SCP updates.
- FinOps: Add a FinOps/Billing account if your finance team needs direct CUR access and cost dashboards separate from security logs.

## Minimal Alternative (If You Must Start Smaller)

If you need to start with fewer accounts temporarily, use this 5-account setup and plan to split within 3–6 months:
- Management (payer)
- Security + Log Archive (combined)
- Shared Services + CI/CD (combined)
- Nonprod (dev + staging)
- Production

Trade-offs: Less isolation for security/logging and platform tools; more risk and migration effort later. Prefer the 8-account model where feasible.

---

Adopting this structure early gives Innovate Inc. strong security posture, clear cost management, and the ability to scale teams and workloads without painful re-architecture.
