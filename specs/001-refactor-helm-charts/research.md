# Research: Refactoring Helm Charts to Remove Hardcoding

This document outlines the technical research, architectural decisions, and best practices for refactoring the subcharts and parent values in the home cluster GitOps configuration.

## Research Questions & Decisions

### 1. How should domains and IP addresses be parameterized across subcharts?

* **Decision**: Introduce a `global.domain` and `global.ingressIP` configuration in the parent `home/values.yaml`. In the subcharts, ingress and routing templates will reference `.Values.global.domain` to construct hostnames dynamically, while falling back to custom values or local overrides if defined.
* **Rationale**: Defining these variables globally in the parent chart allows a single point of configuration for the entire cluster's external domain name, making it easy to swap domains (e.g., from `teegeedoubleyou.work` to `home.arpa`) without modifying individual subcharts.
* **Alternatives Considered**: 
  * *Local parameters in each subchart*: Rejected because changing the domain would require editing 11 separate `values.yaml` files.
  * *Hardcoding fallback values*: Rejected because it violates Core Principle I (Strict Declarative GitOps).

### 2. How should the static files in the `reverse-proxy` chart be refactored?

* **Decision**: Define a structured `proxies` map in the `reverse-proxy` chart's `values.yaml`. Refactor the templates to loop through `.Values.proxies` and render the Certificate, ServersTransport, Service, Endpoints, and IngressRoute resources dynamically.
* **Rationale**: The current implementation has 5 separate static templates with near-identical resource templates, varying only in naming, domain prefixes, and target IP/ports. Consolidating these into dynamic template loops reduces code duplication, enforces DRY principles, and makes adding new proxy targets simple.
* **Alternatives Considered**:
  * *Keeping separate files but parameterizing each*: Rejected because it still duplicates the resource schemas and boilerplate configurations.

### 3. How should the `homepage` configmap bookmarks and services be parameterized?

* **Decision**: Define standard values blocks in `homepage/values.yaml` under `config.bookmarks`, `config.services`, etc. In the ConfigMap template, serialize these values blocks using Helm's `{{- toYaml .Values.config.bookmarks | nindent 4 }}` (or equivalent).
* **Rationale**: This separates layout configuration from template definition. Bookmarks, widgets, and services can be managed purely via values, and template functions can serialize them without hardcoding hostnames.
* **Alternatives Considered**:
  * *Keeping them static*: Rejected because URLs contain hardcoded domains that must be parameterized.

## Technical Choices & Best Practices

- **Helm Helpers (`_helpers.tpl`)**: Standard templates for common labels, annotations, and naming.
- **Dynamic Ingress Hostnames**:
  ```yaml
  host: {{ .Values.ingress.host | default (printf "%s.%s" .Values.ingress.hostPrefix (required "global.domain is required" .Values.global.domain)) }}
  ```
- **Namespace Scoping**: Always use `{{ .Release.Namespace }}` instead of hardcoded `namespace: home`.
- **Resource Constraints**: Ensure resource requests and limits are defined for all workloads, as per Core Principle IV.
