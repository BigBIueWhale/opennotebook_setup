# Open Notebook - Secure Localhost Deployment (SSH-tunnel only) + Highest-Quality Local Models

This setup keeps Open Notebook completely invisible to the public internet and even your LAN by binding all Open Notebook ports to 127.0.0.1 on the server. You access it only through an SSH tunnel.

This setup is opinionated for:
- Ubuntu 24.04 LTS
- RTX 5090 (32 GB VRAM)
- Ollama running on the host, Open Notebook running in Docker
- Maximum correctness over speed

**Version policy (no “latest”, no drift):**
- Open Notebook Docker image: **lfnovo/open_notebook:1.2.4-single**
  - This is the exact version this README is written for.
  - To upgrade in the future, you intentionally change this tag everywhere it appears.

Model policy (simple and strict):
- Chat model: qwen3 (32K context) for day-to-day chat
- All other LLM roles: nemotron-3-nano (220K context) so Open Notebook can stuff huge context without silently truncating
- Embeddings: bge-m3 fp16
- Audio: disabled (no STT/TTS) to avoid surprise downloads and VRAM pressure

Why custom model names?
Open Notebook will call whatever model name you configure. So we create dedicated Ollama models with the exact context limits (and behavior) baked in.

---

## 0) Preconditions (do these once)

On the server (your RTX 5090 box):

- Docker + Docker Compose installed
- Ollama installed and running
- Ollama reachable from containers (host must be reachable from the Docker bridge; commonly `172.17.0.1:11434` on Linux)

---

## 1) Pull the exact Ollama models (be explicit)

Run on the server:

```bash
ollama pull qwen3:32b-q4_K_M
ollama pull nemotron-3-nano:30b-a3b-q4_K_M
ollama pull bge-m3:567m-fp16
```

---

## 2) Create the dedicated Open Notebook Qwen3 chat model (32K, quality defaults, starts in thinking mode)

We create: qwen3:32b-q4_K_M-opennotebook
(This is the model name you will select in the Open Notebook UI for Chat.)

On the server:

```bash
mkdir -p ~/open-notebook/ollama_models
cd ~/open-notebook/ollama_models
```

Create Modelfile.qwen3-32b-q4_K_M-opennotebook:

```bash
cat > Modelfile.qwen3-32b-q4_K_M-opennotebook <<'EOF'
FROM qwen3:32b-q4_K_M

# ===== Quality-first defaults =====
PARAMETER num_ctx 32768
PARAMETER num_predict -1

PARAMETER temperature 0.6
PARAMETER top_p 0.95
PARAMETER top_k 20
PARAMETER min_p 0.0
PARAMETER repeat_penalty 1.0

# ===== System behavior =====
SYSTEM """
/think You are the assistant inside Open Notebook.

Rules:
- Be precise and conservative. If you're not sure, say you're not sure.
- Prefer grounded answers based on the notebook sources when they are available.
"""
EOF
```

Create the model:

```bash
ollama create qwen3:32b-q4_K_M-opennotebook -f Modelfile.qwen3-32b-q4_K_M-opennotebook
```

Sanity check:

```bash
ollama run qwen3:32b-q4_K_M-opennotebook "Say 'ready' and nothing else."
```

---

## 3) Create the dedicated Open Notebook Nemotron long-context model (220K context)

We create: nemotron-3-nano:30b-a3b-q4_K_M-opennotebook-220k
(This is the model name you will select in the Open Notebook UI for: Tools, Transformations, Large Context.)

On the server:

```bash
cd ~/open-notebook/ollama_models
```

Create Modelfile.nemotron-3-nano-30b-a3b-q4_K_M-opennotebook-220k:

```bash
cat > Modelfile.nemotron-3-nano-30b-a3b-q4_K_M-opennotebook-220k <<'EOF'
FROM nemotron-3-nano:30b-a3b-q4_K_M

# The only special requirement: exactly 220K context (fits 32 GB VRAM in this setup)
PARAMETER num_ctx 220000

# Let the model finish (avoid arbitrary cutoffs)
PARAMETER num_predict -1

SYSTEM """
You are the assistant inside Open Notebook.

Rules:
- Be precise and conservative. If you're not sure, say you're not sure.
- Prefer grounded answers based on the notebook sources when they are available.
"""
EOF
```

Create the model:

```bash
ollama create nemotron-3-nano:30b-a3b-q4_K_M-opennotebook-220k -f Modelfile.nemotron-3-nano-30b-a3b-q4_K_M-opennotebook-220k
```

Sanity check:

```bash
ollama run nemotron-3-nano:30b-a3b-q4_K_M-opennotebook-220k "Say 'ready' and nothing else."
```

Optional verification (confirm num_ctx):

```bash
ollama show nemotron-3-nano:30b-a3b-q4_K_M-opennotebook-220k | sed -n '1,120p'
```

---

## 4) Create deployment directories

```bash
mkdir -p ~/open-notebook/notebook_data
mkdir -p ~/open-notebook/surreal_data
cd ~/open-notebook
```

---

## 5) Pull the exact Open Notebook Docker image (pinned)

Run on the server:

```bash
docker pull lfnovo/open_notebook:1.2.4-single
```

---

## 6) Create docker-compose.yml (localhost-only ports, pinned image)

Create/edit:

```bash
nano docker-compose.yml
```

Paste:

```yaml
services:
  open_notebook:
    image: lfnovo/open_notebook:1.2.4-single
    restart: always

    # Strict localhost binding: not reachable from LAN / public internet
    ports:
      - "127.0.0.1:8502:8502"  # Web UI
      - "127.0.0.1:5055:5055"  # API

    env_file:
      - docker.env

    volumes:
      - ./notebook_data:/app/data
      - ./surreal_data:/mydata
```

