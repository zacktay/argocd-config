# Home Kubernetes Cluster Configuration

This repository contains the GitOps configuration for my home Kubernetes cluster. It uses ArgoCD to automatically sync and manage applications defined as Helm charts.

## Core Components

The cluster is built upon a set of core components that provide essential services like networking, storage, and certificate management.

- **Container Orchestration**: Kubernetes (k3s)
- **GitOps**: ArgoCD
- **Package Management**: Helm
- **Storage**: Longhorn for distributed block storage.
- **Networking**: MetalLB for bare-metal load balancing.
- **TLS Certificates**: Cert-Manager for automated certificate management.

---

## Repository Structure

The repository is organized to be used with ArgoCD, with applications defined as Helm charts.

- **`charts/`**: Contains custom Helm charts for applications deployed in the cluster. Each subdirectory is a self-contained chart.
  - `pihole/`: Manages the Pi-hole instance.
  - `uptime-kuma/`: Manages the Uptime Kuma monitoring instance.
  - `metallb-config/`: Manages the MetalLB `IPAddressPool` and `L2Advertisement` Custom Resources.
  - `cert-manager/`: Manages the `Issuer` and `ClusterIssuer` for cert-manager to determine where and how it will request certificates.
  - `scrypted/`: Manages Scrypted, a video integration platform for security cameras.
  - `postgres/`: Manages a Cloud-Native Postgres cluster for applications.

---

## Managed Applications

The following applications are managed via Helm charts within this repository and deployed by ArgoCD:

- **Pi-hole**: Primary Network-wide ad-blocking and DNS.
- **AdGuard**: Secondary Network-wide ad-blocking and DNS. (to be switched to Primary in the future after testing DoH)
- **Uptime Kuma**: A user-friendly monitoring tool.
- **MetalLB Configuration**: Defines the IP address pools for `LoadBalancer` services.
- **Scrypted**: A video integration platform to connect my security cameras to HomeKit / HomeAssistant
- **Postgres**: A relational database for upcoming web applications

---

## Manual Installations (Bootstrapping)

Certain core components must be bootstrapped onto the cluster manually before ArgoCD can manage the applications that depend on them. These are typically installed once using Helm.

### MetalLB

Installs the MetalLB controller and speaker components. Note that only the controller is installed here; the IP address configuration (`IPAddressPools`, etc.) is managed by the `metallb-config` chart in this repository.

```bash
helm repo add metallb https://metallb.github.io/metallb
helm install metallb metallb/metallb --namespace metallb-system --create-namespace
```

### Cert-Manager

Installs the controller for issuing and renewing TLS certificates automatically from providers like Let's Encrypt.

```bash
helm install cert-manager oci://quay.io/jetstack/charts/cert-manager --version v1.20.2 --namespace cert-manager --create-namespace --set crds.enabled=true
```

### Cloud Native Postgres

Installs the cloud native postgres operator onto the cluster

```bash
helm repo add cnpg https://cloudnative-pg.github.io/charts
helm upgrade --install cnpg --namespace cnpg-system --create-namespace cnpg/cloudnative-pg
```
