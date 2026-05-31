# Cluster Nodes

The K3s cluster consists of 4 nodes, each with a dedicated role. The cluster uses SQLite datastore (single server mode) with custom CIDRs.

## Node Architecture

```mermaid
flowchart LR
    subgraph cluster["K3s Cluster - v1.34"]
        CTRL["k3s-controller-1\nControl Plane\n10.89.100.8"]
        GPU["k3s-node-1\nGPU Node\n10.89.100.32"]
        SVC["pocitadlo\nService Node\n158.195.89.102"]
        STOR["sklad\nStorage Node\n10.89.100.10"]
    end

    CTRL --- GPU
    CTRL --- SVC
    CTRL --- STOR
```

## Node Details

| Node | Role | IP | OS | K3s Version | CPU | Memory |
|------|------|----|----|-------------|-----|--------|
| k3s-controller-1 | `control-plane` | 10.89.100.8 | Ubuntu 24.04.3 | v1.34.3 | 45m used | 2.2Gi / 27% |
| k3s-node-1 | `gpu` | 10.89.100.32 | Ubuntu 24.04.3 | v1.34.3 | 56m used | 1.2Gi / 5% |
| pocitadlo | `service` | 158.195.89.102 | Ubuntu 24.04.4 | v1.34.6 | 483m used | 7.3Gi / 2% |
| sklad | `storage` | 10.89.100.10 | Ubuntu 24.04.4 | v1.34.6 | 55m used | 1.4Gi / 18% |

## What Runs Where

```mermaid
flowchart TB
    subgraph ctrl["k3s-controller-1 (control-plane)"]
        direction LR
        T["Traefik"]
        CD["CoreDNS"]
        MS["Metrics Server"]
        LPP["Local-Path Provisioner"]
        CSI_N["SeaweedFS CSI Node"]
        CIP["JupyterHub Image Puller"]
    end

    subgraph gpu_node["k3s-node-1 (gpu)"]
        direction LR
        VDP["Volcano Device Plugin"]
        CSI_N2["SeaweedFS CSI Node"]
        CIP2["JupyterHub Image Puller"]
        JW["Jupyter User Notebooks"]
        RW["Ray Workers"]
        OAP["OAuth2 Proxy"]
    end

    subgraph svc_node["pocitadlo (service)"]
        direction LR
        KC["Keycloak"]
        PG_KC["PostgreSQL x2"]
        ML["MLflow"]
        HUB["JupyterHub Hub + Proxy"]
        KRO["KubeRay Operator"]
        RH["Ray Head"]
        HV["Harbor all components"]
        VS["Volcano Scheduler"]
        VA["Volcano Admission"]
        VC["Volcano Controller"]
        DD["Datasets Dashboard"]
        CSI_C["SeaweedFS CSI Controller"]
    end

    subgraph stor_node["sklad (storage)"]
        direction LR
        SW_M["SeaweedFS Master"]
        SW_V["SeaweedFS Volume"]
        SW_F["SeaweedFS Filer + S3"]
        SW_A["SeaweedFS Admin"]
        CSI_M["SeaweedFS CSI Mount"]
    end
```

## Node Labels

| Label | Applied To | Used By |
|-------|-----------|---------|
| `node-role.kubernetes.io/control-plane` | k3s-controller-1 | Traefik placement |
| `node-role.kubernetes.io/gpu` | k3s-node-1 | GPU workloads, Volcano device plugin |
| `node-role.kubernetes.io/service` | pocitadlo | Harbor, platform services |
| `node-role.kubernetes.io/storage` | sklad | SeaweedFS components |
| `gpu-node: "true"` | k3s-node-1 | Volcano device plugin DaemonSet |
| `sw-backend: "true"` | sklad | SeaweedFS master/filer |
| `sw-volume: "true"` | sklad | SeaweedFS volume server |

## Ansible Inventory

The cluster is provisioned via Ansible (`ansible-k3s-setup/fmfi-hosts.yml`):

```yaml
controller:
  hosts:
    master:
      ansible_host: fmfi-k3s-ctl
      ansible_user: ubuntu

node:
  hosts:
    worker1:
      ansible_host: fmfi-k3s-n1
      ansible_user: ubuntu
    k3s-node-2:
      ansible_host: pocitadlo
      ansible_user: filip
    sklad:
      ansible_host: sklad
      ansible_user: filip
      k3s_node_ip: 10.89.100.10
```
