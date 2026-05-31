# Authentication Flow

All services authenticate through a central **Keycloak** instance using the `compute` realm. Keycloak uses **GitHub** as an external identity provider, restricting access to members of the `fmfi-dai-gpu-servers` GitHub organization.

## Authentication Architecture

```mermaid
architecture-beta
    group ext(cloud)[External Identity]
    service github(internet)[GitHub<br/>fmfi-dai-gpu-servers] in ext

    group kc(cloud)[Keycloak — compute realm]
    service kc_svc(server)[Keycloak 26.5<br/>auth.c.dai.fmph.uniba.sk] in kc
    service pg(database)[PostgreSQL] in kc

    group clients(cloud)[OIDC Clients]
    service jh(server)[JupyterHub] in clients
    service ml(server)[MLflow] in clients
    service ray(server)[Ray Dashboard<br/>oauth2-proxy] in clients
    service ray_cli(server)[Ray CLI<br/>Device Auth] in clients
    service sw(server)[SeaweedFS S3<br/>STS] in clients
    service dd(server)[Datasets<br/>Dashboard] in clients

    github:B --> T:kc_svc
    kc_svc:L --> R:pg
    kc_svc:R --> L:jh
    kc_svc:R --> L:ml
    kc_svc:R --> L:ray
    kc_svc:R --> L:ray_cli
    kc_svc:R --> L:sw
    kc_svc:R --> L:dd
```

## OIDC Client Details

| Client | Type | Flow | Service |
|--------|------|------|---------|
| `jupyterhub` | Confidential | Authorization Code | JupyterHub (GenericOAuthenticator) |
| `mlflow` | Confidential | Authorization Code | MLflow (mlflow-oidc-auth plugin) |
| `ray-proxy` | Confidential | Authorization Code + PKCE (S256) | Ray Dashboard (oauth2-proxy) |
| `ray-cli` | Public | Device Authorization Grant + PKCE | Ray CLI job submission |
| `seaweedfs-s3` | Confidential | STS Token Exchange | SeaweedFS S3 API |
| `datasets-dashboard` | Confidential | Authorization Code | Datasets Dashboard (FastAPI) |

## Browser Auth Flow

```mermaid
sequenceDiagram
    participant U as User Browser
    participant S as Service (e.g. JupyterHub)
    participant K as Keycloak
    participant G as GitHub

    U->>S: Access service URL
    S->>K: Redirect to /auth (OIDC authorize)
    K->>U: Show login page
    U->>K: Click "Sign in with GitHub"
    K->>G: Redirect to GitHub OAuth
    G->>K: Callback with auth code
    K->>G: Exchange for GitHub token
    G->>K: Return user info + orgs
    K->>K: Verify fmfi-dai-gpu-servers membership
    K->>S: Callback with OIDC tokens
    S->>K: Exchange code for tokens
    K->>S: Return ID token + access token
    S->>U: Session established (cookie)
```

## Ray CLI Auth Flow (Device Authorization Grant)

```mermaid
sequenceDiagram
    participant CLI as Ray CLI
    participant K as Keycloak
    participant U as User Browser

    CLI->>K: POST /device (client_id=ray-cli)
    K->>CLI: device_code + verification_uri
    CLI->>U: Print URL + code
    U->>K: Open URL, enter code, approve
    loop Polling
        CLI->>K: POST /token (device_code)
        K-->>CLI: authorization_pending
    end
    U->>K: Complete approval
    CLI->>K: POST /token (device_code)
    K->>CLI: access_token + refresh_token
    CLI->>CLI: Set RAY_AUTH_TOKEN
```

## Keycloak Groups

| Group | Purpose | Services |
|-------|---------|----------|
| `mlflow` | MLflow read access | MLflow |
| `mlflow-admin` | MLflow admin access | MLflow |
| `storage-admins` | Full S3 access | SeaweedFS S3 |

## RBAC Mapping

- Keycloak groups are mapped to service roles via the `groups` OIDC claim
- Each service maps groups to internal roles (e.g., MLflow maps `mlflow-admin` → admin)
- GitHub org membership is the root identity check, enforced at Keycloak level
