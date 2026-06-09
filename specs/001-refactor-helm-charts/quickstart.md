# Quickstart Validation Guide: Refactor Helm Charts

This guide describes how to validate the refactored Helm charts to ensure they conform to Helm best practices, contain no hardcoded domains or IPs, and render valid Kubernetes manifests.

## Prerequisites

- **Helm**: Helm v3 CLI installed.
- **Kubectl**: kubectl CLI (optional, for schema validation).

## Validation Scenario 1: Linting the Charts

Validate that all subcharts and the parent chart pass Helm's syntax and styling lints.

```bash
# Lint the parent umbrella chart and all its local dependencies
helm lint home/
```

**Expected Outcome**:
```text
==> Linting home/
[INFO] Chart.yaml: icon is recommended

1 chart(s) linted, 0 chart(s) failed
```

---

## Validation Scenario 2: Dry-Run Template Rendering

Render the templates locally and check for any syntax or parameter lookup errors.

```bash
# Render all templates with default values
helm template home/ > rendered-manifests.yaml
```

**Expected Outcome**:
- The command completes successfully without rendering errors.
- `rendered-manifests.yaml` contains complete Kubernetes resources.

---

## Validation Scenario 3: Verify Domain Parameterization

Verify that the hardcoded domain `teegeedoubleyou.work` has been successfully replaced by template logic.

```bash
# Search for the hardcoded domain in the rendered output
grep -i "teegeedoubleyou.work" rendered-manifests.yaml
```

**Expected Outcome**:
- No occurrences of `teegeedoubleyou.work` are found when the domain is overridden, OR all occurrences match the customized domain value passed during rendering.

To test overriding the global domain:
```bash
helm template home/ --set global.domain=custom-domain.local > custom-rendered.yaml
grep -i "custom-domain.local" custom-rendered.yaml
```

**Expected Outcome**:
- All ingress hostnames, routing rules, and certificate names are updated to use `custom-domain.local`.
