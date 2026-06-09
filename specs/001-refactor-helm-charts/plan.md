# Implementation Plan: Refactor Helm Charts to Remove Hardcoding

**Branch**: `001-refactor-helm-charts` | **Date**: 2026-06-09 | **Spec**: [spec.md](file:///mnt/secondary/projects/argocd-config/specs/001-refactor-helm-charts/spec.md)

**Input**: Feature specification from `/specs/001-refactor-helm-charts/spec.md`

## Summary

Refactor the existing subcharts and parent configurations within the home cluster configuration repository to remove all hardcoded values (domains, external IPs, and namespaces) and align with Helm best practices. This includes introducing a dynamic loop pattern in the `reverse-proxy` chart and parameterizing ConfigMaps (like the `homepage` dashboard).

## Technical Context

**Language/Version**: Helm v3 (Kubernetes API v1.28+)

**Primary Dependencies**: Traefik (IngressRoute), Cert-Manager (ClusterIssuer), Hashicorp Vault (Secrets)

**Storage**: PersistentVolumeClaims (using `longhorn` storage class)

**Testing**: `helm lint`, `helm template` dry-run validation

**Target Platform**: K3s (Home Kubernetes Cluster)

**Project Type**: Helm parent (umbrella) chart with subcharts

**Performance Goals**: Fast ArgoCD synchronization, zero downtime during rolling upgrades, and zero cluster resource starvation.

**Constraints**: All configurations must be environment-agnostic; no raw credentials/secrets in Git.

**Scale/Scope**: 11 local/remote subcharts managed in `home/charts/`

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principle | Check / Gate Requirements | Status |
|-----------|---------------------------|--------|
| I. Strict Declarative GitOps | All configurations, values, and manifests must be declarative. No manual overrides. | ✅ PASS |
| II. Continuous Linting & Schema Verification | Lint checks must verify formatting. Dry-run checks must ensure manifest correctness. | ✅ PASS |
| III. Consistent UX & Routing Standards | Ingress routing must use parameterized domains and Cert-Manager TLS. | ✅ PASS |
| IV. Resource Quotas & Scheduling Hardening | Request/limit allocations must be defined for all workload pods. | ✅ PASS |
| V. Standardized Monitoring & Observability | Telemetry scrape rules and dashboard paths must be parameterizable. | ✅ PASS |

## Project Structure

### Documentation (this feature)

```text
specs/001-refactor-helm-charts/
├── plan.md              # This file
├── research.md          # Phase 0 output
├── design.md            # Phase 1 design specification (value maps and templates)
├── quickstart.md        # Phase 1 output
└── checklists/
    └── requirements.md  # Spec Quality Checklist
```

### Source Code (repository root)

```text
home/
├── Chart.yaml           # Umbrella chart definition
├── values.yaml          # Global values for umbrella dependencies
└── charts/              # Sub-charts representing individual applications
    ├── bookstack/
    ├── cert-manager/
    ├── headlamp/
    ├── homepage/
    ├── longhorn/
    ├── metallb/
    ├── reverse-proxy/
    ├── scrypted/
    ├── traefik/
    ├── uptime-kuma/
    └── vault/
```

**Structure Decision**: The project uses an Umbrella Chart structure under `home/`. Subcharts are located under `home/charts/`. Global overrides are specified in `home/values.yaml`.

## Complexity Tracking

*No violations identified.*
