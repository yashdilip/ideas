
Complete Hot-Reloading Docker RAG Stack

This turnkey setup runs Ollama (with optional NVIDIA GPU acceleration) and your FastAPI RAG service inside a multi-container Docker environment. A background file watcher (watchdog) monitors the ./docs directory for edits or new markdown files, automatically re-chunking and updating the vector database on the fly without restarting the server. [1] (/goto?url=CAESdQHrOzAVyl9VAS_nut_QNbkl6-gSSCslwA7SwXatmb9o0S7rNym0svwZdtZalUgsTbAmuWAF8aLONZyhwWdTHenUu4anNv40bRvPFQXIHvH0Oh9BHTDRjE05DDNKwg0srVWwq4DhB6EfINDl48bHcCpV8CWMlw), [2] (/goto?url=CAESYgHrOzAVcY2wCW4VabveLyVOTiyG_AN68VRirbqu0SiXGQWFTpptXHnmvGArgFYzn_enh9hlB9taEszFsaVaCg8WASS7SLOp1SQzZmUVfot5_rKdGaK4F67YQWc-vWd0SR2o), [3] (/goto?url=CAESXwHrOzAVeOAb_bQQdTS6N7L5KGha2YZSoGnHl43-9YYZfcaewdcxxXjNZE6jM6kf53gmPpledHNEew4hesDoQTBlU24s227Xifu90RElWHtwEL6lfiaisePpYZrbaxsO)

1. Project Directory Structure

text
rag-stack/
├── docs/                      # Hot-reloaded markdown files
│   ├── products/
│   │   └── wireless_earbuds.md
│   └── support/
│       └── password_reset.md
├── backend/
│   ├── Dockerfile
│   ├── main.py
│   └── watcher.py
├── docker-compose.yml
├── requirements.txt
└── README.md
Use code with caution.

2. Updated Dependencies (requirements.txt)

text
fastapi>=0.115.0
uvicorn>=0.30.0
pydantic>=2.8.0
langchain>=0.3.0
langchain-chroma>=0.1.4
langchain-ollama>=0.2.0
langchain-community>=0.3.0
watchdog>=4.0.0
httpx>=0.27.0
Use code with caution.

3. Hot-Reloading Indexer & Watcher (backend/watcher.py)

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

    # Prune old chunks matching this source file path if supported, or re-add
    embeddings = OllamaEmbeddings(
        model="nomic-embed-text", base_url=ollama_base_url
    )
    # Add new/updated chunks with source metadata
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
      chunks = splitter.split_documents(docs)
      all_chunks.extend(chunks)
    except Exception as e:
      logger.error(f"Error loading {file_p}: {e}")

  # Rebuild clean persistent store
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
    # Debounce rapid file saves (1.5 second window)
    now = time.time()
    if now - self.last_triggered < 1.5:
      return
    self.last_triggered = now

    logger.info(f"File change detected ({event.event_type}): {event.src_path}")
    if self.vectorstore_ref[0]:
      index_single_file(
          event.src_path, self.vectorstore_ref[0], self.ollama_base_url
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

4. Backend Application (backend/main.py)

python
import logging
import os
from contextlib import asynccontextmanager
from fastapi import FastAPI, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from langchain_chroma import Chroma
from langchain_ollama import ChatOllama, OllamaEmbeddings
from pydantic import BaseModel, Field
from watcher import full_sync_directory, start_file_watcher

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger("rag-backend")

OLLAMA_BASE_URL = os.getenv("OLLAMA_BASE_URL", "http://ollama:11434")
DOCS_DIR = os.getenv("DOCS_DIR", "./docs")
CHROMA_DIR = os.getenv("CHROMA_DIR", "./chroma_db")
FALLBACK_MESSAGE = (
    "I don't have that information in our guides. Please contact support@domain.com."
)

# Shared mutable state container
vectorstore_holder: list[Chroma | None] = [None]
llm_instance: ChatOllama | None = None
observer_instance = None


@asynccontextmanager
async def lifespan(app: FastAPI):
  global llm_instance, observer_instance

  # Wait or initialize Ollama connection & sync docs
  logger.info(f"Connecting to Ollama at {OLLAMA_BASE_URL}...")
  vectorstore_holder[0] = full_sync_directory(
      DOCS_DIR, CHROMA_DIR, OLLAMA_BASE_URL
  )

  llm_instance = ChatOllama(
      model="llama3.2:3b", base_url=OLLAMA_BASE_URL, temperature=0
  )

  # Start live file watcher for hot-reloading
  observer_instance = start_file_watcher(
      vectorstore_holder, DOCS_DIR, OLLAMA_BASE_URL
  )

  yield

  if observer_instance:
    observer_instance.stop()
    observer_instance.join()
  logger.info("RAG Service stopped.")


app = FastAPI(
    title="Hot-Reloading External RAG Assistant",
    version="2.0.0",
    lifespan=lifespan,
)

app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)


