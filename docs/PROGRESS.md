# Problem Statement

- Human memory is not built to retain large volumes of fast-moving, passive content — most of what's consumed is forgotten within days
- Short-form content (Reels, Shorts, TikToks) is scattered across multiple platforms with no unified way to organize or reference it later
- Users save content with intent to revisit ("I'll come back to this"), but rarely do — saved items pile up unseen, and users often forget they ever saved it at all

# Objective

Build an AI agent that acts as an external memory layer for the user — capturing, understanding, and retaining social content automatically, so recall doesn't depend on human memory

# Solution

- Capture — user shares a link or content with the agent via universal OS share sheet
- Queue & Acknowledge — payload is queued immediately (non-blocking `200 OK`) so the user keeps scrolling
- Asynchronous Process & Structure — background workers scrape content and multimodal models convert raw media into structured JSON and HyDE questions
- Store — embeddings and structured metadata are saved automatically into vector memory
- Recall — user asks questions later; agent synthesizes answers from stored memory with source attribution
- Manage — user reviews, updates, or deletes memories via natural language

---

# System Architecture: Omnivorous Multimodal RAG

We are building an **"Omnivorous" Multimodal RAG (Retrieval-Augmented Generation) System** — a true Second Brain that acts as a universal catcher for any digital content you consume, transforming passive scrolling into active, searchable knowledge.

By expanding the scope beyond just video to include text posts, articles, and general web links, the architecture shifts from a specialized video-scraper to a generalized context-engine.

## Comprehensive Technical Breakdown

### Phase 1: Ingestion & Queuing (Synchronous)

- [ ] **The App**: Build a Universal OS-level Share Target. When a user taps "Share" on their phone, the app accepts the payload, extracting the URL and `user_id`. Implement direct manual text entry for fleeting thoughts or rules (e.g., _"I prefer Shadcn over Tailwind"_).
- [ ] **API Gateway**: Deploy a lightweight backend endpoint (e.g., Node.js / FastAPI) to receive the payload.
- [ ] **Message Queue (MQ)**: Configure the API to push the payload into a message queue (e.g., Redis, AWS SQS, or RabbitMQ). The API must immediately return a `200 OK` success message so the user's share sheet closes instantly without interruption.

### Phase 2: Dispatch & Scraping (Asynchronous Workers)

- [ ] **The Dispatcher**: Implement a background worker that listens to the MQ, pops the next URL off the queue, and inspects it.
- [ ] **The Router**: Write logic to inspect the URL and route it to dedicated, format-specific scraping workers:
  - [ ] `YT_Worker`: Handles YouTube Shorts and videos (downloads raw `.mp4` and creator captions).
  - [ ] `IG_Worker`: Handles Instagram Reels and posts (downloads media, post captions, and thread comments).
  - [ ] `Web_Worker`: Handles web articles and blogs using a headless browser or services like Firecrawl to extract clean Markdown text, stripping ads and navigation bars.
- [ ] **The Scraper & Retry Queue**: Execute the scraping logic (bypassing anti-bot protections, downloading media, and extracting HTML/text). Implement a fallback where failed scrapes (due to rate limits or network issues) are thrown back into the MQ with a "retry later" tag and a backoff schedule.

### Phase 3: The AI Structurer (Multimodal Processing)

- [ ] **Extraction Models**: Pass the downloaded media through specialized modality models:
  - [ ] **Audio**: Transcribe via Whisper into text.
  - [ ] **Vision**: Pass keyframes (e.g., 1 frame per second) or static images/thumbnails through Vision-Language Models (e.g., LLaVA or GPT-4o) to describe visual actions and diagrams (e.g., _"A flowchart showing React component lifecycles"_).
  - [ ] **Text**: Pre-process and chunk web articles and manual notes into smaller paragraphs.
- [ ] **Structuring**: Use an LLM to synthesize the extracted transcript, visual descriptions, and creator caption, formatting the messy raw data into a clean, standardized JSON object:
  - `title`: Synthesized or extracted headline.
  - `summary`: Comprehensive summary of the content.
  - `key_entities`: Core topics, tools, concepts, or entities mentioned.
  - `original_transcript`: Full audio transcript or extracted article body.
