# Project Analysis: Sanad (سند) RAG API

## Executive Summary

**Sanad** is an **Arabic-first Retrieval-Augmented Generation (RAG) platform** ASP.NET Core Web API. The vision (per README) is to let users upload documents, ask natural-language questions, and receive answers grounded in their own files with citations (file name + page), including graceful "I don't know" handling when evidence is insufficient.

**Current state:** The repository is a **partial skeleton / early-stage implementation**. The data layer (EF Core + pgvector entities, migrations, PDF text extraction) is in place, but the core RAG features (API endpoints, authentication, background processing, vector search, LLM orchestration, SSE streaming) described in the README are **not yet implemented**.

---

## Repository Layout

```
F:\Full Stack DEPI GRADUATION\Sanad-RAG-API-main\
├── .github/
│   └── workflows/
│       └── ci.yml                    # GitHub Actions CI (dotnet 10, build + test)
├── .gitignore                        # Standard Visual Studio / .NET gitignore
├── docker-compose.yml                # PostgreSQL 16 + pgvector (port 5432)
├── README.md                         # Architecture docs, ER diagram, roadmap
├── Sanad.slnx                        # Solution file (4 src + 1 test project)
├── spikes/                           # Experimental / proof-of-concept code
│   └── Spike.Pdf/                    # Console spike testing PdfPig + Arabic text
├── src/
│   ├── Sanad.Api/                    # ASP.NET Core Web API (net10.0)
│   │   ├── Program.cs                # Minimal host: DbContext + Swagger + PDF extractor DI
│   │   ├── appsettings.json
│   │   ├── appsettings.Development.json   # Connection string (localhost:5432)
│   │   ├── Properties/launchSettings.json
│   │   ├── Sanad.Api.http            # HTTP scratch file
│   │   └── Sanad.Api.csproj
│   ├── Sanad.Application/            # Application layer (net10.0 class library)
│   │   └── Interfaces/
│   │       └── IDocumentTextExtractor.cs
│   ├── Sanad.Domain/                 # Domain layer (net10.0 class library)
│   │   ├── Entities/
│   │   │   ├── Document.cs
│   │   │   └── Chunk.cs
│   │   ├── Enums/
│   │   │   └── DocumentStatus.cs
│   │   └── Sanad.Domain.csproj
│   └── Sanad.Infrastructure/         # Infrastructure layer (net10.0 class library)
│       ├── Documents/
│       │   └── PdfTextExtractor.cs   # PDF text extraction via PdfPig
│       ├── Persistence/
│       │   ├── DbContext/
│       │   │   └── SanadDbContext.cs # EF Core IdentityDbContext + pgvector
│       │   └── Identity/
│       │       └── ApplicationUser.cs # Identity user (Guid keys)
│       ├── Migrations/
│       │   ├── 20261022072822_InitialCreate.cs
│       │   ├── ...Designer.cs
│       │   └── SanadDbContextModelSnapshot.cs
│       ├── Class1.cs                 # Leftover placeholder (unused)
│       └── Sanad.Infrastructure.csproj
└── tests/
    └── Sanad.Tests/                  # xUnit test project (net10.0)
        ├── UnitTest1.cs              # Empty placeholder test
        └── Sanad.Tests.csproj
```

---

## Layer-by-Layer Breakdown

### 1. Sanad.Api (Presentation / Composition Root)
- **Target:** `net10.0`, ASP.NET Core Web API
- **Program.cs** registers:
  - `SanadDbContext` with Npgsql + `UseVector()` (pgvector)
  - Swagger/OpenAPI
  - `IDocumentTextExtractor` → `PdfTextExtractor` (scoped)
- **Not present (despite README claims):**
  - No `AddIdentity<ApplicationUser, ...>` / JWT authentication configuration
  - No API controllers or minimal API endpoints
  - No Hangfire background job registration
  - No Semantic Kernel / `Microsoft.Extensions.AI` LLM orchestration
  - No CORS, no health checks, no middleware beyond HTTPS redirection + Swagger

### 2. Sanad.Application
- Single interface: `IDocumentTextExtractor` with method `IReadOnlyList<ExtractedPage> Extract(Stream Pdf)`
- `ExtractedPage(int PageNumber, string Text)` record
- References `Sanad.Domain` only

### 3. Sanad.Domain
Core entities:
- **`Document`** — `Id (Guid)`, `UserId (Guid)`, `FileName`, `StoragePath`, `ContentType`, `SizeBytes`, `Status (DocumentStatus)`, `FailureReason`, `PageCount`, `UploadedAt`, navigation `List<Chunk>`
- **`Chunk`** — `Id (Guid)`, `DocumentId (Guid)`, `Document`, `ChunkIndex`, `PageNumber`, `Text`, `Embedding (Pgvector.Vector)`
- **`DocumentStatus`** enum — `Pending, Processing, Ready, Failed`
- References `Pgvector` 0.3.2

### 4. Sanad.Infrastructure
- **`PdfTextExtractor`** — Implements `IDocumentTextExtractor` using `UglyToad.PdfPig`. Extracts text page-by-page, inserting newlines based on vertical baseline gaps. (Same logic duplicated in the spike.)
- **`SanadDbContext`** — Inherits `IdentityDbContext<ApplicationUser, IdentityRole<Guid>, Guid>`. Exposes `DbSet<Document>` and `DbSet<Chunk>`. In `OnModelCreating`:
  - Enables `vector` PostgreSQL extension
  - Configures `Chunk.Embedding` as `vector(768)` with an **HNSW** index using `vector_cosine_ops`
  - Indexes `Document.UserId` and `Chunk.DocumentId`
