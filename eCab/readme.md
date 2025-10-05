
## Overview

E-Cabinet and BO are independent board meeting products. E-Cabinet lacks AI; it embeds BO's chatbot for enhanced Q&A on board-related documents/data. The requirement is to seamlessly embed a web-based chatbot SDK without impacting E-Cabinet's core functionality. The solution should ensure that E-Cabinet continues to be used for non-AI features, while leveraging BO's chatbot via data replication and embedding.

The solution will be deployed in DEV, QA, PROD on separate bare metal clusters. All components (LLM, Azure Speech Recognition/Synthesis equivalents, E-Cabinet/BO backends, web components) hosted on-premises on bare metal servers. No cloud dependencies; use containerized on-prem alternatives for speech services.

Both systems support multiple tenants, with isolation via tenant IDs in APIs, databases, and WebSocket rooms/namespaces.

This document describes the architecture of an **embedded chatbot SDK** that enables **text**, **voice-to-text**, and **voice-to-voice** conversational interactions inside **E-Cabinet** (the host app), which communicates with **BO** (the chat/meeting/AI platform). The solution supports full meeting lifecycles (pre-meeting, in-meeting, post-meeting) and is designed for **multi-tenant, on-prem bare-metal deployment** across DEV/QA/PROD environments.

---

## Objectives
  * Minimal changes to E-Cabinet’s UI, with a lightweight web SDK that handles all chat and voice features.
  * Secure, standards-based authentication -- TODO.
  * Use of RAG to ground the bot responses in uploaded meeting documents.
  * Real-time STT and TTS features.
  * Support all meeting lifecycle phases: **pre-meeting**, **in-meeting**, and **post-meeting**.
  * PMO tenant isolation (data, indices, access) and environment separation.
  * Observability, scalability, and fault tolerance.
  * Reuse existing BO backend services with minimal changes.

---


## Architecture

The architecture comprises the following major components:

* **BO SDK (in E-Cabinet Frontend)**
  Embedded JavaScript (jQuery-based) that runs in the browser context of E-Cabinet. It handles token fetch, WebSocket connection, voice capture/playback, REST API wrappers.

* **E-Cabinet Backend (ASP.NET)**
  The host application backend provides user login, session management (via OIDC), and a token endpoint for issuing short-lived tokens for BO. It also replicates document uploads, metadata, and meeting-related requests to BO Backend using REST API calls or Message Queues.

* **E-Cabinet Identity Provider (IdP)**
  A centralized IdP that issues ID Tokens / access tokens that are trusted by both E-Cabinet and BO. Details TBD.

* **BO Backend (Node/Python + Socket.io)**
  The core meeting and chat engine. It validates tokens, processes WebSocket chat messages, interacts with REST APIs for meetings, files, insights, action items, etc.

* **WebSocket Gateway (socket.io)**
  A real-time channel for chat messages, heartbeat, “join_room” and other meeting events. It receives the initial handshake (`40{token, userId}`), validates, then routes events to chat services.

* **Document Services**
  This service receives document upload events, fetches blob data, chunks, generates embeddings, and indexes them into a **Vector DB** under tenant-scoped namespaces.

* **Vector DB + Retriever**
  A vector store (Weaviate) that holds document embeddings and supports similarity search queries for RAG.

* **Embedding Service**
  An internal service (on-prem) that transforms text (document chunks) into embeddings, for use by the Document Ingestor or incremental updates.

* **LLM Inference Cluster**
  The Arabic language model (JAIS) serving layer that receives prompts (user + retrieved context) and returns generated answers (possibly streaming).

* **Azure Speech Recognition Engine**
  The speech pipeline, housed on-prem.
  * STT: converts audio chunks/streams into text.
  * TTS: converts generated text responses into audio to play back to the user.

* **Storage & Data Stores**
  * File / object store for meeting attachments, transcripts, audio buffers, models.
  * Multi-tenant document DB for meeting/metadata/account state.
  * Cache (Redis) for ephemeral state, session info.
  * Logging and metrics (EFK, Prometheus)

