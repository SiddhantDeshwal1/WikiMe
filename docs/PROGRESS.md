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

- [ ] **The App**: Build a Universal OS-level Share Target (`ShareTargetModule`). When a user taps "Share" on their phone, the app captures the payload, extracting the URL and `user_id`.
  - **Fire-and-Forget UX**: The client immediately displays a non-blocking toast (*"Saved! Will be searchable shortly."*) and closes the share sheet instantly without waiting or polling.
  - **Quick Thoughts**: Provide direct manual text entry (`ManualNoteWidget`) for fleeting thoughts, rules, or preferences (e.g., _"I prefer Shadcn over Tailwind"_).
- [ ] **API Gateway (Sanitization & Idempotency)**:
  - **URL Sanitization**: Strip tracking parameters, session IDs, and deep-link fragments (e.g., removing `?igsh=...`, `?utm_source=...`, `&t=12s`) to compute a clean `canonical_url`.
  - **Idempotency Check**: Rapid metadata query in Vector DB with filter: `{"user_id": "usr_...", "canonical_url": "..."}`.
  - **Bypass Logic**: If already indexed, immediately return `200 OK (Already Indexed)` and skip queueing, scraping, and embedding.
- [ ] **Message Queue (MQ)**: If unindexed, push payload envelope into `media-ingestion-queue` (Redis / BullMQ). API Gateway immediately returns `202 Accepted` (`{"job_id": "...", "status": "queued"}`).

### Phase 2: Dispatch & Scraping (Asynchronous Workers - Zero Video Download)

- [ ] **The Dispatcher**: Background worker listens to `media-ingestion-queue`, pops the next job envelope, and inspects the domain.
- [ ] **The Router & Extraction Microservices**: Video is **never** downloaded. URL is delegated to dedicated, platform-specific extraction services that return clean text, transcripts, and metadata:
  - [ ] `YT_Worker`: Parses YouTube Shorts and videos, extracting auto-generated captions and transcripts via platform APIs / `yt-dlp` metadata wrappers without downloading `.mp4` video files.
  - [ ] `SocialWorker`: Handles Instagram Reels, TikTok, and social posts. Calls managed scraping APIs to bypass bot challenges, returning post captions, creator metadata, and audio transcripts without downloading raw video media.
  - [ ] `Web_Worker`: Handles web articles and blogs using headless browser / Firecrawl / Readability to extract clean Markdown text, stripping navigation, ads, and boilerplate.
- [ ] **Retry Queue & Dead-Letter Queue (DLQ)**:
  - Implement exponential backoff retry ($2^{\text{retry\_count}} \times 30\text{s}$) up to 3 attempts on rate limits (HTTP 429) or transient failures.
  - If retries exceed 3, route payload to `dlq-failed-ingestion`. The DLQ worker writes a stub record into Vector DB with metadata `status: "SCRAPE_FAILED"`, `error_code`, `source_url`, and `user_id` for client triage.

### Phase 3: The AI Structurer (Text & Transcript Processing)

- [ ] **Text & Transcript Ingestion**: Pass extracted text or transcript through language processing pipelines (no video frame downloading or vision processing required).
- [ ] **The Chunking Strategy (Small-to-Big Preparation)**: For long articles or extensive transcripts (e.g., 4,000-word articles), split the raw text into overlapping 300–500 token chunks.
- [ ] **Structuring**: Use an LLM to synthesize messy transcripts, captions, and text into a standardized JSON object:
  - `title`: Synthesized or extracted headline.
  - `summary`: Comprehensive summary of the content.
  - `key_entities`: Core topics, tools, concepts, or entities mentioned.
  - `primary_topic`: Overarching classification domain.
  - `original_context`: Full transcript or cleaned article markdown.
- [ ] **HyDE Generation (Hypothetical Document Embeddings)**: Prompt LLM to generate 3–5 hypothetical questions answered by this content to maximize semantic retrieval alignment.

### Phase 4: Embedding & Storage (Parent-Child Retrieval Model)

- [ ] **Parent-Child Vectorization Split**: Instead of a single vector per saved item, generate multiple independent vector embeddings using `text-embedding-3-small`:
  - 1 vector for the overall summary.
  - 1 vector for each individual 300–500 token chunk of the text/article.
  - 1 vector for each of the 3–5 HyDE questions.
