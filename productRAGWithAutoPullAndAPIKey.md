
Turnkey Production RAG Stack with Auto-Pull & API Key Security

This production layout adds an Ollama model auto-pull sidecar (https://www.google.com/search?kgmid=%2Fg%2F11ldlzv8sz&q=Ollama+model+auto-pull+sidecar) (using a lightweight curl container or startup script) and FastAPI X-API-Key header authentication via dependency injection. External applications must supply the matching API key on every request. [1] (/goto?url=CAESaAHrOzAV3tIlepOzMn5d0PAjiI9VxhDVXNlCzweGV87NBdH69wUWJXo-wrEZwdZobkiknQFxO06MHX_Vhyb6GGysB7oWNqD9IloRc3Xes0ZkW4m33HnayGSnvxzMEXyYPkUolTwTzQc4), [2] (/goto?url=CAESdQHrOzAVz97rZcXstTZ9CHUifin9TKB96CIXgExmeassEnxURIr4yFNsZjoouAkmFFWllGE6oaFYFH83eXbk1Swb-9X1FUD2-EytkjAZdha83DsiC-4PRM43H2kVVEJ4ifjO3wezQC8sabKpBfIu6nGmHRwBIQ)

1. Directory Structure

text
rag-stack/
├── docs/
│   ├── products/
│   │   └── wireless_earbuds.md
│   └── support/
│       └── password_reset.md
├── backend/
│   ├── Dockerfile
│   ├── main.py
│   └── watcher.py
├── scripts/
│   └── pull-models.sh
├── .env.example
├── .env
├── docker-compose.yml
└── README.md
Use code with caution.

2. Environment Configuration (.env)

ini
APP_API_KEY=your_super_secret_external_app_key_123
OLLAMA_BASE_URL=http://ollama-service:11434
DOCS_DIR=/app/docs
CHROMA_DIR=/app/chroma_db
Use code with caution.

3. Auto-Pull Script (scripts/pull-models.sh)

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

4. Backend Application with API Key Security (backend/main.py)

python
import logging
import os
from contextlib import asynccontextmanager
from fastapi import Depends, FastAPI, HTTPException, Security
from fastapi.middleware.cors import CORSMiddleware
from fastapi.security import APIKeyHeader
from langchain_chroma import Chroma
from langchain_ollama import ChatOllama, OllamaEmbeddings
from pydantic import BaseModel, Field
from starlette.status import HTTP_401_UNAUTHORIZED
from watcher import full_sync_directory, start_file_watcher

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger("rag-backend")

OLLAMA_BASE_URL = os.getenv("OLLAMA_BASE_URL", "http://ollama-service:11434")
DOCS_DIR = os.getenv("DOCS_DIR", "./docs")
CHROMA_DIR = os.getenv("CHROMA_DIR", "./chroma_db")
EXPECTED_API_KEY = os.getenv("APP_API_KEY", "default-dev-key")
FALLBACK_MESSAGE = (
    "I don't have that information in our guides. Please contact support@domain.com."
)

api_key_header_scheme = APIKeyHeader(name="X-API-Key", auto_error=True)


async def verify_api_key(api_key: str = Security(api_key_header_scheme)):
  if api_key != EXPECTED_API_KEY:
    raise HTTPException(
        status_code=HTTP_401_UNAUTHORIZED,
        detail="Invalid or missing X-API-Key header.",
    )
  return api_key


vectorstore_holder: list[Chroma | None] = [None]
llm_instance: ChatOllama | None = None
observer_instance = None


@asynccontextmanager
async def lifespan(app: FastAPI):
  global llm_instance, observer_instance

  logger.info(f"Connecting to Ollama at {OLLAMA_BASE_URL}...")
  vectorstore_holder = full_sync_directory(
      DOCS_DIR, CHROMA_DIR, OLLAMA_BASE_URL
  )

  llm_instance = ChatOllama(
      model="llama3.2:3b", base_url=OLLAMA_BASE_URL, temperature=0
  )

  observer_instance = start_file_watcher(
      vectorstore_holder, DOCS_DIR, OLLAMA_BASE_URL
  )

  yield

  if observer_instance:
    observer_instance.stop()
    observer_instance.join()
  logger.info("RAG Service stopped.")


app = FastAPI(
    title="Secured External RAG Assistant API",
    version="3.0.0",
    lifespan=lifespan,
    dependencies=[Depends(verify_api_key)],  # Enforce auth globally
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
  vs = vectorstore_holder
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
  vectorstore_holder = full_sync_directory(
      DOCS_DIR, CHROMA_DIR, OLLAMA_BASE_URL
  )
  return {"status": "reindexed", "docs_path": DOCS_DIR}


@app.get("/health", dependencies=[])  # Public unauthenticated health check
async def health_check():
  return {
      "status": "ok",
      "vectorstore_ready": vectorstore_holder is not None,
  }
Use code with caution.

5. Multi-Container Orchestration (docker-compose.yml)

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
    healthcheck:
      test: ["CMD-SHELL", "curl -f http://localhost:11434/api/tags || exit 1"]
      interval: 10s
      timeout: 5s
      retries: 10
      start_period: 15s
    environment:
      - OLLAMA_HOST=0.0.0.0:11434
      - OLLAMA_ORIGINS=*

  model-init:
    image: curlimages/curl:latest
    container_name: ollama-model-initializer
    depends_on:
      ollama:
        condition: service_healthy
    volumes:
      - ./scripts/pull-models.sh:/pull-models.sh:ro
    environment:
      - OLLAMA_BASE_URL=http://ollama-service:11434
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

6. Exact Step-by-Step Implementation Guide

1. Create the Project Directory & Copy Filesbash
mkdir -p rag-stack/docs/products rag-stack/docs/support rag-stack/backend rag-stack/scripts
Use code with caution.
2. Make the Shell Script Executablebash
chmod +x rag-stack/scripts/pull-models.sh
Use code with caution.
3. Populate Sample Markdown Files
Save your product and support text files inside rag-stack/docs/.
4. Configure Environment Variables
Copy .env.example to rag-stack/.env and set a strong APP_API_KEY.
5. Build and Launch Everythingbash
cd rag-stack
docker compose up --build -d
Use code with caution.

	• The ollama service starts first.
	• The model-init sidecar waits for Ollama health check, automatically pulls llama3.2:3b and nomic-embed-text, and exits cleanly.
	• The rag-api service boots, indexes ./docs into ./chroma_db, starts the live watchdog file observer, and listens on port 8000.
6. Verify Service Healthbash
curl http://localhost:8000/health
Use code with caution.
7. Call the Secured Assistant from External Applicationsbash
curl -X POST http://localhost:8000/api/chat \
  -H "Content-Type: application/json" \
  -H "X-API-Key: your_super_secret_external_app_key_123" \
  -d '{
    "message": "What is the warranty and battery life of SonicBuds Pro?",
    "session_id": "external-app-user-01"
  }'
Use code with caution.
8. Test Hot-Reloading
Edit rag-stack/docs/products/wireless_earbuds.md on your host machine. Inspect the backend container logs (docker logs -f rag-api-service) to see Incremental update: Processing ... update the vector index automatically.
If you want to add a rate limiter (slowapi) or configure GPU pass-through for NVIDIA/Apple Silicon hardware, let me know.
