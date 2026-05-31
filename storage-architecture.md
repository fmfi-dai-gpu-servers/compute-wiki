# Storage Architecture

Two storage backends serve the cluster: **SeaweedFS** for distributed/object storage and **local-path** for node-local persistent volumes.

## Storage Overview

```mermaid
architecture-beta
    group consumers(cloud)[Storage Consumers]
    service jh(server)[JupyterHub<br/>User Homes] in consumers
    service harbor(database)[Harbor<br/>Image Storage] in consumers
    service mlflow(server)[MLflow<br/>Artifacts] in consumers
    service dd(server)[Datasets<br/>Dashboard] in consumers

    group seaweedfs(cloud)[SeaweedFS — sklad node]
    service s3(server)[S3 Gateway<br/>:8333] in seaweedfs
    service filer(server)[Filer<br/>:8888] in seaweedfs
    service master(server)[Master<br/>:9333] in seaweedfs
    service volume(disk)[Volume Server<br/>Host Path] in seaweedfs

    group csi(cloud)[CSI Driver]
    service csi_ctrl(server)[CSI Controller] in csi
    service csi_node(server)[CSI Node Plugin<br/>DaemonSet] in csi

    group local(cloud)[Local Path Provisioner]
    service lp(disk)[local-path<br/>controller-1 disk] in local

    jh:R --> L:csi_node
    harbor:B --> T:s3
    mlflow:B --> T:lp
    dd:B --> T:s3
    s3:L --> R:filer
    filer:B --> T:master
    filer:L --> R:volume
    csi_ctrl:B --> T:filer
    csi_node:B --> T:filer
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
        S3_Ext["storage.c.dai.fmph.uniba.sk<br/>S3 API"]
        Dash["datasets.c.dai.fmph.uniba.sk<br/>Dashboard"]
    end

    subgraph cluster["Kubernetes — seaweedfs namespace"]
        M["Master (StatefulSet)<br/>Cluster metadata, volume assignment<br/>:9333"]
        V["Volume (StatefulSet)<br/>Actual file data storage<br/>:8080"]
        F["Filer (StatefulSet)<br/>Directory abstraction + S3<br/>:8888 / :8333"]
        A["Admin UI<br/>:23646"]
        DD["Datasets Dashboard<br/>FastAPI web app"]
        CSIC["CSI Controller"]
        CSIN["CSI Node Plugin (DaemonSet)<br/>on all 4 nodes"]
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
| `harbor-registry` | Harbor container image storage | Internal (Harbor → SeaweedFS) |
| `datasets` | Shared dataset storage | Per-user via dashboard |
| `datasets-<username>` | Per-user dataset bucket (20GB quota) | OIDC STS token exchange |
