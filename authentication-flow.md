# Authentication Flow

All services authenticate through a central **Keycloak** instance using the `compute` realm. Keycloak uses **GitHub** as an external identity provider, restricting access to members of the `fmfi-dai-gpu-servers` GitHub organization.

## Authentication Architecture

```mermaid
flowchart TB
    subgraph ext["External Identity"]
        GH["GitHub\nfmfi-dai-gpu-servers"]
    end

    subgraph kc["Keycloak - compute realm"]
        KC["Keycloak 26.5\nauth.c.dai.fmph.uniba.sk"]
        PG["PostgreSQL"]
    end

    subgraph clients["OIDC Clients"]
        JH["JupyterHub"]
        ML["MLflow"]
        RAY["Ray Dashboard\noauth2-proxy"]
        RCLI["Ray CLI\nDevice Auth"]
        SW["SeaweedFS S3\nSTS"]
        DD["Datasets\nDashboard"]
    end

    GH --> KC
    KC --> PG
    KC --> JH
    KC --> ML
    KC --> RAY
    KC --> RCLI
    KC --> SW
    KC --> DD
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
    participant S as Service
    participant K as Keycloak
    participant G as GitHub

    U->>S: Access service URL
    S->>K: Redirect to OIDC authorize
    K->>U: Show login page
    U->>K: Click Sign in with GitHub
    K->>G: Redirect to GitHub OAuth
    G->>K: Callback with auth code
    K->>G: Exchange for GitHub token
    G->>K: Return user info + orgs
    K->>K: Verify org membership
    K->>S: Callback with OIDC tokens
    S->>K: Exchange code for tokens
    K->>S: Return ID token + access token
    S->>U: Session established
```

## Ray CLI Auth Flow (Device Authorization Grant)

```mermaid
sequenceDiagram
    participant CLI as Ray CLI
    participant K as Keycloak
    participant U as User Browser

    CLI->>K: POST /device
    K->>CLI: device_code + verification_uri
    CLI->>U: Print URL + code
    U->>K: Open URL, enter code, approve
    loop Polling
        CLI->>K: POST /token
        K-->>CLI: authorization_pending
    end
    U->>K: Complete approval
    CLI->>K: POST /token
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
- Each service maps groups to internal roles (e.g., MLflow maps `mlflow-admin` to admin)
- GitHub org membership is the root identity check, enforced at Keycloak level
