# ADR-006: Terraform Over Crossplane for Infrastructure-as-Code

**Status:** Accepted, migration complete

## Context

The platform's infrastructure spans a genuinely wide surface: networking (VPCs), IAM roles,
databases (Aurora Postgres), caching (Redis/Valkey), API Gateway, Lambda, SQS, DynamoDB tables,
S3 buckets, WAF rules, KMS keys, and multi-region deployment (separate US and EU production
regions, plus lower environments) — all of which needed to be reproducible, reviewable, and
safely deployable by a small platform team without becoming a full-time job for one person.

We'd started with Crossplane (Kubernetes-native infrastructure management — infrastructure
resources as Kubernetes custom resources, reconciled by controllers running in-cluster) as the
IaC approach. By the time we made this decision, we already had real production experience with
it, so this wasn't a green-field choice — it was a "do we keep going or cut over" decision made
with actual operational data in hand.

## Options considered

1. **Keep Crossplane.**
   - Kept native Kubernetes API-style management, which fit teams already deep in Kubernetes
     tooling.
   - In practice, we found the AWS provider coverage lagged behind what we needed (newer AWS
     resource types/features showed up in Terraform's AWS provider well before Crossplane's),
     debugging a stuck reconciliation loop was often harder than reading a plan diff, and the
     team's actual infra expertise (and the wider hiring/onboarding pool) skewed much more
     toward Terraform than Crossplane.

2. **Terraform**, using standard modules and a plan/apply workflow, orchestrated via a managed
   Terraform-runs platform (Spacelift) for approvals and state management.
   - Mature, first-class AWS provider with fast coverage of new AWS features.
   - `terraform plan` gives a reviewable, human-readable diff before anything is applied — a
     concrete artifact that goes into the PR review, versus a reconciliation loop's eventual
     convergence being harder to review ahead of time.
   - Much larger hiring pool and existing team familiarity, which matters directly for a small
     team's bus factor.

3. **Stay hybrid** (Crossplane for some resources, Terraform for others, split by team or
   resource type).
   - Rejected — running two IaC systems in parallel means two mental models, two review
     processes, and ambiguity about which one "owns" a given piece of infrastructure, which is
     exactly the kind of thing that causes an outage from an infra change nobody thought would
     conflict.

## Decision

We migrated fully to Terraform, with a defined hard deployment-order dependency chain (AWS
IAM roles and account foundation → global resources/DNS → networking/VPC → shared
infrastructure like load balancers and KMS → datastores/API Gateway/integration resources,
several of which can deploy in parallel once the earlier layers exist), orchestrated through
Spacelift for run approval and state management, with infrastructure split across dedicated
repos (core platform infra vs. per-service infra where services own their own narrower
resources).

We made this an explicit platform-wide, dated cutover (documented as the point after which all
new infrastructure work must be in Terraform) rather than a slow, indefinite parallel-run —
partly because we'd learned from the hybrid Crossplane/manual-changes period before it that an
undefined transition period just accumulates undocumented exceptions.

## Tradeoffs

**What we gave up:**
- A real migration cost — every existing Crossplane-managed resource had to be imported into
  Terraform state carefully (state import is unforgiving; get it wrong and you either lose track
  of a resource or, worse, Terraform decides to recreate something that's actually fine and
  serving traffic).
- Kubernetes-native resource management (managing infra the same way you manage app deployments,
  via `kubectl`/GitOps against the cluster) — some teams genuinely liked that model and had
  tooling built around it that needed to change.
- Terraform's plan/apply model is less naturally "continuously reconciling" than Crossplane's
  controller model — drift between actual infrastructure and Terraform state can accumulate
  silently between runs in a way a continuously-reconciling controller would self-heal, and we
  had to add drift-detection tooling to compensate.

**What we got:**
- Faster access to new AWS features and resource types as they shipped, instead of waiting on
  Crossplane provider coverage.
- A genuinely better review experience — `terraform plan` output in a PR is something any
  engineer on the team, not just an infra specialist, can read and sanity-check before approval.
- A much shallower onboarding curve for infra work, since Terraform knowledge is common and
  transferable, versus Crossplane being a narrower, more specialized skill within the team.
- One IaC system, one mental model, one place infra changes are reviewed — removed the
  "which system actually owns this resource" ambiguity that the hybrid period had produced.

## Outcome

The migration completed as planned and was paired with self-healing retry for failed
provisioning and strengthened secret rotation as part of the same modernization push, since we
were already touching the infra layer end-to-end. Since cutover, infra-related incidents caused
by unreviewable or poorly-understood changes dropped noticeably — largely because the plan-diff
review step catches things a reconciliation loop's eventual-convergence behavior would previously
have masked until something actually broke.

## What I'd reconsider today

The state-import phase of the migration was the highest-risk part and took longer than we
initially scoped, mostly because we underestimated how many resources had accumulated
undocumented manual tweaks outside of Crossplane's control during the earlier period. I'd budget
significantly more time for an infrastructure audit before any future large-scale IaC migration,
rather than assuming the existing tool's state is a complete and accurate picture of reality.
