# OpenViking on OpenShift

Deploy [OpenViking](https://github.com/volcengine/OpenViking) — a Context Database for AI Agents — on OpenShift.

## Architecture

```
┌─ OpenShift Namespace: openviking ─────────────────────────────────────┐
│                                                                        │
│  ┌──────────────┐                    ┌─────────────────────┐          │
│  │   Ollama     │◄── embedding ────► │  OpenViking Server  │          │
│  │  (internal)  │   port 11434       │    port 1933        │          │
│  │              │                    │                     │          │
│  │ nomic-embed  │                    │  VLM: Claude Sonnet │          │
│  │   -text      │                    │  4.6 (Anthropic API)│          │
│  └──────┬───────┘                    └──────────┬──────────┘          │
│         │                                       │                     │
│    [PVC 10Gi]                              [PVC 50Gi]                 │
│    ollama-data                          openviking-data                │
│                                                 │                     │
│                                          [Route / TLS]                │
│                                   https://openviking-openviking...    │
└────────────────────────────────────────────────────────────────────────┘
                                          │
                              External Anthropic API
                          (for L0/L1 semantic generation)
```

### Components

| Component | Image | Purpose |
|-----------|-------|---------|
| **OpenViking Server** | `quay.io/aicatalyst/openviking:latest` | Context database with REST API, port 1933 |
| **Ollama** | `docker.io/ollama/ollama:latest` | Self-hosted embedding model (`nomic-embed-text`, 768d), port 11434 |

### Why Ollama?

OpenViking requires an embedding model for vector search. Instead of paying for a cloud embedding API, we self-host an open-source model (nomic-embed-text) using Ollama on OpenShift. The VLM (Sonnet 4.6) is the only external API call.

## Prerequisites

- OpenShift 4.x cluster (tested on 4.20 / ROSA)
- `oc` CLI authenticated with cluster access
- An **Anthropic API key** for Claude Sonnet 4.6 (VLM)
- `podman` (optional, for building the image yourself)

## Deployment

### 1. Edit the secret template

Before deploying, edit `manifests/05-openviking-secret.yaml` and replace the placeholder values:

- `REPLACE_WITH_YOUR_ANTHROPIC_API_KEY` — your Anthropic API key
- `REPLACE_WITH_A_STRONG_ROOT_KEY` — a strong root key for API access (e.g. `openssl rand -hex 16`)

### 2. Deploy everything with Kustomize

```bash
oc apply -k manifests/
```

This creates the namespace, PVCs, secrets, deployments, services, and route in one command.

### 3. Wait for rollout

```bash
oc rollout status deployment/ollama -n openviking --timeout=180s
oc rollout status deployment/openviking -n openviking --timeout=180s
```

### 4. Pull the embedding model

```bash
oc exec deployment/ollama -n openviking -- ollama pull nomic-embed-text
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

### MCP Integration

Use OpenViking as an MCP server with Claude Code, Cursor, or any MCP-compatible client:

```bash
# Claude Code
claude mcp add openviking --transport sse "${OV_URL}/mcp"
```

For Claude Desktop, add to `claude_desktop_config.json`:

```json
{
  "mcpServers": {
    "openviking": {
      "url": "https://YOUR-ROUTE/mcp"
    }
  }
}
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
  "url": "https://YOUR-ROUTE",
  "api_key": "YOUR-ROOT-KEY"
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
> The build compiles Go, Rust, and C++ from source — expect 30-60+ minutes.

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
| `ollama-data` | 10Gi | Embedding model weights |

Default StorageClass is `gp3-csi` (AWS EBS). Change in PVC manifests for other providers.

### Stale Lock Files

If OpenViking fails to start with `lock ... LOCK: Resource temporarily unavailable`, a previous pod left a stale lock:

```bash
oc exec deployment/openviking -n openviking -- rm -f /app/data/vectordb/context/store/LOCK
oc delete pod -l app=openviking -n openviking
```

### VLM Providers

The default config uses Claude Sonnet 4.6 via LiteLLM. To use a different provider, update the `vlm` section in `05-openviking-secret.yaml`:

| Provider | Model Example | Config `provider` |
|----------|---------------|-------------------|
| OpenAI | `gpt-4o` | `openai` |
| Anthropic (via LiteLLM) | `claude-sonnet-4-6` | `litellm` |
| Volcengine | `doubao-seed-2-0-pro-260215` | `volcengine` |
| DeepSeek (via LiteLLM) | `deepseek-chat` | `litellm` |
| Ollama local (via LiteLLM) | `ollama/llama3.1` | `litellm` |

### Embedding Providers

The default uses self-hosted Ollama. Alternatives:

| Provider | Model | Config `provider` |
|----------|-------|-------------------|
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
  02-ollama-pvc.yaml            # Ollama persistent storage (10Gi)
  03-ollama-deployment.yaml     # Ollama deployment (embedding model)
  04-ollama-service.yaml        # Ollama ClusterIP service
  05-openviking-secret.yaml     # Config template (edit before deploying)
  06-openviking-pvc.yaml        # OpenViking persistent storage (50Gi)
  07-openviking-deployment.yaml # OpenViking server deployment
  08-openviking-service.yaml    # OpenViking ClusterIP service
  09-openviking-route.yaml      # OpenShift Route (TLS edge)
```

## Teardown

```bash
oc delete -k manifests/
```

## License

Apache License 2.0