* **CI/CD, Secrets & Infra**
  Tools for deployment, secret management, cluster orchestration, backups, monitoring integration, etc.

### Authentication
Users log in to E-Cabinet as usual. The embedded BO SDK sends a user token (Bearer) and userId to BO backend during initialization. BO backend validates the token against E-Cabinet's IdP (Identity Provider). Once authenticated, users can use meeting features (including in-meeting chat).
  * E-Cabinet and BO are configured as **relying parties** (clients) in the Identity Provider (IdP). When a user logs into E-Cabinet, the IdP issues an **ID Token** (JWT) that includes standard claims.
  * The SDK in the browser gets a token after verifying session.
  * SDK opens a Socket.IO connection to BO’s `wsUrl`.
  * The platform responds with the initial handshake.
  * The SDK sends the initial handshake frame as 40{"token": "<id_token>", "userId": "<sub>"} via Socket.IO.
  * BO validates the token. On success, it sends back success and opens the chat session.
  * If the token expires, the SDK may refresh (via the token endpoint).

### Document Sync and Query Handling
  * Whenever E-Cabinet uploads a document, the E-Cabinet backend publishes a document ingest event to the Message Queue, including metadata like tenant_id, meeting_id, and blob URL.
  * The **Document Listener** fetches the blob from E-Cabinet’s object store, chunks it, and sends chunks to the Embedding Service.
  * Embedding results are upserted into the **Vector DB**, scoped by `tenant_id`.
  * When indexing is complete, BO marks documents as ready for RAG retrieval.
  * On chat message receipt, BO calls:
    1. `vectorDB.search(queryEmbedding, topK)`
    2. Retrieve relevant contexts
    3. Build prompt: user query + retrieved contexts + system instructions
    4. Send to LLM
    5. Return LLM output as chatbot reply
  * For voice input paths, the same flow is used after STT and transcript.

### Chatbot Feature
An embeddable SDK for E-Cabinet's web app, that provides AI-powered chatbot features only (no full BO product integration). It supports:
  * **Text-to-text** messaging.
  * **Voice-to-text** (speech recognition for input).
  * **Voice-to-voice** (speech recognition for input and speech synthesis for output).

Chat interactions occur over secure WebSocket (socket.io). Sample exchange includes:
  * Handshake (e.g., sid, pingInterval).
  * Token submission.
  * Joining rooms (group/private modes via chatMode in metadata).
  * Starting chat sessions.
  * Ping/pong for keep-alive.
  * Messages include metadata like meetingType:"in-meeting".
BO backend provides RAG-based LLM responses using previously replicated documents.

### Meeting Features
E-Cabinet handles user operations (e.g., logins, create meeting, document upload) updates its own backend, and replicates necessary data to BO's backend. Documents uploaded to E-Cabinet are stored locally and pushed to BO (via API or queue) for indexing into a vector DB. This ensures BO has access to documents for RAG in the embedded chatbot.

#### Pre-Meeting
  * **Create / Update Drafts**: The SDK calls `POST /meeting-drafts` or `PATCH /meetings/{id}` to establish meeting metadata (title, agenda, participants).
  * **Upload Documents**: Files uploaded from E-Cabinet via the SDK are sent to BO with `POST /files/upload/azure` (multipart/form-data) and stored in object store.
  * **Insights / Agenda Suggestions**: SDK may invoke `GET /meetings/{meetingId}/get-insights` to retrieve AI-generated agenda items or summaries.

