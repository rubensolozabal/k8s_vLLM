# k8s_vLLM

Production-ready Kubernetes deployment for serving LLMs with [vLLM](https://github.com/vllm-project/vllm), monitored by Prometheus + Grafana and tracked with MLflow.

## Architecture

```
                        ┌────────────────────────────────┐
                        │        Kubernetes Cluster       │
                        │                                 │
  Client Notebook       │   ┌─────────┐   ┌─────────┐   │
  ─────────────────►    │   │  vLLM   │   │  vLLM   │   │
  (LangChain + MLflow)  │   │ Pod #1  │   │ Pod #2  │   │
                        │   │ (GPU)   │   │ (GPU)   │   │
                        │   └────┬────┘   └────┬────┘   │
                        │        │              │        │
                        │   ┌────┴──────────────┴────┐   │
                        │   │    vllm Service (LB)   │   │
                        │   └────────────────────────┘   │
                        │                                 │
                        │   ┌──────────┐  ┌───────────┐  │
                        │   │Prometheus│─►│  Grafana   │  │
                        │   │(per-pod) │  │(dashboard) │  │
                        │   └──────────┘  └───────────┘  │
                        │                                 │
                        │   ┌──────────┐                  │
                        │   │  MLflow  │                  │
                        │   │(tracking)│                  │
                        │   └──────────┘                  │
                        └────────────────────────────────┘
```

## Stack

| Component | Purpose | Image |
|-----------|---------|-------|
| **vLLM** | LLM inference server (2 replicas, 1 GPU each) | `vllm/vllm-openai:latest` |
| **MLflow** | Experiment tracking & trace logging | `ghcr.io/mlflow/mlflow:latest` |
| **Prometheus** | Metrics collection (per-pod scraping) | `prom/prometheus:latest` |
| **Grafana** | Dashboards (latency, throughput, tokens) | `grafana/grafana:latest` |

## Prerequisites

- Kubernetes cluster with **2+ GPU nodes** (NVIDIA)
- [NVIDIA GPU Operator](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/) installed
- GPU worker nodes labeled with `accelerator=nvidia` (see step 2 below)
- `kubectl` configured to access the cluster
- [uv](https://docs.astral.sh/uv/) package manager
- Python ≥ 3.13.7

## Quick Start

### 1. Clone & Setup Environment

```bash
git clone https://github.com/rubensolozabal/k8s_vLLM.git
cd k8s_vLLM

# Create virtual environment and install dependencies
uv sync
source .venv/bin/activate
```

### 2. Label GPU Nodes

The vLLM deployment uses a `nodeSelector` to schedule pods only on GPU-equipped nodes. You must label your GPU worker nodes **before** deploying:

```bash
# List nodes and identify GPU workers
kubectl get nodes -o wide

# Label each GPU node (replace with your actual node names)
kubectl label node <gpu-node-1> accelerator=nvidia
kubectl label node <gpu-node-2> accelerator=nvidia

# Verify labels
kubectl get nodes -l accelerator=nvidia
```

> **Important:** Without this label, vLLM pods will remain in `Pending` state indefinitely. The scheduler error will show: *"node(s) didn't match Pod's node affinity/selector"*.

### 3. Configure HuggingFace Token

vLLM needs a HuggingFace token to download gated models. Create the secret **before** deploying:

**Option A: Edit the YAML** — replace `YOUR_TOKEN_HERE` in `vllm.yaml` with your token, then apply normally.

**Option B: Create via CLI** (recommended — avoids committing secrets):

```bash
kubectl create namespace ai-stack
kubectl create secret generic hf-token -n ai-stack \
  --from-literal=HF_TOKEN=hf_<YOUR_TOKEN>
```

> Get a token at [huggingface.co/settings/tokens](https://huggingface.co/settings/tokens). The token needs **read** access to the model repository.

### 4. Deploy to Kubernetes

Deploy the stack in order — the namespace `ai-stack` is created automatically:

```bash
# Deploy vLLM inference servers (2 replicas with GPU)
# If you created the secret via CLI (Option B), the Secret in vllm.yaml will be skipped
kubectl apply -f vllm.yaml

# Deploy MLflow tracking server
kubectl apply -f mlflow.yaml

# Deploy Prometheus + Grafana monitoring
kubectl apply -f monitoring.yaml
```

### 5. Verify Deployment

```bash
# Check all pods are running
kubectl get pods -n ai-stack

# Expected output:
# NAME                          READY   STATUS    AGE
# grafana-xxx                   1/1     Running   ...
# mlflow-xxx                    1/1     Running   ...
# prometheus-xxx                1/1     Running   ...
# vllm-xxx-1                    1/1     Running   ...
# vllm-xxx-2                    1/1     Running   ...
```

> **Note:** vLLM pods take ~5-10 minutes to start (model download + loading). Monitor with:
> ```bash
> kubectl get pods -n ai-stack -w
> ```

### 6. Access Services

The services are deployed as `ClusterIP` by default. Choose an access method:

#### Option A: Port Forwarding (from bastion/local)

```bash
# Forward all services to localhost
kubectl port-forward -n ai-stack svc/grafana 3000:3000 --address 0.0.0.0 &
kubectl port-forward -n ai-stack svc/mlflow 5000:5000 --address 0.0.0.0 &
kubectl port-forward -n ai-stack svc/prometheus 9090:9090 --address 0.0.0.0 &
kubectl port-forward -n ai-stack svc/vllm 8000:80 --address 0.0.0.0 &
```

> ⚠️ Port-forward creates a single tunnel to one pod — **no load balancing**. Use NodePort for real distribution.

#### Option B: NodePort (recommended for multi-replica load balancing)

```bash
# Patch services to NodePort
kubectl patch svc vllm -n ai-stack -p '{"spec": {"type": "NodePort"}}'
kubectl patch svc mlflow -n ai-stack -p '{"spec": {"type": "NodePort"}}'
kubectl patch svc grafana -n ai-stack -p '{"spec": {"type": "NodePort"}}'
kubectl patch svc prometheus -n ai-stack -p '{"spec": {"type": "NodePort"}}'

# Get assigned ports
kubectl get svc -n ai-stack
```

Then access via `http://<node-ip>:<node-port>`.

### 7. Run the Client Notebook

Open `client.ipynb` in JupyterLab or VS Code and update the connection variables in cell 1:

```python
NODE_IP = "<your-node-ip>"
API_URL = f"http://{NODE_IP}:<vllm-nodeport>/v1"     # e.g., http://10.0.0.87:31583/v1
TRACKING_URL = f"http://localhost:5000"                # or http://<node-ip>:<mlflow-nodeport>
```

The notebook includes three usage patterns:
- **Conversation** — multi-turn chat with message history
- **Chat** — interactive chat loop with `input()`
- **Streaming** — token-by-token streaming output

## Monitoring

### Grafana Dashboard

Access Grafana (default login: `admin` / `admin`).

The pre-configured **MLExpert vLLM Dashboard** includes:

| Panel | Description |
|-------|-------------|
| Total Requests Completed | Request count over selected time range |
| Total Generated/Prompt Tokens | Token usage stats |
| End-to-End Latency | P50, P95, and average latency **per pod** |
| Time To First Token (TTFT) | TTFT P50, P95, and average **per pod** |
| Token Throughput | Prompt and generation tokens/sec **per pod** |
| Workload Distribution | Heatmaps of prompt and generation lengths |

Use the **Pod** dropdown at the top to filter by individual vLLM replica or view all.

### Prometheus Queries

Example queries:

```promql
# Requests per pod
sum by (pod) (rate(vllm:request_success_total[5m]))

# P95 latency per pod
histogram_quantile(0.95, sum by (le, pod) (rate(vllm:e2e_request_latency_seconds_bucket[5m])))

# Token throughput per pod
sum by (pod) (rate(vllm:generation_tokens_total[5m]))
```

### MLflow

The notebook uses `mlflow.langchain.autolog()` to automatically log:
- Input prompts and output responses
- Token usage (input, output, total)
- Execution time per trace

## vLLM Configuration

| Parameter | Value |
|-----------|-------|
| Model | `Qwen/Qwen3-4B-Instruct-2507` |
| Replicas | 2 |
| GPU per pod | 1x NVIDIA |
| GPU memory utilization | 90% |
| Max context length | 4096 tokens |
| Pod anti-affinity | One pod per node |

### Load Balancing

The Kubernetes Service distributes requests across vLLM replicas via **iptables** (random ~50/50 per TCP connection). Each `model.invoke()` call may hit a different pod. Both pods serve the same model — the client maintains conversation state locally.

## Project Structure

```
├── client.ipynb       # LangChain client notebook (conversation, chat, streaming)
├── vllm.yaml          # vLLM deployment (2 GPU replicas + Service + Ingress)
├── mlflow.yaml        # MLflow tracking server (SQLite + artifacts)
├── monitoring.yaml    # Prometheus (per-pod scraping) + Grafana (dashboard)
├── pyproject.toml     # Python dependencies
└── uv.lock            # Locked dependency versions
```