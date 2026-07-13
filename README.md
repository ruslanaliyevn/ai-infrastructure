# AI Infrastructure

A fully self-hosted AI infrastructure stack running on local GPU hardware — covering LLM serving, observability, gateway management, and production inference with NVIDIA NIM.

This repository contains configuration files, deployment references, and stack documentation accompanying a series of technical articles on building production-grade AI infrastructure.

---

## Stack Overview

```
Consumer Applications (Open WebUI, Python scripts, agents)
                        │
                        ▼
              LiteLLM Gateway (:4000)
              ├── Token accounting
              ├── Rate limiting
              ├── Virtual API keys
              └── Langfuse tracing callback
                        │
          ┌─────────────┴─────────────┐
          ▼                           ▼
   NVIDIA NIM (:8000)          Ollama (:11434)
   Production inference        Experimentation
   fp8 / vLLM backend          llama.cpp backend
          │                           │
          └─────────────┬─────────────┘
                        ▼
                 GPU (CUDA)
                        │
                        ▼
              DCGM Exporter (:9400)
              GPU hardware metrics
                        │
                        ▼
            Prometheus (:9090) + Grafana (:3000)
            Unified observability dashboard
                        │
                        ▼
              Langfuse (:3031)
              LLM request tracing
              Token usage & cost accounting
```

---

## Components

| Component | Role | Port |
|---|---|---|
| NVIDIA NIM | Production LLM inference (vLLM, fp8) | 8000 |
| Ollama | Local model serving, experimentation | 11434 |
| LiteLLM | Unified OpenAI-compatible gateway | 4000 |
| Langfuse | LLM observability, tracing, cost accounting | 3031 |
| DCGM Exporter | GPU hardware metrics (Prometheus format) | 9400 |
| Prometheus | Metrics collection and storage | 9090 |
| Grafana | Visualization and dashboards | 3000 |

---

## Repository Structure

```
ai-infrastructure/
├── nim/
│   ├── README.md               # NIM deployment reference
│   └── docker-run-examples.sh  # Container run commands
├── litellm/
│   ├── docker-compose.yml
│   ├── config/
│   │   └── litellm_config.yaml
│   └── .env.example
├── langfuse/
│   ├── docker-compose.yml
│   └── .env.example
├── monitoring/
│   ├── docker-compose.yml
│   ├── prometheus/
│   │   └── prometheus.yml
│   └── grafana/
│       └── dashboards/
└── README.md
```

---

## Quick Start

### 1. Langfuse — LLM Observability Backend

```bash
git clone https://github.com/langfuse/langfuse.git
cd langfuse
```

Generate secrets and configure:

```bash
PG_PASS=$(openssl rand -base64 24 | tr -d '/+=')

cat > .env << EOF
NEXTAUTH_SECRET=$(openssl rand -base64 32)
NEXTAUTH_URL=http://<YOUR_SERVER_IP>:3031
SALT=$(openssl rand -base64 32)
ENCRYPTION_KEY=$(openssl rand -hex 32)
POSTGRES_PASSWORD=${PG_PASS}
DATABASE_URL=postgresql://postgres:${PG_PASS}@postgres:5432/postgres
CLICKHOUSE_PASSWORD=$(openssl rand -base64 24 | tr -d '/+=')
MINIO_ROOT_PASSWORD=$(openssl rand -base64 24 | tr -d '/+=')
REDIS_AUTH=$(openssl rand -base64 24 | tr -d '/+=')
EOF

docker compose up -d
```

### 2. LiteLLM — Unified Gateway

```bash
cd litellm
cp .env.example .env
# Fill in LANGFUSE_PUBLIC_KEY, LANGFUSE_SECRET_KEY, LITELLM_MASTER_KEY
docker compose up -d
```

### 3. NVIDIA NIM — Production Inference

```bash
export NGC_API_KEY=nvapi-xxxxxxxxxxxxxxxxxxxx

docker run -it --rm \
  --gpus all \
  --shm-size=16GB \
  -e NGC_API_KEY=$NGC_API_KEY \
  -e NIM_PASSTHROUGH_ARGS="--gpu-memory-utilization 0.5 --max-model-len 8192" \
  -v ~/.nim/cache:/opt/nim/.cache \
  -p 8000:8000 \
  nvcr.io/nim/meta/llama-3.1-8b-instruct:latest
```

### 4. Monitoring — DCGM + Prometheus + Grafana

```bash
cd monitoring
docker compose up -d
```

---

## LiteLLM Configuration

`litellm/config/litellm_config.yaml`:

```yaml
model_list:
  # NIM — production inference
  - model_name: llama-3.1-8b-nim
    litellm_params:
      model: openai/meta/llama-3.1-8b-instruct
      api_base: http://<SERVER_IP>:8000/v1
      api_key: dummy

  # Ollama — experimentation
  - model_name: qwen2.5-7b
    litellm_params:
      model: ollama_chat/qwen2.5:7b
      api_base: http://host.docker.internal:11434

  - model_name: llama3.1-8b
    litellm_params:
      model: ollama_chat/llama3.1:8b
      api_base: http://host.docker.internal:11434

litellm_settings:
  callbacks: ["langfuse_otel"]

general_settings:
  master_key: os.environ/LITELLM_MASTER_KEY
```

