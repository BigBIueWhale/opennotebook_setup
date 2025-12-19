This is the **Secure Localhost Deployment**. We will bind all ports to `127.0.0.1` so the application is completely invisible to the public internet (and even your LAN). You will access it via an SSH Tunnel.

We will also configure it to use **BGE-M3** and **Qwen3** exclusively, with no audio services enabled to preserve VRAM.

### Prerequisite: Prepare Models

On your RTX 5090 server, ensure the specific models are pulled:

```bash
ollama pull qwen3:32b
ollama pull bge-m3:567m-fp16

```

---

### 1. Create Deployment Directory

```bash
mkdir -p ~/open-notebook/notebook_data
mkdir -p ~/open-notebook/surreal_data
cd ~/open-notebook

```

### 2. Create Secure `docker-compose.yml`

This configuration binds ports strictly to **localhost**.

**Create the file:**

```bash
nano docker-compose.yml

```

**Paste this content:**

```yaml
services:
  open_notebook:
    image: lfnovo/open_notebook:v1-latest-single
    ports:
      # Binds strictly to localhost. NOT accessible from LAN or Internet.
      - "127.0.0.1:8502:8502"  # Frontend
      - "127.0.0.1:5055:5055"  # API
    environment:
      # --- NETWORK CONFIGURATION ---
      # Since we are tunneling, your browser will see this at localhost:5055
      - API_URL=http://localhost:5055
      
      # --- OLLAMA CONNECTION ---
      # Connects to your existing Ollama user service via Docker bridge
      - OLLAMA_API_BASE=http://172.17.0.1:11434
      
      # --- DUMMY KEYS ---
      - OPENAI_API_KEY=sk-dummy-key-for-validation
      
      # --- SYSTEM CONFIG ---
      # Generous timeouts for local model swapping
      - API_CLIENT_TIMEOUT=1200
      - ESPERANTO_LLM_TIMEOUT=600
      
      # --- DATABASE ---
      - SURREAL_URL=ws://localhost:8000/rpc
      - SURREAL_USER=root
      - SURREAL_PASSWORD=root
      - SURREAL_NAMESPACE=open_notebook
      - SURREAL_DATABASE=production

    volumes:
      - ./notebook_data:/app/data
      - ./surreal_data:/mydata
    restart: always

```

### 3. Start the Server

```bash
docker compose up -d

```

---

### 4. Connect via SSH Tunnel

Since the ports are blocked from the outside, you must create a tunnel from your **client computer** (laptop/desktop) to access the server.

Run this on your **local machine** (not the server):

```bash
# Replace 'user' and 'your-server-ip' with your actual details
ssh -L 8502:localhost:8502 -L 5055:localhost:5055 user@your-server-ip

```

*Leave this terminal window open.* This maps your local ports 8502 and 5055 securely to the server's localhost ports.

---

### 5. Configure Open Notebook

Now open your browser on your local machine and go to: **[http://localhost:8502](https://www.google.com/search?q=http://localhost:8502)**

Navigate to **⚙️ Settings** > **🤖 Models** and configure as follows:

#### A. Language Models (Qwen3)

1. **Provider**: `Ollama`
2. **Model Name**: `qwen3:32b`
3. Click **Save**.
4. Set as **Default** for: `Chat`, `Tools`, `Transformations`, and `Large Context`.

#### B. Embedding Models (BGE-M3)

1. **Provider**: `Ollama`
2. **Model Name**: `bge-m3:567m-fp16`
3. Click **Save**.
4. Set as **Default**.

#### C. Audio Services (Disable)

1. Check the **Text-to-Speech** and **Speech-to-Text** sections.
2. Ensure **NO models** are selected as default.
3. Do not add any providers here. This ensures Open Notebook never attempts to load audio models into your VRAM.

---

### 6. Verify Behavior

1. **Add a Source**: When you add a file/link, Open Notebook calls the API. Your Ollama logs (`journalctl --user -u ollama -f`) should show `bge-m3` loading to embed the content.
2. **Chat**: When you ask a question, Ollama will unload `bge-m3` and load `qwen3:32b`.

**Security Status:**

* **Web UI/API**: Only accessible via SSH Tunnel.
* **Data**: Stored locally on the server.
* **AI Processing**: 100% local via Ollama.
* **Audio**: Disabled to prioritize text quality and VRAM.
