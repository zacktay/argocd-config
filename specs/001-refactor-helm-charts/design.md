# Design Specification: Refactoring Helm Charts

This document defines the concrete values structures, YAML snippets, schemas, and templates to be used during the refactoring process.

---

## 1. Global Values Schema (`home/values.yaml`)

Define the following global config block in the parent `home/values.yaml` file:

```yaml
global:
  domain: teegeedoubleyou.work
  ingressIP: 192.168.1.20
  clusterIssuer: letsencrypt-cluster-issuer
  ingressClassName: traefik
  storageClassName: longhorn
```

---

## 2. Reverse Proxy Config Mapping (`home/charts/reverse-proxy/values.yaml`)

Define the `proxies` map under `home/charts/reverse-proxy/values.yaml`:

```yaml
proxies:
  plex:
    enabled: true
    ip: 192.168.1.5
    port: 32400
    scheme: https
    portName: https
  proxmox:
    enabled: true
    ip: 192.168.1.10
    port: 8006
    scheme: https
    portName: https
  synology:
    enabled: true
    ip: 192.168.1.5
    port: 5001
    scheme: https
    portName: https
  unifi:
    enabled: true
    ip: 192.168.1.1
    port: 443
    scheme: https
    portName: https
  wireguard:
    enabled: true
    ip: 192.168.1.254
    port: 10086
    scheme: http
    portName: http
```

### 2.1 Unified Template Logic for `reverse-proxy`

#### Naming and Labels (`templates/_helpers.tpl`)
Create standard chart helpers:
```yaml
{{/* Common labels */}}
{{- define "reverse-proxy.labels" -}}
helm.sh/chart: {{ printf "%s-%s" .Chart.Name .Chart.Version | replace "+" "_" }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end }}
```

#### Certificate (`templates/certificate.yaml`)
```yaml
{{- range $name, $config := .Values.proxies }}
{{- if $config.enabled }}
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: {{ $name }}-ui-cert
  namespace: {{ $.Values.namespaceOverride | default $.Release.Namespace }}
spec:
  secretName: {{ $name }}-ui-tls
  issuerRef:
    name: {{ $.Values.global.clusterIssuer | default "letsencrypt-cluster-issuer" }}
    kind: ClusterIssuer
  dnsNames:
  - {{ $name }}.{{ required "global.domain is required" $.Values.global.domain }}
---
{{- end }}
{{- end }}
```

#### Service & Endpoints (`templates/service.yaml` & `templates/endpoints.yaml`)
Alternatively, combine them or keep separate files but iterate:
```yaml
{{- range $name, $config := .Values.proxies }}
{{- if $config.enabled }}
apiVersion: v1
kind: Service
metadata:
  name: {{ $name }}-ui-backend
  namespace: {{ $.Values.namespaceOverride | default $.Release.Namespace }}
spec:
  clusterIP: None
  ports:
  - name: {{ $config.portName }}
    port: {{ $config.port }}
    targetPort: {{ $config.port }}
---
apiVersion: v1
kind: Endpoints
metadata:
  name: {{ $name }}-ui-backend
  namespace: {{ $.Values.namespaceOverride | default $.Release.Namespace }}
subsets:
- addresses:
  - ip: {{ $config.ip }}
  ports:
  - name: {{ $config.portName }}
    port: {{ $config.port }}
---
{{- end }}
{{- end }}
```

#### IngressRoute (`templates/ingressroute.yaml`)
```yaml
{{- range $name, $config := .Values.proxies }}
{{- if $config.enabled }}
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: {{ $name }}-ui-ingressroute
  namespace: {{ $.Values.namespaceOverride | default $.Release.Namespace }}
spec:
  entryPoints:
    - websecure
  routes:
  - match: Host(`{{ $name }}.{{ required "global.domain is required" $.Values.global.domain }}`)
    kind: Rule
    services:
    - name: {{ $name }}-ui-backend
      port: {{ $config.port }}
      scheme: {{ $config.scheme }}
      serversTransport: {{ $name }}-transport
  tls:
    secretName: {{ $name }}-ui-tls
---
apiVersion: traefik.io/v1alpha1
kind: ServersTransport
metadata:
  name: {{ $name }}-transport
  namespace: {{ $.Values.namespaceOverride | default $.Release.Namespace }}
spec:
  insecureSkipVerify: true
  disableHTTP2: true
---
{{- end }}
{{- end }}
```

---

## 3. Homepage Configuration (`home/charts/homepage/values.yaml`)

Define the bookmark, widget, and service blocks using string representations (so they can be evaluated via `tpl` for domain resolution) or directly:

