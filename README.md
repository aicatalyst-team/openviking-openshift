# OpenViking on OpenShift

Deploy [OpenViking](https://github.com/volcengine/OpenViking) -- a Context Database for AI Agents -- on OpenShift with fully self-hosted model serving via OpenShift AI and vLLM.

## Architecture

```
┌─ OpenShift AI (openviking-models namespace) ───────────────────────────┐
│                                                                        │
│  ┌─────────────────────┐         ┌──────────────────────┐             │
│  │  vLLM Embedding     │         │  vLLM VLM            │             │
│  │  Qwen3-Embedding    │         │  Qwen3-32B           │             │
│  │  -0.6B              │         │                      │             │
│  │  /v1/embeddings     │         │  /v1/chat/completions │             │
│  └─────────────────────┘         └──────────────────────┘             │
│         A100 80GB GPU (shared)                                         │
└────────────────────────────────────────────────────────────────────────┘
                    ▲                           ▲
                    │ embedding                 │ L0/L1 generation
                    │                           │
┌─ OpenViking (openviking namespace) ───────────────────────────────────┐
│                                                                        │
│  ┌─────────────────────┐                                              │
│  │  OpenViking Server  │────────────── [Route / TLS]                  │
│  │    port 1933        │                                              │
│  │                     │                                              │
│  └──────────┬──────────┘                                              │
│        [PVC 50Gi]                                                      │
│     openviking-data                                                    │
└────────────────────────────────────────────────────────────────────────┘
```

### Components

| Component | Serving | Purpose |
|-----------|---------|---------|
| **OpenViking Server** | `quay.io/aicatalyst/openviking:latest` | Context database with REST API, port 1933 |
| **Qwen3-Embedding-0.6B** | vLLM via OpenShift AI (KServe) | Self-hosted embedding model, OpenAI-compatible `/v1/embeddings` |
| **Qwen3-32B** | vLLM via OpenShift AI (KServe) | Self-hosted VLM for L0/L1 semantic generation, OpenAI-compatible `/v1/chat/completions` |

### Why OpenShift AI?

OpenViking requires both an embedding model and a VLM for semantic generation. Instead of relying on external API calls, this deployment uses OpenShift AI to serve both models entirely on-cluster:

- **Managed model serving via KServe** -- declarative InferenceService resources handle lifecycle, scaling, and routing
- **GPU-optimized vLLM inference** -- high-throughput serving with continuous batching and PagedAttention
- **OpenAI-compatible APIs** -- both models expose standard `/v1/` endpoints, no client-side changes needed
- **Fully self-hosted** -- no external API calls required; all data stays within your cluster
- **Enterprise-grade with autoscaling** -- KServe can scale replicas based on request load, including scale-to-zero

## Prerequisites

- OpenShift 4.x cluster with the **OpenShift AI** operator installed (tested on 4.20 / ROKS)
- NVIDIA GPU node (A100 80GB recommended -- fits both models on a single device with GPU time-slicing, ~62 GB total VRAM: 5% GPU for embeddings + 85% for VLM)
- `oc` CLI authenticated with cluster access
- `podman` (optional, for building the image yourself)

No Anthropic API key or other external API credentials are needed. The entire stack is self-hosted.

## Deployment

### 1. Deploy OpenShift AI model serving

Apply the OpenShift AI manifests to create the model serving namespace, serving runtimes, and inference services:

```bash
oc apply -k manifests/openshift-ai/
```

This creates the `openviking-models` namespace with two vLLM-based InferenceServices. Ensure the model storage (e.g., PVC or S3 bucket) is configured for your environment and that the model weights are accessible.

Wait for the inference services to become ready:

```bash
oc wait --for=condition=Ready inferenceservice/qwen3-embedding -n openviking-models --timeout=600s
oc wait --for=condition=Ready inferenceservice/qwen3-32b -n openviking-models --timeout=600s
```

### 2. Edit the secret template

Before deploying OpenViking, edit `manifests/02-openviking-secret.yaml` and replace the placeholder values:

- `REPLACE_WITH_EMBEDDING_ROUTE` -- the route URL for the embedding InferenceService
- `REPLACE_WITH_VLM_ROUTE` -- the route URL for the VLM InferenceService
- `REPLACE_WITH_A_STRONG_ROOT_KEY` -- a strong root key for API access (e.g. `openssl rand -hex 16`)

You can retrieve the model routes with:

```bash
oc get inferenceservice qwen3-embedding -n openviking-models -o jsonpath='{.status.url}'
oc get inferenceservice qwen3-32b -n openviking-models -o jsonpath='{.status.url}'
```

### 3. Deploy OpenViking

```bash
oc apply -k manifests/
```

This creates the namespace, PVC, secret, deployment, service, and route in one command.

### 4. Wait for rollout

```bash
oc rollout status deployment/openviking -n openviking --timeout=180s
```

### 5. Verify

```bash
# Get route URL
OV_URL=$(oc get route openviking -n openviking -o jsonpath='https://{.spec.host}')
echo "OpenViking URL: ${OV_URL}"

# Health check (no auth required)
curl -sk "${OV_URL}/health"
# {"status":"ok","healthy":true,"version":"0.0.0"}
```

Retrieve your root API key for subsequent requests:

```bash
ROOT_KEY=$(oc get secret openviking-ov-conf -n openviking \
  -o jsonpath='{.data.ov\.conf}' | base64 -d | python3 -c \
  "import sys,json; print(json.load(sys.stdin)['server']['root_api_key'])")
echo "Root API Key: ${ROOT_KEY}"
```

## Usage

### Adding Resources

Add documents, URLs, or GitHub repos as context for your AI agents:

```bash
# Add a URL
curl -sk -X POST "${OV_URL}/api/v1/resources" \
  -H "Content-Type: application/json" \
  -H "X-API-Key: ${ROOT_KEY}" \
  -d '{"path": "https://raw.githubusercontent.com/volcengine/OpenViking/main/README.md"}'

# Add a GitHub repository
curl -sk -X POST "${OV_URL}/api/v1/resources" \
  -H "Content-Type: application/json" \
  -H "X-API-Key: ${ROOT_KEY}" \
  -d '{"path": "https://github.com/volcengine/OpenViking"}'
```

### Browsing the Virtual Filesystem

OpenViking organizes all context into a `viking://` virtual filesystem:

```bash
# List all resources
curl -sk "${OV_URL}/api/v1/fs/ls?uri=viking://resources/" \
  -H "X-API-Key: ${ROOT_KEY}"

# Tree view (depth 2)
curl -sk "${OV_URL}/api/v1/fs/tree?uri=viking://resources/&depth=2" \
  -H "X-API-Key: ${ROOT_KEY}"

# Read a specific file (L2 full content)
curl -sk "${OV_URL}/api/v1/content/read?uri=viking://resources/README/Overview.md" \
  -H "X-API-Key: ${ROOT_KEY}"

# Get L0 abstract (~100 tokens)
curl -sk "${OV_URL}/api/v1/content/abstract?uri=viking://resources/README" \
  -H "X-API-Key: ${ROOT_KEY}"

# Get L1 overview (~2K tokens)
curl -sk "${OV_URL}/api/v1/content/overview?uri=viking://resources/README" \
  -H "X-API-Key: ${ROOT_KEY}"
```

### Semantic Search

```bash
# Search across all resources
curl -sk -X POST "${OV_URL}/api/v1/search/find" \
  -H "Content-Type: application/json" \
  -H "X-API-Key: ${ROOT_KEY}" \
  -d '{"query": "how does authentication work"}'

# Grep within a specific path
curl -sk -X POST "${OV_URL}/api/v1/search/grep" \
  -H "Content-Type: application/json" \
  -H "X-API-Key: ${ROOT_KEY}" \
  -d '{"pattern": "config", "uri": "viking://resources/README"}'
```

### Session Management

Sessions let agents accumulate memories from conversations:

```bash
# Create a session
curl -sk -X POST "${OV_URL}/api/v1/sessions" \
  -H "Content-Type: application/json" \
  -H "X-API-Key: ${ROOT_KEY}" \
  -d '{"session_id": "my-session"}'

# Add a message
curl -sk -X POST "${OV_URL}/api/v1/sessions/my-session/messages" \
  -H "Content-Type: application/json" \
  -H "X-API-Key: ${ROOT_KEY}" \
  -d '{"role": "user", "content": "I prefer Python over JavaScript"}'

# Commit session (extracts long-term memories)
curl -sk -X POST "${OV_URL}/api/v1/sessions/my-session/commit" \
  -H "Content-Type: application/json" \
  -H "X-API-Key: ${ROOT_KEY}"
```

### Python SDK

```bash
pip install openviking
```

```python
import openviking as ov

client = ov.SyncHTTPClient(
    url="https://YOUR-ROUTE",
    api_key="YOUR-ROOT-KEY"
)
client.initialize()

# Add a resource
client.add_resource("https://github.com/your-org/your-repo")
client.wait_processed()

# Search
results = client.find("how does config work")
for r in results.resources:
    print(f"  {r.uri} (score: {r.score:.4f})")

client.close()
```

### CLI

```bash
pip install openviking

# Configure CLI
mkdir -p ~/.openviking
cat > ~/.openviking/ovcli.conf <<EOF
{
  "url": "${OV_URL}",
  "api_key": "${ROOT_KEY}"
}
EOF

# Use
ov status
ov add-resource https://github.com/your-org/your-repo
ov ls viking://resources/
ov find "how does config work"
```

## Building the Image

To build and push the OpenViking image to your own registry using Podman:

```bash
cd /path/to/OpenViking

# Build for linux/amd64 (required for x86_64 OpenShift clusters)
podman build --platform linux/amd64 \
  -t quay.io/aicatalyst/openviking:latest \
  -t quay.io/aicatalyst/openviking:$(git rev-parse --short HEAD) \
  -f Dockerfile .

# Push
podman login quay.io
podman push quay.io/aicatalyst/openviking:latest
podman push quay.io/aicatalyst/openviking:$(git rev-parse --short HEAD)
```

> **Note:** Building on Apple Silicon (ARM64) for amd64 uses QEMU emulation.
> The build compiles Go, Rust, and C++ from source -- expect 30-60+ minutes.

## OpenShift-Specific Notes

### Security Context Constraints (SCC)

All manifests comply with the `restricted-v2` SCC (OpenShift default):

- `runAsNonRoot: true`
- `allowPrivilegeEscalation: false`
- `capabilities.drop: ["ALL"]`
- `seccompProfile.type: RuntimeDefault`
- No `fsGroup: 0` (not permitted by restricted SCC)
- `HOME` and temp dirs redirected to writable `emptyDir` volumes

### Storage

| PVC | Size | Purpose |
|-----|------|---------|
| `openviking-data` | 50Gi | Vector index + AGFS file storage |

Default StorageClass is `gp3-csi` (AWS EBS). Change in PVC manifests for other providers.

### Stale Lock Files

If OpenViking fails to start with `lock ... LOCK: Resource temporarily unavailable`, a previous pod left a stale lock:

```bash
oc exec deployment/openviking -n openviking -- rm -f /app/data/vectordb/context/store/LOCK
oc delete pod -l app=openviking -n openviking
```

### VLM Providers

The default config uses Qwen3-32B served via vLLM on OpenShift AI, accessed through the OpenAI-compatible `/v1/chat/completions` endpoint. To use a different provider, update the `vlm` section in `02-openviking-secret.yaml`:

| Provider | Model Example | Config `provider` |
|----------|---------------|-------------------|
| vLLM (OpenShift AI) | `Qwen3-32B` | `openai` (compatible API) |
| OpenAI | `gpt-4o` | `openai` |
| Anthropic (via LiteLLM) | `claude-sonnet-4-6` | `litellm` |
| Volcengine | `doubao-seed-2-0-pro-260215` | `volcengine` |
| DeepSeek (via LiteLLM) | `deepseek-chat` | `litellm` |

### Embedding Providers

The default uses Qwen3-Embedding-0.6B served via vLLM on OpenShift AI, accessed through the OpenAI-compatible `/v1/embeddings` endpoint. To use a different provider, update the `embedding` section in `02-openviking-secret.yaml`:

| Provider | Model | Config `provider` |
|----------|-------|-------------------|
| vLLM (OpenShift AI) | `Qwen3-Embedding-0.6B` | `openai` (compatible API) |
| Ollama (self-hosted) | `nomic-embed-text` | `openai` (compatible API) |
| OpenAI | `text-embedding-3-large` | `openai` |
| Jina | `jina-embeddings-v5-text-small` | `jina` |
| Voyage | `voyage-3.5-lite` | `voyage` |

## API Quick Reference

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| GET | `/health` | No | Liveness check |
| GET | `/ready` | No | Readiness check |
| POST | `/api/v1/resources` | Yes | Add resource (URL, file, repo) |
| GET | `/api/v1/fs/ls?uri=` | Yes | List directory |
| GET | `/api/v1/fs/tree?uri=&depth=` | Yes | Tree view |
| GET | `/api/v1/content/read?uri=` | Yes | Read file content (L2) |
| GET | `/api/v1/content/abstract?uri=` | Yes | Get L0 abstract |
| GET | `/api/v1/content/overview?uri=` | Yes | Get L1 overview |
| POST | `/api/v1/search/find` | Yes | Semantic search |
| POST | `/api/v1/search/grep` | Yes | Pattern search |
| POST | `/api/v1/sessions` | Yes | Create session |
| POST | `/api/v1/sessions/{id}/messages` | Yes | Add message |
| POST | `/api/v1/sessions/{id}/commit` | Yes | Commit session |
| GET | `/api/v1/observer/system` | Yes | System status |

Auth is via `X-API-Key` header or `Authorization: Bearer` header.

## Manifest Files

```
manifests/
  kustomization.yaml            # Kustomize config (oc apply -k manifests/)
  01-namespace.yaml             # Namespace
  02-openviking-secret.yaml     # Config template (edit before deploying)
  03-openviking-pvc.yaml        # OpenViking persistent storage (50Gi)
  04-openviking-deployment.yaml # OpenViking server deployment
  05-openviking-service.yaml    # OpenViking ClusterIP service
  06-openviking-route.yaml      # OpenShift Route (TLS edge)
  openshift-ai/
    kustomization.yaml              # Kustomize config for model serving
    01-namespace.yaml               # Model serving namespace
    02-serving-runtime-embedding.yaml # vLLM ServingRuntime for embeddings
    03-serving-runtime-vlm.yaml     # vLLM ServingRuntime for VLM
    04-inference-service-embedding.yaml # Qwen3-Embedding-0.6B InferenceService
    05-inference-service-vlm.yaml   # Qwen3-32B InferenceService
```

## Teardown

```bash
oc delete -k manifests/
oc delete -k manifests/openshift-ai/
```

## License

Apache License 2.0
