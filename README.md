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
  - `adguard/`: Manages the AdGuard instance.
  - `uptime-kuma/`: Manages the Uptime Kuma monitoring instance.
  - `metallb-config/`: Manages the MetalLB `IPAddressPool` and `L2Advertisement` Custom Resources.
  - `cert-manager/`: Manages the `Issuer` and `ClusterIssuer` for cert-manager to determine where and how it will request certificates.
  - `scrypted/`: Manages Scrypted, a video integration platform for security cameras.
  - `vault/`: Manages the Ingress for Hashicorp Vault

---

## Managed Applications

The following applications are managed via Helm charts within this repository and deployed by ArgoCD:

- **Pi-hole**: Primary Network-wide ad-blocking and DNS.
- **AdGuard**: Secondary Network-wide ad-blocking and DNS. (to be switched to Primary in the future after testing DoH)
- **Uptime Kuma**: A user-friendly monitoring tool.
- **MetalLB Configuration**: Defines the IP address pools for `LoadBalancer` services.
- **Cert-Manager**: Defines the `Issuer` and `ClusterIssuer` for the certificates to be used within the cluster
- **Scrypted**: A video integration platform to connect my security cameras to HomeKit / HomeAssistant
- **Vault**: Defines the `Ingress` for Hashicorp Vault

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
helm upgrade trust-manager oci://quay.io/jetstack/charts/trust-manager --install --namespace cert-manager --wait
```

### Hashicorp vault

Installs the vault

```bash
kubectl create namespace vault
helm repo add hashicorp https://helm.releases.hashicorp.com
helm upgrade --install vault hashicorp/vault --namespace vault -f vault-values.yaml
```
Initialize the Vault

```bash
kubectl exec -ti vault-0 -n vault -- vault operator init
```
Unseal the Vault using the unseal keys obtained during initialization
```bash
kubectl exec -ti vault-0 -n vault -- vault operator unseal
```

Once Unsealed, join the Vault to the cluster (do this for the other nodes)

```bash
kubectl -n vault exec -it vault-1 -- vault operator raft join https://vault-0.vault-internal.vault.svc.cluster.local:8200
kubectl -n vault exec -it vault-1 -- vault operator unseal
```

Auto-unseal using AWS KMS - https://developer.hashicorp.com/vault/docs/configuration/seal/awskms#awskms-example 
```yaml
seal "awskms" {
  region     = "ap-southeast-1"
  access_key = "YOUR_ACCESS_KEY"
  secret_key = "YOUR_SECRET_KEY"
  kms_key_id = "YOUR_KMS_KEY_ID"
}
```
