
Complete Start-to-Finish RAG Assistant Setup Guide

This end-to-end guide deploys a production-ready, self-hosted RAG Microservice on a single machine or server. It features Ollama with GPU pass-through, automatic model initialization, hot-reloading markdown documentation, X-API-Key authentication, and SlowAPI rate limiting (15 requests/minute per client IP).

Phase 1: Host System Preparation

1. Install Docker & Docker Compose (version 2.20+).
2. Install NVIDIA Container Toolkit (Linux / Ubuntu / WSL2):bash
curl -fsSL https://github.io | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg
curl -s -L https://github.io | \
  sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
  sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list
sudo apt-get update && sudo apt-get install -y nvidia-container-toolkit
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker
Use code with caution.
3. Verify Host GPU Driver (driver 535+ required): [1] (/goto?url=CAESowEB6zswFS-A-JVbJzgDsMaWlUiNPEyA4KNbHUlbrzGMVjvoKsHCAJroh9DGs6_pD97KbZ4AC_ENqBIO1nNW3_8zgkwCtfTGRRAg7TV5-P1oCPV6g6xRDrXjmnK_gG35WZ9L22uZJhSXLuy6-foFJR2f1wt9ANfPPuXJ-6IyZH2zYrpqbBc_dj9xZUBicJ1R51RCo8tLnzXARfhaJBSdMDq89fVf), [2] (/goto?url=CAESYgHrOzAVnXS77TykjLeFzZNNQKWXyPjrrk5ygyEMcqdE-h51ld7cdJcg_umGh287y4s4RprXRdNpKM6NCzKt5i2vI-XYeIs_Mx7NoIaqlpZvBnLEPVrX8OtfVpZ6rniwpSAu), [3] (/goto?url=CAESgwEB6zswFfoiMpTWHX9vah6qGrUNn2RVDk0i64uHGLrK-2eQIFuTozo8e3X6p6zc_dPJTYqo75wBasBil5wMpvnmnbR6K4A86Ko6PbJQ0cvxMJuzFXzKxFIBA8m9PnHkIICeUX3gTC9FUpwFrMLst6zg4e-N3W5sOfajHi6AjxKYZroSCQ)
bash
nvidia-smi
Use code with caution.

Phase 2: Project Layout & Files

Create the exact directory tree on your host machine:
text
rag-stack/
├── docs/
│   ├── products/
│   │   └── wireless_earbuds.md
│   └── support/
│       ├── password_reset.md
│       └── user_guidelines.md
├── backend/
│   ├── Dockerfile
│   ├── main.py
│   └── watcher.py
├── scripts/
│   └── pull-models.sh
├── .env
├── docker-compose.yml
└── README.md
Use code with caution.

1. docs/products/wireless_earbuds.md

markdown
# Product: SonicBuds Pro
- SKU: SB-PRO-500
- Price: $79.99
- In Stock: Yes
- Battery Life: 8 hours (32 hours with charging case)
- Water Resistance: IPX5 (sweat and splash resistant, not waterproof)
- FAQ: Can I swim with them? No.
Use code with caution.

2. docs/support/password_reset.md

markdown
# Topic: How to Reset Password
- Steps: Go to login screen -> Click "Forgot Password" -> Enter registered email -> Open 30-minute token link.
- Troubleshooting: Check spam folder or email support@domain.com if token expires.
Use code with caution.

3. scripts/pull-models.sh

bash
#!/usr/bin/env bash
set -e

OLLAMA_HOST="${OLLAMA_BASE_URL:-http://localhost:11434}"
echo "Waiting for Ollama service at ${OLLAMA_HOST}..."

until curl -sf "${OLLAMA_HOST}/api/tags" > /dev/null; do
  sleep 2
done

echo "Ollama is live. Pulling required models..."
curl -s -X POST "${OLLAMA_HOST}/api/pull" -d '{"name": "llama3.2:3b"}'
curl -s -X POST "${OLLAMA_HOST}/api/pull" -d '{"name": "nomic-embed-text"}'

