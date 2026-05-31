# Network Topology

## Access Path

Users connect to the cluster through SSH tunnels. There is no VPN; all remote access is via SSH port forwarding to the K3s API server.

```mermaid
flowchart LR
    subgraph external["Developer Machine"]
        Laptop["Workstation"]
    end

    subgraph tunnel["SSH Tunnel"]
        SSH["SSH port-forward :6443"]
    end

    subgraph cluster["K3s Cluster"]
        API["K3s API - 10.89.100.8:6443"]
        Traefik["Traefik - :80/:443"]
    end

    subgraph dns["DNS / TLS"]
        Wildcard["*.c.dai.fmph.uniba.sk"]
        LE["Let's Encrypt"]
    end

    Laptop --> SSH --> API --> Traefik
    Wildcard --> Traefik
    Traefik --> LE
```

## SSH Tunnel Setup

The `remote-kubectl.sh` helper script establishes an SSH tunnel for kubectl access:

```bash
ssh -fN -L 6443:127.0.0.1:6443 fmfi-k3s-ctl
```

The kubeconfig (`~/.kube/fmfi.yaml`) points to `localhost:6443`, which forwards to the K3s controller.

## Cluster Networking

```mermaid
flowchart TB
    subgraph cluster["K3s Cluster"]
        subgraph pods["Pod Network - 172.16.0.0/16"]
            P1["Pods on controller-1 - 172.16.0.x"]
            P2["Pods on k3s-node-1 - 172.16.1.x"]
            P3["Pods on pocitadlo - 172.16.2.x"]
            P4["Pods on sklad - 172.16.3.x"]
        end

        subgraph svcs["Service Network - 172.17.0.0/16"]
            S1["kube-dns: 172.17.0.10"]
            S2["kubernetes API: 172.17.0.1"]
            S3["Service ClusterIPs"]
        end
    end

    subgraph external["External"]
        DNS["*.c.dai.fmph.uniba.sk"]
        Users["Browser / kubectl"]
    end

    Users -->|SSH tunnel :6443| P1
    DNS -->|A/CNAME records| P1
    P1 --> S1 & S2
```

## Network Details

| Network | CIDR | Purpose |
|---------|------|---------|
| Pod Network | `172.16.0.0/16` | Inter-pod communication (VXLAN overlay) |
| Service Network | `172.17.0.0/16` | ClusterIP services |
| Node Network | `10.89.100.0/24` | Physical node IPs (internal FMFI) |

## Firewall Rules (UFW)

| Port | Protocol | Purpose |
|------|----------|---------|
| 6443 | TCP | Kubernetes API server |
| 80 | TCP | HTTP (redirect to HTTPS) |
| 443 | TCP | HTTPS (Traefik entrypoint) |
| 10250 | TCP | Kubelet API |
| 8472 | UDP | VXLAN overlay network |
| 30000-32767 | TCP | NodePort range |
| 10001 | TCP | Ray Client (TCP IngressRoute) |

## Domain Routing

All services share the wildcard domain `*.c.dai.fmph.uniba.sk`. Traefik routes by Host header:

| Domain | Service |
|--------|---------|
| `auth.c.dai.fmph.uniba.sk` | Keycloak |
| `jhub.c.dai.fmph.uniba.sk` | JupyterHub |
| `mlflow.c.dai.fmph.uniba.sk` | MLflow |
| `ray.c.dai.fmph.uniba.sk` | Ray Dashboard |
| `ray-client.c.dai.fmph.uniba.sk` | Ray Client (TCP) |
| `harbor.c.dai.fmph.uniba.sk` | Harbor Registry |
| `storage.c.dai.fmph.uniba.sk` | SeaweedFS S3 API |
| `datasets.c.dai.fmph.uniba.sk` | Datasets Dashboard |
| `traefik.c.dai.fmph.uniba.sk` | Traefik Dashboard |
