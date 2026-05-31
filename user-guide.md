# Compute Platform — User Guide

This guide covers everything you need to use the GPU-accelerated ML platform at FMFI DAI. All services are accessible through your web browser and from your Python code.

## Table of Contents

- [Quick Start](#quick-start)
- [Authentication & Access](#authentication--access)
- [JupyterHub — Notebooks](#jupyterhub--notebooks)
- [MLflow — Experiment Tracking](#mlflow--experiment-tracking)
- [Ray Cluster — Distributed Compute](#ray-cluster--distributed-compute)
- [Datasets & Storage](#datasets--storage)
- [Quick Reference](#quick-reference)

---

## Quick Start

### Prerequisites

- A **GitHub account** that is a member of the [`fmfi-dai-gpu-servers`](https://github.com/orgs/fmfi-dai-gpu-servers) organization
- Your org membership must be **public** (GitHub settings → Organizations → make visible)
- A modern web browser

### All Services at a Glance

| Service | URL | What It Does |
|---------|-----|--------------|
| JupyterHub | [jhub.c.dai.fmph.uniba.sk](https://jhub.c.dai.fmph.uniba.sk) | Interactive notebooks (CPU & GPU) |
| MLflow | [mlflow.c.dai.fmph.uniba.sk](https://mlflow.c.dai.fmph.uniba.sk) | Track experiments, metrics, and models |
| Ray Dashboard | [ray.c.dai.fmph.uniba.sk](https://ray.c.dai.fmph.uniba.sk) | Distributed compute jobs (CPU & GPU) |
| Datasets | [datasets.c.dai.fmph.uniba.sk](https://datasets.c.dai.fmph.uniba.sk) | Upload, browse, and manage datasets |
| S3 Storage | [storage.c.dai.fmph.uniba.sk](https://storage.c.dai.fmph.uniba.sk) | S3-compatible object storage API |

### First-Time Login

Every service uses the same login: click the login button → sign in with GitHub → done. Your account is created automatically on first login. See [Authentication & Access](#authentication--access) for details.

---

## Authentication & Access

All services (except where noted) use **single sign-on** through GitHub. You log in once with your GitHub account and get access to everything.

### How Login Works

```
Your Browser → Service URL → Keycloak Login → "Sign in with GitHub" → GitHub → Back to Service ✓
```

1. Navigate to any service URL (e.g., `https://jhub.c.dai.fmph.uniba.sk`)
2. You'll be redirected to a Keycloak login page
3. Click **"Sign in with GitHub"**
4. Authorize on GitHub (first time only)
5. You're redirected back to the service, logged in

> **Important:** You must be a **public** member of the `fmfi-dai-gpu-servers` GitHub organization. If your membership is private, login will fail. To fix this, go to GitHub → Your profile → Organizations → toggle visibility to Public.

### Your Username

Your username across all services is your **GitHub username**. This determines:
- Your JupyterHub home directory
- Your dataset bucket name (`datasets-<username>`)
- Your identity in MLflow and Ray

### Troubleshooting Login Issues

| Problem | Solution |
|---------|----------|
| "Access denied" after GitHub login | Ensure your `fmfi-dai-gpu-servers` org membership is **public** |
| Redirect loop | Clear browser cookies for `*.c.dai.fmph.uniba.sk` and try again |
| "Page not found" | Ensure you're on the university network or have SSH tunnel active |
| Session expired | Log in again — sessions time out periodically for security |

<details>
<summary>📖 What is Keycloak?</summary>

Keycloak is an open-source identity and access management system. It sits between you and all platform services, handling authentication so you don't need separate accounts for each service. Think of it as "Sign in with Google" but for our platform — you sign in with GitHub once and get access to everything.

</details>

---

## JupyterHub — Notebooks

**URL:** [jhub.c.dai.fmph.uniba.sk](https://jhub.c.dai.fmph.uniba.sk)

JupyterHub gives you personal Jupyter notebooks running on the cluster. Choose a profile based on your workload, get a persistent home directory, and optionally use GPU acceleration.

<details>
<summary>📖 What is JupyterHub?</summary>

JupyterHub is a multi-user version of Jupyter Notebook/JupyterLab. Instead of running notebooks on your laptop, they run on the cluster where they can access GPU hardware, shared datasets, and more computing power. Your notebooks and files persist between sessions.

</details>

### Choosing a Profile

When you log in, you'll see a spawner page with profile options:

| Profile | CPU | RAM | GPU | Best For |
|---------|-----|-----|-----|----------|
| **Light-weight notebook** | 1 | 1 GB | — | Quick scripts, data exploration |
| **Medium notebook** | 1 | 4 GB | — | Larger datasets, training small models |
| **Shared CUDA notebook** | 1.5 | 3.5 GB | ~2.4 GB VRAM (shared) | GPU inference, light training |

Select a profile and click **Start**. Your notebook will launch in a few seconds (images are pre-pulled to nodes).

### Multiple Notebooks (Named Servers)

You can run **up to 2 concurrent notebooks** with different profiles. For example:
- One lightweight notebook for writing code
- One GPU notebook for running training

To start a named server: on the JupyterHub home page, instead of clicking "My Server", type a name in the server name field and click "Add Named Server".

### Persistent Storage

| Property | Value |
|----------|-------|
| Home directory size | 10 GB |
| Persistence | Survives restarts — your files are kept |
| Access | ReadWriteMany — accessible from all your notebooks |

Your home directory (`/home/jovyan`) is stored on distributed storage and persists even when your notebook server is stopped. Files you save today will be there tomorrow.

### Auto-Shutdown (Culling)

Notebooks are automatically stopped to free resources:

| Condition | Timeout |
|-----------|---------|
| **Idle** (no activity) | 30 minutes |
| **Maximum lifetime** | 12 hours (even if active) |

When your notebook is culled, your files are preserved — only the running process is stopped. Simply start a new server to continue where you left off.

### Using the GPU

When you select the **Shared CUDA notebook** profile:

- You get access to a **shared NVIDIA TITAN Xp GPU**
- Your allocation: **~2.4 GB VRAM** and **~20% of GPU compute**
- The GPU is time-sliced among users, so performance varies with load
- The image comes with **CUDA 12.6** and common ML libraries pre-installed

**Check GPU availability in your notebook:**

```python
import torch

print(f"CUDA available: {torch.cuda.is_available()}")
print(f"GPU name: {torch.cuda.get_device_name(0)}")
print(f"GPU memory: {torch.cuda.get_device_properties(0).total_mem / 1e9:.1f} GB")
```

### Tips

- **Pre-installed packages**: CPU images use `jupyter/base-notebook` (minimal). The GPU image is `cschranz/gpu-jupyter:v1.9_cuda-12.6_ubuntu-24.04_slim` which includes PyTorch, TensorFlow, and common data science libraries.
- **Install packages**: Use `pip install --user <package>` to install additional packages. They persist in your home directory.
- **Large files**: Store datasets in your S3 bucket (see [Datasets & Storage](#datasets--storage)) rather than in your notebook home to save space.

---

## MLflow — Experiment Tracking

**URL:** [mlflow.c.dai.fmph.uniba.sk](https://mlflow.c.dai.fmph.uniba.sk)

MLflow lets you track machine learning experiments — log parameters, metrics, and artifacts, then compare runs and register models through a web UI or Python API.

<details>
<summary>📖 What is MLflow?</summary>

MLflow is an open-source platform for managing the ML lifecycle. Think of it as a lab notebook for machine learning: every time you train a model, you log what parameters you used, what metrics you achieved, and what the model artifacts look like. You can then compare experiments, reproduce results, and promote the best models to production.

</details>

### Accessing MLflow

- **Web UI**: Navigate to [mlflow.c.dai.fmph.uniba.sk](https://mlflow.c.dai.fmph.uniba.sk) — log in with GitHub SSO
- **From code**: Use the tracking URI below

### User Roles

| Role | How to Get It | Capabilities |
|------|---------------|--------------|
| **User** | Automatic on login | View experiments, log runs (after admin grants permissions) |
| **Admin** | Assigned by platform admin | Full access, manage permissions, generate API tokens |

> **Note:** New users have limited access by default. Ask an admin to grant you READ/EDIT permissions on relevant experiments.

### Connecting from Python

#### Step 1: Get an Access Token

1. Log in to the [MLflow web UI](https://mlflow.c.dai.fmph.uniba.sk)
2. Navigate to **`/oidc/permissions`**
3. Click **"Generate Access Token"**
4. Copy the token

#### Step 2: Configure MLflow

```python
import mlflow

mlflow.set_tracking_uri("https://mlflow.c.dai.fmph.uniba.sk")
```

Or set environment variables:

```bash
export MLFLOW_TRACKING_URI="https://mlflow.c.dai.fmph.uniba.sk"
export MLFLOW_TRACKING_USERNAME="<your-username>"
export MLFLOW_TRACKING_PASSWORD="<your-access-token>"
```

#### Step 3: Track Experiments

```python
import mlflow

mlflow.set_tracking_uri("https://mlflow.c.dai.fmph.uniba.sk")
mlflow.set_experiment("my-classification-experiment")

with mlflow.start_run(run_name="random-forest-v1"):
    # Log parameters
    mlflow.log_param("n_estimators", 100)
    mlflow.log_param("max_depth", 10)

    # Train your model here...
    accuracy = 0.95
    f1_score = 0.93

    # Log metrics
    mlflow.log_metric("accuracy", accuracy)
    mlflow.log_metric("f1_score", f1_score)

    # Log an artifact (e.g., a plot or data file)
    # mlflow.log_artifact("confusion_matrix.png")

    print(f"Run ID: {mlflow.active_run().info.run_id}")
```

#### Using from JupyterHub Notebooks

In your Jupyter notebook:

```python
import mlflow

mlflow.set_tracking_uri("https://mlflow.c.dai.fmph.uniba.sk")

# If you have env vars set (MLFLOW_TRACKING_USERNAME/PASSWORD),
# you can skip explicit auth. Otherwise, set them:
import os
os.environ["MLFLOW_TRACKING_USERNAME"] = "your-username"
os.environ["MLFLOW_TRACKING_PASSWORD"] = "your-access-token"

# Now use MLflow normally
mlflow.set_experiment("gpu-training")
with mlflow.start_run():
    mlflow.log_param("model", "resnet50")
    mlflow.log_metric("loss", 0.023)
```

### System Metrics

MLflow automatically logs **system metrics** (CPU, memory, GPU usage) for your runs. You can view these in the web UI under each run's metrics tab.

### Tips

- **Experiment organization**: Create separate experiments for different projects or model types
- **Artifact storage**: Artifacts (model files, plots, etc.) are stored on the cluster. Keep individual artifacts reasonably sized.
- **Compare runs**: Use the web UI's chart view to compare metrics across runs side-by-side

---

## Ray Cluster — Distributed Compute

**URL:** [ray.c.dai.fmph.uniba.sk](https://ray.c.dai.fmph.uniba.sk)

Ray provides distributed computing for scaling Python workloads. Submit jobs through the web dashboard or the CLI, and leverage GPU workers for parallel training or inference.

<details>
<summary>📖 What is Ray?</summary>

Ray is a framework for running distributed Python applications. Instead of running code on a single machine, Ray lets you spread work across multiple workers. This is useful for:
- **Parallel hyperparameter search**: Train many model variants simultaneously
- **Distributed training**: Split large training jobs across GPU workers
- **Batch inference**: Process large datasets in parallel
- **Data processing**: Scale Pandas/NumPy workloads with Ray Data

</details>

### Accessing the Dashboard

Navigate to [ray.c.dai.fmph.uniba.sk](https://ray.c.dai.fmph.uniba.sk) — log in with GitHub SSO. The dashboard lets you:
- View cluster status and available resources
- Submit and monitor jobs
- View logs and task details
- Inspect actors and placement groups

### CLI Authentication

For command-line usage, authenticate using the Device Authorization Grant flow:

```bash
# Get your token (interactive — requires opening a browser URL)
python ray_auth.py

# The script prints a URL and code — open the URL in your browser, enter the code
# On success, it prints export statements. Use eval to activate them:
eval $(python ray_auth.py)
```

After authentication, two environment variables are set:
- `RAY_ADDRESS=https://ray.c.dai.fmph.uniba.sk`
- `RAY_AUTH_TOKEN=<your-token>`

> **Note:** `ray_auth.py` is available in the `services/kube-ray/` directory. Tokens expire — re-run the script when needed.

### Submitting Jobs

#### From the CLI

```bash
# After running eval $(python ray_auth.py)
ray job submit --address https://ray.c.dai.fmph.uniba.sk -- python my_script.py
```

#### From Python

```python
import os
from ray.job_submission import JobSubmissionClient

token = os.environ["RAY_AUTH_TOKEN"]

client = JobSubmissionClient(
    address="https://ray.c.dai.fmph.uniba.sk",
    headers={"Authorization": f"Bearer {token}"},
)

# Submit a job
job_id = client.submit_job(entrypoint="python train.py")
print(f"Submitted job: {job_id}")

# Check status
print(client.get_job_status(job_id))

# Get logs
print(client.get_job_logs(job_id))

# List all jobs
for job in client.list_jobs():
    print(f"{job.job_id}: {job.status}")
```

#### With pip dependencies

```bash
ray job submit --address https://ray.c.dai.fmph.uniba.sk \
  --runtime-env-json '{"pip": ["torch", "transformers"]}' \
  -- python train.py
```

### Connecting from JupyterHub

From a notebook running on the cluster, connect using the Ray Client endpoint:

```python
import ray

ray.init("ray://ray-client.c.dai.fmph.uniba.sk:10001")
print(f"Cluster resources: {ray.cluster_resources()}")
```

### GPU Workers

The Ray cluster can scale from 0 to **4 GPU workers**, each with:

| Resource | Per Worker |
|----------|------------|
| CPU | 1.5 cores |
| RAM | 3.5 GB |
| GPU | 1 vGPU (~2.4 GB VRAM, shared) |

Workers **auto-scale**: they spin up when jobs arrive and shut down after **120 seconds** of idle time. If no one is using Ray, no GPU resources are consumed.

### Sample GPU Job

```python
import ray
import time

ray.init()

print(f"Cluster resources: {ray.cluster_resources()}")
print(f"Available GPUs: {ray.available_resources().get('GPU', 0)}")


@ray.remote(num_gpus=0.5)
def gpu_task(task_id: int) -> dict:
    import torch
    return {
        "task_id": task_id,
        "cuda_available": torch.cuda.is_available(),
        "device_name": torch.cuda.get_device_name(0) if torch.cuda.is_available() else None,
        "gpu_count": torch.cuda.device_count(),
    }


@ray.remote
def cpu_task(task_id: int) -> int:
    total = sum(i * i for i in range(10_000_000))
    return task_id


print("\n--- CPU tasks (4 parallel) ---")
start = time.time()
futures = [cpu_task.remote(i) for i in range(4)]
results = ray.get(futures)
print(f"Results: {results}")
print(f"Completed in {time.time() - start:.2f}s")

print("\n--- GPU tasks (2 parallel, 0.5 GPU each) ---")
try:
    start = time.time()
    futures = [gpu_task.remote(i) for i in range(2)]
    results = ray.get(futures)
    for r in results:
        print(f"  Task {r['task_id']}: cuda={r['cuda_available']}, device={r['device_name']}")
    print(f"Completed in {time.time() - start:.2f}s")
except Exception as e:
    print(f"GPU tasks failed (no GPU workers available?): {e}")

print("\nDone.")
```

Submit it:

```bash
ray job submit --address https://ray.c.dai.fmph.uniba.sk -- python sample_gpu_job.py
```

### Tips

- **Worker startup time**: The first job after idle may take 30-60 seconds while workers spin up
- **GPU task sizing**: Each worker has 1 vGPU. Use `num_gpus=0.5` to run 2 tasks per worker
- **Images**: Workers use `ray-ml:2.46.0` with PyTorch and CUDA pre-installed (Python 3.11)
- **Ray version**: 2.46.0

---

## Datasets & Storage

**Dashboard:** [datasets.c.dai.fmph.uniba.sk](https://datasets.c.dai.fmph.uniba.sk)
**S3 API:** [storage.c.dai.fmph.uniba.sk](https://storage.c.dai.fmph.uniba.sk)

Every user gets a personal S3 bucket for storing and sharing datasets. Access it through the web dashboard or programmatically with Python.

<details>
<summary>📖 What is S3 / Object Storage?</summary>

S3 (Simple Storage Service) is a standard way to store files in the cloud. Instead of a regular filesystem with folders, S3 uses "buckets" that hold "objects" (files). You interact with it via an API, typically using the `boto3` Python library. Our platform uses SeaweedFS, which is compatible with the S3 API — so any code that works with AWS S3 will work here too.

</details>

### The Datasets Dashboard

Navigate to [datasets.c.dai.fmph.uniba.sk](https://datasets.c.dai.fmph.uniba.sk) and log in with GitHub SSO. The dashboard provides:

- **File browser** — Navigate your bucket with a folder-based UI, breadcrumb navigation
- **Upload** — Drag-and-drop or click-to-browse file upload with progress bar
- **Download** — One-click download for any file
- **Create folders** — Organize your data
- **Delete** — Remove files and folders
- **Storage stats** — See your usage, quota, and file count
- **S3 credentials** — View your access key, secret key, and endpoint

Your personal bucket (`datasets-<username>`) is created automatically on first access with **20 GB** of storage. Versioning is enabled — old versions of files are kept for 365 days.

### Getting S3 Credentials

In the dashboard, click **"Show Credentials"** to reveal:
- **Access Key** — Your S3 access key
- **Secret Key** — Your S3 secret key (click to reveal, click to copy)
- **Endpoint** — `https://storage.c.dai.fmph.uniba.sk`
- **Bucket** — `datasets-<username>`
- **Region** — `us-east-1`

### Using S3 from Python

#### Install boto3

```bash
pip install boto3
```

#### Connect to Your Bucket

```python
import boto3

s3 = boto3.client(
    "s3",
    endpoint_url="https://storage.c.dai.fmph.uniba.sk",
    aws_access_key_id="<your-access-key>",
    aws_secret_access_key="<your-secret-key>",
    region_name="us-east-1",
)

BUCKET = "datasets-<username>"  # Replace <username> with your GitHub username
```

#### List Files and Folders

```python
resp = s3.list_objects_v2(Bucket=BUCKET, Prefix="", Delimiter="/")

print("Folders:")
for folder in resp.get("CommonPrefixes", []):
    print(f"  📁 {folder['Prefix']}")

print("Files:")
for obj in resp.get("Contents", []):
    size_mb = obj["Size"] / (1024 * 1024)
    print(f"  📄 {obj['Key']} ({size_mb:.1f} MB)")
```

#### Upload a File

```python
s3.upload_file("local_data.csv", BUCKET, "project-name/data.csv")
print("Uploaded!")
```

#### Download a File

```python
s3.download_file(BUCKET, "project-name/data.csv", "downloaded_data.csv")
print("Downloaded!")
```

#### Delete a File

```python
s3.delete_object(Bucket=BUCKET, Key="project-name/data.csv")
print("Deleted!")
```

#### Upload from JupyterHub

In a notebook, you can use the `sw_s3.py` helper which handles authentication automatically:

```python
from sw_s3 import get_s3_client, upload, download, list_datasets

# Get an authenticated client (auto-detects OIDC token)
s3 = get_s3_client()

# List your files
print(list_datasets(s3))

# Upload
upload(s3, "local_file.csv", "data/file.csv")

# Download
download(s3, "data/file.csv", "local_copy.csv")
```

#### List Files in a Subfolder

```python
resp = s3.list_objects_v2(Bucket=BUCKET, Prefix="project-name/", Delimiter="/")

for obj in resp.get("Contents", []):
    print(f"{obj['Key']} — {obj['Size'] / 1e6:.1f} MB")
```

### Storage Quotas

| Property | Value |
|----------|-------|
| Default quota | **20 GB** per user |
| Quota enforcement | Server-side + dashboard pre-upload check |
| Over-quota behavior | Uploads are rejected with a clear error message |
| Quota increase | Contact the platform administrator |

### Tips

- **Use from JupyterHub**: Use the same S3 endpoint `https://storage.c.dai.fmph.uniba.sk` from notebooks
- **Organize with prefixes**: Use `/` in key names to create folder-like structures (e.g., `project-a/train/images/001.jpg`)
- **Versioning**: Old file versions are kept for 365 days. Overwriting a file preserves the previous version.

---

## Quick Reference

### All Service URLs

| Service | URL | Auth Method |
|---------|-----|-------------|
| JupyterHub | `https://jhub.c.dai.fmph.uniba.sk` | GitHub SSO |
| MLflow | `https://mlflow.c.dai.fmph.uniba.sk` | GitHub SSO |
| MLflow Tokens | `https://mlflow.c.dai.fmph.uniba.sk/oidc/permissions` | Admin: generate tokens |
| Ray Dashboard | `https://ray.c.dai.fmph.uniba.sk` | GitHub SSO (browser) |
| Ray Client (TCP) | `ray://ray-client.c.dai.fmph.uniba.sk:10001` | Token (CLI) |
| Datasets Dashboard | `https://datasets.c.dai.fmph.uniba.sk` | GitHub SSO |
| S3 API | `https://storage.c.dai.fmph.uniba.sk` | Access key / OIDC token |

### CLI Quick Commands

```bash
# Ray: authenticate for CLI usage
eval $(python ray_auth.py)

# Ray: submit a job
ray job submit --address https://ray.c.dai.fmph.uniba.sk -- python script.py

# Ray: submit with dependencies
ray job submit --address https://ray.c.dai.fmph.uniba.sk \
  --runtime-env-json '{"pip": ["torch"]}' \
  -- python train.py
```

### Python Configuration Snippets

```python
# MLflow
import mlflow
mlflow.set_tracking_uri("https://mlflow.c.dai.fmph.uniba.sk")

# Ray (from cluster notebooks)
import ray
ray.init("ray://ray-client.c.dai.fmph.uniba.sk:10001")

# S3
import boto3
s3 = boto3.client("s3", endpoint_url="https://storage.c.dai.fmph.uniba.sk",
                   aws_access_key_id="...", aws_secret_access_key="...",
                   region_name="us-east-1")
```

### Resource Limits Summary

| Resource | Limit |
|----------|-------|
| JupyterHub notebooks per user | 2 (named servers) |
| JupyterHub idle timeout | 30 minutes |
| JupyterHub max server lifetime | 12 hours |
| JupyterHub home storage | 10 GB per user |
| Ray GPU workers | Up to 4 (auto-scaling from 0) |
| Ray GPU per worker | ~2.4 GB VRAM (shared) |
| Ray worker idle timeout | 120 seconds |
| Dataset storage | 20 GB per user |

### Getting Help

If you run into issues:
1. Check the [Troubleshooting](#troubleshooting-login-issues) section above
2. Verify your GitHub org membership is public
3. Contact the platform administrator