---

## 7) Create docker.env (the important bits)

Create/edit:

```bash
nano docker.env
```

Paste:

```bash
# =========================
# Networking (tunnel-friendly)
# =========================
# Your browser will access the API through the SSH tunnel as localhost:5055
API_URL=http://localhost:5055

# Keep internal default explicit (helps avoid surprises)
INTERNAL_API_URL=http://localhost:5055

# =========================
# Ollama (host -> container)
# =========================
# Open Notebook is running in Docker; Ollama is on the host.
# On typical Linux Docker setups, the host bridge IP is 172.17.0.1.
OLLAMA_API_BASE=http://172.17.0.1:11434

# =========================
# Dummy key (some apps validate presence even if unused)
# =========================
OPENAI_API_KEY=sk-dummy-key-for-validation

# =========================
# Timeouts (local models can take time)
# =========================
API_CLIENT_TIMEOUT=1200
ESPERANTO_LLM_TIMEOUT=600

# =========================
# SurrealDB (single-container image runs it locally)
# =========================
SURREAL_URL=ws://localhost:8000/rpc
SURREAL_USER=root
SURREAL_PASSWORD=root
SURREAL_NAMESPACE=open_notebook
SURREAL_DATABASE=production
```

---

## 8) Start Open Notebook

```bash
docker compose up -d
```

Verify containers:

```bash
docker compose ps
```

---

## 9) Connect from your laptop/desktop via SSH tunnel (required)

Run this on your client machine:

```bash
ssh -N -L 8502:localhost:8502 -L 5055:localhost:5055 user@your-server-ip
```

Leave it running.

Now open (on your client machine):

```text
http://localhost:8502
```

---

## 10) Configure Open Notebook UI (this is the part that must be correct)

Open Notebook has a Models page in the sidebar. Configure these model slots exactly.

### 10.1 Chat model (Qwen3 32K)

Models -> Chat model:

* Provider: Ollama
* Model name: qwen3:32b-q4_K_M-opennotebook
* Save, then set as Default

This keeps normal chat fast and high quality, but it is still 32K context.

### 10.2 Everything else (Nemotron 220K)

Models -> Tools:

* Provider: Ollama
* Model name: nemotron-3-nano:30b-a3b-q4_K_M-opennotebook-220k
* Save, then set as Default

Models -> Transformations:

* Provider: Ollama
* Model name: nemotron-3-nano:30b-a3b-q4_K_M-opennotebook-220k
* Save, then set as Default

Models -> Large Context:

* Provider: Ollama
* Model name: nemotron-3-nano:30b-a3b-q4_K_M-opennotebook-220k
* Save, then set as Default

This is the core fix for the "Open Notebook exceeded my model context and chopped the beginning" problem:

* Any role that tends to ingest lots of text uses the 220K model.

### 10.3 Embeddings (bge-m3 fp16)

Models -> Embeddings:

* Provider: Ollama
* Model name: bge-m3:567m-fp16
* Save, then set as Default

### 10.4 Disable audio completely (VRAM + stability)

Models:

* Speech-to-Text: do not configure a provider/model
* Text-to-Speech: do not configure a provider/model

This prevents Open Notebook from trying to download or load audio models.

---

## 11) Embeddings: chunk size and what to do for bge-m3

Open Notebook embeddings work like this:

* When you enable embeddings for a source, Open Notebook chunks the extracted text into chunks of about 1000 words and embeds each chunk.
* Those embeddings power semantic search and retrieval for the Ask/research flows.

For bge-m3 (which supports long inputs), the default 1000-word chunks are already "large" compared to typical RAG chunking. Do not try to make chunks smaller unless you have a specific reason (smaller chunks usually increase recall but often hurt precision and create noisy retrieval).

Practical guidance for best results with the fixed chunking:

* Always embed (so every source becomes searchable).
* Keep source text clean (headings and paragraph breaks matter). If a PDF extracts with garbage line breaks, fix extraction (or re-upload a cleaner version) before embedding.
* After switching embedding models, re-embed everything so the vector index is not a mix of incompatible embedding spaces.

Re-embed after switching embedding models:

* Open the notebook and use its Advanced actions to trigger Re-embed content.
* Fallback: remove and re-add sources (slower, but works).

---

## 12) Verification checklist (do not skip)

### 12.1 Confirm Open Notebook is truly localhost-only on the server

On the server:

```bash
ss -ltnp | egrep '(:8502|:5055)'
```

You should see 127.0.0.1:8502 and 127.0.0.1:5055 (not 0.0.0.0).

### 12.2 Confirm Open Notebook API is up

On the server:

```bash
curl -s http://127.0.0.1:5055/docs >/dev/null && echo "API docs reachable"
```

### 12.3 Confirm Ollama is up from host perspective

On the server:

```bash
curl -s http://127.0.0.1:11434/api/tags | head
```

### 12.4 Confirm the right models are actually used

On the server, watch what is loaded:

```bash
ollama ps
```

Then in the UI:

1. Add a Source with embeddings enabled -> should load/use `bge-m3:567m-fp16`
2. Normal chat -> should load/use `qwen3:32b-q4_K_M-opennotebook`
3. Run a Transformation or anything tool-heavy / long-context -> should load/use `nemotron-3-nano:30b-a3b-q4_K_M-opennotebook-220k`

---

## 13) Security status (what this deployment guarantees)

* Web UI/API: only reachable via SSH tunnel
* Data: stored on the server under `~/open-notebook/`
* LLM + embeddings: 100% local via Ollama
* Audio: disabled (no surprise VRAM pressure)

This setup is intentionally simple:

* One small-ish chat model (32K) for normal use
* One massive-context model (220K) for everything that can blow up context
* One embedding model everywhere
* No audio stack
