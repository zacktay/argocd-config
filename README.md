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
  -  `cert-manager/`: Manages the `Issuer` and `ClusterIssuer` for cert-manager to determine where and how it will request certificates

---

## Managed Applications

The following applications are managed via Helm charts within this repository and deployed by ArgoCD:

- **Pi-hole**: Network-wide ad-blocking and DNS.
- **Uptime Kuma**: A user-friendly monitoring tool.
- **MetalLB Configuration**: Defines the IP address pools for `LoadBalancer` services.

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

### GitHub Actions Runner Controller (ARC)

Installs the controller for managing self-hosted GitHub Actions runners within the Kubernetes cluster.

```bash
helm install arc --namespace arc-systems --create-namespace oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set-controller
```
