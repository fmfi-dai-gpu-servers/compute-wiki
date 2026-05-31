# Storage Architecture

Two storage backends serve the cluster: **SeaweedFS** for distributed/object storage and **local-path** for node-local persistent volumes.

## Storage Overview

```mermaid
flowchart TB
    subgraph consumers["Storage Consumers"]
        JH["JupyterHub\nUser Homes"]
        HV["Harbor\nImage Storage"]
        MLF["MLflow\nArtifacts"]
        DD["Datasets\nDashboard"]
    end

    subgraph seaweedfs["SeaweedFS - sklad node"]
        S3["S3 Gateway :8333"]
        Filer["Filer :8888"]
        Master["Master :9333"]
        Volume["Volume Server\nHost Path"]
    end

    subgraph csi["CSI Driver"]
        CSI_C["CSI Controller"]
        CSI_N["CSI Node Plugin\nDaemonSet"]
    end

    subgraph local["Local Path Provisioner"]
        LP["local-path\ncontroller-1 disk"]
    end

    JH --> CSI_N
    HV --> S3
    MLF --> LP
    DD --> S3
    S3 --> Filer
    Filer --> Master
    Filer --> Volume
    CSI_C --> Filer
    CSI_N --> Filer
```

## Storage Classes

| Class | Provisioner | Access | Binding | Expand | Use Case |
|-------|-------------|--------|---------|--------|----------|
| `local-path` (default) | rancher.io/local-path | RWO | WaitForFirstConsumer | No | DBs, internal state |
| `seaweedfs-storage` | seaweedfs-csi-driver | RWX | Immediate | Yes | User homes, shared data |

## SeaweedFS Architecture

```mermaid
flowchart TB
    subgraph ingress["External Access"]
        S3_Ext["storage.c.dai.fmph.uniba.sk\nS3 API"]
        Dash["datasets.c.dai.fmph.uniba.sk\nDashboard"]
    end

    subgraph cluster["Kubernetes - seaweedfs namespace"]
        M["Master (StatefulSet)\nCluster metadata, volume assignment\n:9333"]
        V["Volume (StatefulSet)\nActual file data storage\n:8080"]
        F["Filer (StatefulSet)\nDirectory abstraction + S3\n:8888 / :8333"]
        A["Admin UI\n:23646"]
        DD["Datasets Dashboard\nFastAPI web app"]
        CSIC["CSI Controller"]
        CSIN["CSI Node Plugin (DaemonSet)\non all 4 nodes"]
    end

    subgraph storage["Physical Storage on sklad"]
        MV["/mnt/seaweed-data/master/"]
        VV["/mnt/seaweed-data/volume/"]
        FV["/mnt/seaweed-data/filer/"]
    end

    S3_Ext --> F
    Dash --> DD
    DD --> F
    M --> MV
    V --> VV
    F --> FV
    F --> M
    F --> V
    CSIC --> F
    CSIN --> F
```

## Persistent Volumes

| Claim | Namespace | Size | Class | Mode | Purpose |
|-------|-----------|------|-------|------|---------|
| `hub-db-dir` | jupyterhub | 1Gi | local-path | RWO | JupyterHub hub DB |
| `claim-fiil123` | jupyterhub | 10Gi | seaweedfs-storage | RWX | User home directory |
| `claim-fiil123--gpu` | jupyterhub | 10Gi | seaweedfs-storage | RWX | GPU notebook home |
| `claim-fiil123--gpu02` | jupyterhub | 10Gi | seaweedfs-storage | RWX | GPU notebook home 2 |
| `postgres-pvc` | keycloak | 10Gi | local-path | RWO | Keycloak PostgreSQL |
| `postgres-pvc` | mlflow | 10Gi | local-path | RWO | MLflow PostgreSQL |
| `mlflow-artifacts-pvc` | mlflow | 10Gi | local-path | RWO | MLflow artifact store |
| `database-data-harbor-database-0` | harbor | 2Gi | local-path | RWO | Harbor PostgreSQL |
| `data-harbor-redis-0` | harbor | 1Gi | local-path | RWO | Harbor Redis |
| `data-harbor-trivy-0` | harbor | 5Gi | local-path | RWO | Trivy vulnerability DB |
| `harbor-jobservice` | harbor | 1Gi | local-path | RWO | Harbor job logs |
| `traefik` | traefik-system | 128Mi | local-path | RWO | Let's Encrypt ACME certs |

## S3 Buckets

| Bucket | Purpose | Access |
|--------|---------|--------|
| `harbor-registry` | Harbor container image storage | Internal (Harbor to SeaweedFS) |
| `datasets` | Shared dataset storage | Per-user via dashboard |
| `datasets-<username>` | Per-user dataset bucket (20GB quota) | OIDC STS token exchange |