```yaml
config:
  bookmarks:
    - Developer:
        - Github:
            - icon: github.png
              href: https://github.com/zacktay/
              description: Github
        - Coursera:
            - icon: coursera.png
              href: https://www.coursera.org/programs/b10b-jan26-12m-3ry38/
              description: Coursera
        - Gemini:
            - icon: google.png
              href: https://gemini.google.com/
              description: Gemini
    - Social:
        - Gmail:
            - icon: gmail.png
              href: https://mail.google.com/
              description: Gmail
        - Reddit:
            - icon: reddit.png
              href: https://reddit.com/
              description: Reddit
        - AWS re:Post:
            - icon: aws.png
              href: https://repost.aws/
              description: AWS re:Post
    - Entertainment:
        - Plex:
            - icon: plex.png
              href: http://plex.{{ .Values.global.domain }}
              description: Plex Media Server
        - YouTube:
            - icon: youtube.png
              href: https://youtube.com/
              description: YouTube
        - Netflix:
            - icon: netflix.png
              href: https://netflix.com/
              description: Netflix

  services:
    - Infrastructure & Management:
        - Proxmox:
            icon: proxmox.png
            href: https://proxmox.{{ .Values.global.domain }}
            description: Hypervisor
        - Longhorn:
            icon: longhorn.png
            href: https://longhorn.{{ .Values.global.domain }}
            description: Block Storage
        - Headlamp:
            icon: headlamp.png
            href: https://headlamp.{{ .Values.global.domain }}/c/main/
            description: Kubernetes UI
        - Unifi:
            icon: unifi.png
            href: https://unifi.{{ .Values.global.domain }}/
            description: Unifi Management Console
    - Monitoring & Routing:
        - Traefik:
            icon: traefik.png
            href: https://traefik.{{ .Values.global.domain }}
            description: Reverse Proxy
        - Uptime Kuma:
            icon: uptime-kuma.png
            href: https://uptime-kuma.{{ .Values.global.domain }}
            description: Uptime Monitoring
    - DNS & Security:
        - Pihole-1:
            icon: pi-hole.png
            href: https://pihole-1.{{ .Values.global.domain }}/admin/
            description: Primary DNS
        - Pihole-2:
            icon: pi-hole.png
            href: https://pihole-2.{{ .Values.global.domain }}/admin/
            description: Secondary DNS
        - Vault:
            icon: hashicorp-vault.png
            href: https://vault.{{ .Values.global.domain }}
            description: Secrets Management
        - Wireguard:
            icon: wireguard.png
            href: https://wireguard.{{ .Values.global.domain }}
            description: VPN
    - Smart Home:
        - Home Assistant:
            icon: home-assistant.png
            href: https://ha.{{ .Values.global.domain }}
            description: Home Automation
        - Scrypted:
            icon: scrypted.png
            href: https://scrypted.{{ .Values.global.domain }}/endpoint/@scrypted/core/public/#/device
            description: NVR
    - Wiki & Apps:
        - Bookstack:
            icon: bookstack.png
            href: https://bookstack.{{ .Values.global.domain }}
            description: Documentation
        - Synology:
            icon: synology.png
            href: https://synology.{{ .Values.global.domain }}/
            description: File Sharing

  widgets:
    - search:
        provider: google
        target: _blank
        showSearchIcon: true
        focus: true
    - datetime:
        text_size: xl
        format:
          timeStyle: short
          dateStyle: full
    - kubernetes:
        cluster:
          show: true
          cpu: true
          memory: true
          showLabel: true
          label: "cluster"
        nodes:
          show: true
          cpu: true
          memory: true
          showLabel: true
```

### 3.1 Homepage ConfigMap Template (`templates/configmap.yaml`)

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: homepage
  namespace: {{ .Values.namespaceOverride | default .Release.Namespace }}
  labels:
    app.kubernetes.io/name: homepage
data:
  kubernetes.yaml: |
    mode: cluster
  settings.yaml: ""
  ingress: "true"
  traefik: "true"
  custom.css: ""
  custom.js: ""
  bookmarks.yaml: |
    {{- tpl (toYaml .Values.config.bookmarks) . | nindent 4 }}
  services.yaml: |
    {{- tpl (toYaml .Values.config.services) . | nindent 4 }}
  widgets.yaml: |
    {{- toYaml .Values.config.widgets | nindent 4 }}
```

### 3.2 Homepage Deployment Template (`templates/deployment.yaml`)

- Rename the file `templates/deploment.yaml` -> `templates/deployment.yaml`.
- Replace the environment variable `HOMEPAGE_ALLOWED_HOSTS` template section:
```yaml
            - name: HOMEPAGE_ALLOWED_HOSTS
              value: "{{ printf \"$(MY_POD_IP):3000,localhost:3000,%s:80,%s.%s\" .Values.global.ingressIP .Values.ingress.hostPrefix (required \"global.domain is required\" .Values.global.domain) }}"
```

---

## 4. Generic Subchart Ingress Pattern

For all standard application charts (`bookstack`, `headlamp`, `scrypted`, `traefik`, `uptime-kuma`, `vault`):

In their `values.yaml`:
```yaml
global:
  domain: ""
namespaceOverride: ""
ingress:
  enabled: true
  hostPrefix: "app-name"
  # Optional: host override
  host: ""
```

In their `ingress.yaml` template:
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: {{ include "chart-name.fullname" . }}
  namespace: {{ .Values.namespaceOverride | default .Release.Namespace }}
  # ...
spec:
  # ...
  rules:
    - host: {{ .Values.ingress.host | default (printf "%s.%s" .Values.ingress.hostPrefix (required "global.domain is required" .Values.global.domain)) }}
```

---

## 5. Vault Operator CRD Namespace Exception

As per the user clarification, the Vault operator CRD templates:
- `home/charts/vault/templates/vaultauth.yaml`
- `home/charts/vault/templates/vaultconnection.yaml`
- `home/charts/vault/templates/vaultstaticsecret.yaml`

MUST keep their hardcoded namespaces (`home` and `cert-manager`) because they represent fixed cross-namespace system integrations that bind Vault authentication and connections between the primary applications and the TLS certificate manager. Generalizing these would break this dependency.
