# Architecture Overview

The platform is a GPU-accelerated ML/AI environment running on a 4-node K3s Kubernetes cluster at FMFI DAI. All services share a single domain (`*.c.dai.fmph.uniba.sk`) with TLS via Let's Encrypt.

## High-Level Architecture

```mermaid
architecture-beta
    group users(cloud)[Users]
    service developer(user)[Developer] in users

    group ingress(cloud)[Ingress / TLS]
    service traefik(server)[Traefik v3] in ingress

    group identity(cloud)[Identity]
    service keycloak(server)[Keycloak] in identity

    group platform(cloud)[Platform Services]
    service harbor(database)[Harbor Registry] in platform
    service seaweedfs(disk)[SeaweedFS S3] in platform

    group ml(cloud)[ML Platform]
    service jupyterhub(server)[JupyterHub] in ml
    service mlflow(server)[MLflow] in ml
    service ray(server)[Kube-Ray] in ml

    group gpu(cloud)[GPU Layer]
    service volcano(server)[Volcano vGPU] in gpu

    developer:R --> L:traefik
    traefik:R --> L:jupyterhub
    traefik:R --> L:mlflow
    traefik:R --> L:ray
    traefik:B --> T:harbor
    traefik:B --> T:keycloak

    keycloak:R --> L:jupyterhub
    keycloak:R --> L:mlflow
    keycloak:R --> L:ray
    keycloak:B --> T:seaweedfs

    seaweedfs:R --> L:harbor
    seaweedfs:B --> L:jupyterhub

    volcano:T --> B:ray
    volcano:T --> B:jupyterhub

    harbor:B --> T:ray
    harbor:B --> T:mlflow
```

## Service Dependency Graph

```mermaid
flowchart LR
    subgraph infra["Infrastructure Layer"]
        T[Traefik<br/>Ingress/TLS]
        K[Keycloak<br/>OIDC Auth]
        SW[SeaweedFS<br/>Storage S3/CSI]
        H[Harbor<br/>Container Registry]
    end

    subgraph scheduling["Scheduling Layer"]
        V[Volcano<br/>Batch/vGPU Scheduler]
    end

    subgraph ml["ML/AI Services"]
        JH[JupyterHub<br/>Notebooks]
        ML[MLflow<br/>Experiment Tracking]
        RC[Ray Cluster<br/>Distributed Compute]
    end

    T --> JH & ML & RC & H & K
    K -.->|OIDC| JH & ML & RC & SW
    SW -.->|S3 images| H
    SW -.->|CSI RWX| JH
    V -.->|vGPU scheduling| JH & RC
    H -.->|container images| RC & ML
```

## Key Characteristics

- **Single cluster**: 4 nodes, 1 control plane, 3 workers with specialized roles
- **Central auth**: Keycloak with `compute` realm, 6 OIDC clients, GitHub as external IdP
- **GPU sharing**: 1 NVIDIA TITAN Xp split into up to 25 virtual GPUs via Volcano/Hami
- **Distributed storage**: SeaweedFS provides both S3 API and CSI (ReadWriteMany) mounts
- **Private registry**: Harbor stores all custom and mirrored images, backed by SeaweedFS S3