echo "All models ready."
Use code with caution.
Make executable on host: chmod +x scripts/pull-models.sh

4. requirements.txt

text
fastapi>=0.115.0
uvicorn>=0.30.0
pydantic>=2.8.0
slowapi>=0.1.9
langchain>=0.3.0
langchain-chroma>=0.1.4
langchain-ollama>=0.2.0
langchain-community>=0.3.0
watchdog>=4.0.0
httpx>=0.27.0
Use code with caution.

5. backend/watcher.py

python
import logging
import time
from pathlib import Path
from langchain_chroma import Chroma
from langchain_community.document_loaders import TextLoader
from langchain_ollama import OllamaEmbeddings
from langchain_text_splitters import RecursiveCharacterTextSplitter
from watchdog.events import FileSystemEventHandler
from watchdog.observers import Observer

logger = logging.getLogger("rag-watcher")


def index_single_file(
    file_path: str, vectorstore: Chroma, ollama_base_url: str
):
  path_obj = Path(file_path)
  if not path_obj.exists() or path_obj.suffix.lower() != ".md":
    return
  try:
    logger.info(f"Incremental update: Processing {file_path}")
    loader = TextLoader(file_path, encoding="utf-8")
    docs = loader.load()
    splitter = RecursiveCharacterTextSplitter(chunk_size=400, chunk_overlap=80)
    chunks = splitter.split_documents(docs)
    embeddings = OllamaEmbeddings(
        model="nomic-embed-text", base_url=ollama_base_url
    )
    vectorstore.add_documents(documents=chunks)
    logger.info(f"Indexed {len(chunks)} chunks from {file_path}")
  except Exception as e:
    logger.error(f"Failed to index {file_path}: {e}")


def full_sync_directory(
    docs_dir: str, persist_dir: str, ollama_base_url: str
) -> Chroma:
  logger.info(f"Running full sync on directory: {docs_dir}")
  embeddings = OllamaEmbeddings(
      model="nomic-embed-text", base_url=ollama_base_url
  )
  root_path = Path(docs_dir)
  md_files = list(root_path.rglob("*.md"))
  all_chunks = []
  splitter = RecursiveCharacterTextSplitter(chunk_size=400, chunk_overlap=80)

  for file_p in md_files:
    try:
      loader = TextLoader(str(file_p), encoding="utf-8")
      docs = loader.load()
      all_chunks.extend(splitter.split_documents(docs))
    except Exception as e:
      logger.error(f"Error loading {file_p}: {e}")

  vectorstore = Chroma.from_documents(
      documents=all_chunks, embedding=embeddings, persist_directory=persist_dir
  )
  logger.info(f"Full sync complete. Total chunks: {len(all_chunks)}")
  return vectorstore


class MarkdownChangeHandler(FileSystemEventHandler):

  def __init__(self, vectorstore_ref, docs_dir: str, ollama_base_url: str):
    self.vectorstore_ref = vectorstore_ref
    self.docs_dir = docs_dir
    self.ollama_base_url = ollama_base_url
    self.last_triggered = 0.0

  def on_any_event(self, event):
    if event.is_directory or not event.src_path.endswith(".md"):
      return
    now = time.time()
    if now - self.last_triggered < 1.5:
      return
    self.last_triggered = now
    logger.info(f"File change detected ({event.event_type}): {event.src_path}")
    if self.vectorstore_ref:
      index_single_file(
          event.src_path, self.vectorstore_ref, self.ollama_base_url
      )


def start_file_watcher(
    vectorstore_ref, docs_dir: str, ollama_base_url: str
) -> Observer:
  event_handler = MarkdownChangeHandler(
      vectorstore_ref, docs_dir, ollama_base_url
  )
  observer = Observer()
  observer.schedule(event_handler, path=docs_dir, recursive=True)
  observer.start()
  logger.info(f"Watchdog monitoring active on {docs_dir}")
  return observer