### In-Meeting (Live chat & voice)
  * **Start the Meeting**: Using `PATCH /meetings/{meetingId}` or equivalent to mark meeting status “started.”
  * **Chat (Text)**: SDK emits chat messages via WebSocket events (`"chat_message"` or `"42[...]` frames) to BO, which persists and broadcasts.
  * **Voice → Text**: SDK captures microphone audio, streams or posts to STT engine → receives transcript → sends as chat message.
  * **Bot Response + RAG**: BO receives message, queries the Vector DB for relevant document context, constructs prompt, calls LLM, and returns the answer (via WebSocket).
  * **Text → Voice**: If configured, BO sends chatbot replies over TTS endpoint; SDK receives audio, decodes, and plays back.
  * **Action Items**: SDK uses `POST /action-items` to create tasks mid-meeting; or poll via `GET /action-items/meeting/{meetingId}`.

### Post-Meeting
  * **Summaries**: E-Cabinet calls `POST /internal/meetings/summary` (or similar) to persist meeting summary.
  * **Retrieve Meeting Metadata**: `GET /meetings/{meetingId}` returns full meeting record including summary.
  * **Action Items**: `GET /action-items/meeting/{meetingId}` to list them; `PUT /action-items/status` to update statuses.
  * **Download Files**: `POST /files/download/azure` to get meeting attachments (blobs or presigned URLs).

Note: All calls include Authorization: Bearer <token> and tenant_id in headers/body.


### Overall Architecture

```mermaid
flowchart LR
  subgraph UserSide[E-Cabinet App on User Browser]
    UI[jQuery UI]
    SDK[Chatbot SDK]
    Mic[Mic / Speaker Access]
  end

  subgraph E-Cabinet[E-Cabinet Backend]
    E-CabinetAPI[REST APIs]
    E-CabinetAuth[Auth Middleware]
  end

  subgraph IdP[E-Cabinet Identity Provider]
    Auth[< TBD >]
  end

  subgraph BO[On-Prem BO Backend]
    MQ[Message Queue]
    Listener[Queue Listener]
    BOAPI[REST APIs]
    WS[WebSocket]
    JWTVal[Auth Validator]
  end

  subgraph AI[On-Prem BO AI Services]
    TTS[Azure Speech Recognition]
    LLM[LLM - Jais]
    Embedding[Embedding Model]
    RAG[Vector DB + RAG]
  end

  subgraph Storage[On-Prem BO File & DB Storage]
    Docs[Document Store]
    DB[Data Store]
  end

  %% Connections
  UI --> SDK
  SDK -->|Login / Session| E-CabinetAPI
  E-CabinetAPI --> E-CabinetAuth --> Auth

  SDK -->|WebSocket + Token| WS
  WS --> JWTVal --> Auth

  E-CabinetAPI --> BOAPI

  BOAPI --> RAG
  RAG --> LLM

  BOAPI --> WS
  WS --> SDK

  E-CabinetAPI --> MQ --> Listener
  Listener --> DB
  Listener --> Docs

  %% Styling subgraphs
  style E-Cabinet fill:#eefcdc
  style IdP fill:#eefcdc
  style UserSide fill:#eefcdc

```

### Pre-Meeting Flow
User creates or updates a meeting, uploads files, generates agenda/insights.
```mermaid
sequenceDiagram
  participant User as User (Browser + SDK in E-Cabinet)
  participant E-Cabinet as E-Cabinet Backend
  participant IdP as E-Cabinet IdP
  participant BO as BO Backend

  User->>E-Cabinet: Login
  E-Cabinet->>IdP: Validate credentials
  IdP-->>E-Cabinet: ID Token / Access Token
  E-Cabinet-->>User: Session established

  User->>E-Cabinet: Create meeting draft
  E-Cabinet->>BO: POST /meeting-drafts
  BO-->>E-Cabinet: Meeting draft created

  User->>E-Cabinet: Upload agenda/docs
  E-Cabinet->>BO: POST /files/upload/azure
  BO-->>E-Cabinet: File uploaded (docId)

  User->>E-Cabinet: Request agenda insights
  E-Cabinet->>BO: GET /meetings/{meetingId}/get-insights
  BO-->>E-Cabinet: Insights response

  E-Cabinet-->>User: Show agenda, files, participants

```