---

## Prometheus Scrape Configuration

`monitoring/prometheus/prometheus.yml`:

```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: "nim-llm"
    static_configs:
      - targets: ["<SERVER_IP>:8000"]
    metrics_path: "/metrics"

  - job_name: "dcgm"
    static_configs:
      - targets: ["<SERVER_IP>:9400"]

  - job_name: "litellm"
    static_configs:
      - targets: ["<SERVER_IP>:4000"]
    metrics_path: "/metrics"
```

---

## Monitoring — Key Metrics

### NIM / vLLM

| Metric | Description |
|---|---|
| `vllm:num_requests_running` | Active in-flight requests |
| `vllm:num_requests_waiting` | Requests queued for GPU capacity |
| `vllm:gpu_cache_usage_perc` | KV cache utilization (0.0 – 1.0) |
| `vllm:e2e_request_latency_seconds` | Full request latency distribution |
| `vllm:prompt_tokens_total` | Cumulative input tokens |
| `vllm:generation_tokens_total` | Cumulative output tokens |

### DCGM — GPU Hardware

| Metric | Description |
|---|---|
| `DCGM_FI_DEV_GPU_UTIL` | GPU compute utilization (%) |
| `DCGM_FI_DEV_MEM_COPY_UTIL` | Memory bandwidth utilization (%) |
| `DCGM_FI_DEV_FB_USED` | VRAM used (MiB) |
| `DCGM_FI_DEV_FB_FREE` | VRAM free (MiB) |
| `DCGM_FI_DEV_GPU_TEMP` | GPU temperature (°C) |
| `DCGM_FI_DEV_POWER_USAGE` | Power draw (W) |

---

## Observability Layer

Two complementary metric sources provide full inference visibility:

```
DCGM Exporter        →  GPU hardware level
                         VRAM, utilization, temperature, power

NIM /metrics         →  Inference engine level
                         requests, KV cache, latency, tokens

Both → Prometheus → Grafana dashboard
```

When request latency increases:
- `vllm:gpu_cache_usage_perc` at 1.0 → KV cache saturated, requests queuing
- `DCGM_FI_DEV_GPU_UTIL` at 100% → GPU compute bottleneck
- Both low → issue is in network or application layer

---

## Articles

| Article | Description |
|---|---|
| [AI Infrastructure Basics: GPU Memory and Why It All Matters](https://medium.com/@ruslanaliyevn0/ai-infrastructure-basics-gpu-memory-and-why-it-all-matters-7417f466c115) | GPU memory, quantization, compute fundamentals |
| [Building a Local AI Infrastructure](https://medium.com/@ruslanaliyevn0/building-a-local-ai-infrastructure-gpu-setup-nvidia-stack-and-monitoring-ffc6e65d9800) | GPU setup, NVIDIA stack, DCGM monitoring |
| [Building an LLM Observability Stack](https://medium.com/@ruslanaliyevn0/building-an-llm-observability-stack-with-ollama-litellm-langfuse-e1e80b3c26ec) | Ollama + LiteLLM + Langfuse deployment |
| [RAG: What It Is and How It Works](https://medium.com/@ruslanaliyevn0/rag-retrieval-augmented-generation-nədir-nə-üçün-i̇stifadə-olunur-və-necə-i̇şləyir-e2de8291426d) | RAG architecture and implementation |
| [NVIDIA NIM: The Infrastructure Layer](https://medium.com/@ruslanaliyevn0/nvidia-nim-the-infrastructure-layer-between-running-a-model-and-operating-an-inference-service-33c594e5c5be) | NIM architecture, model profiles, NIM 2.0 |
| [NVIDIA NIM in Practice](https://medium.com/@ruslanaliyevn0/deploying-nvidia-nim-a-complete-walkthrough-from-ngc-key-to-running-inference-4bafd66ea36c) | GPU memory, KV cache, deployment walkthrough |

---

## Environment Variables

### LiteLLM

```env
LANGFUSE_PUBLIC_KEY=pk-lf-...
LANGFUSE_SECRET_KEY=sk-lf-...
LANGFUSE_OTEL_HOST=http://<SERVER_IP>:3031
LITELLM_MASTER_KEY=sk-...
LITELLM_DB_PASSWORD=...
UI_USERNAME=admin
UI_PASSWORD=...
```

### NIM

```env
NGC_API_KEY=nvapi-...
NIM_PASSTHROUGH_ARGS=--gpu-memory-utilization 0.5 --max-model-len 8192
```

---

## Hardware Requirements

| Component | Minimum | Recommended |
|---|---|---|
| GPU VRAM | 8GB | 24GB+ |
| System RAM | 16GB | 32GB+ |
| Storage | 50GB | 200GB+ |
| CUDA | 11.8+ | 12.0+ |

---

*Built and maintained alongside production AI infrastructure work. All components are open-source and fully self-hosted — no data leaves the network.*