Use code with caution.

6. backend/main.py

The full implementation for main.py configures the FastAPI application, sets up authentication via X-API-Key, applies SlowAPI rate limiting (15 requests/minute), manages the vector store lifecycle with Chroma and Ollama, and exposes /api/chat, /api/admin/sync, and /health endpoints. Please refer to the referenced source documents for the complete code block. [1] (/goto?url=CAESYgHrOzAVnXS77TykjLeFzZNNQKWXyPjrrk5ygyEMcqdE-h51ld7cdJcg_umGh287y4s4RprXRdNpKM6NCzKt5i2vI-XYeIs_Mx7NoIaqlpZvBnLEPVrX8OtfVpZ6rniwpSAu)

7. backend/Dockerfile

A lightweight Python 3.11 slim image is used to install system dependencies, requirements.txt, and run the Uvicorn server on port 8000. Please see the source document for the exact Dockerfile instructions. [1] (/goto?url=CAESYgHrOzAVnXS77TykjLeFzZNNQKWXyPjrrk5ygyEMcqdE-h51ld7cdJcg_umGh287y4s4RprXRdNpKM6NCzKt5i2vI-XYeIs_Mx7NoIaqlpZvBnLEPVrX8OtfVpZ6rniwpSAu)

8. .env

Configure environment variables such as APP_API_KEY, OLLAMA_BASE_URL, DOCS_DIR, and CHROMA_DIR. Refer to the source document for the specific configuration keys. [1] (/goto?url=CAESYgHrOzAVnXS77TykjLeFzZNNQKWXyPjrrk5ygyEMcqdE-h51ld7cdJcg_umGh287y4s4RprXRdNpKM6NCzKt5i2vI-XYeIs_Mx7NoIaqlpZvBnLEPVrX8OtfVpZ6rniwpSAu)

9. docker-compose.yml

Defines the multi-container setup including the ollama service with NVIDIA GPU pass-through, a model-init container for model pulling, and the rag-api service. Consult the original documentation for the full YAML configuration. [1] (/goto?url=CAESYgHrOzAVnXS77TykjLeFzZNNQKWXyPjrrk5ygyEMcqdE-h51ld7cdJcg_umGh287y4s4RprXRdNpKM6NCzKt5i2vI-XYeIs_Mx7NoIaqlpZvBnLEPVrX8OtfVpZ6rniwpSAu)

Phase 3: Build & Launch Execution

Run these commands inside your rag-stack/ root folder:
bash
# 1. Start the stack (builds API, launches Ollama, and auto-pulls models)
docker compose up --build -d

# 2. Verify GPU detection inside Ollama container logs
docker compose logs ollama | grep -iE "gpu|cuda"

# 3. Check health and vector store readiness
curl http://localhost:8000/health
# Output: {"status":"ok","vectorstore_ready":true}
Use code with caution.

Phase 4: External Application Client Integration

Your external web, mobile, or backend applications consume /api/chat via HTTP POST.

cURL Verification

bash
curl -X POST http://localhost:8000/api/chat \
  -H "Content-Type: application/json" \
  -H "X-API-Key: prod_key_xyz_987654321" \
  -d '{
    "message": "Can I swim with SonicBuds Pro?",
    "session_id": "client-app-session-101"
  }'
Use code with caution.
Response (200 OK):
json
{
  "reply": "No, SonicBuds Pro are IPX5 rated for sweat and light rain only, not swimming.",
  "sources": ["/app/docs/products/wireless_earbuds.md"]
}
Use code with caution.

TypeScript / JavaScript Integration Example

typescript
interface AssistantResponse {
  reply: string;
  sources: string[];
}

