# Feature Specification: Refactor Helm Charts to Remove Hardcoding

**Feature Branch**: `001-refactor-helm-charts`

**Created**: 2026-06-09

**Status**: Draft

**Input**: User description: "Refactor existing helm charts to remove any form of hardcoding. Templates should follow helm best practices"

## Clarifications

### Session 2026-06-09

- Q: How should the templates resolve the namespace when it is not explicitly provided at deployment time? → A: Add a `.Values.namespaceOverride` field defaulting to `home`. Templates resolve namespace using `{{ .Values.namespaceOverride | default .Release.Namespace }}`.
- Q: How should the distinct namespaces in the Vault chart templates (vaultstaticsecret.yaml, vaultauth.yaml, vaultconnection.yaml) be parameterized? → A: Keep namespaces static (hardcoded "home" and "cert-manager" in templates) since they represent fixed cross-namespace system bindings.


## User Scenarios & Testing *(mandatory)*

### User Story 1 - Refactor Static Service/Endpoint Wrappers (Priority: P1)

As a cluster administrator, I want to deploy external reverse-proxy services without maintaining duplicate, static YAML templates for every target server, so that I can manage targets dynamically from `values.yaml`.

**Why this priority**: The `reverse-proxy` chart currently has duplicate, static resource definitions (Plex, Proxmox, Synology, UniFi, WireGuard) which duplicate logic and make maintaining host/IP configurations tedious and error-prone.

**Independent Test**: Modifying the proxy targets in `values.yaml` dynamically creates or removes the corresponding Certificate, ServersTransport, Service, Endpoints, and IngressRoute resources when rendered.

**Acceptance Scenarios**:

1. **Given** a list of proxy configurations under `proxies:` in `values.yaml`, **When** I run `helm template`, **Then** Helm should output a full set of Kubernetes resources (Service, Endpoints, IngressRoute, etc.) for each enabled proxy.
2. **Given** an entry in `proxies:` is disabled (`enabled: false`), **When** I run `helm template`, **Then** no resources for that entry should be generated.

---

### User Story 2 - Parameterize Dashboard Configurations (Priority: P1)

As a cluster administrator, I want dashboard apps (like `homepage`) to have their configuration maps generated dynamically from `values.yaml`, so that changes to bookmarks, services, and widgets don't require editing raw ConfigMap templates.

**Why this priority**: Hardcoded URLs and hostnames in ConfigMaps prevent deployment to other environments (e.g. testing/staging) and violate the DRY principle.

**Independent Test**: Bookmark and service links in the homepage configmap can be configured and updated purely through `values.yaml`.

**Acceptance Scenarios**:

1. **Given** customizable YAML block properties in `values.yaml` (e.g. `config.bookmarks`, `config.services`), **When** I run `helm template`, **Then** the generated ConfigMap contains those exact properties serialized correctly.

---

### User Story 3 - Generalize Domains, IPs, and Namespaces (Priority: P1)

As a cluster administrator, I want all domain names, IP addresses, and namespaces in all charts to be parameterized using values and release context variables, so that the entire chart suite is portable and environment-agnostic.

**Why this priority**: Hardcoded domains (`teegeedoubleyou.work`), internal IP subnets, and namespaces (`namespace: home`) couple the configuration tightly to a specific environment instance, violating Core Principle I (Strict Declarative GitOps).

**Independent Test**: Overriding the domain value at render time updates all ingresses, certificates, and routes to use the new domain.

**Acceptance Scenarios**:

1. **Given** a target root domain configured in `values.yaml` (e.g., `global.domain` or `.Values.domain`), **When** I run `helm template`, **Then** all generated Ingresses and Certificates use hostnames under that root domain.
2. **Given** the chart is deployed to a different namespace (e.g. `helm install -n staging`), **When** I check the rendered manifests, **Then** no hardcoded `namespace: home` declarations exist, and resources are correctly assigned to the release namespace.

---

### Edge Cases

- **Missing domain/IP settings**: The chart templates must fall back to safe defaults (e.g., `.Values.domain | default "home.arpa"`) or fail rendering with a clear error message (using `required`) if critical values like target IPs are omitted.
- **Port mapping discrepancies**: Proxy targets may use different protocols (HTTP vs HTTPS) or custom ports. The templated Service/Endpoint wrapper must dynamically map the target port and protocol scheme based on values.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: Every chart MUST parameterize its root domain name, replacing any hardcoded occurrences of `teegeedoubleyou.work` with a configurable value.
- **FR-002**: Workloads and configurations MUST support namespace customization via `{{ .Values.namespaceOverride | default .Release.Namespace }}`, defaulting to `home` if not specified. (Exception: Vault operator CRD templates vaultauth.yaml, vaultconnection.yaml, and vaultstaticsecret.yaml in the `vault` chart MUST keep their hardcoded namespaces `home` and `cert-manager`).
- **FR-003**: The `reverse-proxy` chart templates MUST be replaced with a single set of dynamic template loops that iterate over a `.Values.proxies` structure.
- **FR-004**: The `homepage` chart's ConfigMap bookmarks, services, and widgets MUST be dynamically serialized from the chart's `values.yaml`.
- **FR-005**: All templates MUST implement standard Helm labels and annotations using helper templates (`_helpers.tpl`) where possible.
- **FR-006**: All charts MUST pass `helm lint` validation.
- **FR-007**: Resource request and limit settings MUST be defined for all workloads, in accordance with Core Principle IV (Resource Quotas & Scheduling Hardening).

### Key Entities

- **Global Configuration (`values.yaml`)**: The main interface for supplying domains, external IPs, and application configurations.
- **Helper Templates (`_helpers.tpl`)**: Standardized Helm files used to construct names, labels, and common URLs.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: 100% of Helm charts in the repository pass `helm lint` with zero errors or warnings.
- **SC-002**: 100% of hardcoded domains (`teegeedoubleyou.work`), external IPs, and namespaces are parameterized.
- **SC-003**: The file count in `reverse-proxy/templates/` is reduced from 5 static target files to 1 dynamic template loop.
- **SC-004**: Rendered output from `helm template` produces valid Kubernetes YAML files matching the functional behavior of the previous static configurations.

## Assumptions

- The default domain used when no domain is provided will be `home.arpa`.
- ArgoCD sync setups will not be broken by the namespaces dynamically resolving to the release namespace.
