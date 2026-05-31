# Compute Platform Wiki

Architecture and infrastructure documentation for the GPU-accelerated ML platform running at FMFI DAI.

## Contents

| Page | Description |
|------|-------------|
| [Architecture Overview](architecture-overview.md) | Big-picture service architecture and data flow |
| [Network Topology](network-topology.md) | Physical network, SSH access, cluster CIDRs, DNS |
| [Cluster Nodes](cluster-nodes.md) | K3s node roles, resources, and pod placement |
| [Storage Architecture](storage-architecture.md) | SeaweedFS, CSI driver, storage classes, PVCs |
| [Authentication Flow](authentication-flow.md) | Keycloak OIDC, registered clients, auth per service |
| [GPU Stack](gpu-stack.md) | NVIDIA runtime, Volcano vGPU, device sharing |
| [Service Map](service-map.md) | Per-service details: domains, images, Helm charts |
