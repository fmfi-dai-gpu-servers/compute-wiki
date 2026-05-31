# Service Map

Complete inventory of all deployed services in the K3s cluster.

## Service Overview

```mermaid
architecture-beta
    group infra(cloud)[Infrastructure]
    service traefik(server)[Traefik<br/>Ingress/TLS] in infra
    service kc(server)[Keycloak<br/>Identity] in infra
    service harbor(database)[Harbor<br/>Registry] in infra
    service sw(disk)[SeaweedFS<br/>Storage] in infra
    service volcano(server)[Volcano<br/>GPU Scheduler] in infra

    group ml(cloud)[ML/AI Platform]
    service jhub(server)[JupyterHub<br/>Notebooks] in ml
    service mlflow(server)[MLflow<br/>Experiments] in ml
    service ray(server)[Kube-Ray<br/>Distributed Compute] in ml

    group datasets(cloud)[Data]
    service dash(server)[Datasets<br/>Dashboard] in datasets

    traefik:R --> L:jhub
    traefik:R --> L:mlflow
    traefik:R --> L:ray
    traefik:B --> T:kc
    traefik:B --> T:harbor
    traefik:B --> T:sw
    kc:L --> R:jhub
    kc:L --> R:mlflow
    kc:L --> R:ray
    kc:L --> R:dash
    sw:R --> L:harbor
    sw:R --> L:jhub
    volcano:T --> B:jhub
    volcano:T --> B:ray
    harbor:T --> B:ray
    harbor:T --> B:mlflow
```

## Service Details

### Traefik — Ingress Controller

| Field | Value |
|-------|-------|
| Namespace | `traefik-system` |
| Domain | `traefik.c.dai.fmph.uniba.sk` (dashboard) |
| Chart | `traefik/traefik` v38.0.2 |
| Image | `traefik:v3.6.5` |
| Node | k3s-controller-1 |
| TLS | Let's Encrypt (HTTP challenge) |
| Entrypoints | `web` (80→HTTPS redirect), `websecure` (443), `rayclient` (10001/TCP), `metrics` (9100) |
| Dashboard Auth | BasicAuth |

### Keycloak — Identity Management

| Field | Value |
|-------|-------|
| Namespace | `keycloak` |
| Domain | `auth.c.dai.fmph.uniba.sk` |
| Chart | `codecentric/keycloakx` v7.1.9 |
| Image | `quay.io/keycloak/keycloak:26.5.5` |
| Node | pocitadlo |
| Realm | `compute` |
| Database | PostgreSQL 15 (local-path PVC, 10Gi) |
| External IdP | GitHub (fmfi-dai-gpu-servers org) |
| Clients | jupyterhub, mlflow, ray-proxy, ray-cli, seaweedfs-s3, datasets-dashboard |

### Harbor — Container Registry

| Field | Value |
|-------|-------|
| Namespace | `harbor` |
| Domain | `harbor.c.dai.fmph.uniba.sk` |
| Chart | `goharbor/harbor` v1.18.3 (app v2.14.3) |
| Node | pocitadlo |
| Image Storage | SeaweedFS S3 (bucket: `harbor-registry`) |
| Scanner | Trivy (built-in) |
| Components | core, portal, registry, jobservice, trivy, database, redis (1 replica each) |

### SeaweedFS — Distributed Storage

| Field | Value |
|-------|-------|
| Namespace | `seaweedfs` |
| S3 Domain | `storage.c.dai.fmph.uniba.sk` |
| Dashboard | `datasets.c.dai.fmph.uniba.sk` |
| Chart | `seaweedfs/seaweedfs` v4.21.0 |
| CSI Chart | `seaweedfs-csi-driver` v0.2.15 |
| Node | sklad (master/volume/filer), all nodes (CSI) |
| S3 Port | 8333 |
| Filer Port | 8888 |
| Master Port | 9333 |
| StorageClass | `seaweedfs-storage` (RWX, CSI) |
| Host Paths | `/mnt/seaweed-data/{master,volume,filer}/` |

### JupyterHub — Multi-user Notebooks