### In-Meeting Flow
Meeting is started, users join chat, share docs, get live AI insights, and create action items.

#### Text Chat
```mermaid
sequenceDiagram
  participant User as User (Browser + SDK in E-Cabinet)
  participant E-Cabinet as E-Cabinet Backend
  participant BO as BO Backend
  participant WS as BO WebSocket (Chat)
  participant STT as ASR Engine (on-prem)
  participant RAG as Vector DB + Retriever
  participant LLM as JAIS LLM (on-prem)

  %% Meeting Start
  User->>E-Cabinet: Join meeting UI
  E-Cabinet->>BO: PATCH /meetings/{meetingId} (status=started)
  BO-->>E-Cabinet: Meeting marked started

  %% WebSocket Setup
  User->>WS: Connect WebSocket
  WS->>BO: Validate token
  BO-->>WS: Auth success
  WS-->>User: Connection established

  %% --- TEXT CHAT ---
  User->>WS: Send message "What’s next on agenda?"
  WS->>BO: Save chat message
  BO->>LLM: Prompt with user query + RAG context
  LLM-->>BO: Generated response
  BO-->>WS: Chatbot reply
  WS-->>User: Show response text

```
#### Voice Chat
```mermaid
sequenceDiagram
  participant User as User (Browser + SDK in E-Cabinet)
  participant E-Cabinet as E-Cabinet Backend
  participant BO as BO Backend
  participant WS as BO WebSocket (Chat)
  participant STT as ASR Engine (on-prem)
  participant RAG as Vector DB + Retriever
  participant LLM as JAIS LLM (on-prem)

  %% Meeting Start
  User->>E-Cabinet: Join meeting UI
  E-Cabinet->>BO: PATCH /meetings/{meetingId} (status=started)
  BO-->>E-Cabinet: Meeting marked started

  %% WebSocket Setup
  User->>WS: Connect WebSocket
  WS->>BO: Validate token
  BO-->>WS: Auth success
  WS-->>User: Connection established

  %% --- VOICE CHAT (Voice to Text) ---
  User->>SDK: Speak question via mic
  SDK->>STT: Stream audio
  STT-->>SDK: Transcript text
  SDK->>WS: Send transcript text
  WS->>BO: Store message + process
  BO->>RAG: Retrieve context
  RAG-->>BO: Relevant docs
  BO->>LLM: Generate answer
  LLM-->>BO: Response text
  BO-->>WS: Deliver chatbot text response
  WS-->>User: Show response text

  %% --- VOICE TO VOICE ---
  alt Voice reply enabled
    BO->>STT: Convert chatbot text to audio
    STT-->>SDK: Audio stream
    SDK-->>User: Play voice response
  end

```

### Post-Meeting Flow
MoM, action items and follow-ups.
```mermaid
sequenceDiagram
  participant User as User (Browser + SDK in E-Cabinet)
  participant E-Cabinet as E-Cabinet Backend
  participant BO as BO Backend

  E-Cabinet->>BO: POST /internal/meetings/summary
  BO-->>E-Cabinet: Summary saved

  User->>E-Cabinet: Retrieve meeting summary
  E-Cabinet->>BO: GET /meetings/{meetingId}
  BO-->>E-Cabinet: Meeting details + summary

  User->>E-Cabinet: Fetch action items
  E-Cabinet->>BO: GET /action-items/meeting/{meetingId}
  BO-->>E-Cabinet: Action items list

  User->>E-Cabinet: Update action item status
  E-Cabinet->>BO: PUT /action-items/status
  BO-->>E-Cabinet: Status updated

```
---
## Multi-Tenancy & Environment Isolation


---

## Deployment architecture
<TBD>

---

## CI/CD Pipeline
<TBD>

---

## Hardware Sizing
<TBD>

---
## Networking and Security
<TBD>

---
## Backup, DR & Data retention
<TBD>

---
## Observability
<TBD>

---
## Glossary & Definitions
<TBD>

---