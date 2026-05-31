# Architecture Overview

The platform is a GPU-accelerated ML/AI environment running on a 4-node K3s Kubernetes cluster at FMFI DAI. All services share a single domain (`*.c.dai.fmph.uniba.sk`) with TLS via Let's Encrypt.

## High-Level Architecture

```mermaid
flowchart TB
    subgraph users["Users"]
        Developer["Developer"]
    end

    subgraph ingress["Ingress / TLS"]
        Traefik["Traefik v3"]
    end

    subgraph identity["Identity"]
        Keycloak["Keycloak"]
    end

    subgraph platform["Platform Services"]
        Harbor["Harbor Registry"]
        SeaweedFS["SeaweedFS S3"]
    end

    subgraph ml["ML Platform"]
        JupyterHub["JupyterHub"]
        MLflow["MLflow"]
        Ray["Kube-Ray"]
    end

    subgraph gpu["GPU Layer"]
        Volcano["Volcano vGPU"]
    end

    Developer --> Traefik
    Traefik --> JupyterHub
    Traefik --> MLflow
    Traefik --> Ray
    Traefik --> Harbor
    Traefik --> Keycloak

    Keycloak --> JupyterHub
    Keycloak --> MLflow
    Keycloak --> Ray
    Keycloak --> SeaweedFS

    SeaweedFS --> Harbor
    SeaweedFS --> JupyterHub

    Volcano --> Ray
    Volcano --> JupyterHub

    Harbor --> Ray
    Harbor --> MLflow
```

## Service Dependency Graph

```mermaid
flowchart LR
    subgraph infra["Infrastructure Layer"]
        T["Traefik - Ingress/TLS"]
        K["Keycloak - OIDC Auth"]
        SW["SeaweedFS - Storage S3/CSI"]
        H["Harbor - Container Registry"]
    end

    subgraph scheduling["Scheduling Layer"]
        V["Volcano - Batch/vGPU Scheduler"]
    end

    subgraph ml["ML/AI Services"]
        JH["JupyterHub - Notebooks"]
        ML["MLflow - Experiment Tracking"]
        RC["Ray Cluster - Distributed Compute"]
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