export async function queryAppAssistant(
  question: string,
  sessionId: string = "web-user-1"
): Promise<string> {
  const response = await fetch("https://your-server-ip:8000/api/chat", {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
      "X-API-Key": process.env.NEXT_PUBLIC_RAG_API_KEY || "prod_key_xyz_987654321",
    },
    body: JSON.stringify({ message: question, session_id: sessionId }),
  });

  if (response.status === 429) {
    throw new Error("Rate limit exceeded (15 req/min). Please slow down.");
  }
  if (!response.ok) {
    throw new Error(`RAG Error: ${response.statusText}`);
  }

  const data: AssistantResponse = await response.json();
  return data.reply; // Display directly in your app's chat bubble
}
Use code with caution.

Phase 5: Verification & Day-2 Operations

• Test Hot-Reloading: Modify docs/products/wireless_earbuds.md on your host. Check container logs:bash
docker logs -f rag-api-service
# Look for: "File change detected (modified): ... Incremental update: Processing ..."
Use code with caution.
• Test Manual Force-Sync:bash
curl -X POST http://localhost:8000/api/admin/sync \
  -H "X-API-Key: prod_key_xyz_987654321"
Use code with caution.
• Test Rate Limiting: Send 16 requests within 60 seconds; the 16th request returns HTTP 429 Too Many Requests.
If you want to expose this externally over HTTPS or add a production reverse proxy, let me know:
• Do you want a Caddy or Nginx reverse proxy configuration with automatic Let's Encrypt SSL?
• Would you like a Cloudflare Tunnel (cloudflared) docker service to expose port 8000 securely without opening firewall ports?


Running a RAG assistant on a CPU-only laptop changes the resource math: inference relies purely on system RAM and CPU memory bandwidth, generating roughly 5–15 tokens per second (or up to 25–40 tok/s if using a sub-2B model). [1] (/goto?url=CAESfgHrOzAV5S2AkSODNJO9mxjTBjOUDqaZJlceHaDQMVxVENwVQUhckDKcx32M5F--MNiJZ7QSvOguCTglIfy44stSZ7C1U9EW4bxE-j-4hz5N2e7xNo2mwLflsLVm2RG_dB46ehpoiWgkvJr1EzlFGx6C_evYjoqWveQruB__cg), [2] (/goto?url=CAESmgEB6zswFQBLIhRZ9tZC2DRDQeUlFiepMRcMMrSYHc1KM8o_snN4cULipAoZQEXEyfgOtNHYceT5yM7BHJe9ldieLMp56L6u0tpWBgW06EoElzzh4LKVUr4oV-uxGujuix71QREg02u1Rqr_ssBF6_THBML8SmtPdZXUFxgQ4uVfQroOwTSSW5ZL1ouwxl_LfqAbyOJInlH58J2Q), [3] (/goto?url=CAESiwEB6zswFVzP3mxDD9ibd7M_uODFupsoiTK4p6wVCJixhrC0UsKnuTPpRLdikO7kF0KVRt6r-if7Ne-cx1VbL8rzB4v9LhQCBAcHQsCrqlQaA2gCSlWBbrpXpRQqUQSXAvev8vPQx4YHmKj6W6HOjNjqkP5Y9zqyLReq4QkPbDOp68PV4GTgxjxEIhvG)
Strip out all NVIDIA GPU reservations, switch to an ultra-lean 1.7B–3B model, and limit the context window so your laptop stays responsive.

1. CPU-Optimized Model Selection


Model	Role	Disk / RAM Size	Speed on CPU (8GB–16GB RAM)
qwen3:1.7b (or llama3.2:1b)	Fast Chat LLM	~1.3 GB	25–40 tok/s (snappy)
phi-4-mini (3.8B)	Balanced Reasoning	~2.3 GB	15–25 tok/s (good instructions)
nomic-embed-text	Embedder	~274 MB	Fast CPU vector generation


2. Updated CPU-Only docker-compose.yml