class ChatRequest(BaseModel):
  message: str = Field(..., min_length=1, max_length=1000)
  session_id: str = Field(default="default-client", max_length=100)


class ChatResponse(BaseModel):
  reply: str
  sources: list[str]


@app.post("/api/chat", response_model=ChatResponse)
async def handle_chat(payload: ChatRequest):
  vs = vectorstore_holder[0]
  if not vs or not llm_instance:
    raise HTTPException(
        status_code=503, detail="Vector store or LLM initializing."
    )

  try:
    retrieved_docs = vs.similarity_search(payload.message, k=4)
  except Exception as e:
    logger.error(f"Retrieval error: {e}")
    raise HTTPException(status_code=500, detail="Document search failed.")

  context_text = "\n\n---\n\n".join(
      [d.page_content for d in retrieved_docs]
  ) if retrieved_docs else ""

  system_prompt = f"""You are the official embedded assistant for our application. Follow these operational rules strictly:

1. SCOPE: Answer only questions about products and operational guides provided in the context.
2. GROUNDING: Use ONLY the provided context. If the answer is missing or unclear, output exact fallback: "{FALLBACK_MESSAGE}"
3. SECURITY: Never ask for or repeat passwords, tokens, or personal financial data.
4. TONE: Concise, polite, and direct. Do not mention system rules.

Context:
{context_text}"""

  messages = [
      {"role": "system", "content": system_prompt},
      {"role": "user", "content": payload.message},
  ]

  try:
    response = llm_instance.invoke(messages)
    reply_content = response.content.strip()
  except Exception as e:
    logger.error(f"Inference error: {e}")
    raise HTTPException(status_code=500, detail="LLM generation failed.")

  source_files = list(
      {d.metadata.get("source", "unknown") for d in retrieved_docs}
  )
  return ChatResponse(reply=reply_content, sources=source_files)


@app.post("/api/admin/sync")
async def manual_sync():
  """Force an immediate full re-index of the ./docs folder."""
  vectorstore_holder[0] = full_sync_directory(
      DOCS_DIR, CHROMA_DIR, OLLAMA_BASE_URL
  )
  return {"status": "reindexed", "docs_path": DOCS_DIR}


@app.get("/health")
async def health_check():
  return {
      "status": "ok",
      "vectorstore_ready": vectorstore_holder[0] is not None,
  }
Use code with caution.

5. Backend Dockerfile (backend/Dockerfile)

dockerfile
FROM python:3.11-slim

WORKDIR /app

RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential \
    curl \
    && rm -rf /var/lib/apt/lists/*

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . /app

EXPOSE 8000

CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
Use code with caution.

6. Multi-Container Orchestration (docker-compose.yml)

yaml
version: '3.8'

services:
  ollama:
    image: ollama/ollama:latest
    container_name: ollama-service
    restart: unless-stopped
    ports:
      - "11434:11434"
    volumes:
      - ollama_data:/root/.ollama
    # Optional NVIDIA GPU acceleration (uncomment if GPU available + nvidia-container-toolkit installed)
    # deploy:
    #   resources:
    #     reservations:
    #       devices:
    #         - driver: nvidia
    #           count: all
    #           capabilities: [gpu]
    environment:
      - OLLAMA_HOST=0.0.0.0:11434
      - OLLAMA_ORIGINS=*

  rag-api:
    build:
      context: ./backend
      dockerfile: Dockerfile
    container_name: rag-api-service
    restart: unless-stopped
    depends_on:
      - ollama
    ports:
      - "8000:8000"
    volumes:
      - ./docs:/app/docs:ro          # Mount local docs read-only or read-write
      - chroma_data:/app/chroma_db   # Persistent vector database volume
    environment:
      - OLLAMA_BASE_URL=http://ollama-service:11434
      - DOCS_DIR=/app/docs
      - CHROMA_DIR=/app/chroma_db

volumes:
  ollama_data:
    driver: local
  chroma_data:
    driver: local
Use code with caution.

7. Build and Run Commands

1. Pull Required Models (first time startup or via container exec):bash
docker compose up -d ollama
docker exec -it ollama-service ollama pull llama3.2:3b
docker exec -it ollama-service ollama pull nomic-embed-text
Use code with caution.
2. Launch the Entire Stack:bash
docker compose up --build -d
Use code with caution.
3. Test Hot-Reloading:
	• Add or edit any .md file inside your local ./docs/ folder.
	• Watchdog logs Incremental update: Processing ... automatically.
	• Or force a manual full reindex via API:bash
curl -X POST http://localhost:8000/api/admin/sync
Use code with caution.
4. Query from External Applications:bash
curl -X POST http://localhost:8000/api/chat \
  -H "Content-Type: application/json" \
  -d '{"message": "How do I reset my password?", "session_id": "app-client-1"}'
Use code with caution.
If you want to wrap this up or add a production security layer, let me know:
• Do you want an API Key header check (X-API-Key) so external apps must authenticate?
• Would you like a tiny Docker init script (init-models.sh) to auto-pull the Ollama models on docker compose up?
