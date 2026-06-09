# Tasks: Refactor Helm Charts to Remove Hardcoding

**Input**: Design documents from `/specs/001-refactor-helm-charts/`

**Prerequisites**: plan.md (required), spec.md (required for user stories), research.md, quickstart.md

**Tests**: Tests are dry-run template render validations and lint checks.

**Organization**: Tasks are grouped by user story to enable independent implementation and testing of each story.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which user story this task belongs to (e.g., US1, US2, US3)
- Include exact file paths in descriptions

## Path Conventions

- Paths assume parent chart at `home/` and subcharts under `home/charts/`

---

## Phase 1: Setup

**Purpose**: Baseline backup and initial lint validation

- [x] T001 Run baseline lint checks using `helm lint home/`
- [x] T002 Generate baseline rendered templates using `helm template home/ > rendered-baseline.yaml`

---

## Phase 2: Foundational

**Purpose**: Core infrastructure that must be complete before any user story can be implemented

- [x] T003 [P] Define global variables (`global.domain`, `global.ingressIP`, `global.clusterIssuer`, `global.ingressClassName`) in [home/values.yaml](file:///mnt/secondary/projects/argocd-config/home/values.yaml) (see [design.md#1-global-values-schema](file:///mnt/secondary/projects/argocd-config/specs/001-refactor-helm-charts/design.md#1-global-values-schema))
- [x] T004 [P] Create or update common helper templates (`_helpers.tpl`) in subcharts to support global domain suffix rendering (see [design.md#21-unified-template-logic-for-reverse-proxy](file:///mnt/secondary/projects/argocd-config/specs/001-refactor-helm-charts/design.md#21-unified-template-logic-for-reverse-proxy))

---

## Phase 3: User Story 1 - Refactor Static Service/Endpoint Wrappers (Priority: P1)

**Goal**: Consolidate multiple static wrappers into a dynamic template loop under the `reverse-proxy` chart.

**Independent Test**: Modifying proxy configs in `values.yaml` automatically creates/removes corresponding services, endpoints, and ingress routes.

### Implementation for User Story 1

- [x] T005 [US1] Define proxies map structure and populate baseline values in [home/charts/reverse-proxy/values.yaml](file:///mnt/secondary/projects/argocd-config/home/charts/reverse-proxy/values.yaml) (see [design.md#2-reverse-proxy-config-mapping](file:///mnt/secondary/projects/argocd-config/specs/001-refactor-helm-charts/design.md#2-reverse-proxy-config-mapping))
- [x] T006 [US1] Replace static templates in `home/charts/reverse-proxy/templates/` with a unified parameterized `certificate.yaml` loop template (see [design.md#certificate-templatescertificateyaml](file:///mnt/secondary/projects/argocd-config/specs/001-refactor-helm-charts/design.md#certificate-templatescertificateyaml))
- [x] T007 [US1] Replace static templates in `home/charts/reverse-proxy/templates/` with a unified dynamic service-endpoints loop template (see [design.md#service--endpoints-templatesserviceyaml--templatesendpointsyaml](file:///mnt/secondary/projects/argocd-config/specs/001-refactor-helm-charts/design.md#service--endpoints-templatesserviceyaml--templatesendpointsyaml))
- [x] T008 [US1] Replace static templates in `home/charts/reverse-proxy/templates/` with a parameterized `ingressroute.yaml` loop template (see [design.md#ingressroute-templatesingressrouteyaml](file:///mnt/secondary/projects/argocd-config/specs/001-refactor-helm-charts/design.md#ingressroute-templatesingressrouteyaml))
- [x] T009 [US1] Remove deprecated static target files from `home/charts/reverse-proxy/templates/` (plex.yaml, proxmox.yaml, synology.yaml, unifi.yaml, wireguard.yaml)

**Checkpoint**: User Story 1 is complete. `reverse-proxy` chart rendered manifests should be functionally identical to the baseline but dynamically generated.

---

## Phase 4: User Story 2 - Parameterize Dashboard Configurations (Priority: P1)

**Goal**: Parameterize bookmarks, services, and widgets in the `homepage` dashboard chart.

**Independent Test**: ConfigMap layout updates can be managed fully from `homepage/values.yaml`.

### Implementation for User Story 2

- [x] T010 [US2] Move hardcoded dashboard configurations (bookmarks, services, widgets, settings) to [home/charts/homepage/values.yaml](file:///mnt/secondary/projects/argocd-config/home/charts/homepage/values.yaml) (see [design.md#3-homepage-configuration](file:///mnt/secondary/projects/argocd-config/specs/001-refactor-helm-charts/design.md#3-homepage-configuration))
- [x] T011 [US2] Refactor [home/charts/homepage/templates/configmap.yaml](file:///mnt/secondary/projects/argocd-config/home/charts/homepage/templates/configmap.yaml) to dynamically serialize config values using `toYaml` (see [design.md#31-homepage-configmap-template-templatesconfigmapyaml](file:///mnt/secondary/projects/argocd-config/specs/001-refactor-helm-charts/design.md#31-homepage-configmap-template-templatesconfigmapyaml))
- [x] T012 [US2] Rename `deploment.yaml` to `deployment.yaml` and parameterize the `HOMEPAGE_ALLOWED_HOSTS` environment variable in [home/charts/homepage/templates/deployment.yaml](file:///mnt/secondary/projects/argocd-config/home/charts/homepage/templates/deployment.yaml) (see [design.md#32-homepage-deployment-template-templatesdeploymentyaml](file:///mnt/secondary/projects/argocd-config/specs/001-refactor-helm-charts/design.md#32-homepage-deployment-template-templatesdeploymentyaml))

**Checkpoint**: User Story 2 is complete. Dashboard configs are dynamically generated.

---

## Phase 5: User Story 3 - Generalize Domains, IPs, and Namespaces (Priority: P1)

**Goal**: Remove all remaining hardcoded domains, IPs, and namespaces across all remaining subcharts.

**Independent Test**: Modifying the global domain propagates across all ingress hostnames and certificates.

### Implementation for User Story 3

- [x] T013 [P] [US3] Parameterize domains/ingresses in [home/charts/bookstack/values.yaml](file:///mnt/secondary/projects/argocd-config/home/charts/bookstack/values.yaml) and [home/charts/bookstack/templates/ingress.yaml](file:///mnt/secondary/projects/argocd-config/home/charts/bookstack/templates/ingress.yaml) (see [design.md#4-generic-subchart-ingress-pattern](file:///mnt/secondary/projects/argocd-config/specs/001-refactor-helm-charts/design.md#4-generic-subchart-ingress-pattern))
- [x] T014 [P] [US3] Parameterize domains/ingresses in [home/charts/headlamp/values.yaml](file:///mnt/secondary/projects/argocd-config/home/charts/headlamp/values.yaml) and [home/charts/headlamp/templates/ingress.yaml](file:///mnt/secondary/projects/argocd-config/home/charts/headlamp/templates/ingress.yaml) (see [design.md#4-generic-subchart-ingress-pattern](file:///mnt/secondary/projects/argocd-config/specs/001-refactor-helm-charts/design.md#4-generic-subchart-ingress-pattern))
- [x] T015 [P] [US3] Parameterize domains/ingresses in [home/charts/scrypted/values.yaml](file:///mnt/secondary/projects/argocd-config/home/charts/scrypted/values.yaml) and [home/charts/scrypted/templates/ingress.yaml](file:///mnt/secondary/projects/argocd-config/home/charts/scrypted/templates/ingress.yaml) (see [design.md#4-generic-subchart-ingress-pattern](file:///mnt/secondary/projects/argocd-config/specs/001-refactor-helm-charts/design.md#4-generic-subchart-ingress-pattern))
- [x] T016 [P] [US3] Parameterize domains/ingresses in [home/charts/traefik/values.yaml](file:///mnt/secondary/projects/argocd-config/home/charts/traefik/values.yaml) and [home/charts/traefik/templates/ingress.yaml](file:///mnt/secondary/projects/argocd-config/home/charts/traefik/templates/ingress.yaml) (see [design.md#4-generic-subchart-ingress-pattern](file:///mnt/secondary/projects/argocd-config/specs/001-refactor-helm-charts/design.md#4-generic-subchart-ingress-pattern))
- [x] T017 [P] [US3] Parameterize domains/ingresses in [home/charts/uptime-kuma/values.yaml](file:///mnt/secondary/projects/argocd-config/home/charts/uptime-kuma/values.yaml) and [home/charts/uptime-kuma/templates/ingress.yaml](file:///mnt/secondary/projects/argocd-config/home/charts/uptime-kuma/templates/ingress.yaml) (see [design.md#4-generic-subchart-ingress-pattern](file:///mnt/secondary/projects/argocd-config/specs/001-refactor-helm-charts/design.md#4-generic-subchart-ingress-pattern))
- [x] T018 [P] [US3] Parameterize domains/ingresses in [home/charts/vault/values.yaml](file:///mnt/secondary/projects/argocd-config/home/charts/vault/values.yaml) and [home/charts/vault/templates/ingress.yaml](file:///mnt/secondary/projects/argocd-config/home/charts/vault/templates/ingress.yaml) (see [design.md#4-generic-subchart-ingress-pattern](file:///mnt/secondary/projects/argocd-config/specs/001-refactor-helm-charts/design.md#4-generic-subchart-ingress-pattern))
- [x] T019 [US3] Parameterize domain settings in [home/charts/cert-manager/templates/letsencrypt-certificate.yaml](file:///mnt/secondary/projects/argocd-config/home/charts/cert-manager/templates/letsencrypt-certificate.yaml)
- [x] T020 [US3] Parameterize parent chart values for remote subchart dependencies (pihole-1, pihole-2) in [home/values.yaml](file:///mnt/secondary/projects/argocd-config/home/values.yaml)
- [x] T021 [US3] Scan and replace all hardcoded `namespace: home` declarations with `{{ .Values.namespaceOverride | default .Release.Namespace }}` across all custom subchart template files (excluding Vault operator CRD templates vaultauth.yaml, vaultconnection.yaml, and vaultstaticsecret.yaml in the vault chart)

**Checkpoint**: All user stories are complete. There are no remaining instances of hardcoding.

---

## Phase N: Polish & Cross-Cutting Concerns

- [x] T022 Validate final linting using `helm lint home/`
- [x] T023 Render all templates with `helm template home/ > rendered-final.yaml` and compare structure against `rendered-baseline.yaml`
- [x] T024 Perform final validation steps listed in specs/001-refactor-helm-charts/quickstart.md

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: Can start immediately.
- **Foundational (Phase 2)**: Depends on Setup (Phase 1).
- **User Stories (Phases 3-5)**: All depend on Foundational (Phase 2) completion.
  - Can proceed in parallel or sequentially.
- **Polish (Phase N)**: Depends on completion of all user stories.

### Parallel Opportunities

- Foundational tasks (T003, T004) can run in parallel.
- All User Story 3 sub-chart parameterizations (T013 to T018) can run in parallel.

---

## Implementation Strategy

### MVP First (User Story 1 & 2)

1. Setup and Foundational.
2. Refactor static proxy wrappers (User Story 1).
3. Parameterize homepage ConfigMap (User Story 2).
4. Run template rendering check to ensure validity.

### Full Delivery

1. Proceed to generalize all other subcharts (User Story 3).
2. Clean up namespaces and run final diff tests.
