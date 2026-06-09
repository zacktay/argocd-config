<!--
SYNC IMPACT REPORT
- Version change: [CONSTITUTION_VERSION] -> 1.0.0
- Modified principles:
  * [PRINCIPLE_1_NAME] -> I. Strict Declarative GitOps
  * [PRINCIPLE_2_NAME] -> II. Continuous Linting & Schema Verification
  * [PRINCIPLE_3_NAME] -> III. Consistent UX & Routing Standards
  * [PRINCIPLE_4_NAME] -> IV. Resource Quotas & Scheduling Hardening
  * [PRINCIPLE_5_NAME] -> V. Standardized Monitoring & Observability
- Added sections:
  * Additional Security Standards (formerly SECTION_2)
  * GitOps Deployment Workflow (formerly SECTION_3)
- Removed sections:
  * None
- Templates requiring updates:
  * .specify/templates/plan-template.md (✅ updated / no changes required)
  * .specify/templates/spec-template.md (✅ updated / no changes required)
  * .specify/templates/tasks-template.md (✅ updated / no changes required)
- Follow-up TODOs:
  * None
-->

# Home Kubernetes Cluster Configuration Constitution

## Core Principles

### I. Strict Declarative GitOps

All cluster configurations, Helm values, and Kubernetes manifests MUST be fully declared in
this repository. Manual modifications (e.g. direct `kubectl edit` or editing resources
in-cluster) are strictly prohibited. Configuration files MUST be clean, structured, and
modular. Values must be parameterized via `values.yaml` files, avoiding any hardcoded
environment-specific credentials or hostnames. Secrets MUST NOT be checked into version control
and must be managed via Hashicorp Vault.

### II. Continuous Linting & Schema Verification

Before merging any changes to the main branch, Helm charts and manifests MUST pass automated
linting (`helm lint`), formatting, and structural checks. All generated manifests MUST be
dry-run tested using `helm template` or `kubectl apply --dry-run=client` to ensure syntactic
and schema correctness. Where feasible, validation must target a staging namespace or testing
cluster before production rollout.

### III. Consistent UX & Routing Standards

Every service exposing a user interface MUST configure a consistent ingress using automated
TLS certificates from Cert-Manager. Hostnames must strictly follow the standard internal domain
pattern (e.g., `[app].home.arpa` or subdomain convention). Ingress rules must support clean
paths and redirect HTTP to HTTPS. Dashboard interfaces must load reliably with correct assets
and provide clear navigation links back to central cluster status tools (e.g., Headlamp).

### IV. Resource Quotas & Scheduling Hardening

Every workload (Deployment, StatefulSet, DaemonSet) MUST define explicit CPU and memory request
and limit profiles. No pod may run without resource limits. Pods MUST configure appropriate
health checks (startup, liveness, and readiness probes) with reasonable timeouts and threshold
values. Rolling update strategies must ensure service availability during upgrades.

### V. Standardized Monitoring & Observability

Every application deployment MUST expose telemetry endpoint configurations or annotate services
for Prometheus scraping. A corresponding monitoring endpoint MUST be registered in Uptime Kuma
to verify service availability. Standardized alerting channels (e.g., webhook notifications for
container restarts, ingress failures, or TLS expiration) must be defined for all critical apps.

## Additional Security Standards

All sensitive credentials, API keys, and certificates MUST be stored in Hashicorp Vault and
injected dynamically into pods using the Vault Secrets Operator or External Secrets Operator.
Under no circumstances may raw secret values be committed to Git. All namespaces must enforce
basic isolation rules where possible.

## GitOps Deployment Workflow

Changes to cluster state are driven by commits to the `main` branch. Developers must commit
modifications to feature branches, verify them via local dry-runs, and submit a Pull Request.
ArgoCD will automatically detect changes and reconcile the cluster state once merged. Manual
sync override is permitted only for emergency recovery and must be logged.

## Governance

This constitution defines the non-negotiable standards for configuring, deploying, and
maintaining the Home Kubernetes Cluster configuration. All pull requests, code modifications,
and architecture updates must comply with these core principles. The constitution version will
be updated using semantic versioning whenever new standards are introduced or existing ones
are modified.

**Version**: 1.0.0 | **Ratified**: 2026-06-09 | **Last Amended**: 2026-06-09