- [ ] **Metadata Linking Mechanism**: Every child vector is tagged with the exact same `item_id`, `user_id`, and `canonical_url`. The heavy "Parent" JSON payload (full summary and original context) is attached to the primary Summary vector (or minimally duplicated across chunks).
- [ ] **Vector DB Insertion**: Persist vectors into Vector DB (Pinecone / Qdrant) enforcing user-level isolation (`user_id`).
- [ ] **Fire-and-Forget Job Completion**: Mark job as complete asynchronously in the database. In keeping with the Lean MVP, no active WebSockets or push notifications are required; client queries only failed jobs via the Dead Reports dashboard.

### Phase 5: Conversational Retrieval Engine (Local SLM + Jev)

- [ ] **Query Condensation**: For multi-turn chats, use a fast LLM to rewrite user queries into standalone, unambiguous retrieval queries resolving all coreferences.
- [ ] **Parent-Child Deduplicated Search**: Execute similarity search across child vectors (summaries, chunks, and HyDE questions). Group vector hits by `item_id`, deduplicate matches, and retrieve the single comprehensive "Parent" JSON payload for each matching item.
- [ ] **Jev Discriminative Filtering**: Evaluate retrieved candidate parent chunks using the TypeSafe AI Jev API (`Noul` probability scoring $\ge 0.80$) to eliminate hallucinations and isolate the top verified chunks.
- [ ] **SLM Synthesis**: Pass verified parent context chunks, chat history, and prompt to a conversational model (e.g., Llama 3.1 8B via vLLM or Groq) to synthesize concise answers with source citations.
- [ ] **Streaming Output**: Stream tokens in real time via Server-Sent Events (SSE) to the client chat UI.

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
    - **Fire-and-Forget Lean UX**: Immediately shows an OS toast (*"Saved! Will be searchable shortly."*) and dismisses the share sheet upon dispatching the request. Does not keep active WebSockets or poll for completion.
  - **Conversational Chat Interface (`ChatStreamView`)**: Markdown-enabled chat view supporting real-time Server-Sent Events (SSE) token rendering, source citation cards, and confirmation modal dialogs.
  - **Quick Thought Composer (`ManualNoteWidget`)**: Lightweight text input allowing users to store direct text constraints, preferences, and facts.
  - **Dead Reports Screen (`TriageDashboard`)**: UI view displaying ingest operations that failed scraping after all retries (`status: "SCRAPE_FAILED"`), allowing the user to view the failed URL and manually paste the raw text or transcript.
  - **Local Token & State Store (`SecureStorage`)**: Keychain/Keystore wrapper storing authentication tokens (JWTs) and active session IDs (`session_id`).

### 1.2 Data Structures & Client Schemas

#### Ingestion Share Payload (`SharePayloadDTO`)

```json
{
  "user_id": "usr_98a7f6c2",
  "source_url": "https://www.instagram.com/reel/C8xYz12345/?igsh=NzBmMjdhZWRiYQ==",
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

| Action              | Protocol                  | Endpoint                           | Headers                                                           | Success Response                                                                                         |
| :------------------ | :------------------------ | :--------------------------------- | :---------------------------------------------------------------- | :------------------------------------------------------------------------------------------------------- |
| Enqueue Content     | POST (HTTP/1.1 or HTTP/2) | `/api/v1/ingest`                   | Authorization: Bearer &lt;JWT&gt;, Content-Type: application/json | 202 Accepted (`{"job_id": "job_...", "status": "queued"}`) OR 200 OK (`{"status": "already_indexed"}`) |
| Conversational Chat | POST (Streaming)          | `/api/v1/chat`                     | Authorization: Bearer &lt;JWT&gt;, Accept: text/event-stream      | 200 OK (Chunked data: `{"token": "..."}\n\n`)                                                            |
| Fetch Dead Reports  | GET                       | `/api/v1/dead-reports`             | Authorization: Bearer &lt;JWT&gt;                                 | 200 OK (`{"failed_items": [...]}`)                                                                       |
| Resolve Dead Report | POST                      | `/api/v1/dead-reports/:id/resolve` | Authorization: Bearer &lt;JWT&gt;                                 | 200 OK (`{"status": "requeued"}`)                                                                        |

---

## 2. The Backend / Broker Layer (API Gateway & Queue Orchestrator)

The Backend orchestrates synchronous network traffic, enforces authentication, executes rapid URL sanitization and idempotency filtering, decouples long-running media processing via an asynchronous broker, and manages chat session memory.

### 2.1 Services & Components

- **API Gateway (`api-gateway`)**: Built using FastAPI or Node.js (NestJS/Express).
  - Handles CORS, rate limiting, request validation, and JWT verification.
  - **URL Sanitizer**: Strips query tracking parameters (`?igsh=...`, `?utm_source=...`, `&t=12s`, session hashes) to produce clean `canonical_url`.
  - **Idempotency Gate**: Intercepts payloads and performs a fast metadata lookup against the Vector DB (`user_id` + `canonical_url`) before enqueuing.
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
    "source_url": "https://www.instagram.com/reel/C8xYz12345/?igsh=NzBmMjdhZWRiYQ==",
    "canonical_url": "https://www.instagram.com/reel/C8xYz12345/",
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
  { "role": "user", "content": "I am looking for a React UI toolkit." },
  {
    "role": "assistant",
    "content": "You previously saved notes recommending Shadcn UI."
  },
  {
    "role": "user",
    "content": "What was the recommended frontend component library I saved recently?"
  }
]
```

