# GPU Architecture & CUDA Compatibility

Detailed record of the 2026-10 GPU platform consolidation: why "the CUDA
:image" failed on half the cluster, the fix (cu128 everywhere + TITAN Xp
masked — the Xp was later removed from the VM and the masking reverted,
2026-10-12), the exact mechanisms applied, and how to verify / reverse each
piece. For the general stack layers see [gpu-stack.md](gpu-stack.md).

## Hardware Inventory

| Node | GPU | Micro-arch | VRAM | Driver | Schedulable via |
|------|-----|-----------|------|--------|-----------------|
| k3s-node-02 | NVIDIA RTX 5090 | sm_120 (Blackwell) | 32G | 580.173.02 | `accelerator=nvidia-gpu` (ray), `gpu-node=true` (jupyter) |
| k3s-node-03 | NVIDIA RTX 5090 | sm_120 (Blackwell) | 32G | 580.173.02 | same |
| k3s-node-1 | NVIDIA TITAN V | sm_70 (Volta) | 16G | 580.173.02 | `gpu-node=true` (jupyter only) |
| k3s-node-1 | NVIDIA TITAN Xp | sm_61 (Pascal) | 12G | — | removed from the VM 2026-10-12 (previously masked, not schedulable) |

All GPUs are shared through Volcano + hami vGPU (hami-core mode): a pod
requests `volcano.sh/vgpu-number/-cores/-memory` and gets a fraction of one
physical GPU.

## The Problem (RCA)

Symptom: "the GPU CUDA image works on the Titans but not the 5090s" — and
vice versa. Root cause: **two platform images built on different CUDA
toolkit generations, each supporting a disjoint end of the GPU arch
spectrum.** Drivers, container toolkit, hami, and storage were all healthy.

Verified behavior matrix (live tests on this cluster, torch `matmul`;
2026-10, while the Xp was still installed):

| torch build | TITAN V sm_70 | TITAN Xp sm_61 | RTX 5090 sm_120 |
|---|---|---|---|
| 2.6.0+cu126 (old jupyter image v1.9) | OK | OK | **FAIL** `no kernel image is available` |
| 2.8.0+cu128 (ray-worker image) | OK | **FAIL** | OK |
| 2.9.1+cu128 (jupyter image v1.10 "cuda-12.9") | OK | **FAIL** | OK |

Why: official PyTorch wheels compile kernels for a contiguous arch range —
cu126 wheels span sm_50–sm_90 (Pascal yes, Blackwell no: CUDA 12.6
predates RTX 5090), cu128 wheels span sm_70–sm_120 (Blackwell yes, Pascal
no: CUDA 12.8 dropped Pascal support). **No single official build covers
both sm_61 and sm_120.**

## Decision (2026-10-08): standardize on cu128, mask the Xp

Two options existed:

1. **One arch pool**: cu128 everywhere (TITAN V + 5090s), mask the TITAN Xp
   out of scheduling. One image, one user experience. — **chosen**
2. **Two arch pools**: keep cu126 and cu128 images and split scheduling per
   node arch. Doubles image maintenance, users must know which pool they
   landed on; rejected as high-effort/confusing.

