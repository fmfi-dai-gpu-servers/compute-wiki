# GPU Stack

The cluster provides GPU acceleration through a layered stack: NVIDIA drivers → container runtime → Volcano device plugin → vGPU scheduling → ML workloads. A single **NVIDIA TITAN Xp** (12GB VRAM) on `k3s-node-1` is shared among multiple concurrent users.

## GPU Stack Layers

```mermaid
flowchart TB
    subgraph workloads["ML Workloads"]
        JH_GPU["JupyterHub<br/>GPU Notebook"]
        RAY_GPU["Ray Workers<br/>GPU Tasks"]
    end

    subgraph scheduling["Scheduling Layer"]
        V_S["Volcano Scheduler<br/>deviceshare plugin"]
        V_DP["Volcano Device Plugin<br/>DaemonSet"]
    end

    subgraph runtime["Container Runtime"]
        N_RT["NVIDIA RuntimeClass"]
        CT["containerd + NVIDIA<br/>Container Toolkit"]
    end

    subgraph hardware["Hardware"]
        DRV["NVIDIA Host Drivers"]
        GPU["NVIDIA TITAN Xp<br/>12GB VRAM"]
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
architecture-beta
    group gpu(cloud)[NVIDIA TITAN Xp — 12GB VRAM]
    service vgpu1(server)[vGPU 1<br/>5 cores / 2.4GB] in gpu
    service vgpu2(server)[vGPU 2<br/>5 cores / 2.4GB] in gpu
    service vgpu3(server)[vGPU 3<br/>5 cores / 2.4GB] in gpu
    service dots(server)[...] in gpu
    service vgpu25(server)[vGPU 25<br/>max splits] in gpu

    group consumers(cloud)[Consumers]
    service jh(server)[Jupyter<br/>GPU Notebook] in consumers
    service ray(server)[Ray<br/>GPU Worker] in consumers

    jh:R --> L:vgpu1
    ray:R --> L:vgpu2
    ray:R --> L:vgpu3
```

## Scheduling Flow

```mermaid
sequenceDiagram
    participant User as User (JupyterHub/Ray)
    participant K8s as Kubernetes API
    participant V as Volcano Scheduler
    participant DP as Device Plugin
    participant GPU as Physical GPU

    User->>K8s: Submit pod with vGPU requests
    Note over K8s: volcano.sh/vgpu-number: 1<br/>volcano.sh/vgpu-cores: 5<br/>volcano.sh/vgpu-memory: 2400<br/>runtimeClassName: nvidia
    K8s->>V: Schedule via Volcano
    V->>DP: Check available vGPU resources
    DP->>GPU: Query GPU memory/compute
    GPU->>DP: Report available resources
    DP->>V: Report allocatable vGPUs
    V->>V: deviceshare plugin: allocate vGPU<br/>(binpack policy)
    V->>K8s: Pod scheduled on k3s-node-1
    K8s->>DP: Create container with vGPU allocation
    DP->>GPU: Partition GPU memory/compute
    Note over DP: hami-core mode:<br/>Memory isolation + compute percentage
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

1. **enqueue** — Queue management
2. **allocate** — Main allocation (deviceshare for vGPU)
3. **backfill** — Fill gaps with lower-priority jobs

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
