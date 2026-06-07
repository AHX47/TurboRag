# TurboRag
<img width="1024" height="572" alt="image" src="https://github.com/user-attachments/assets/3420df50-3880-4e64-8514-6dfdcb0f6994" />
**TurboRag** is a fully offline, CPU, RAM RAG (Retrieval Augmented Generation) engine. It combines:

- **TurboVec** – a quantized (Q4) vector index (8× smaller than float32, faster than FAISS)
- <img width="1024" height="297" alt="image" src="https://github.com/user-attachments/assets/a428a12f-c386-4756-800a-d95e5c344b77" />
- **TurboQuant** – a lightweight quantization toolkit algorithms for embeddings and LLMs  
- **llama‑cpp‑python** – runs all models as Q4_K_M GGUF files on CPU  
- **FastAPI** – optional REST API  
- **FastMCP** – MCP server for agent integration  
- **Multi‑language SDKs** – Python, Node.js, Rust, C/C++

> **Projects referenced in this README:**  
> - [TurboVec](https://github.com/RyanCodrai/turbovec) – core quantized vector index  
> - [TurboQuant](https://github.com/RyanCodrai/turboquant) – quantization toolkit for models

---

## Features

- **No GPU required, no internet at runtime** – everything runs offline on CPU.
- **Tiny memory footprint** – Gemma Embedding 300M (≈150 MB) + Qwen 0.5B (300 MB).
- **TurboVec Q4 index** – 8× compression, fast brute‑force search.
- **Optional VLM** – SmolVLM (90‑320 MB) for vision+text tasks.
- **Built‑in SQLite document store** – metadata and chunk storage.
- **REST API** – `/index`, `/search`, `/ask` endpoints.
- **MCP server** – stdio (Claude Desktop) or SSE transport.
- **LangChain integration** – use as a vector store.
- **Multi‑language SDKs** – Python, Node.js, Rust, C/C++.

---

## Quick Start

### 1. System dependencies

```bash
# Ubuntu / Debian
sudo apt update && sudo apt install build-essential cmake

# macOS
xcode-select --install
brew install cmake
```

### 2. Install Rust (required for TurboVec)

```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
source "$HOME/.cargo/env"
```

### 3. Build TurboVec from source

TurboRag relies on **TurboVec** for vector indexing. Build it as follows:

```bash
# Download TurboVec
wget https://github.com/RyanCodrai/turbovec/archive/main.zip
unzip main.zip && cd turbovec-main

# Build Python binding
pip install maturin
cd turbovec-python
maturin develop --release
cd ../..
```

> **Note:** The official TurboVec repository is at [https://github.com/RyanCodrai/turbovec](https://github.com/RyanCodrai/turbovec).

### 4. Install TurboRag

```bash
git clone https://github.com/yourusername/turborag.git
cd turborag
pip install -r requirements.txt
pip install -e .
```

### 5. Download models (using TurboQuant)

TurboRag uses **TurboQuant** to produce quantized GGUF models. The primary embedding model is:

- **gemma-embedding-300m** – Q4_K_M, 2048‑dim, required.

Download it directly:

```bash
mkdir -p models
wget -O models/embeddinggemma-300m-q4_k_m.gguf \
  "https://huggingface.co/sabafallah/embeddinggemma-300m-Q4_K_M-GGUF/resolve/main/embeddinggemma-300m-q4_k_m.gguf"
```

For the LLM (Qwen 0.5B Q4_K_M):

```bash
wget -O models/qwen-0.5b-Q4_K_M.gguf \
  https://huggingface.co/.../qwen-0.5b-Q4_K_M.gguf   # replace with actual URL
```

To quantize your own models with **TurboQuant**:

```bash
git clone https://github.com/RyanCodrai/turboquant.git
cd turboquant
pip install -e .
turboquant convert --model gemma-300m --format gguf --qtype Q4_K_M
```

### 6. Verify installation

```python
python -c "from turbovec import IdMapIndex; from turborag import TurboRag; print('OK')"
```

---

## Usage

### Python API

```python
from turborag import TurboRag

rag = TurboRag.create(
    embed_model="models/embeddinggemma-300m-q4_k_m.gguf",
    llm_model="models/qwen-0.5b-Q4_K_M.gguf",
)
rag.add_document("Paris is the capital of France.")
answer, sources = rag.ask("What is the capital of France?")
print(answer)  # "Paris"
```

### REST API

```bash
# Start server
uvicorn turborag.api.server:app --host 127.0.0.1 --port 8000

# Index a document
curl -X POST http://localhost:8000/index \
  -H "Content-Type: application/json" \
  -d '{"text": "Paris is the capital of France."}'

# Ask a question
curl -X POST http://localhost:8000/ask \
  -H "Content-Type: application/json" \
  -d '{"question": "What is the capital of France?", "k": 5}'
```

### MCP Server

```bash
# SSE transport
python -m turborag.mcp.server --transport sse --port 8001

# stdio for Claude Desktop
python -m turborag.mcp.server --transport stdio
```

### LangChain Integration

```python
from turborag.integrations.langchain import TurboRagVectorStore, make_llamacpp_embeddings

embeddings = make_llamacpp_embeddings("models/embeddinggemma-300m-q4_k_m.gguf")
store = TurboRagVectorStore(embeddings=embeddings)
store.add_texts(["doc1", "doc2"])
docs = store.similarity_search("my query", k=3)
```

---

## SDKs

### Python SDK

```python
from turborag.sdk.python.turborag_sdk import TurboRagClient

client = TurboRagClient("http://127.0.0.1:8000")
client.index("Paris is the capital of France.")
resp = client.ask("What is the capital of France?")
print(resp.answer)
```

### Node.js SDK

```js
const { TurboRagClient } = require('./sdk/nodejs/src/index');
const client = new TurboRagClient('http://127.0.0.1:8000');
await client.index('Paris is the capital of France.');
const { answer } = await client.ask('What is the capital of France?');
console.log(answer);
```

### Rust SDK

```rust
use turborag_sdk::TurboRagClient;
let client = TurboRagClient::new("http://127.0.0.1:8000");
client.index("Paris is the capital of France.", None).await?;
let resp = client.ask("What is the capital of France?", 5, None).await?;
println!("{}", resp.answer);
```

### C SDK

```c
#include "turborag.h"
turborag_client_t *c = turborag_create("http://127.0.0.1:8000", NULL);
turborag_index(c, "Paris is the capital of France.", NULL);
turborag_ask_result_t r = turborag_ask(c, "What is the capital?", 5);
printf("%s\n", r.answer);
turborag_destroy(c);
```

---

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│  REST API (FastAPI)  │  MCP Server (FastMCP)           │
├─────────────────────────────────────────────────────────┤
│  TurboRag Core                                          │
│  ┌──────────┐  ┌───────────┐  ┌─────────┐  ┌────────┐ │
│  │ Embedder │  │ TurboVec  │  │   LLM   │  │SQLite  │ │
│  │ (Gemma)  │  │IdMapIndex │  │ (Qwen)  │  │Store   │ │
│  └──────────┘  └───────────┘  └─────────┘  └────────┘ │
├─────────────────────────────────────────────────────────┤
│  TurboQuant – quantization of models                   │
├─────────────────────────────────────────────────────────┤
│  SDKs: Python · Node.js · Rust · C/C++                 │
└─────────────────────────────────────────────────────────┘
```

---

## Models (all GGUF, CPU-only)

| Model | Size | Role | Download / Source |
|-------|------|------|-------------------|
| **gemma-embedding-300m** | ~300 MB | Text → 2048-dim vectors (required) | [HF:embeddinggemma-300m-GGUF] | Required
| Qwen 0.5B | ~500 MB | Fast Q&A generation | |[HF:qwen2.5-0.5b-instruct-GGUF] | optimal
| DeepSeek 1.3B | ~1.5 GB | Reasoning / longer answers | |[HF:deepseek-coder-1.3b-instruct-GGUF] |optimal
| SmolVLM 135M | ~150 MB | Ultra-light vision+text | |[HF:SmolLM2-135M-Instruct-GGUF] |optimal
| SmolVLM 256M | ~300 MB | Balanced VLM | |[HF:SmolVLM-256M-Instruct-GGUF] |optimal
| SmolVLM 500M | ~600 MB | Best vision quality | |[HF:SmolVLM-500M-Instruct-GGUF] | optimal

---

## Requirements

- Python 3.10+
- Rust (for TurboVec build) – [rustup.rs](https://rustup.rs/)
- ~1 GB RAM minimum (Gemma + Qwen)
- ~2 GB disk space for models and indexes
- No internet required at runtime

---

## Related Projects

- **[TurboVec](https://github.com/RyanCodrai/turbovec)** – Quantized vector index used by TurboRag.
- <img src="https://github.com/RyanCodrai/turbovec/blob/main/docs/header.png"/>
- **[TurboQuant](https://arxiv.org/abs/2504.19874)** – Google Research's TurboQuant algorithm
- <img width="300" height="168" alt="image" src="https://github.com/user-attachments/assets/16a2ccaf-37e5-4834-b46c-4ca395c4d98b" />
">

---

## License

MIT