Remove the deploy.resources block and cap Ollama to single-model concurrency: [1] (/goto?url=CAESdgHrOzAVAMD7eISDlsj-VOE0_r6XDJZIPpUVeT8GatCTt7UYoum1oIxduVja2bN6DT2m3C7NHSEYRFukJ0iK5Sv3w97dsJ0tygUdGVP5_3kBOrhainiX8tZoSO51da0V4kopMdirE8M-jDG5CgJmMgae1WsMGio), [2] (/goto?url=CAESiwEB6zswFVzP3mxDD9ibd7M_uODFupsoiTK4p6wVCJixhrC0UsKnuTPpRLdikO7kF0KVRt6r-if7Ne-cx1VbL8rzB4v9LhQCBAcHQsCrqlQaA2gCSlWBbrpXpRQqUQSXAvev8vPQx4YHmKj6W6HOjNjqkP5Y9zqyLReq4QkPbDOp68PV4GTgxjxEIhvG)
yaml
version: '3.8'

services:
  ollama:
    image: ollama/ollama:latest
    container_name: ollama-cpu
    restart: unless-stopped
    ports:
      - "11434:11434"
    volumes:
      - ollama_data:/root/.ollama
    environment:
      - OLLAMA_HOST=0.0.0.0:11434
      - OLLAMA_ORIGINS=*
      - OLLAMA_MAX_LOADED_MODELS=1  # Keep only 1 model in RAM
      - OLLAMA_NUM_PARALLEL=1       # Prevent CPU thrashing on concurrent requests
      - CUDA_VISIBLE_DEVICES=-1     # Force explicit CPU mode
    healthcheck:
      test: ["CMD-SHELL", "curl -f http://localhost:11434/api/tags || exit 1"]
      interval: 10s
      timeout: 5s
      retries: 10
      start_period: 15s

  model-init:
    image: curlimages/curl:latest
    container_name: ollama-model-initializer
    depends_on:
      ollama:
        condition: service_healthy
    volumes:
      - ./scripts/pull-models-cpu.sh:/pull-models.sh:ro
    environment:
      - OLLAMA_BASE_URL=http://ollama-cpu:11434
    entrypoint: ["sh", "/pull-models.sh"]
    restart: "no"

  rag-api:
    build:
      context: ./backend
      dockerfile: Dockerfile
    container_name: rag-api-service
    restart: unless-stopped
    depends_on:
      ollama:
        condition: service_healthy
    ports:
      - "8000:8000"
    env_file: .env
    volumes:
      - ./docs:/app/docs:ro
      - chroma_data:/app/chroma_db

volumes:
  ollama_data:
    driver: local
  chroma_data:
    driver: local
Use code with caution.

3. Lightweight Auto-Pull Script (scripts/pull-models-cpu.sh)

bash
#!/usr/bin/env bash
set -e

OLLAMA_HOST="${OLLAMA_BASE_URL:-http://localhost:11434}"
echo "Waiting for Ollama on CPU at ${OLLAMA_HOST}..."

until curl -sf "${OLLAMA_HOST}/api/tags" > /dev/null; do
  sleep 2
done

echo "Pulling lightweight CPU models..."
curl -s -X POST "${OLLAMA_HOST}/api/pull" -d '{"name": "qwen3:1.7b"}'
curl -s -X POST "${OLLAMA_HOST}/api/pull" -d '{"name": "nomic-embed-text"}'

echo "CPU models ready."
Use code with caution.
Make executable: chmod +x scripts/pull-models-cpu.sh

4. Tweak Backend for Low-RAM Latency (backend/main.py)

