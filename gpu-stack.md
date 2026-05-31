# GPU Stack

The cluster provides GPU acceleration through a layered stack: NVIDIA drivers, container runtime, Volcano device plugin, vGPU scheduling, and ML workloads. A single **NVIDIA TITAN Xp** (12GB VRAM) on `k3s-node-1` is shared among multiple concurrent users.

## GPU Stack Layers

```mermaid
flowchart TB
    subgraph workloads["ML Workloads"]
        JH_GPU["JupyterHub\nGPU Notebook"]
        RAY_GPU["Ray Workers\nGPU Tasks"]
    end

    subgraph scheduling["Scheduling Layer"]
        V_S["Volcano Scheduler\ndeviceshare plugin"]
        V_DP["Volcano Device Plugin\nDaemonSet"]
    end

    subgraph runtime["Container Runtime"]
        N_RT["NVIDIA RuntimeClass"]
        CT["containerd + NVIDIA\nContainer Toolkit"]
    end

    subgraph hardware["Hardware"]
        DRV["NVIDIA Host Drivers"]
        GPU["NVIDIA TITAN Xp\n12GB VRAM"]
    end

    workloads --> scheduling
    scheduling --> runtime
    runtime --> hardware

    JH_GPU --> V_S
    RAY_GPU --> V_S
    V_S --> V_DP
    V_DP --> N_RT
    N_RT --> CT
    CT --> DRV
    DRV --> GPU
```

## vGPU Architecture

```mermaid
flowchart LR
    subgraph gpu["NVIDIA TITAN Xp - 12GB VRAM"]
        vgpu1["vGPU 1\n5 cores / 2.4GB"]
        vgpu2["vGPU 2\n5 cores / 2.4GB"]
        vgpu3["vGPU 3\n5 cores / 2.4GB"]
        dots["..."]
        vgpu25["vGPU N\nmax 25 splits"]
    end

    subgraph consumers["Consumers"]
        JH["Jupyter\nGPU Notebook"]
        RAY["Ray\nGPU Worker"]
    end

    JH --> vgpu1
    RAY --> vgpu2
    RAY --> vgpu3
```

## Scheduling Flow

```mermaid
sequenceDiagram
    participant User as User
    participant K8s as Kubernetes API
    participant V as Volcano Scheduler
    participant DP as Device Plugin
    participant GPU as Physical GPU

    User->>K8s: Submit pod with vGPU requests
    Note over K8s: volcano.sh/vgpu-number: 1, vgpu-cores: 5, vgpu-memory: 2400, runtimeClassName: nvidia
    K8s->>V: Schedule via Volcano
    V->>DP: Check available vGPU resources
    DP->>GPU: Query GPU memory/compute
    GPU->>DP: Report available resources
    DP->>V: Report allocatable vGPUs
    V->>V: deviceshare: allocate vGPU (binpack)
    V->>K8s: Pod scheduled on k3s-node-1
    K8s->>DP: Create container with vGPU allocation
    DP->>GPU: Partition GPU memory/compute
    Note over DP: hami-core mode: Memory isolation + compute percentage
```

## vGPU Resources

Custom resources used in pod specs:

| Resource | Unit | Description |
|----------|------|-------------|
| `volcano.sh/vgpu-number` | count | Number of vGPUs requested |
| `volcano.sh/vgpu-memory` | MB | VRAM allocation per vGPU |
| `volcano.sh/vgpu-cores` | percentage | Compute cores (out of 100) |
| `volcano.sh/vgpu-mode` | string | Operating mode (`hami-core`) |

## Current Configuration

| Parameter | Value |
|-----------|-------|
| Device split count | 25 (max vGPUs per GPU) |
| Device memory scaling | 1.0 |
| Operating mode | `hami-core` |
| Runtime class | `nvidia` |
| Device plugin image | `projecthami/volcano-vgpu-device-plugin:v1.11.0` |
| Volcano version | v1.13.1 |

## Scheduler Config

Volcano uses these scheduling plugins in order:

1. **enqueue** -- Queue management
2. **allocate** -- Main allocation (deviceshare for vGPU)
3. **backfill** -- Fill gaps with lower-priority jobs

The `deviceshare` plugin uses a **binpack** policy to pack vGPU workloads efficiently.

## Workload Examples

### JupyterHub GPU Profile
```yaml
resources:
  limits:
    volcano.sh/vgpu-number: 1
    volcano.sh/vgpu-cores: 5
    volcano.sh/vgpu-memory: 2400
  requests:
    volcano.sh/vgpu-number: 1
    volcano.sh/vgpu-cores: 5
    volcano.sh/vgpu-memory: 2400
schedulerName: volcano
runtimeClassName: nvidia
```

### Ray Worker GPU
```yaml
resources:
  limits:
    volcano.sh/vgpu-number: 1
    volcano.sh/vgpu-cores: 5
    volcano.sh/vgpu-memory: 2400
  requests:
    volcano.sh/vgpu-number: 1
    volcano.sh/vgpu-cores: 5
    volcano.sh/vgpu-memory: 2400
schedulerName: volcano
runtimeClassName: nvidia
```