- [ ] **HyDE Generation (Hypothetical Document Embeddings)**: Prompt the LLM to generate 3–5 hypothetical questions that this piece of content answers, dramatically improving future semantic retrieval alignment.

### Phase 4: Embedding & Storage

- [ ] **Chunking & Vectorization**: Pass the structured summary and the synthetic HyDE questions to an embedding model (e.g., OpenAI's `text-embedding-3-small`) to convert them into high-dimensional vector representations.
- [ ] **Vector DB Insertion**: Store the vectors in a Vector DB (e.g., Pinecone or Qdrant) alongside rich metadata:
  - `source_url`: Link to the original reel, video, or article.
  - `content_type`: `video`, `article`, `social_post`, or `manual`.
  - `timestamp_saved`: Ingestion timestamp.
  - `user_id`: Identifier for user-level data isolation.
  - `creator_name`: Creator or author handle.
- [ ] **Job Completion**: Once vectors and metadata are persisted, mark the background job as `"Complete"` in the backend.

### Phase 5: Conversational Retrieval Engine (Local SLM + Jev)

- [ ] **Query Expansion**: Convert the user's question into a vector and use a fast LLM to rewrite the raw query into optimized search terms.
- [ ] **Hybrid & HyDE Search**: Execute a similarity search in the Vector DB across both content summaries and the synthetic HyDE questions to pull the top $K$ (e.g., 20) relevant chunks.
- [ ] **Jev Filtering**: Pass the retrieved chunks to the TypeSafe AI Jev API. Use the `Noul` (Yes/No probability) primitive to score relevance, filtering the results down to only the highly relevant chunks (e.g., top 5) to eliminate hallucinations.
- [ ] **API Reasoning (Optional)**: Send the filtered chunks to a cloud API (like Claude 3.5 Sonnet) to analyze the facts, resolve contradictions, and format a strict JSON payload of raw facts and URLs.
- [ ] **SLM Synthesis**: Host a local Small Language Model (e.g., Llama 3.1 8B via vLLM) on a cloud GPU. Pass the raw facts/JSON to the SLM to format the text, apply custom guardrails, adjust verbosity, and append a "Sources" list.
- [ ] **Streaming Output**: Connect the SLM's output stream via Server-Sent Events (SSE) or WebSockets back to the mobile app for real-time delivery.

---

# High Level Architecture

## E-Brain: Complete Multi-Layer System Architecture Specification

This document provides the complete, production-grade technical specification for the E-Brain architecture across all five core layers. Every component, data schema, network protocol, service interface, and data flow path is specified in detail to serve as the direct blueprint for system architecture, sequence, and deployment diagrams.

---

## 1. The Frontend Layer (Client & OS Integration)

The Frontend acts as the capture edge and conversational presentation layer. It abstracts asynchronous ingestion away from the user, ensuring immediate responsiveness while managing local state and streaming sessions.

### 1.1 Applications & Native Sub-Modules
- **Mobile Client App (`client-mobile`)**: Cross-platform application (React Native / Flutter) or native mobile app (Swift/Kotlin).
  - **OS Share Sheet Receiver (`ShareTargetModule`)**: An OS extension module registered in `AndroidManifest.xml` (`ACTION_SEND` intent-filter) and iOS `ShareViewController` (`NSExtensionActivationRule`). Intercepts incoming URLs and shared text without launching the full application UI.
  - **Conversational Chat Interface (`ChatStreamView`)**: Markdown-enabled chat view supporting real-time Server-Sent Events (SSE) token rendering, source citation cards, and confirmation modal dialogs.
  - **Quick Thought Composer (`ManualNoteWidget`)**: Lightweight text input allowing users to store direct text constraints, preferences, and facts.
  - **Dead Reports Screen (`TriageDashboard`)**: UI view displaying ingest operations that failed scraping after all retries, allowing the user to view the failed URL and manually paste the raw text or transcript.
  - **Local Token & State Store (`SecureStorage`)**: Keychain/Keystore wrapper storing authentication tokens (JWTs) and active session IDs (`session_id`).

### 1.2 Data Structures & Client Schemas

#### Ingestion Share Payload (`SharePayloadDTO`)
```json
{
  "user_id": "usr_98a7f6c2",
  "source_url": "https://www.instagram.com/reel/C8xYz12345/",
  "raw_text": null,
  "content_type": "url",
  "client_timestamp": "2026-10-11T13:30:00Z"
}
```

#### Manual Note Payload (`ManualNoteDTO`)
```json
{
  "user_id": "usr_98a7f6c2",
  "source_url": null,
  "raw_text": "I prefer using Shadcn UI over standard Tailwind CSS components for clean dashboard architecture.",
  "content_type": "manual_note",
  "client_timestamp": "2026-10-11T13:31:00Z"
}
```

#### Client Chat Request (`ChatQueryDTO`)
```json
{
  "user_id": "usr_98a7f6c2",
  "session_id": "ses_4b82e1",
  "query": "What was the recommended frontend component library I saved recently?",
  "client_timestamp": "2026-10-11T13:35:00Z"
}
```

### 1.3 Network Protocols & Endpoints

| Action | Protocol | Endpoint | Headers | Success Response |
| :--- | :--- | :--- | :--- | :--- |
| Enqueue Content | POST (HTTP/1.1 or HTTP/2) | `/api/v1/ingest` | Authorization: Bearer &lt;JWT&gt;, Content-Type: application/json | 202 Accepted (`{"job_id": "job_...", "status": "queued"}`) |
| Conversational Chat | POST (Streaming) | `/api/v1/chat` | Authorization: Bearer &lt;JWT&gt;, Accept: text/event-stream | 200 OK (Chunked data: `{"token": "..."}\n\n`) |
| Fetch Dead Reports | GET | `/api/v1/dead-reports` | Authorization: Bearer &lt;JWT&gt; | 200 OK (`{"failed_items": [...]}`) |
| Resolve Dead Report | POST | `/api/v1/dead-reports/:id/resolve` | Authorization: Bearer &lt;JWT&gt; | 200 OK (`{"status": "requeued"}`) |

---

## 2. The Backend / Broker Layer (API Gateway & Queue Orchestrator)

The Backend orchestrates synchronous network traffic, enforces authentication, decouples long-running media processing via an asynchronous broker, and manages chat session memory.

### 2.1 Services & Components
- **API Gateway (`api-gateway`)**: Built using FastAPI or Node.js (NestJS/Express). Handles CORS, rate limiting, request validation, and JWT verification.
- **Message Broker (`redis-broker`)**: Redis instance configured with persistence (AOF/RDB) hosting BullMQ or Celery queues.
  - **Queue: `media-ingestion-queue`**: Holds raw incoming URLs waiting for scraping and extraction.
  - **Queue: `dlq-failed-ingestion`**: Dead-letter queue holding jobs that failed all retry attempts.
- **Session Cache (`redis-session-store`)**: Redis Key-Value store managing sliding-window chat conversation turns.
- **Task Dispatcher (`queue-dispatcher`)**: Background service consuming jobs from `media-ingestion-queue` and invoking the Extraction Microservice via gRPC.

### 2.2 Data Structures & Internal Queue Payloads

#### Ingestion Job Envelope (`QueueJobMessage`)
```json
{
  "job_id": "job_01h8x9p3q7",
  "user_id": "usr_98a7f6c2",
  "payload": {
    "source_url": "https://www.instagram.com/reel/C8xYz12345/",
    "raw_text": null,
    "content_type": "url"
  },
  "retry_count": 0,
  "max_retries": 3,
  "enqueued_at": "2026-10-11T13:30:01Z"
}
```

#### Redis Chat History Schema
- **Key Convention**: `chat:history:{user_id}:{session_id}`
- **Data Structure**: Redis LIST storing stringified JSON objects.
- **Retention**: TTL configured to 86,400 seconds (24 hours).
- **Format**:
```json
[
  {"role": "user", "content": "I am looking for a React UI toolkit."},
  {"role": "assistant", "content": "You previously saved notes recommending Shadcn UI."},
  {"role": "user", "content": "What was the recommended frontend component library I saved recently?"}
]
```

### 2.3 Internal Flow Logic
- `POST /api/v1/ingest` validates incoming JWT against public auth keys.
- If payload contains `raw_text` (manual note), Gateway bypasses Phase 2 (Microservices) and pushes directly to Phase 3 (AI Structurer).
- If payload contains `source_url`, Gateway pushes `QueueJobMessage` to `media-ingestion-queue`.
- Gateway immediately issues an HTTP 202 Accepted response to the client with `job_id`.

---

## 3. The Microservice Layer (Extraction Pipeline)

An isolated cluster of extraction workers designed to bypass platform limitations, manage third-party scraper API quotas, and isolate anti-bot dependencies from the main application logic.

### 3.1 Services & Responsibilities
- **Media Extraction gRPC Service (`extractor-service`)**: Exposes RPC endpoints over HTTP/2 with binary serialization.
- **Worker Implementations**:
  - **`YouTubeWorker`**: Parses YouTube Shorts and videos. Invokes `yt-dlp` or third-party wrappers to fetch auto-generated transcripts and captions without downloading full `.mp4` video streams.
  - **`SocialWorker`**: Handles Instagram Reels and TikTok URLs. Interfaces with managed scraping APIs (e.g., Firecrawl, ScrapFly, or Supadata) to bypass WebGL/Canvas fingerprinting and extract video captions, author metadata, and audio transcripts.
  - **`WebArticleWorker`**: Parses blog posts and standard web links. Fetches raw HTML, strips boilerplate, navigation menus, and ads, and converts DOM content into sanitized Markdown.
- **Dead-Letter Coordinator (`dlq-handler`)**: Intercepts unrecoverable errors (e.g., private video, 404 deleted content, exhausted proxy retries) and registers the failure in the vector store for user triage.

### 3.2 Protocol Buffer Interface (`MediaExtractor.proto`)
```protobuf
syntax = "proto3";

package ebrain.extractor;

service MediaExtractor {
  // Extracts clean text, transcript, and metadata from a target URL
  rpc ExtractMedia (MediaExtractRequest) returns (MediaExtractResponse);
}

message MediaExtractRequest {
  string job_id = 1;
  string user_id = 2;
  string source_url = 3;
  int32 retry_count = 4;
}

message MediaMetadata {
  string author = 1;
  string platform = 2; // "youtube", "instagram", "web"
  string raw_title = 3;
  int64 duration_seconds = 4;
  string published_timestamp = 5;
}

message MediaExtractResponse {
  string job_id = 1;
  bool success = 2;
  string extracted_text = 3; // Full transcript or clean markdown
  MediaMetadata metadata = 4;
  string error_code = 5;     // E.g., "RATE_LIMITED", "RESOURCE_PRIVATE", "UNREACHABLE"
  string error_message = 6;
}
```

### 3.3 Microservice Processing Flow
- `queue-dispatcher` establishes a persistent gRPC channel with `extractor-service` and calls `rpc ExtractMedia`.
- Router inspects the domain of `source_url`:
  - Match `youtube.com` / `youtu.be` $\rightarrow$ delegate to `YouTubeWorker`.
  - Match `instagram.com` / `tiktok.com` $\rightarrow$ delegate to `SocialWorker`.
  - Default fallback $\rightarrow$ delegate to `WebArticleWorker`.
- If the scraper receives an HTTP 429 (Rate Limited) or bot challenge:
  - Worker returns `success: false` with `error_code: "RATE_LIMITED"`.
  - Dispatcher increments `retry_count` and re-enqueues the job to Redis with an exponential backoff delay ($2^{\text{retry\_count}} \times 30\text{ seconds}$).
- If `retry_count >= 3`:
  - Job is routed to `dlq-failed-ingestion`.
  - Payload is logged as a Dead Report in the database layer with `status: "SCRAPE_FAILED"`.

---

## 4. The AI Layer (Gateway, Structuring, Jev & Synthesis)

The AI layer unifies multimodal inputs, normalizes raw text into structured schema, optimizes retrieval alignment through HyDE, filters candidate noise via discriminative decision gates, and synthesizes answers.

### 4.1 Sub-Components & Tooling
- **AI Gateway (`litellm-proxy`)**: A self-hosted or containerized LiteLLM Proxy service.
  - Exposes an OpenAI-compatible interface (`/v1/chat/completions`, `/v1/embeddings`) to the backend.
  - Translates internal model aliases (`ebrain-structurer`, `ebrain-embedding`, `ebrain-synthesis`) to underlying providers (Google Gemini Flash, Groq Llama 3.1, or local vLLM instances).
  - Centralizes API key rotation, load balancing, and fallbacks.
- **The Structurer Engine**: Parses raw scraped text/transcripts into standardized knowledge schemas.
- **HyDE Generator**: Generates synthetic user queries likely to match the current content.
- **Query Condenser (Coreference Resolver)**: Transforms multi-turn chat history into a standalone retrieval query.
- **Jev Discriminative Filter**: A high-speed, non-generative classification step (TypeSafe AI Jev model) that evaluates candidate text chunks against the user query, outputting a scalar confidence score.
- **Streaming Synthesizer (SLM/LLM)**: Fast conversational model that synthesizes verified chunks into a concise response and appends formatted source citations.
- **Tool Calling Evaluator**: Detects user intent to update or delete indexed memories, emitting structured function calls.

### 4.2 AI Schemas & Prompts

#### Structured Content Schema (`StructuredExtractionJSON`)
```json
{
  "title": "Configuring Shadcn UI with Tailwind CSS",
  "summary": "Step-by-step guidance on initializing Shadcn UI in a Next.js project, highlighting component reusability and styling separation.",
  "key_entities": ["Shadcn UI", "Tailwind CSS", "Next.js", "Radix Primitives"],
  "primary_topic": "Frontend Development",
  "original_context": "Full raw transcript or cleaned markdown text goes here..."
}
```

#### HyDE Output Schema (`HydeGenerationJSON`)
```json
{
  "hypothetical_questions": [
    "How do I set up Shadcn UI in a React project?",
    "What is the best component library to use with Tailwind CSS?",
    "How does Shadcn differ from traditional component packages?"
  ]
}
```

#### Structurer System Prompt
```plaintext
You are the E-Brain Structurer. You receive raw transcripts or article markdown.
Extract the core facts into a standardized JSON object.
Return strictly valid JSON matching this schema:
{
  "title": string,
  "summary": string,
  "key_entities": string[],
  "primary_topic": string,
  "original_context": string
}
Do not include any conversational preamble or Markdown fences.
```

#### Query Condenser System Prompt
```plaintext
Given the following conversation history and a follow-up user query, rewrite the follow-up query into a standalone, unambiguous search query.
Resolve all pronouns (e.g., "it", "that library", "his split") into concrete named entities.

History:
{chat_history}

Follow-up Query: {user_query}

Standalone Query:
```

#### Synthesis System Prompt
```plaintext
You are E-Brain, a personal knowledge assistant.
Answer the user's question using ONLY the provided verified context chunks below.
Keep your response concise, clear, and actionable.
At the end of your response, list the source URLs provided in the context under a "Sources:" heading.
If the context does not contain the answer, explicitly state that you cannot find it in saved memories.

Context:
{verified_context_chunks}
```

#### Memory Deletion Tool Schema (`ToolDefinition`)
```json
{
  "type": "function",
  "function": {
    "name": "delete_memory",
    "description": "Triggered when a user explicitly requests to forget or delete a saved memory, note, or resource.",
    "parameters": {
      "type": "object",
      "properties": {
        "search_term": {
          "type": "string",
          "description": "The topic, library, or keyword identifying the memory to delete (e.g., 'Shadcn note')."
        },
        "confirmation_required": {
          "type": "boolean",
          "description": "Set to true to prompt the user before permanent deletion."
        }
      },
      "required": ["search_term", "confirmation_required"]
    }
  }
}
```

---

## 5. The Database Layer (Vectors, Payloads & Isolation)

The database layer serves as both the semantic search index and the primary document repository. It enforces multi-tenant isolation and manages high-throughput, low-latency lookups.

### 5.1 Architecture & Components
- **Repository Pattern (`VectorStore` Interface)**: Abstract class decoupling database vendor SDKs from business logic:
```python
class VectorStore(ABC):
    @abstractmethod
    async def upsert(self, id: str, vector: list[float], payload: dict) -> bool: ...
    @abstractmethod
    async def search(self, vector: list[float], filter_query: dict, limit: int) -> list[Record]: ...
    @abstractmethod
    async def delete(self, filter_query: dict) -> bool: ...
```
- **Database Adapters**:
  - **`PineconeAdapter`**: Uses the Pinecone Serverless REST/gRPC API for initial testing and zero-cost operation.
  - **`QdrantAdapter`**: Production adapter communicating via gRPC with a managed or self-hosted Qdrant cluster.
- **Embedding Specifications**:
  - **Model**: `text-embedding-3-small` (1536 dimensions) or `text-embedding-004` (768 dimensions).
  - **Distance Metric**: Cosine Similarity.
  - **Storage Requirement**: In-memory HNSW index backed by NVMe storage for millisecond vector nearest-neighbor traversal.

### 5.2 Document & Metadata Storage Schema
Every vector record holds its complete payload metadata inside the vector engine, avoiding the need for an external relational database during initial deployment.

#### Complete Record Payload Schema
```json
{
  "id": "vec_550e8400-e29b-41d4-a716-446655440000",
  "vector": [0.0124, -0.0452, 0.0891, "...(1536 float elements)..."],
  "payload": {
    "user_id": "usr_98a7f6c2",
    "item_id": "item_88fbc12a",
    "source_url": "https://www.instagram.com/reel/C8xYz12345/",
    "content_type": "video",
    "status": "INDEXED",
    "title": "Configuring Shadcn UI with Tailwind CSS",
    "summary": "Step-by-step guidance on initializing Shadcn UI in a Next.js project...",
    "key_entities": ["Shadcn UI", "Tailwind CSS", "Next.js"],
    "primary_topic": "Frontend Development",
    "raw_context": "Full raw transcript or cleaned markdown text goes here...",
    "hyde_questions": [
      "How do I set up Shadcn UI in a React project?",
      "What is the best component library to use with Tailwind CSS?"
    ],
    "created_at": "2026-10-11T13:30:15Z"
  }
}
```

#### Dead Report Error Record Schema
```json
{
  "id": "vec_dead_771a33b9",
  "vector": [0.0, 0.0, "...(dummy or zero vector)..."],
  "payload": {
    "user_id": "usr_98a7f6c2",
    "item_id": "item_failed_9921",
    "source_url": "https://www.instagram.com/reel/PRIVATE_OR_DELETED/",
    "content_type": "video",
    "status": "SCRAPE_FAILED",
    "error_code": "RESOURCE_PRIVATE",
    "created_at": "2026-10-11T13:31:00Z"
  }
}
```

### 5.3 Tenant Isolation & Search Execution
To guarantee that users never access another user's private data, all vector operations enforce hard metadata filtering at the database engine level.

#### Hard Filter Query Structure
```json
{
  "filter": {
    "must": [
      {
        "key": "user_id",
        "match": {
          "value": "usr_98a7f6c2"
        }
      },
      {
        "key": "status",
        "match": {
          "value": "INDEXED"
        }
      }
    ]
  }
}
```

---

## 6. End-to-End Sequence & Data Flow Paths

These four end-to-end execution paths define the entire runtime life cycle of data in the system.

### Summary Flow Paths
- **Flow A: Ingest Pipeline**:
  `Client ──► Gateway ──► Redis Queue ──► Dispatcher ──► gRPC Extractor ──► AI Structurer ──► Vector DB`
- **Flow B: Chat Retrieval**:
  `Client ──► Gateway ──► Redis History ──► Condenser ──► Vector Search ──► Jev Filter ──► Synthesizer ──► Client (SSE)`
- **Flow C: Deletion Flow**:
  `Client ──► Gateway ──► LLM Tool Call ──► Confirmation ──► Vector DB (Delete Filter)`
- **Flow D: Dead Report Recovery**:
  `Scraper Fail (3x) ──► DLQ ──► Vector DB (SCRAPE_FAILED) ──► Client Triage UI ──► Manual Resolve`

### Flow A: Asynchronous Content Ingestion & Indexing
```
[Client] -> POST /api/v1/ingest (URL + JWT)
    │
    ▼
[API Gateway]
    ├── Validate JWT
    ├── Construct QueueJobMessage
    ├── LPUSH to Redis: media-ingestion-queue
    └── Return 202 Accepted to [Client]
          │
          ▼
[Task Dispatcher] (Pops from Redis)
    │
    ▼
[Media Extraction Microservice] (gRPC)
    ├── Invoke YouTubeWorker / SocialWorker / WebArticleWorker
    ├── Bypass bot guards / Extract raw transcript
    └── Return MediaExtractResponse (Protobuf)
          │
          ▼
[AI Layer: Structurer & HyDE] (via LiteLLM Proxy)
    ├── Call Structurer LLM -> Generate StructuredContentJSON
    ├── Call HyDE LLM -> Generate 3-5 Hypothetical Questions
    └── Call Embedding Model -> Generate 1536-dim vector for summary + HyDE
          │
          ▼
[Database Layer]
    └── VectorStore.upsert(id, vector, payload: { user_id, source_url, status: "INDEXED", ... })
```

### Flow B: Conversational Retrieval & SSE Streaming
```
[Client] -> POST /api/v1/chat (User Query + session_id)
    │
    ▼
[API Gateway]
    ├── Fetch last 5 messages from Redis (chat:history:{user_id}:{session_id})
    │
    ▼
[AI Layer: Query Condenser]
    ├── Send (History + User Query) to LiteLLM Proxy
    └── Receive condensed standalone query (e.g., "Shadcn UI documentation and setup")
          │
          ▼
[Database Layer: Vector Search]
    ├── Embed standalone query via LiteLLM
    ├── VectorStore.search(query_vector, filter: { user_id: current_user, status: "INDEXED" }, limit: 20)
    └── Return Top 20 raw candidate records
          │
          ▼
[AI Layer: Jev Discriminative Filter]
    ├── For each candidate: Evaluate (Candidate Chunk, Query) via Jev API
    ├── Receive Noul scalar probability scores [0.0 - 1.0]
    └── Filter candidates: Keep only chunks where score >= 0.80 (Select Top 5)
          │
          ▼
[AI Layer: Streaming Synthesizer]
    ├── Assemble Prompt: System Prompt + Top 5 Verified Chunks + Chat History
    ├── Call Synthesizer LLM with streaming enabled
    │
    ▼ (Server-Sent Events: text/event-stream)
[API Gateway] ---> Pipes SSE stream token-by-token ---> [Client Markdown UI]
    │
    ▼ (On Stream Complete)
[API Gateway]
    └── RPUSH user message and completed assistant response to Redis chat history
```

### Flow C: Interactive Memory Deletion via LLM Tool Calling
```
[Client] -> "Delete that note I saved about Shadcn UI"
    │
    ▼
[API Gateway] -> Calls Synthesizer with `delete_memory` tool definition
    │
    ▼
[AI Synthesizer] -> Emits Tool Call: delete_memory(search_term="Shadcn UI", confirmation_required=true)
    │
    ▼
[API Gateway]
    ├── Intercepts tool call (halts text output)
    └── Emits confirmation SSE event to Client: 
        `event: confirm_action, data: {"action_id": "act_991", "message": "Delete note: 'Configuring Shadcn UI'?"}`
          │
          ▼
[Client UI] -> Shows confirmation modal -> User taps "Confirm Delete"
    │
    ▼
[Client] -> POST /api/v1/actions/act_991/confirm
    │
    ▼
[API Gateway]
    ├── VectorStore.delete(filter: { user_id: current_user, search_term: "Shadcn UI" })
    └── Returns 200 OK -> Client removes item from view
```

### Flow D: Scrape Failure, DLQ, and Dead Reports Manual Recovery
```
[Extractor Microservice] -> Scrape attempt fails (e.g., HTTP 403 / Bot Block)
    │
    ├── Retry 1 (delay: 30s) -> Fails
    ├── Retry 2 (delay: 60s) -> Fails
    └── Retry 3 (delay: 120s) -> Fails
          │
          ▼
[Task Dispatcher]
    ├── Routes payload to dlq-failed-ingestion
    └── Calls VectorStore.upsert with payload:
        { user_id: "usr_...", status: "SCRAPE_FAILED", source_url: "...", error_code: "RATE_LIMITED" }
          │
          ▼
[Client UI: Dead Reports Screen]
    ├── Calls GET /api/v1/dead-reports -> Fetches records with status: "SCRAPE_FAILED"
    ├── User clicks failed item and pastes the manual transcript or video summary
    └── Calls POST /api/v1/dead-reports/:id/resolve (includes pasted raw text)
          │
          ▼
[API Gateway]
    └── Directly routes pasted text to AI Structurer (Flow A) -> Indexes vector -> Removes failed status
```

---

# Roadmap & Tasks

- [x] **V1_001: Improving the Onboarding**
  - [x] Standardize git commit format with `GENAI=YES/NO` convention in `RULES.md` and `GEMINI.md`
  - [x] Establish post-task summary generation (`GIT.md`) and archiving into `docs/history/`
  - [x] Configure `.gitignore` to ignore root `GIT.md`
  - [x] Establish sprint tracking workflow (`SPRINT.md`) and task registration policy in `docs/PROGRESS.md`
  - [x] Add pre-commit policy violation guardrails and task ID validation
  - [x] Create `git-commit` skill (`.agents/skills/git-commit/SKILL.md`) enforcing pre-commit verification and archiving
  - [x] Document system overview and phased technical architecture in `docs/PROGRESS.md`
  - [x] Design multi-layer high-level architecture specification and PlantUML diagram (`docs/diagrams/high_level_design.puml`)
  - [x] Break down architectural phases into roadmap tasks (`V1_002` to `V1_006`)

- [ ] **V1_002: Ingestion & Queuing Layer**
  - [ ] Implement mobile OS share target integration to extract shared URL and `user_id`
  - [ ] Add direct manual text note entry within the mobile client
  - [ ] Implement lightweight ingestion API Gateway endpoint (FastAPI / Node.js)
  - [ ] Integrate message queue (Redis / SQS / RabbitMQ) for payload queueing
  - [ ] Implement non-blocking `200 OK` immediate response on enqueue

- [ ] **V1_003: Worker Dispatch & Content Scraping Pipeline**
  - [ ] Build background Dispatcher service consuming from message queue
  - [ ] Implement URL Router delegating payloads to format-specific workers
  - [ ] Implement `YT_Worker` for YouTube Shorts / video download and caption extraction
  - [ ] Implement `IG_Worker` for Instagram Reels / post media, caption, and comment scraping
  - [ ] Implement `Web_Worker` for web articles / blogs markdown extraction (headless browser / Firecrawl)
  - [ ] Implement rate-limit detection and backoff retry queueing (`retry later` tag)

- [ ] **V1_004: Multimodal AI Structuring & HyDE Generation**
  - [ ] Implement audio transcription pipeline using Whisper
  - [ ] Implement frame extraction and Vision-Language Model processing (LLaVA / GPT-4o)
  - [ ] Implement text chunking and cleaning for web articles and manual notes
  - [ ] Build multimodal LLM fusion structuring into standard JSON schema (`title`, `summary`, `key_entities`, `original_transcript`)
  - [ ] Implement HyDE generation module (synthesizing 3–5 hypothetical questions per content item)

- [ ] **V1_005: Unified Vector Embedding & Persistent Storage**
  - [ ] Implement chunking and embedding generation using `text-embedding-3-small` for summaries and HyDE questions
  - [ ] Configure and initialize Vector Database collections/indexes (Pinecone / Qdrant)
  - [ ] Persist vectors with comprehensive metadata (`source_url`, `content_type`, `timestamp_saved`, `user_id`, `creator_name`)
  - [ ] Implement job state tracker to mark background jobs as `Complete`

- [ ] **V1_006: Conversational Retrieval Engine & Source Attribution**
  - [ ] Implement user query embedding pipeline
  - [ ] Implement hybrid search querying both content summaries and synthetic HyDE questions
  - [ ] Implement conversational answer synthesis grounding responses in retrieved context
  - [ ] Build source attribution module generating direct reference links to original media/URLs
