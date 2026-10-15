
Hardened RAG Stack with SlowAPI Rate Limiting & GPU Pass-Through

This final production update integrates SlowAPI (capping requests per client and API key) and enables NVIDIA Container Toolkit GPU pass-through inside Docker Compose, preventing local VRAM exhaustion and brute-force abuse. [1] (/goto?url=CAESowEB6zswFcmuJfROeGXigfAJZz3YQnKWSObMs5hAG97H5oCSgyMUKVm7EMAEdERBnNrajziJ1WquTTi4UgvS0_vs8GVfD4g1UIKkyE4g8KgmBXd5biGhCKRXga4hCh5O81fOjCx-4jnjyP6OlYRS8FGVXGz7KbbhZvjq5LrNXnuZC_gGyb2qwtIErbazgEW1zIHsNejNROEQnObKvbhcDfysURnH), [2] (/goto?url=CAESdwHrOzAVKPTVi5UQWNcdmMGWfb72wbg37LFON94PH4C716LIu2mlE0HuZ8y0f1hiVtGVKceHaXou-GCV6l8BALvdZrJUlr3DlsXPFF4P96lREEcnjEC1GkYj0hhkKEsK25a78okH4NLehs3_Hg5ZwUKzHxRzqadP), [3] (/goto?url=CAESYwHrOzAVefd0bPrwg24pk2Cpwyp1504WtR5ghGUahoRSJT5SRvAX8-78wAflSRFABTtNwEKMz-NOfJSLQKX9d0yvy4cuJnaFhGcI3m6HP0eXVmU61Fx0z7OCHU7638pIPdG_EA)

1. Updated Dependencies (requirements.txt)

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

2. Backend Application with Rate Limiting (backend/main.py)

The FastAPI backend implements API key validation via headers, directory syncing, and SlowAPI rate limiting (e.g., 30/minute global and 15/minute on /api/chat). It uses LangChain with Chroma and ChatOllama (llama3.2:3b) with strict grounding and fallback rules. For the full implementation code, please refer to the referenced source files. [1] (/goto?url=CAESdwHrOzAVKPTVi5UQWNcdmMGWfb72wbg37LFON94PH4C716LIu2mlE0HuZ8y0f1hiVtGVKceHaXou-GCV6l8BALvdZrJUlr3DlsXPFF4P96lREEcnjEC1GkYj0hhkKEsK25a78okH4NLehs3_Hg5ZwUKzHxRzqadP), [2] (/goto?url=CAESYwHrOzAVefd0bPrwg24pk2Cpwyp1504WtR5ghGUahoRSJT5SRvAX8-78wAflSRFABTtNwEKMz-NOfJSLQKX9d0yvy4cuJnaFhGcI3m6HP0eXVmU61Fx0z7OCHU7638pIPdG_EA)

3. Docker Compose with NVIDIA GPU Reservations (docker-compose.yml)

The docker-compose.yml configures the ollama service with environment variables for concurrency and NVIDIA device reservations under deploy.resources.reservations, alongside a model initializer and the FastAPI application service. Full configuration details are available in the source documentation. [1] (/goto?url=CAESowEB6zswFcmuJfROeGXigfAJZz3YQnKWSObMs5hAG97H5oCSgyMUKVm7EMAEdERBnNrajziJ1WquTTi4UgvS0_vs8GVfD4g1UIKkyE4g8KgmBXd5biGhCKRXga4hCh5O81fOjCx-4jnjyP6OlYRS8FGVXGz7KbbhZvjq5LrNXnuZC_gGyb2qwtIErbazgEW1zIHsNejNROEQnObKvbhcDfysURnH)

4. Host System Setup & Verification (GPU + Docker)

1. Install NVIDIA Container Toolkit on Host (Ubuntu/Debian):
Configure the repository, install the toolkit, configure the Docker runtime, and restart the Docker daemon using standard installation commands. [1] (/goto?url=CAESfAHrOzAVWeqL-UoUY_BpuMu8a0f_mEVVnvv0S7DOGZqFiuaCroSvd5nqijxrIh5PTOvD-IhJfMDdCE6ViXdd2MKsJ2kUEgQzWIN5-RvmxoIliALwACOxaC-BKD06r_JkNbVv5PTu29mbazV5ZaECsmN_xNIOJfIIRkzgzLM)
2. Verify GPU Visibility Inside Container:
Bring up the Ollama container and run nvidia-smi inside it to confirm hardware access.
3. Launch Stack & Test Rate Limit Exceeded (HTTP 429):
Start the stack using Docker Compose and fire rapid requests to /api/chat to verify that requests beyond the 15/minute cap return 429 Too Many Requests.
If you want to complete the production deployment, let me know:
• Do you need an Nginx reverse proxy or Cloudflare Tunnel snippet for HTTPS termination?
• Would you like a client-side React/JS SDK wrapper with automatic retry logic on 429 and 503?
