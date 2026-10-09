# GPU Stack

The cluster provides GPU acceleration through a layered stack: NVIDIA drivers, container runtime, Volcano device plugin, vGPU scheduling, and ML workloads. Three GPUs are schedulable — **2× NVIDIA RTX 5090** (32GB, `k3s-node-02/03`) and **1× NVIDIA TITAN V** (16GB, `k3s-node-1`) — each shared among multiple concurrent users via vGPU slicing. A fourth GPU (TITAN Xp) is permanently masked for CUDA-architecture reasons; see [gpu-cuda-compatibility.md](gpu-cuda-compatibility.md).

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
        GPU["3 schedulable GPUs\n2× RTX 5090 (sm_120)\n1× TITAN V (sm_70)"]
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
    subgraph gpu5["k3s-node-02/03 - RTX 5090 32GB"]
        vgpu1["vGPU 1\n5 cores / 2.4GB"]
        vgpu2["vGPU 2\n5 cores / 2.4GB"]
        dots5["..."]
        vgpu25a["vGPU N\nmax 25 splits"]
    end
    subgraph gpuV["k3s-node-1 - TITAN V 16GB (jupyter only)"]
        vgV1["vGPU 1"]
        dotsV["..."]
        vgV25["max 25 splits"]
    end
    subgraph xpmask["k3s-node-1 - TITAN Xp - MASKED"]
        XM["not registered with hami\n(pci-stub at boot)"]
    end

    subgraph consumers["Consumers"]
        JH["Jupyter\nGPU Notebook"]
        RAY["Ray\nGPU Worker"]
    end

    JH --> vgpu1
    RAY --> vgpu2
    RAY --> vgV1


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
    V->>K8s: Pod scheduled on a GPU node (5090s via accelerator=nvidia-gpu, TITAN V via gpu-node=true)
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
| Operating mode | `hami-core` |
| Runtime class | `nvidia` |
| Device plugin image | `projecthami/volcano-vgpu-device-plugin:v1.11.0` |
| Volcano version | v1.13.1 |
| Platform CUDA line | cu128 (torch 2.8 on ray workers, torch 2.9 on jupyter) — covers sm_70…sm_120; see [gpu-cuda-compatibility.md](gpu-cuda-compatibility.md) |
| Ray worker nodes | `k3s-node-02/03` only (`accelerator=nvidia-gpu`) |
| Jupyter GPU nodes | `k3s-node-1/02/03` (`gpu-node=true`) |

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
