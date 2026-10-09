# Cluster Nodes

The K3s cluster consists of 6 nodes, each with a dedicated role. The cluster uses SQLite datastore (single server mode) with custom CIDRs.

## Node Architecture

```mermaid
flowchart LR
    subgraph cluster["K3s Cluster - v1.34"]
        CTRL["k3s-controller-1\nControl Plane\n10.89.100.8"]
        GPU1["k3s-node-1\nGPU Node (Titans)\n10.89.100.32"]
        GPU2["k3s-node-02\nGPU Node (RTX 5090)\n10.89.100.33"]
        GPU3["k3s-node-03\nGPU Node (RTX 5090)\n10.89.100.34"]
        SVC["pocitadlo\nService Node\n158.195.89.102"]
        STOR["sklad\nStorage Node\n10.89.100.10"]
    end

    CTRL --- GPU1
    CTRL --- GPU2
    CTRL --- GPU3
    CTRL --- SVC
    CTRL --- STOR
```

## Node Details

| Node | Role | IP | OS | GPU | Notes |
|------|------|----|----|-----|-------|
| k3s-controller-1 | `control-plane` | 10.89.100.8 | Ubuntu 24.04.3 | — | k3s data on root LV |
| k3s-node-1 | `gpu` | 10.89.100.32 | Ubuntu 24.04.3 | TITAN V (schedulable) + TITAN Xp (masked) | jupyter pool only |
| k3s-node-02 | `gpu` | 10.89.100.33 | Ubuntu 24.04.3 | RTX 5090 32G | k3s data on 7.3T `k3s-data` LV |
| k3s-node-03 | `gpu` | 10.89.100.34 | Ubuntu 24.04.3 | RTX 5090 32G | k3s data on 7.3T `k3s-data` LV |
| pocitadlo | `service` | 158.195.89.102 | Ubuntu 24.04.4 | — | holds all local-path PVs |
| sklad | `storage` | 10.89.100.10 | Ubuntu 24.04.4 | — | SeaweedFS; 62G root — never pull GPU images |

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
    end

    subgraph gpu_node1["k3s-node-1 (gpu — jupyter pool)"]
        direction LR
        VDP["Volcano Device Plugin"]
        CSI_N1["SeaweedFS CSI Node"]
        JW1["Jupyter User Notebooks\n(TITAN V slices)"]
    end

    subgraph gpu_node23["k3s-node-02/03 (gpu — ray + jupyter)"]
        direction LR
        VDP2["Volcano Device Plugin"]
        CSI_N2["SeaweedFS CSI Node"]
        JW2["Jupyter Notebooks\n(RTX 5090 slices)"]
        RW["Ray Workers"]
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
| `node-role.kubernetes.io/gpu` | k3s-node-1, -02, -03 | GPU workloads, Volcano device plugin |
| `node-role.kubernetes.io/service` | pocitadlo | Harbor, platform services |
| `node-role.kubernetes.io/storage` | sklad | SeaweedFS components |
| `gpu-node: "true"` | k3s-node-1, -02, -03 | Volcano device plugin DaemonSet, jupyter GPU pool |
| `accelerator: nvidia-gpu` | k3s-node-02, -03 only | ray workers, future 5090-only workloads |
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
    worker1:                      # k3s-node-1, Titans
      ansible_host: fmfi-k3s-n1
      ansible_user: ubuntu
    k3s-node-2:                   # pocitadlo, service node
      ansible_host: pocitadlo
      ansible_user: filip
    sklad:
      ansible_host: sklad
      ansible_user: filip
      k3s_node_ip: 10.89.100.10
    worker2:                      # k3s-node-02, RTX 5090
      ansible_host: fmfi-k3s-n2
      ansible_user: spravovatel
    worker3:                      # k3s-node-03, RTX 5090
      ansible_host: fmfi-k3s-n3
      ansible_user: spravovatel
```