| Field | Value |
|-------|-------|
| Namespace | `jupyterhub` |
| Domain | `jhub.c.dai.fmph.uniba.sk` |
| Chart | `jupyterhub/jupyterhub` v4.3.3 (app v5.4.4) |
| Node | pocitadlo (hub/proxy/scheduler), all (image puller) |
| Auth | Keycloak OIDC (client: `jupyterhub`) |
| User Storage | SeaweedFS CSI (10Gi RWX per user) |
| Named Servers | enabled (max 2 per user) |
| Idle Culling | 30min idle, 12h max age |
| GPU Profiles | Shared CUDA (1 vGPU, 5 cores, 2.4GB via Volcano) |

### MLflow — Experiment Tracking

| Field | Value |
|-------|-------|
| Namespace | `mlflow` |
| Domain | `mlflow.c.dai.fmph.uniba.sk` |
| Chart | `community-charts/mlflow` v1.8.1 (app v3.7.0) |
| Image | Custom: `harbor.c.dai.fmph.uniba.sk/services/mlflow-oidc:latest` |
| Node | pocitadlo |
| Auth | Keycloak OIDC (client: `mlflow`, mlflow-oidc-auth plugin) |
| Database | PostgreSQL (local-path PVC, 10Gi) |
| Artifacts | local-path PVC (10Gi, planned migration to SeaweedFS S3) |
| CI/CD | GitHub Actions → Harbor |

### Kube-Ray — Distributed Compute

| Field | Value |
|-------|-------|
| Namespace | `ray-system` |
| Dashboard | `ray.c.dai.fmph.uniba.sk` |
| Client (TCP) | `ray-client.c.dai.fmph.uniba.sk:10001` |
| Operator Chart | `kuberay/kuberay-operator` v1.5.1 |
| Cluster Chart | `kuberay/ray-cluster` v1.6.1 |
| Ray Version | 2.46.0 |
| Node | pocitadlo (head, operator), k3s-node-1 (workers) |
| Auth (Browser) | oauth2-proxy → Keycloak (client: `ray-proxy`) |
| Auth (CLI) | Device Auth Grant → Keycloak (client: `ray-cli`) |
| GPU | 0-4 workers, 1 vGPU each (Volcano) |
| Images | From Harbor (mirrored) |

### Volcano — Batch/vGPU Scheduler

| Field | Value |
|-------|-------|
| Namespace | `volcano-system` |
| Chart | `volcano` v1.13.1 |
| Node | pocitadlo (scheduler/admission/controller), k3s-node-1 (device plugin) |
| Device Plugin | `projecthami/volcano-vgpu-device-plugin:v1.11.0` |
| Runtime Class | `nvidia` |
| Validating Webhooks | jobs, cronjobs, podgroups, queues, hypernodes |
| CRDs | commands, cronjobs, jobs, jobflows, jobtemplates, hypernodes, podgroups, queues |

## Helm Releases

| Release | Namespace | Chart | Revision |
|---------|-----------|-------|----------|
| traefik | traefik-system | traefik-38.0.2 | 2 |
| keycloak | keycloak | keycloakx-7.1.9 | 5 |
| harbor | harbor | harbor-1.18.3 | 4 |
| jupyterhub | jupyterhub | jupyterhub-4.3.3 | 5 |
| mlflow | mlflow | mlflow-1.8.1 | 4 |
| kuberay-operator | ray-system | kuberay-operator-1.5.1 | 2 |
| ray | ray-system | ray-cluster-1.6.1 | 20 |
| ray-auth-proxy | ray-system | oauth2-proxy-10.6.0 | 5 |
| seaweedfs | seaweedfs | seaweedfs-4.21.0 | 17 |
| seaweedfs-csi-driver | seaweedfs | seaweedfs-csi-driver-0.2.15 | 1 |
| volcano | volcano-system | volcano-1.13.1 | 3 |

## Namespaces

| Namespace | Purpose | Pods |
|-----------|---------|------|
| `kube-system` | K3s system components | 3 (coredns, metrics-server, local-path-provisioner) |
| `traefik-system` | Ingress controller | 1 |
| `keycloak` | Identity management | 2 |
| `harbor` | Container registry | 8 |
| `seaweedfs` | Distributed storage + CSI | 10 |
| `volcano-system` | Batch scheduler + GPU | 4 |
| `jupyterhub` | Multi-user notebooks | 5 + user pods |
| `mlflow` | Experiment tracking | 2 |
| `ray-system` | Distributed compute | 3 + workers |
