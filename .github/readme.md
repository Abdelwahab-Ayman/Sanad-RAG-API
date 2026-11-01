# Sanad (سند)

**Turn documents into answers you can verify.**

Sanad is an Arabic-first Retrieval-Augmented Generation (RAG) platform.
Users upload documents, ask questions in natural language, and get answers
grounded in their own files, with citations (file name + page) so every
answer can be checked against its source.

## Why Sanad?
Generic chatbots answer from what the model memorized, and they hallucinate.
Sanad answers only from the uploaded documents, and when the documents don't
contain enough evidence, it says so instead of guessing.

## Features
- Document upload with background processing (text extraction, chunking, embeddings)
- Arabic-aware semantic search using vector similarity (pgvector)
- RAG-based Q&A with streaming responses (SSE)
- Source citations for every answer
- Insufficient-evidence handling: "I don't know" instead of hallucination
- JWT authentication with per-user document isolation
- Dockerized setup with CI via GitHub Actions

## Tech Stack
| Layer | Technology |
|---|---|
| API | ASP.NET Core Web API, C# |
| AI orchestration | Semantic Kernel / Microsoft.Extensions.AI |
| Database | PostgreSQL + pgvector, EF Core |
| Background jobs | Hangfire |
| Auth | ASP.NET Core Identity + JWT |
| DevOps | Docker, GitHub Actions |

## How It Works
1. **Ingest**: upload a file, extract text, split into chunks, generate embeddings, store vectors.
2. **Retrieve**: embed the question, find the most similar chunks.
3. **Generate**: build a prompt from the retrieved chunks and ask the LLM to answer only from them.
4. **Cite**: return the answer with the source file and page for each chunk used.

## Architecture
## Database Design

```mermaid
erDiagram
    USER ||--o{ DOCUMENT : uploads
    USER ||--o{ CONVERSATION : owns
    DOCUMENT ||--o{ CHUNK : "split into"
    CONVERSATION ||--o{ MESSAGE : contains
    MESSAGE ||--o{ MESSAGE_SOURCE : cites
    CHUNK ||--o{ MESSAGE_SOURCE : "used in"

    USER {
        guid Id PK
        string Email
        string PasswordHash
        datetime CreatedAt
    }
    DOCUMENT {
        guid Id PK
        guid UserId FK
        string FileName
        string StoragePath
        string ContentType
        long SizeBytes
        int Status
        string FailureReason
        int PageCount
        datetime UploadedAt
    }
    CHUNK {
        guid Id PK
        guid DocumentId FK
        int ChunkIndex
        int PageNumber
        string Text
        vector Embedding
    }
    CONVERSATION {
        guid Id PK
        guid UserId FK
        string Title
        datetime CreatedAt
    }
    MESSAGE {
        guid Id PK
        guid ConversationId FK
        int Role
        string Content
        bool InsufficientEvidence
        datetime CreatedAt
    }
    MESSAGE_SOURCE {
        guid Id PK
        guid MessageId FK
        guid ChunkId FK
        float Score
    }
```

## Live Demo
(link here)

## Evaluation
Tested on 20 hand-written Arabic questions with known answers:
- Correct answer: X/20
- Correct source cited: X/20
- Correctly refused when no answer exists: X/5

## Getting Started
(docker compose up instructions here)

## Roadmap
- Multi-tenant workspaces
- Conversation history
- Support for more file types
