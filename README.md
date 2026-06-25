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

The repository is organized as an umbrella Helm chart (`home`) to manage all application configurations.

- **`home/`**: The core directory containing the umbrella Helm chart.
  - **`Chart.yaml`**: Defines metadata and dependencies (both local sub-charts and upstream external charts).
  - **`values.yaml`**: Contains global and chart-specific configurations for the entire deployment stack.
  - **`charts/`**: Custom local sub-charts and vendored upstream charts.
    - `bookstack/`: Manages the BookStack instance.
    - `cert-manager/`: Manages cluster issuers (`letsencrypt-cluster-issuer`, custom home-ca, selfsigned) and certificates.
    - `homepage/`: Manages the custom dashboard (Homepage).
    - `longhorn/`: Configures access to the Longhorn storage dashboard.
    - `metallb/`: Defines MetalLB `IPAddressPool` and `L2Advertisement` custom resources.
    - `reverse-proxy/`: Manages Ingress resources for external/non-cluster services (e.g., Plex, Proxmox, Synology, UniFi, WireGuard).
    - `scrypted/`: Manages the Scrypted video integration platform.
    - `traefik/`: Configures Traefik entrypoints and routing configurations.
    - `uptime-kuma/`: Manages the Uptime Kuma monitoring tool.
    - `vault/`: Configures Vault Secrets Operator integrations (`VaultConnection`, `VaultAuth`, `VaultStaticSecret`) and Ingress.

---

## Managed Applications

The following applications are managed via Helm charts in this repository and deployed to the home cluster:

- **Pi-hole (Dual Instance)**: Network-wide ad-blocking and DNS via primary (`pihole-1`) and secondary (`pihole-2`) instances.
- **BookStack**: Wiki and documentation platform.
- **Homepage**: Customizable application portal and dashboard.
- **Headlamp**: Kubernetes Web UI for cluster management.
- **Uptime Kuma**: Monitoring and status check page.
- **MetalLB**: Defines IP address pools for `LoadBalancer` services.
- **Cert-Manager**: Issues TLS certificates using Let's Encrypt or custom CA.
- **Scrypted**: Connects smart security cameras to HomeKit/Home Assistant.
- **Traefik & Reverse Proxy**: Ingress controllers and secure routing for cluster apps and external servers (Plex, Proxmox, Synology NAS, UniFi, WireGuard).
- **Vault Secrets Operator**: Automatically synchronizes secrets from HashiCorp Vault.

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

Install Vault Secret Operator
```bash
helm install vault-secrets-operator hashicorp/vault-secrets-operator --namespace vault-secrets-operator-system --create-namespace --version 1.4.0
```

Enable Kubernetes Auth
```bash
vault write auth/kubernetes/config \
  token_reviewer_jwt="$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)" \
  kubernetes_host="https://$KUBERNETES_PORT_443_TCP_ADDR:443" \
  kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt
```

Create Policy for Vault Operator
```bash
kubectl exec -it vault-0 -n vault -- /bin/sh
vault policy write vso-reader - <<EOF
path "secret/data/*" {
  capabilities: [permissions_to_grant_here]
}
EOF
```