The TITAN Xp was not broken hardware — while still installed it was masked
out of scheduling. On 2026-10-12 it was removed from the k3s-node-1 VM
entirely (host-side passthrough removed) and the masking machinery was
reverted. If it ever returns it needs a cu126-era stack (see
[Reversal](#reversal)) — it is simply impossible to serve one shared CUDA
image across Pascal and Blackwell simultaneously.

## What Changed — consolidation (2026-10-08/09)

| Component | Before | After |
|---|---|---|
| Ray worker image | `ray-ml:2.46.0.deprecated-py311-gpu` (cu126, deprecated upstream) | `services/ray-worker:2.46.0-cu128-v1` (torch 2.8.0) |
| Ray worker scheduling | any gpu node | `nodeSelector: accelerator=nvidia-gpu` → 5090 nodes only |
| Ray runtimeClass | injected by helm post-renderer script | native chart value `worker.runtimeClassName: nvidia` (chart 1.6.1) |
| Jupyter CUDA profile image | `cschranz/gpu-jupyter:v1.9_cuda-12.6_...` | `harbor.../mirrors/cschranz/gpu-jupyter:v1.10_cuda-12.9_ubuntu-24.04_slim` (torch 2.9.1+cu128) |
| Jupyter pre-pullers | all nodes pull all profile images | hook puller on `gpu-node=true` only; continuous puller disabled |
| k3s-node-1 GPUs registered with hami | TITAN V + TITAN Xp | TITAN V only |

Deployed: ray helm rev 24 (2026-10-08), jupyterhub helm rev 15 (2026-10-09).

Follow-up on 2026-10-12: the TITAN Xp was removed from the k3s-node-1 VM
entirely (host-side passthrough removed), so the masking machinery
documented below was reverted the same day:

| Component | Before | After (2026-10-12) |
|---|---|---|
| k3s-node-1 Xp PCI passthrough | present in VM, masked (unbind + pci-stub) | removed from the VM |
| `/etc/default/grub` | `GRUB_CMDLINE_LINUX_DEFAULT="pci-stub.ids=10de:1b02"` | `""` + `update-grub` |
| `/etc/initramfs-tools/modules` | `pci-stub` line | removed + `update-initramfs -u -k all` |
| hami configmap `volcano-vgpu-node-config` | `filterdevices.uuid` = Xp uuid (inert) | `filterdevices` deleted |
| node-1 hami registration | TITAN V only (12288 MB) | unchanged by the revert |

## TITAN Xp Masking — Full Mechanism

> **Status 2026-10-12: historical.** The Xp has been removed from the VM
> (host-side passthrough removed), and all masking mechanisms below were
> reverted the same day: the runtime unbind/driver_override died with the
> device, `pci-stub.ids` was dropped from `GRUB_CMDLINE_LINUX_DEFAULT`
> (+ `update-grub`), `pci-stub` was removed from
> `/etc/initramfs-tools/modules` (+ `update-initramfs -u -k all`), and the
> `filterdevices` entry was deleted from configmap
> `volcano-vgpu-node-config`. Plugin pod restarted afterwards;
> `Reporting devices` → TITAN V only, node allocatable still 12288.

Requirements: remove the Xp from *scheduling* while keeping the TITAN V
fully usable on the same node. Attempts, in order:

### Attempt 1 — hami node annotation `volcano.sh/vgpu-number: 1` on the node: FAILED

Not enforced anywhere; node kept advertising 2 vGPUs / 24576 MB. Annotation
removed. (This annotation is meaningful on *pods* as a request, not on
nodes as a cap.)

### Attempt 2 — hami `filterdevices` in configmap `volcano-vgpu-node-config`: FAILED

Both documented forms were tested (index and uuid):

```json
"filterdevices": { "uuid": ["GPU-982f3ee9-b8d5-7274-2594-6cd76b61d3c0"], "index": [] }
"filterdevices": { "uuid": [], "index": [1] }
```

The plugin **loads** the filter — log line `FilterDevice: &{[GPU-982f3ee9-...] []}`
proves it — but then registers **both** GPUs anyway:

```
Reporting devices GPU-75545d5c-...NVIDIA TITAN V,...:GPU-982f3ee9-...NVIDIA TITAN Xp,...
```

and the Volcano scheduler still assigned the Xp to test pods. Conclusion:
`filterdevices` is non-functional for NVML devices in
`projecthami/volcano-vgpu-device-plugin:v1.11.0` (operatingmode
`hami-core`). The uuid entry was left in the configmap (inert, documenting
intent) until the Xp left the VM; it was deleted on 2026-10-12.

### Working mechanism — unbind the Xp from the nvidia driver

If the driver doesn't see the GPU, NVML doesn't report it, hami registers
nothing for it, the scheduler can never place a pod on it. Runtime
procedure on k3s-node-1 (Xp = PCI `0000:00:11.0`, TITAN V = `0000:00:10.0`):

```bash
sudo modprobe vfio-pci                                              # give it a harmless driver to bind
echo vfio-pci | sudo tee /sys/bus/pci/devices/0000:00:11.0/driver_override
echo 0000:00:11.0  | sudo tee /sys/bus/pci/drivers/nvidia/unbind     # nvidia lets go
echo 0000:00:11.0  | sudo tee /sys/bus/pci/drivers/vfio-pci/bind     # park it (optional)
```

Notes from the live run:
- `driver_override` **must be set before unbind**, otherwise the nvidia
  driver re-claims the device.
- In practice the final `bind` to vfio-pci hung once and left the device
  simply *unbound* — which is equally masked (the override prevents any
  re-claim). Do not assume the bind succeeded; check with `lspci -k`.
- After unbinding, **restart the volcano device plugin pod** on the node —
  it only re-reads devices at startup:

```bash
fk -n volcano-system delete pod -l name=volcano-device-plugin --field-selector spec.nodeName=k3s-node-1
# NB: label is name= (DS selector, verified 2026-10-12); an app= selector matches nothing
```

- Restart any other DaemonSet pod that caches NVML handles if needed.

### Verification (all observed post-mask)

```bash
ssh ubuntu@fmfi-k3s-n1 'nvidia-smi --query-gpu=index,name --format=csv,noheader'
# → 0, NVIDIA TITAN V          (Xp gone)

fk -n volcano-system logs <plugin-pod-on-node-1> -c volcano-device-plugin | grep "Reporting devices"
# → only GPU-75545d5c-...NVIDIA TITAN V

fk get node k3s-node-1 -o jsonpath='{.status.allocatable.volcano\.sh/vgpu-memory}'
# → 12288                     (was 24576 with both GPUs)

fk get nodes -o custom-columns='NODE:.metadata.name,VGPU-MEM:.status.allocatable.volcano\.sh/vgpu-memory'
# observed steady-state: node-02/03 = 32607 (RTX 5090), node-1 = 12288
```

End-to-end: two concurrent vGPU test pods (volcano scheduler, nodeSelector
k3s-node-1) both logged `GPU0 NVIDIA TITAN V sm_70 matmul=OK`.

### Reboot persistence (grub pci-stub)

The runtime unbind dies at reboot. Persistent mechanism on k3s-node-1
(Ubuntu 24.04, grub2):

1. `/etc/default/grub`:
   `GRUB_CMDLINE_LINUX_DEFAULT="pci-stub.ids=10de:1b02"`
   ⚠️ The Xp's PCI device ID here is **`1b02`** (verified via
   `cat /sys/bus/pci/devices/0000:00:11.0/device` → `0x1b02`); do not trust
   datasheets — some TITAN Xp variants are `1b06`.
2. `sudo update-grub` — arg confirmed present in `/boot/grub/grub.cfg`.
3. `pci-stub` is a **module, not builtin** in Ubuntu generic kernels. If it
   only loads from the root filesystem it races the nvidia driver. Force it
   into every initramfs: `echo pci-stub >> /etc/initramfs-tools/modules &&
   sudo update-initramfs -u -k all` (all kernels — node has 6.8.0-138/139/142
   installed; next boot uses 142 while 138 currently runs).
   Verified: `lsinitramfs /boot/initrd.img-6.8.0-142-generic | grep pci-stub`.

At boot: pci-stub (from initramfs, before root pivot) claims `10de:1b02`,
nvidia never sees it, NVML/hami expose only the TITAN V. The runtime
driver_override from the original masking remains set on the device —
harmless; clear it during a reversal.

Reverted 2026-10-12 when the Xp left the VM: the grub arg was removed
(`grep -c pci-stub.ids /boot/grub/grub.cfg` → 0), the `pci-stub` line was
deleted from `/etc/initramfs-tools/modules` and both kernels' initramfs
rebuilt without it (`lsinitramfs` → 0 matches each; the kernel set is now
6.8.0-146/138 — the 139/142 kernels referenced above are since gone).

### Reversal (revive the Xp)

Historical procedure (while the masking was in place, pre-2026-10-12):

```bash
# on k3s-node-1, after removing pci-stub.ids from GRUB_CMDLINE_LINUX_DEFAULT + update-grub:
echo > /sys/bus/pci/devices/0000:00:11.0/driver_override      # clear override
echo 0000:00:11.0 > /sys/bus/pci/drivers/nvidia/bind           # rebind
# then restart the volcano device plugin pod on the node
```

Since 2026-10-12 the Xp is no longer passed through to the k3s-node-1 VM at
all and the masking is fully reverted (grub, initramfs, configmap). Reviving
it means re-adding the PCI passthrough on the VM host; on the guest side a
reboot plus the usual plugin restart/verification (below) suffices. The Xp
then needs workloads with a cu126-era stack (e.g.
`gpu-jupyter:v1.9_cuda-12.6`) — see the compatibility matrix above.

### Revert record (2026-10-12)

What was actually found and done when the Xp was removed from the VM:

- Pre-revert state on the node: `/proc/cmdline` still carried
  `pci-stub.ids=10de:1b02` (booted before the revert; inert — the stub
  matched no device), `pci_stub` loaded with 0 uses, PCI `0000:00:11.0`
  absent from sysfs (so the runtime unbind/driver_override needed no
  clearing), TITAN V `driver_override` = `(null)`.
- Sweep for other residue was clean: `/etc/modules`, `/etc/modules-load.d/`,
  `/etc/rc.local`, `/etc/systemd/system/`, `/etc/udev/rules.d/`.
- Grub + initramfs reverted as listed under
  [Reboot persistence](#reboot-persistence-grub-pci-stub) above.
- Configmap cleaned via jq + apply:
  `jq '.data["config.json"] |= (fromjson | .nodeconfig |= map(del(.filterdevices)) | tostring)'`
  piped into `fk apply -f -`.
- Node-1 plugin pod deleted and recreated: fresh pod logged
  `FilterDevice: <nil>` and `Reporting devices GPU-75545d5c-...NVIDIA TITAN V`
  only; node-1 `volcano.sh/vgpu-memory` allocatable stayed 12288, node-02/03
  unchanged at 32607.

## JupyterHub Rollout Notes (rev 15)

Three stacked failures were resolved on the way to the deploy; documented
because each will recur in future image bumps:

1. **Harbor proxy-cache transient 500s**: the `mirrors` project proxies
   docker.io; manifest HEADs returned `500 Internal Server Error` for a
   window (upstream/rate-limit blip). Nodes with the image cached never
   re-ask, so only the remaining nodes kept failing. Self-heals; retry.
2. **Wedged helm release**: cancelling a helm upgrade client mid-hook
   leaves server-side hooks running and the release in `pending-upgrade`;
   subsequent upgrades fail with "another operation (install/upgrade/
   rollback) is in progress". Recover with
   `fhelm rollback jupyterhub -n jupyterhub` (returns to last deployed,
   then upgrade fresh). Always let the pre-pull hook finish instead of
   cancelling.
3. **Service nodes cannot hold the GPU image** (the real blocker): the
   GPU image extracts to ~20GB; sklad has a 62G root with ~25G free —
   containerd aborts with `no space left on device` mid-cuDNN-layer.
   Service nodes (sklad, pocitadlo, controller) never run GPU notebooks,
   so the pre-pullers were restricted in `services/jupyter-hub/config.yaml`:

```yaml
prePuller:
  hook:
    nodeSelector:      # chart schema allows nodeSelector here
      gpu-node: 'true'
  continuous:
    enabled: false     # schema has NO nodeSelector for continuous; disable
```

Verification: release `deployed` (rev 15), fresh hub pod Running, and the
live hub secret's `values.yaml` serves the new image:
`fk -n jupyterhub get secret hub -o json | grep -o 'gpu-jupyter:[^"]*'`.

Cluster-op reminder: use `fk` / `fhelm` wrappers only — they maintain the
KUBECONFIG and ssh tunnel; plain `helm` with `KUBECONFIG` set dies with
`dial tcp 127.0.0.1:6443: connect: connection refused`.

## Ongoing Verification Recipe

After any node reboot or hami/plugin change, confirm the GPU surface:

```bash
# 1. per-GPU-node registration (find pods: fk -n volcano-system get pods -l name=volcano-device-plugin -o wide)
fk -n volcano-system logs <plugin-pod> -c volcano-device-plugin | grep "Reporting devices"
# expect: node-02/03 → one RTX 5090 each; node-1 → TITAN V only (unchanged by the 2026-10-12 Xp removal)

# 2. node allocatable
fk get nodes -o custom-columns='NODE:.metadata.name,VGPU-MEM:.status.allocatable.volcano\.sh/vgpu-memory'
# expect: node-02/03 = 32607 (RTX 5090); node-1 = 12288 (TITAN V)

# 3. quick CUDA sanity through the whole stack (volcano + hami + runtimeClass)
kubectl run cudasmoke --image=harbor.c.dai.fmph.uniba.sk/mirrors/cschranz/gpu-jupyter:v1.10_cuda-12.9_ubuntu-24.04_slim \
  --overrides='{"spec":{"schedulerName":"volcano","runtimeClassName":"nvidia",
  "nodeSelector":{"gpu-node":"true"},
  "containers":[{"name":"cudasmoke","command":["python","-c","import torch;print(torch.cuda.get_device_name(0), torch.cuda.get_device_capability(0))"],
  "resources":{"limits":{"volcano.sh/vgpu-number":1,"volcano.sh/vgpu-cores":5,"volcano.sh/vgpu-memory":2400}}}]}}' \
  -n gpu-workloads --restart=Never
```