### 2.3 Internal Flow Logic

- `POST /api/v1/ingest` validates incoming JWT against public auth keys.
- If payload contains `raw_text` (manual note), Gateway bypasses Phase 2 (Microservices) and pushes directly to Phase 3 (AI Structurer).
- If payload contains `source_url`:
  1. **URL Sanitization**: Strips tracking queries and fragments to compute `canonical_url`.
  2. **Idempotency Check**: Queries Vector DB metadata filter: `{"user_id": user_id, "canonical_url": canonical_url}`.
  3. **Bypass Logic**: If record already exists in Vector DB, Gateway immediately returns `200 OK` (`{"status": "already_indexed"}`) and completely bypasses the Redis queue, extraction, and embedding phases.
  4. **Enqueue**: If not found, Gateway constructs `QueueJobMessage` with `canonical_url`, pushes to Redis `media-ingestion-queue`, and returns `202 Accepted` (`{"job_id": job_id, "status": "queued"}`).

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

### 5.2 Document & Metadata Storage Schema (Parent-Child Retrieval Model)

To solve vector dilution and handle long articles without information loss, the database layer implements the **Parent-Child Retrieval Model (Small-to-Big Retrieval)**. 

#### 1. The Chunking Strategy
Raw text from long articles (e.g., 4,000 words) or extended transcripts is partitioned by the AI Structurer into overlapping 300–500 token chunks.

#### 2. The Vectorization Split
Instead of creating a single diluted vector per saved URL, the Embedding Layer creates multiple independent vectors for the same content item:
- **1 Summary Vector**: Embeds the high-level synthesis for thematic matching.
- **$N$ Chunk Vectors**: Embeds each individual 300–500 token chunk for exact technical or factual retrieval.
- **3–5 HyDE Vectors**: Embeds synthetic user questions for conversational intent matching.

#### 3. The Linking Mechanism (Metadata)
Every child vector is persisted in the Vector DB tagged with the exact same `item_id`, `canonical_url`, and `user_id`. The full "Parent" JSON payload (complete summary and full text) is attached to the primary summary vector:

```json
{
  "id": "vec_550e8400_summary",
  "vector": [0.0124, -0.0452, 0.0891, "...(1536 float elements)..."],
  "payload": {
    "item_id": "item_88fbc12a",
    "user_id": "usr_98a7f6c2",
    "vector_role": "summary",
    "source_url": "https://www.instagram.com/reel/C8xYz12345/?igsh=NzBmMjdhZWRiYQ==",
    "canonical_url": "https://www.instagram.com/reel/C8xYz12345/",
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

#### Chunk Child Vector Schema (`vector_role: "chunk"`)
```json
{
  "id": "vec_550e8400_chunk_03",
  "vector": [0.0512, 0.0118, -0.0763, "...(1536 float elements)..."],
  "payload": {
    "item_id": "item_88fbc12a",
    "user_id": "usr_98a7f6c2",
    "vector_role": "chunk",
    "chunk_index": 3,
    "chunk_text": "To configure Tailwind with Shadcn components, add the path to the components directory inside your tailwind.config.js content array...",
    "canonical_url": "https://www.instagram.com/reel/C8xYz12345/",
    "status": "INDEXED"
  }
}
```

#### HyDE Child Vector Schema (`vector_role: "hyde"`)
```json
{
  "id": "vec_550e8400_hyde_01",
  "vector": [-0.0219, 0.0815, 0.0411, "...(1536 float elements)..."],
  "payload": {
    "item_id": "item_88fbc12a",
    "user_id": "usr_98a7f6c2",
    "vector_role": "hyde",
    "question": "How do I set up Shadcn UI in a React project?",
    "canonical_url": "https://www.instagram.com/reel/C8xYz12345/",
    "status": "INDEXED"
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
    "source_url": "https://www.instagram.com/reel/PRIVATE_OR_DELETED/?igsh=xyz",
    "canonical_url": "https://www.instagram.com/reel/PRIVATE_OR_DELETED/",
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
  `Client ──► Gateway (Sanitize & Idempotency) ──► Redis Queue ──► Dispatcher ──► gRPC Extractor (No Video Download) ──► AI Structurer (Parent-Child) ──► Vector DB`
- **Flow B: Chat Retrieval**:
  `Client ──► Gateway ──► Redis History ──► Condenser ──► Vector Search ──► Deduplicate by item_id ──► Jev Filter ──► Synthesizer ──► Client (SSE)`
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
    ├── URL Sanitization: Strip tracking queries (?igsh=..., ?utm_source=...) -> canonical_url
    ├── Idempotency Check: Fast Vector DB query (user_id + canonical_url)
    │     ├── [If Found] -> Return 200 OK ({"status": "already_indexed"}) -> Halt
    │     └── [If Not Found] -> Continue
    ├── Construct QueueJobMessage with canonical_url
    ├── LPUSH to Redis: media-ingestion-queue
    └── Return 202 Accepted to [Client] (Client shows "Saved! Will be searchable shortly." toast)
          │
          ▼
[Task Dispatcher] (Pops from Redis)
    │
    ▼
[Media Extraction Microservice] (gRPC - Zero Video Download)
    ├── Invoke YouTubeWorker / SocialWorker / WebArticleWorker
    ├── Bypass bot guards / Extract raw transcript, captions, or clean markdown (Never downloads .mp4)
    └── Return MediaExtractResponse (Protobuf)
          │
          ▼
[AI Layer: Structurer, Chunker & HyDE] (via LiteLLM Proxy)
    ├── Call Structurer LLM -> Generate StructuredContentJSON (Summary, Entities, Topic)
    ├── Chunker: Split raw text/articles into overlapping 300-500 token chunks
    ├── Call HyDE LLM -> Generate 3-5 Hypothetical Questions
    └── Call Embedding Model -> Generate independent 1536-dim vectors:
          ├── 1x Summary Vector
          ├── Nx Chunk Vectors
          └── 3-5x HyDE Vectors
          │
          ▼
[Database Layer]
    └── VectorStore.upsert(all child vectors sharing item_id, canonical_url, user_id, status: "INDEXED")
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
    └── Return Top 20 raw candidate child records (may include chunks, HyDE questions, summaries)
          │
          ▼
[Deduplication Gate (Backend)]
    ├── Identify all unique item_ids across the Top 20 candidate child hits
    ├── Retrieve the unified "Parent" JSON payload for each unique item_id
    └── Produce deduplicated list of Parent context candidates (e.g., Top 5-8 distinct items)
          │
          ▼
[AI Layer: Jev Discriminative Filter]
    ├── For each candidate parent chunk: Evaluate (Parent Chunk, Query) via Jev API
    ├── Receive Noul scalar probability scores [0.0 - 1.0]
    └── Filter candidates: Keep only items where score >= 0.80 (Select Top 5)
          │
          ▼
[AI Layer: Streaming Synthesizer]
    ├── Assemble Prompt: System Prompt + Top 5 Verified Parent Chunks + Chat History
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

# System Implementation Phases & Steps

This section translates the complete high-level system architecture into seven modular, independent execution phases. Each phase is defined by its architectural scope, discrete actionable steps with unique step IDs and checkboxes (`- [ ]`), and strict verification criteria / expected outcomes.

---

## Phase 01: Database Layer & Parent-Child Vector Store Foundation (`PHASE_01`)

### Overview & Scope
Implements the core data repository layer, abstract vector store interfaces, multi-tenant isolation (`user_id`), the Parent-Child (Small-to-Big) multi-vector storage schema, and fast metadata filtering for URL idempotency and dead-letter failure logging.

### Dependencies & Independence
- **Prerequisites**: None (Foundational layer).
- **Independence**: Can be fully developed, mocked, and unit-tested in isolation using local/mock vector adapters before connecting upstream services.

### Actionable Implementation Steps
- [ ] **Step 1.1 [STEP_01_01]**: Define the abstract `VectorStore` repository interface (`upsert`, `search`, `delete`, `query_by_metadata`).
- [ ] **Step 1.2 [STEP_01_02]**: Implement the `PineconeAdapter` and `QdrantAdapter` supporting cosine distance metric, 1536-dimensional embeddings (`text-embedding-3-small`), and HNSW indexing.
- [ ] **Step 1.3 [STEP_01_03]**: Implement the Parent-Child vector storage schema:
  - Summary Vector (`vector_role: "summary"`) carrying the primary document payload (`title`, `summary`, `key_entities`, `primary_topic`, `raw_context`, `hyde_questions`).
  - Chunk Child Vectors (`vector_role: "chunk"`) carrying `chunk_index`, `chunk_text`, and linking metadata.
  - HyDE Child Vectors (`vector_role: "hyde"`) carrying `question` and linking metadata.
- [ ] **Step 1.4 [STEP_01_04]**: Implement strict multi-tenant metadata filtering ensuring all queries enforce `user_id` and document status (`INDEXED` vs `SCRAPE_FAILED`).
- [ ] **Step 1.5 [STEP_01_05]**: Build rapid metadata lookup indexes on `canonical_url` + `user_id` for instantaneous upstream idempotency checks.
- [ ] **Step 1.6 [STEP_01_06]**: Write automated test suites validating multi-vector upsert, parent payload retrieval by `item_id`, idempotency queries, and hard-delete purges.

### Expected Outcome (Definition of Done)
A standalone, fully tested database repository layer capable of persisting child vectors linked by `item_id`, querying parent payloads without relational database overhead, enforcing strict tenant isolation, and validating URL indexing state in sub-50ms.

---

## Phase 02: Ingestion Edge, API Gateway & Message Queue Broker (`PHASE_02`)

### Overview & Scope
Establishes the synchronous entry point of the system. Implements the API Gateway with JWT authentication, URL sanitization (tracking parameter stripping), rapid Vector DB idempotency bypass, and enqueuing jobs to Redis with non-blocking client responses.

### Dependencies & Independence
- **Prerequisites**: `PHASE_01` (for idempotency metadata checks).
- **Independence**: Focuses exclusively on network ingress, payload sanitization, caching, and queuing; downstream extraction microservices can be mocked.

### Actionable Implementation Steps
- [ ] **Step 2.1 [STEP_02_01]**: Initialize API Gateway service (FastAPI / Node.js) with CORS headers, rate-limiting guards, and JWT authentication middleware.
- [ ] **Step 2.2 [STEP_02_02]**: Implement URL Sanitizer utility to normalize incoming URLs into clean `canonical_url` representations by stripping tracking parameters (`?igsh=...`, `?utm_source=...`, `&t=12s`, session tokens, and deep-link fragments).
- [ ] **Step 2.3 [STEP_02_03]**: Implement Idempotency Gate in the gateway: queries `VectorStore` (`user_id` + `canonical_url`) and returns `200 OK (Already Indexed)` immediately if previously indexed.
- [ ] **Step 2.4 [STEP_02_04]**: Configure Redis message broker hosting `media-ingestion-queue` and `dlq-failed-ingestion` using BullMQ or Celery with persistent storage (AOF/RDB).
- [ ] **Step 2.5 [STEP_02_05]**: Implement `POST /api/v1/ingest` endpoint:
  - Handles direct manual text notes (`raw_text`), routing directly to AI structuring.
  - Enqueues unindexed URL job envelopes (`QueueJobMessage`) with `canonical_url` into Redis and immediately returns `202 Accepted` (`{"job_id": "...", "status": "queued"}`).
- [ ] **Step 2.6 [STEP_02_06]**: Write integration tests simulating duplicate URL submissions, sanitization regex edge cases, and verifying Redis queue insertion latency (< 100ms).

### Expected Outcome (Definition of Done)
An authenticated, high-throughput API Gateway that halts duplicate URLs at the door, normalizes tracking links, buffers valid requests into Redis queues, and releases the client with non-blocking HTTP 202/200 acknowledgments.

---

## Phase 03: Distributed Extraction Microservices Cluster (Zero Video Download) (`PHASE_03`)

### Overview & Scope
Implements an isolated cluster of format-specific extraction workers communicating over gRPC that extract clean text, transcripts, captions, and metadata directly from URLs without ever downloading raw video media. Includes queue dispatching and exponential backoff retry mechanisms.

### Dependencies & Independence
- **Prerequisites**: `PHASE_02` (Redis queue contracts).
- **Independence**: Operates as standalone background microservices behind gRPC interfaces (`MediaExtractor.proto`); can be tested using mock URL batches without AI structuring.

### Actionable Implementation Steps
- [ ] **Step 3.1 [STEP_03_01]**: Define Protocol Buffer interface (`MediaExtractor.proto`) exposing `rpc ExtractMedia(MediaExtractRequest) returns (MediaExtractResponse)`.
- [ ] **Step 3.2 [STEP_03_02]**: Implement `YouTubeWorker`: extracts auto-generated captions, video titles, and creator transcripts via platform APIs and `yt-dlp` metadata extraction without downloading `.mp4` video streams.
- [ ] **Step 3.3 [STEP_03_03]**: Implement `SocialWorker`: extracts post captions, creator metadata, and audio transcripts for Instagram Reels and TikTok via managed anti-bot scraping APIs (Supadata / ScrapFly) without downloading video media.
- [ ] **Step 3.4 [STEP_03_04]**: Implement `WebArticleWorker`: extracts clean Markdown text from web links and blogs using headless browser / Firecrawl / Readability, stripping ads, scripts, and navigation chrome.
- [ ] **Step 3.5 [STEP_03_05]**: Implement `queue-dispatcher` worker consuming from Redis `media-ingestion-queue`, routing URLs by domain, and invoking the gRPC extraction service.
- [ ] **Step 3.6 [STEP_03_06]**: Implement exponential backoff retry handler ($2^{\text{retry\_count}} \times 30\text{s}$) up to 3 attempts on rate limits (HTTP 429) or transient network errors.
- [ ] **Step 3.7 [STEP_03_07]**: Implement Dead-Letter Queue coordinator (`dlq-handler`): routes exhausted jobs (retry $\ge$ 3) to `dlq-failed-ingestion` and writes a stub record into `VectorStore` with `status: "SCRAPE_FAILED"`, `error_code`, `source_url`, and `user_id`.

### Expected Outcome (Definition of Done)
A resilient microservice extraction cluster capable of scraping clean transcripts and markdown across YouTube, Instagram, TikTok, and web links with zero video download, backed by a persistent queue dispatcher and automated DLQ error logging.

---

## Phase 04: AI Structurer, Parent-Child Chunking & HyDE Vectorization (`PHASE_04`)

### Overview & Scope
Translates raw scraped text and transcripts into standardized JSON schemas, divides long content into overlapping token chunks, generates hypothetical retrieval questions (HyDE), and generates multi-vector embeddings for parent-child storage.

### Dependencies & Independence
- **Prerequisites**: `PHASE_01` (VectorStore) and `PHASE_03` (extracted text/transcripts).
- **Independence**: Can be evaluated and tuned independently using static text fixtures against LLM prompt benchmarks.

### Actionable Implementation Steps
- [ ] **Step 4.1 [STEP_04_01]**: Deploy LiteLLM Proxy Gateway (`litellm-proxy`) managing API key rotation, provider load balancing, and internal model aliases (`ebrain-structurer`, `ebrain-embedding`, `ebrain-synthesis`).
- [ ] **Step 4.2 [STEP_04_02]**: Implement Structurer LLM prompt formatting raw transcripts and markdown into strictly validated `StructuredExtractionJSON` (`title`, `summary`, `key_entities`, `primary_topic`, `original_context`).
- [ ] **Step 4.3 [STEP_04_03]**: Implement Parent-Child Chunker utility: splits raw text/articles into overlapping 300–500 token chunks with token-boundary preservation.
- [ ] **Step 4.4 [STEP_04_04]**: Implement HyDE Generator LLM prompt: synthesizes 3–5 high-probability hypothetical user questions answered by the content.
- [ ] **Step 4.5 [STEP_04_05]**: Implement multi-vector embedding pipeline using `text-embedding-3-small`:
  - 1x vector for the synthesized summary.
  - Nx vectors for the individual 300–500 token chunks.
  - 3–5x vectors for the synthetic HyDE questions.
- [ ] **Step 4.6 [STEP_04_06]**: Wire the AI Structurer, Chunker, and Embedding pipeline to the Dispatcher and `VectorStore`, completing end-to-end ingestion (Flow A).
- [ ] **Step 4.7 [STEP_04_07]**: Write end-to-end integration tests verifying that submitting a URL results in multi-vector points linked by `item_id` with `status: "INDEXED"` in `VectorStore`.

### Expected Outcome (Definition of Done)
An end-to-end automated ingestion pipeline (Flow A) that takes raw extracted content, normalizes schemas, chunks long text, generates synthetic retrieval vectors, and writes parent-child vector clusters into the database.

---

## Phase 05: Conversational Retrieval Engine, Jev Filtering & SSE Streaming (`PHASE_05`)

### Overview & Scope
Powers the interactive conversational recall experience. Manages multi-turn conversation context in Redis, resolves coreferences into standalone queries, executes parent-child vector retrieval with item-level deduplication, applies discriminative relevance filtering via TypeSafe AI Jev, and synthesizes grounded answers streamed over Server-Sent Events (SSE).

### Dependencies & Independence
- **Prerequisites**: `PHASE_01` (VectorStore) and `PHASE_04` (embedded knowledge base).
- **Independence**: Conversational retrieval endpoints can be tested independently against pre-indexed test vectors using synthetic chat dialogues.

### Actionable Implementation Steps
- [ ] **Step 5.1 [STEP_05_01]**: Configure Redis session cache (`redis-session-store`) managing sliding 5-turn chat history (`chat:history:{user_id}:{session_id}`) with 24-hour TTL.
- [ ] **Step 5.2 [STEP_05_02]**: Implement Query Condenser LLM prompt: rewrites multi-turn user follow-up questions into standalone, unambiguous search queries resolving all pronouns and coreferences.
- [ ] **Step 5.3 [STEP_05_03]**: Implement Vector Search execution: embeds standalone query via LiteLLM and queries `VectorStore` for Top 20 raw candidate child records with hard `user_id` isolation.
- [ ] **Step 5.4 [STEP_05_04]**: Implement Deduplication Gate: extracts unique `item_id`s across the Top 20 child hits, fetches the unified Parent JSON payload for each unique item, and consolidates parent context candidates.
- [ ] **Step 5.5 [STEP_05_05]**: Integrate TypeSafe AI Jev API discriminative filter: evaluates candidate parent chunks against the query using `Noul` probability scoring ($\ge 0.80$) to eliminate noise and isolate top 5 verified chunks.
- [ ] **Step 5.6 [STEP_05_06]**: Implement Streaming Synthesizer LLM prompt: grounds answer strictly on the verified parent chunks, adds custom guardrails, and appends formatted markdown source citations.
- [ ] **Step 5.7 [STEP_05_07]**: Implement `POST /api/v1/chat` streaming endpoint delivering tokens via Server-Sent Events (`text/event-stream`), committing completed turns to Redis history upon stream completion.

### Expected Outcome (Definition of Done)
A real-time conversational search and retrieval pipeline (Flow B) that correctly resolves pronouns across turns, retrieves and deduplicates parent context, eliminates hallucinations via Jev scoring, and streams grounded answers with source citations via SSE.

---

## Phase 06: Memory Management & Triage Dashboard APIs (`PHASE_06`)

### Overview & Scope
Delivers natural-language memory maintenance and administrative control. Enables users to forget or delete saved memories via LLM tool calling with confirmation handshakes, and provides endpoints for retrieving and resolving failed scrapes via the Dead Reports dashboard.

### Dependencies & Independence
- **Prerequisites**: `PHASE_01` (VectorStore deletion & DLQ records) and `PHASE_05` (LLM conversational runtime).
- **Independence**: Tool-calling and dead-report endpoints can be verified independently with targeted API integration tests.

### Actionable Implementation Steps
- [ ] **Step 6.1 [STEP_06_01]**: Define `delete_memory` function calling schema in Synthesizer model to detect user intent to remove or forget saved memories.
- [ ] **Step 6.2 [STEP_06_02]**: Implement Confirmation Handshake in API Gateway: intercepts `delete_memory` tool call, pauses text generation, and emits `confirm_action` SSE event with action token.
- [ ] **Step 6.3 [STEP_06_03]**: Implement `POST /api/v1/actions/:id/confirm` endpoint: purges all child and parent vector records matching the target `item_id` / metadata in `VectorStore`.
- [ ] **Step 6.4 [STEP_06_04]**: Implement `GET /api/v1/dead-reports` endpoint: queries `VectorStore` for records with `status: "SCRAPE_FAILED"` under the authenticated `user_id`.
- [ ] **Step 6.5 [STEP_06_05]**: Implement `POST /api/v1/dead-reports/:id/resolve` endpoint: accepts user-pasted manual text/transcripts, routes directly to AI Structurer (Flow A), re-indexes the memory as `status: "INDEXED"`, and purges the failure record.
- [ ] **Step 6.6 [STEP_06_06]**: Write automated tests for Flow C (delete confirmation cycle) and Flow D (DLQ record recovery).

### Expected Outcome (Definition of Done)
Fully functional operational APIs allowing conversational memory deletion with confirmation safety (Flow C), as well as dead-letter report inspection and manual text recovery (Flow D).

---

## Phase 07: Client Mobile App & OS Share Target Integration (`PHASE_07`)

### Overview & Scope
Builds the native/cross-platform client interface (`client-mobile`). Integrates OS-level share targets for friction-free capture, quick thought entry, markdown chat with real-time SSE streaming, and a Dead Reports triage dashboard.

### Dependencies & Independence
- **Prerequisites**: `PHASE_02`, `PHASE_05`, and `PHASE_06` backend endpoints.
- **Independence**: Can be developed concurrently with backend mocks, then pointed to live gateway endpoints.

### Actionable Implementation Steps
- [ ] **Step 7.1 [STEP_07_01]**: Implement OS Share Sheet Receiver (`ShareTargetModule`) in Android (`ACTION_SEND` intent filter) and iOS (`ShareViewController`) to extract shared URLs and text without launching the full app.
- [ ] **Step 7.2 [STEP_07_02]**: Implement Fire-and-Forget Share UX: show instant OS toast notification (*"Saved! Will be searchable shortly."*) upon enqueuing and immediately dismiss share sheet.
- [ ] **Step 7.3 [STEP_07_03]**: Implement Quick Thought composer widget (`ManualNoteWidget`) for capturing fleeting notes and direct constraints.
- [ ] **Step 7.4 [STEP_07_04]**: Implement Conversational Chat View (`ChatStreamView`) supporting markdown rendering, real-time SSE token streaming, source citation cards, and confirmation modal dialogs.
- [ ] **Step 7.5 [STEP_07_05]**: Implement Dead Reports Dashboard (`TriageDashboard`): lists failed ingest operations from `GET /api/v1/dead-reports` with input fields to paste manual transcripts/notes and trigger resolution.
- [ ] **Step 7.6 [STEP_07_06]**: Implement Local Token & State Store (`SecureStorage`) wrapping Keychain/Keystore for JWT authentication and session management.
- [ ] **Step 7.7 [STEP_07_07]**: Conduct end-to-end physical device testing across Android and iOS verifying share target speed (< 1s to dismiss), streaming recall, and triage resolution.

### Expected Outcome (Definition of Done)
A production-ready mobile application providing seamless OS share sheet ingestion, zero-lag fire-and-forget saving, real-time streaming conversational recall, and manual triage resolution for failed content.

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
  - [x] Refine architecture specifications with zero-video-download, parent-child retrieval, and fire-and-forget UX
  - [x] Update PlantUML high-level design diagram (`docs/diagrams/high_level_design.puml`) with idempotency check and deduplication flows
  - [x] Structure modular implementation phases and step-by-step checklists (`PHASE_01` to `PHASE_07`) in `docs/PROGRESS.md`

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