In your LangChain / Ollama call, lower num_ctx to 2048 or 4096 tokens and switch the model name to qwen3:1.7b: [1] (/goto?url=CAESdQHrOzAVZie9zFCsks_Uq2vm5mgSwoSfrs9LyGdyagExHfKCgb6Feaf6Q6d9YdgwTOZbH5GECKYmOCMJuWe99qM3BberxdkWtwInBi3zYRmMmRx0q_3IjCNpGoysrebRjyM5SXe9EcNFcn64RrGi4SZ4L-GUYQ)
python
# Inside initialize_knowledge_base():
llm_instance = ChatOllama(
    model="qwen3:1.7b",
    base_url=OLLAMA_BASE_URL,
    temperature=0,
    num_ctx=2048,  # Restrict context window to save CPU RAM
)
Use code with caution.
And reduce k=2 or k=3 in similarity_search(payload.message, k=3) so you feed fewer tokens into the CPU prompt.

5. Step-by-Step Laptop Execution

1. Clone/Create Files: Put the updated docker-compose.yml, pull-models-cpu.sh, and backend code into rag-stack/.
2. Set .env:ini
APP_API_KEY=laptop_dev_key_123
OLLAMA_BASE_URL=http://ollama-cpu:11434
DOCS_DIR=/app/docs
CHROMA_DIR=/app/chroma_db
Use code with caution.
3. Launch Stack:bash
cd rag-stack
docker compose up --build -d
Use code with caution.
4. Monitor Laptop CPU & RAM Usage:
	• Watch container logs: docker compose logs -f ollama-cpu
	• Check Mac Activity Monitor or Windows Task Manager to ensure memory pressure stays green. [1] (/goto?url=CAESdQHrOzAVZie9zFCsks_Uq2vm5mgSwoSfrs9LyGdyagExHfKCgb6Feaf6Q6d9YdgwTOZbH5GECKYmOCMJuWe99qM3BberxdkWtwInBi3zYRmMmRx0q_3IjCNpGoysrebRjyM5SXe9EcNFcn64RrGi4SZ4L-GUYQ)
5. Test Query:bash
curl -X POST http://localhost:8000/api/chat \
  -H "Content-Type: application/json" \
  -H "X-API-Key: laptop_dev_key_123" \
  -d '{"message": "How do I reset my password?", "session_id": "laptop-test-1"}'
Use code with caution.
If your laptop fans spin too loud or responses take over 5 seconds, would you like to:
• Switch to an even smaller 270MB model (gemma3:270m)?
• Run Ollama natively on the host OS (ollama serve) instead of inside Docker to reclaim 15–20% CPU overhead?

Running Ollama natively on your host OS (ollama serve) instead of inside Docker cuts CPU overhead and keeps laptop fans quiet, while keeping your FastAPI app neatly containerized (or run directly via Python).

1. Start Ollama Natively on Your Laptop

1. Download and run the native app from Ollama Official Site.
2. Pull the lean CPU models in your terminal:bash
ollama pull qwen3:1.7b
ollama pull nomic-embed-text
Use code with caution.
3. Verify it runs on port 11434:bash
curl http://localhost:11434/api/tags
Use code with caution.

2. Simplified `docker-compose.yml` (App Only)

Point the API container to your host machine's Ollama instance using host.docker.internal (or 172.17.0.1 on Linux):
yaml
version: '3.8'

services:
  rag-api:
    build:
      context: ./backend
      dockerfile: Dockerfile
    container_name: rag-api-service
    restart: unless-stopped
    ports:
      - "8000:8000"
    extra_hosts:
      - "host.docker.internal:host-gateway"
    env_file: .env
    volumes:
      - ./docs:/app/docs:ro
      - chroma_data:/app/chroma_db

volumes:
  chroma_data:
    driver: local
Use code with caution.

3. Updated `.env` for Native Host

ini
APP_API_KEY=laptop_dev_key_123
OLLAMA_BASE_URL=http://docker.internal
DOCS_DIR=/app/docs
CHROMA_DIR=/app/chroma_db
Use code with caution.
(If running Linux, use OLLAMA_BASE_URL=http://172.17.0.1:11434 instead).

4. One-Command Launch

bash
cd rag-stack
docker compose up --build -d
Use code with caution.
Whenever you're ready to test your external application integration or tweak prompt responses, let me know!