- **`ApplicationUser`** — Empty subclass of `IdentityUser<Guid>`
- **Migrations** — `InitialCreate` creates Identity tables (AspNetUsers, AspNetRoles, etc.) plus `Documents` and `Chunks` with the HNSW vector index.
- **`Class1.cs`** — Empty placeholder class, serves no purpose.
- Missing (per README roadmap): Hangfire integration, file storage abstraction, embedding generation service.

### 5. tests / Sanad.Tests
- xUnit project referencing `Sanad.Api`
- Contains only a placeholder `UnitTest1.Test1` with an empty body — **no actual test coverage**
- Packages: coverlet.collector, xunit, xunit.runner.visualstudio

### 6. spikes / Spike.Pdf
- .NET console experiment validating `PdfPig` text extraction, counting Arabic Unicode characters (`\u0600-\u06FF`) and presentation forms (`\uFB50-\uFEFF`), writing output to `.extracted.txt`
- Not referenced by the solution; a standalone scratch project

### 7. DevOps / Config
- **docker-compose.yml** — Spins up `pgvector/pgvector:pg16` with user `sanad` / db `sanad`, port 5432
- **ci.yml** — GitHub Actions: checkout → setup-dotnet 10.0.302 → restore → build (Release) → test (Release)
- **appsettings.Development.json** — Contains the connection string; `appsettings.json` has none (production connection string would be missing at runtime unless provided via environment variable)

---

## Tech Stack (claimed vs. actual)

| Layer | README Claims | Actually in Code |
|-------|--------------|------------------|
| API | ASP.NET Core Web API, C# | ✅ Minimal host, Swagger only |
| AI orchestration | Semantic Kernel / Microsoft.Extensions.AI | ❌ Not referenced |
| Database | PostgreSQL + pgvector, EF Core | ✅ DbContext + migrations + vector index |
| Background jobs | Hangfire | ❌ Not referenced |
| Auth | ASP.NET Core Identity + JWT | ⚠️ Identity DbContext exists, but Identity/JWT services not configured |
| DevOps | Docker, GitHub Actions | ✅ docker-compose + CI workflow |

---

## Key Gaps & Issues (relative to the README vision)

1. **No API endpoints whatsoever** — No document upload, no chat/Q&A, no citation retrieval.
2. **Authentication not wired up** — `SanadDbContext` is an `IdentityDbContext`, but `Program.cs` never calls `AddIdentity`, `AddAuthentication(JwtBearerDefaults...)`, or configures JWT. Per-user document isolation (`UserId` on `Document`) cannot be enforced yet.
3. **No background processing** — No Hangfire server/client, no document ingestion pipeline (extract → chunk → embed → store).
4. **No vector search** — Despite the HNSW index and `vector(768)` column, there is no service/repository to perform cosine-similarity queries against chunks.
5. **No LLM integration** — No Semantic Kernel, no `Microsoft.Extensions.AI`, no prompt building, no answer generation.
6. **No SSE streaming** — Mentioned in README; not implemented.
7. **Insufficient-evidence handling** — No logic to detect when retrieved chunks don't answer the question.
8. **Tests are empty** — Zero meaningful test coverage.
9. **Connection string only in Development config** — Production/staging would fail without an env var.
10. **Leftover `Class1.cs`** — Dead code in Infrastructure.
11. **Duplicate text-extraction logic** — The spike duplicates `PdfTextExtractor.ExtractText`; no shared library.

---

## Strengths of the Current Foundation

- Clean layered architecture (Api → Application → Domain → Infrastructure) with correct dependency direction.
- EF Core + Npgsql + pgvector already configured with proper HNSW cosine index for semantic search.
- Identity infrastructure (Guid-based users) is defined and migration-ready.
- PDF text extraction is implemented and a spike confirms Arabic character handling considerations.
- Dockerized database and CI pipeline are set up.
- `.NET 10` throughout, modern minimal APIs/Swagger setup.

---

## Recommended Next Steps (to match the README)

1. **Wire up Identity + JWT** in `Program.cs` (`AddIdentity`, `AddAuthentication`, token issuer/signing key).
2. **Add API endpoints** (minimal APIs or controllers): document upload, document status, chat/question, list documents.
3. **Add Hangfire** for background ingestion (extract text → chunk → generate embeddings → save chunks with vectors).
4. **Add a vector-search service** querying `Chunks` via `vector_cosine_ops` / cosine distance.
5. **Integrate Semantic Kernel / Microsoft.Extensions.AI** for RAG prompt construction and LLM calls with citation tracking.
6. **Add SSE streaming endpoint** for chat responses.
7. **Implement insufficient-evidence** logic (e.g., score threshold → "I don't know").
8. **Write real unit/integration tests** replacing `UnitTest1`.
9. **Remove `Class1.cs`** and consolidate the spike logic into the shared extractor.
10. **Add connection string to base `appsettings.json`** or document required env var.

If you'd like, I can help implement any of these next steps — for example, adding the missing API endpoints, wiring up JWT authentication, or building the vector search service. Just let me know which area you want to tackle first.
