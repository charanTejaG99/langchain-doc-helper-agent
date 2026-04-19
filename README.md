# LangChain Documentation Helper

A RAG (Retrieval-Augmented Generation) chatbot that answers questions about LangChain by crawling the official documentation, indexing it into a vector store, and retrieving relevant context at query time.

## Architecture

```
┌─────────────┐     ┌──────────────┐     ┌─────────────┐     ┌──────────┐
│  Streamlit   │────▶│  LangChain   │────▶│  Pinecone   │     │  Azure   │
│   Chat UI    │◀────│    Agent     │◀────│ VectorStore │     │ OpenAI   │
│  (main.py)   │     │ (core.py)    │────▶│             │     │  GPT-5   │
└─────────────┘     └──────────────┘     └─────────────┘     └──────────┘
                           │
                    ┌──────┴──────┐
                    │   Langfuse  │
                    │   Tracing   │
                    └─────────────┘
```

### Components

| File | Description |
|---|---|
| `main.py` | Streamlit chat UI with message history, source citations, and session management |
| `backend/core.py` | RAG pipeline — initializes the LLM agent, retrieves context from Pinecone, and generates answers |
| `ingestion.py` | Documentation ingestion pipeline — crawls LangChain docs via Tavily, chunks them, and indexes into Pinecone |
| `logger.py` | Colored console logging utilities for the ingestion pipeline |

## Tech Stack

- **LLM**: Azure OpenAI GPT-5 (via `langchain-openai`)
- **Embeddings**: Azure OpenAI `text-embedding-3-small`
- **Vector Store**: Pinecone
- **Web Crawling**: Tavily (`TavilyCrawl`)
- **Text Splitting**: LangChain `RecursiveCharacterTextSplitter` (4000 chunk size, 200 overlap)
- **Frontend**: Streamlit
- **Observability**: Langfuse tracing
- **Language**: Python 3.11+

## Prerequisites

- Python >= 3.11
- [uv](https://docs.astral.sh/uv/) (recommended) or pip
- An Azure OpenAI deployment with GPT-5 and `text-embedding-3-small`
- A Pinecone index
- A Tavily API key
- (Optional) A Langfuse account for tracing

## Setup

1. **Clone the repository**

   ```bash
   git clone <repo-url>
   cd langchain-doc-helper
   ```

2. **Create a virtual environment and install dependencies**

   ```bash
   uv venv --python 3.11
   source .venv/bin/activate
   uv pip install -e .
   ```

3. **Configure environment variables**

   Create a `.env` file in the project root:

   ```env
   # Azure OpenAI
   AZURE_OPENAI_API_KEY=<your-api-key>
   AZURE_OPENAI_ENDPOINT=<your-azure-endpoint>
   OPENAI_API_VERSION=2024-12-01-preview

   # Pinecone
   PINECONE_API_KEY=<your-pinecone-api-key>
   INDEX_NAME=<your-pinecone-index-name>

   # Tavily
   TAVILY_API_KEY=<your-tavily-api-key>

   # Langfuse (optional)
   LANGFUSE_SECRET_KEY=<your-secret-key>
   LANGFUSE_PUBLIC_KEY=<your-public-key>
   LANGFUSE_BASE_URL=https://cloud.langfuse.com
   ```

## Usage

### 1. Ingest documentation

Crawl the LangChain docs and index them into Pinecone. This only needs to be run once (or when you want to refresh the index):

```bash
python ingestion.py
```

### 2. Run the Streamlit app

```bash
streamlit run main.py
```

The app will be available at `http://localhost:8501`.

### 3. Run the backend directly (without UI)

```bash
python backend/core.py
```

This runs a single test query and prints the result to the console.

## How It Works

1. **Ingestion** (`ingestion.py`): Uses Tavily to crawl `python.langchain.com`, extracts page content, splits it into chunks using `RecursiveCharacterTextSplitter`, and indexes the chunks into Pinecone with Azure OpenAI embeddings. Batches are processed concurrently with `asyncio.gather`.

2. **Query** (`backend/core.py`): When a user asks a question, a LangChain agent retrieves the top matching document chunks from Pinecone, passes them as context to Azure OpenAI GPT-5, and generates a cited answer. All calls are traced via Langfuse.

3. **UI** (`main.py`): A Streamlit chat interface that maintains conversation history, displays the assistant's answers with expandable source citations, and provides a sidebar button to clear the session.
