# Open Notebook — Secure Localhost Deployment (SSH-tunnel only) + Highest-Quality Local Models

This setup keeps **Open Notebook completely invisible** to the public internet and even your LAN by binding **all Open Notebook ports to `127.0.0.1`** on the server. You access it **only** through an SSH tunnel.

It also forces Open Notebook to use:
- **LLM:** `opennotebook-qwen3:32b-q4_K_M` (a custom Ollama model we create to bake in your exact “high quality” defaults + `/think`)
- **Embeddings:** `bge-m3:567m-fp16`
- **Audio:** disabled (no STT/TTS providers/models) to preserve VRAM and avoid surprise downloads/loads

> Why the custom LLM name?  
> Open Notebook will call *whatever* model name you configure. So we make a dedicated Ollama model whose defaults match your quality-first parameters and whose system behavior starts in **thinking mode** via `/think`.

---

## 0) Preconditions (do these once)

On the **server** (your RTX 5090 box):

- Docker + Docker Compose installed
- Ollama installed and running

---

## 1) Pull the exact Ollama models (be explicit)

Run on the **server**:

```bash
ollama pull qwen3:32b-q4_K_M
ollama pull bge-m3:567m-fp16
````

---

## 2) Create the dedicated “Open Notebook” Qwen3 model (bakes in `/think` + your parameters)

We create: **`opennotebook-qwen3:32b-q4_K_M`** (this is the model name you will select in the Open Notebook UI).

On the **server**:

```bash
mkdir -p ~/open-notebook/ollama_models
cd ~/open-notebook/ollama_models
```

Create `Modelfile.opennotebook-qwen3-32b-q4_K_M`:

```bash
cat > Modelfile.opennotebook-qwen3-32b-q4_K_M <<'EOF'
FROM qwen3:32b-q4_K_M

# ===== Your quality-first defaults =====
# Highest context window setting supported by Qwen3-32B before quality loss
PARAMETER num_ctx 32768

# IMake sure the LLM can talk for as long as it sees fit
PARAMETER num_predict -1

# Sampling: conservative / high-quality bias (your stated preferences)
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
- Prefer grounded answers based on the user's notebook sources when they are available.
"""
EOF
```

Now create the model:

```bash
ollama create opennotebook-qwen3:32b-q4_K_M -f Modelfile.opennotebook-qwen3-32b-q4_K_M
```

Sanity check:

```bash
ollama run opennotebook-qwen3:32b-q4_K_M "Say 'ready' and nothing else."
```

---

## 3) Create deployment directories

```bash
mkdir -p ~/open-notebook/notebook_data
mkdir -p ~/open-notebook/surreal_data
cd ~/open-notebook
```

---

## 4) Create `docker-compose.yml` (localhost-only ports)

Create/edit:

```bash
nano docker-compose.yml
```

Paste:

```yaml
services:
  open_notebook:
    image: lfnovo/open_notebook:v1-latest-single
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

## 5) Create `docker.env` (the important bits)

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

## 6) Start Open Notebook

```bash
docker compose up -d
```

Verify containers:

```bash
docker compose ps
```

---

## 7) Connect from your laptop/desktop via SSH tunnel (required)

Run this on your **client machine**:

```bash
ssh -N -L 8502:localhost:8502 -L 5055:localhost:5055 user@your-server-ip
```

Leave it running.

Now open (on your client machine):

```text
http://localhost:8502
```

---

## 8) Configure Open Notebook UI (this is the part that must be correct)

Open Notebook's UI has a **Models** page in the sidebar. Do this there.

### 8.1 Add / select the Ollama LLM everywhere

In **Models**:

* For **Chat model**:

  * Provider: `Ollama`
  * Model name: `opennotebook-qwen3:32b-q4_K_M`
  * Save, then set as Default

Repeat the same model for:

* **Tools**
* **Transformations**
* **Large Context**

Yes, all four should be the same model here. This avoids silent “fallback” behavior where one role ends up using a different model.

### 8.2 Set the embedding model

In **Models** → **Embeddings**:

* Provider: `Ollama`
* Model name: `bge-m3:567m-fp16`
* Save, then set as Default

### 8.3 Disable audio completely (VRAM + stability)

In **Models**:

* **Speech-to-Text**: do not configure a provider/model
* **Text-to-Speech**: do not configure a provider/model

This prevents Open Notebook from ever trying to download or load audio models.

---

## 9) Make embedding behavior deterministic (important for search quality)

When you add sources to a notebook, Open Notebook can embed immediately or skip embeddings.

For **search quality**, do this:

* Set your ingestion behavior to **always embed** (so every source becomes searchable semantically).
* If you previously embedded with a different embedding model: you *must* re-embed, otherwise your vector index is a mix of incompatible vector spaces.

### Re-embed after switching embedding models

Open the notebook and use its **Advanced** actions to trigger **Re-embed content** (Open Notebook added a dedicated re-embedding action specifically for this scenario).

If you can't find it quickly: the practical fallback is to remove and re-add sources (but the goal is to use the built-in re-embed action).

---

## 10) Verification checklist (do not skip)

### 10.1 Confirm Open Notebook is truly localhost-only on the server

On the **server**:

```bash
ss -ltnp | egrep '(:8502|:5055)'
```

You should see `127.0.0.1:8502` and `127.0.0.1:5055` (not `0.0.0.0`).

### 10.2 Confirm Open Notebook can reach Ollama

On the **server**:

```bash
curl -s http://127.0.0.1:5055/docs >/dev/null && echo "API docs reachable"
```

(And check Ollama is up from host perspective:)

```bash
curl -s http://127.0.0.1:11434/api/tags | head
```

### 10.3 Confirm the right models are actually used

On the **server**, watch what's loaded:

```bash
ollama ps
```

Then in the UI:

1. Add a Source → it should load/use `bge-m3:567m-fp16`
2. Chat → it should load/use `opennotebook-qwen3:32b-q4_K_M`

---

## What you *do not* need to change when switching to `bge-m3:567m-fp16`

If you are already using Ollama successfully:

* You do **not** need to change Open Notebook “context” settings just because you changed the embedding model.
* You **do** need to:

  1. Set the embedding model to `bge-m3:567m-fp16`
  2. Re-embed existing content so your index is consistent

That's it.

---

## Security status (what this deployment guarantees)

* Web UI/API: **only reachable via SSH tunnel**
* Data: stored on the server under `~/open-notebook/`
* LLM + embeddings: **100% local** via Ollama
* Audio: disabled (no surprise VRAM pressure)

If your goal is “maximum correctness over speed”, this is the cleanest way to make Open Notebook behave deterministically: one LLM everywhere, one embedding model everywhere, and a forced `/think` default baked into the LLM.
