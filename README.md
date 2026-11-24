# TRUTH//PRINT — Honest ATS Resume Platform (Web Edition)

![TypeScript](https://img.shields.io/badge/TypeScript-100%25-blue?style=flat-square&logo=typescript)
![Next.js](https://img.shields.io/badge/Next.js-App_Router-black?style=flat-square&logo=next.js)
![Supabase](https://img.shields.io/badge/Supabase-Postgres_%2B_pgvector-3ECF8E?style=flat-square&logo=supabase)
![Turborepo](https://img.shields.io/badge/Monorepo-npm_%2B_Turborepo-EF4444?style=flat-square&logo=turborepo)

**TRUTH//PRINT** is a **100% TypeScript** monorepo for building, tailoring, and compiling ATS-compliant LaTeX/PDF resumes **without AI hallucinations**.
* **Zero Hosted AI Billing:** Powered by a **Bring Your Own Key (BYOK)** architecture.
* **Token-Optimized:** Uses diff-only JSON patching and pre-flight minification to cut token burn by 40–65%.
* **Truth-Gated:** Every AI-suggested edit must trace back to a verified `source_ref` in the user's Master Vault.

> **Before writing code:** Read [`docs/architecture_decision_records.md`](docs/architecture_decision_records.md) (ADRs `0001`–`0012`) to understand module boundaries and system constraints.

---

## 1. Repository Structure & Collaborator Rules

We use **npm workspaces** + **Turborepo** for fast, path-filtered builds. Frontend routes are thin entrypoints (`page.tsx`) that compose feature-level components (`one file = one section/component`).

```text
ats-platform/
├── .env.example
├── .gitignore
├── package.json                              # npm workspaces + Turborepo scripts
├── turbo.json
├── tsconfig.base.json
├── dependency-cruiser.config.ts              # Enforces ADR-0001 & ADR-0002 boundaries
│
├── apps/
│   ├── web/                                  # Next.js (App Router) Mobile-First PWA
│   │   ├── next.config.ts
│   │   ├── tailwind.config.ts
│   │   ├── tsconfig.json
│   │   ├── package.json
│   │   └── src/
│   │       ├── app/                          # Thin Next.js Route Entrypoints ONLY
│   │       │   ├── layout.tsx                # Root layout, fonts, Navbar, Toast
│   │       │   ├── globals.css               # Tailwind + newsprint/brutalist utilities
│   │       │   ├── page.tsx                  # Route: / (Landing Atelier)
│   │       │   ├── vault/page.tsx            # Route: /vault
│   │       │   ├── tailor/page.tsx           # Route: /tailor
│   │       │   ├── diagnostic/page.tsx       # Route: /diagnostic
│   │       │   ├── keys/page.tsx             # Route: /keys
│   │       │   ├── versions/page.tsx         # Route: /versions
│   │       │   └── export/page.tsx           # Route: /export
│   │       ├── features/                     # One File = One Section / Component
│   │       │   ├── landing/
│   │       │   │   ├── components/           # HeroSection, TokenSurgeryHud, TruthGuardLab, etc.
│   │       │   │   └── data/lab-roles.ts
│   │       │   ├── vault/
│   │       │   │   ├── components/           # VaultHeaderGauge, ExperienceForm, IndiaConventionsPanel, etc.
│   │       │   │   └── hooks/useMasterVault.ts
│   │       │   ├── tailor/
│   │       │   │   ├── components/           # WorkerStatusBanner, JdInputPanel, DiffReviewPanel, AtsPdfSheet, etc.
│   │       │   │   └── hooks/useTailoringSession.ts
│   │       │   ├── diagnostic/
│   │       │   │   ├── components/           # FormatReadinessGauge, JobMatchGauge, AuditLedgerTable, FormulaModal, etc.
│   │       │   │   └── hooks/useDiagnosticAudit.ts
│   │       │   ├── keys/
│   │       │   │   ├── components/           # ByokSummaryStats, ProviderSelector, KeyConfigForm, SessionUsageLedger, etc.
│   │       │   │   └── hooks/useByokVault.ts
│   │       │   ├── versions/
│   │       │   │   ├── components/           # VersionTreeDag, SnapshotNode, SnapshotInspector
│   │       │   │   └── hooks/useVersionHistory.ts
│   │       │   └── export/
│   │       │       ├── components/           # CompilePdfCard, ExportTexCard, AtsPreflightChecklist, LatexCodeViewer
│   │       │       └── hooks/useLatexExport.ts
│   │       ├── components/                   # Global Shared UI (Navbar, PreviewDrawer, Toast)
│   │       └── lib/                          # Pure TS Utilities (api-client, audio-fx, byok-vault, indexed-db)
│   │
│   ├── api/                                  # Modular Monolith (Node.js / TypeScript)
│   │   ├── db/migrations/0001_init.sql       # Supabase Postgres + pgvector schema
│   │   └── src/
│   │       ├── main.ts
│   │       ├── shared/                       # outbox.ts, telemetry.ts, db.ts
│   │       └── modules/                      # Hexagonal modules (domain, application, ports, adapters)
│   │           ├── identity/
│   │           ├── profile/
│   │           ├── resume/
│   │           ├── job-descriptions/
│   │           ├── scoring/
│   │           ├── tailoring/
│   │           └── rendering/
│   │
│   └── worker/                               # Async AI & LaTeX/PDF Workers (BullMQ + Upstash Redis)
│       └── src/
│           ├── main.ts
│           ├── queues/bullmq.ts
│           └── processors/                   # ai-tailoring, scoring, latex-render
│
├── packages/
│   ├── contracts/                            # Shared TypeScript API types & OpenAPI spec
│   ├── resume-schema/                        # Single source of truth (Zod + JSON Schema)
│   ├── ai-skills/                            # schema-pruner, prompt-minifier, diff-patcher, token-counter, truth-guard
│   └── config/                               # Shared TSConfig, ESLint, and Tailwind presets
│
├── docs/                                     # ADRs & UI design templates
└── evals/                                    # Token-metered LLM regression tests
```

### Strict Boundary Rules (Enforced in CI)
1. **Apps never import from other apps:** `apps/web`, `apps/api`, and `apps/worker` may **only** import from `packages/*`.
2. **Modules never cross-query:** Inside `apps/api`, modules talk through public interfaces or domain events—never by importing another module's database tables.
3. **Contracts first:** Any API change starts in `packages/contracts/` and `packages/resume-schema/` before touching frontend or backend handlers.

---

## 2. Frontend (`apps/web`) — Routes & Client Architecture

Built with **Next.js (App Router)**, **React**, **TypeScript**, and **Tailwind CSS** using a tactile newsprint/brutalist design system.

### App Routes (`src/app/`)
| Route | View Name | What It Does |
|---|---|---|
| `/` | **Landing Atelier** | Product overview, live Token Surgery HUD, and interactive Truth-Guard comparison lab. |
| `/vault` | **Master Experience Vault** | Unconstrained career catalog. Toggle items between **RAG Memory** and **1-Page Active Draft**, plus India-specific fields (Notice Period, private CTC). |
| `/tailor` | **Tailoring Studio** | 3-column workspace: Job Description parser, Truth-Gated Diff Review (Accept/Reject), and live ATS PDF / `.tex` preview. |
| `/diagnostic` | **ATS Diagnostic** | Deterministic format readiness audit (100 pts) + hybrid semantic job-match score and honest gap analysis. |
| `/keys` | **BYOK Secret Vault** | Client-side API key management (Gemini, Groq, OpenAI, Anthropic, local Ollama) and session token usage ledger. |
| `/versions` | **Version Tree (DAG)** | Immutable resume snapshot timeline. Compare deltas between tailored branches and restore previous states. |
| `/export` | **Export & Compilation Tray** | Dual-format export: compiled single-column ATS PDF and sanitized raw `.tex` source. |

### Key Shared Components & Client Libraries
* **`src/components/`**
  * `Navbar.tsx`: Top navigation bar, view switcher, and mechanical audio toggle.
  * `PreviewDrawer.tsx`: Slide-in drawer for inspecting compiled ATS PDFs and raw `.tex` side-by-side.
  * `Toast.tsx`: Non-blocking status notifications.
* **`src/lib/`**
  * `byok-vault.ts`: Encrypts and stores user API keys strictly in browser `sessionStorage`/`localStorage`. **Keys are never saved to the server.**
  * `indexed-db.ts`: Offline-first draft persistence and optional client-side vector storage.
  * `audio-fx.ts`: Web Audio API synthesizer for mechanical click and stamp feedback.
  * `api-client.ts`: Typed fetch wrapper aligned with `packages/contracts`.

---

## 3. Backend (`apps/api` & `apps/worker`)

### Modular Monolith (`apps/api/src/modules/`)
Every module follows **Hexagonal Architecture** (`domain/`, `application/`, `ports/`, `adapters/`):

* **`identity/`** — Phone OTP & Google OAuth (OIDC), user sessions, and DPDP consent tracking.
* **`profile/`** — Manages the user's Master Experience Vault and private fields (e.g., expected CTC).
* **`resume/`** — Immutable `resume_version` DAG management. Edits and tailored outputs always create new versions.
* **`job-descriptions/`** — JD ingestion, SHA-256 hash generation (for idempotency), and keyword extraction.
* **`scoring/`** — Hybrid ATS scoring engine (deterministic keyword/format checks + `pgvector` similarity + LLM rubric).
* **`tailoring/`** — Orchestrates diff-only tailoring requests and enforces the **Truthfulness Guard**.
* **`rendering/`** — Converts canonical JSON into sanitized LaTeX (`.tex`) and coordinates PDF compilation.

### Shared Infrastructure (`apps/api/src/shared/`)
* **`outbox.ts`**: Transactional outbox pattern—writes the task row and enqueue event in one DB transaction so async jobs are never lost.
* **`telemetry.ts`**: OpenTelemetry tracing with strict PII-free logging (no resume content or contact info in logs).

### Async Workers (`apps/worker/src/`)
* **`queues/bullmq.ts`**: Upstash Redis-backed queue consumer with bounded concurrency, exponential backoff, and dead-letter queues.
* **`processors/`**:
  * `ai-tailoring.processor.ts`: Executes ephemeral BYOK LLM calls and validates diff patches.
  * `scoring.processor.ts`: Runs async semantic and rubric evaluations.
  * `latex-render.processor.ts`: Compiles `.tex` into single-column ATS PDFs.

---

## 4. Database (`apps/api/db/migrations/0001_init.sql`)

Powered by **Supabase (PostgreSQL + `pgvector`)** hosted in an India-first region (Mumbai) per ADR-0007. Uses **UUIDv7** primary keys and `user_id` indexing across all user-owned tables.

| Section | Tables | Purpose |
|---|---|---|
| **Identity & Privacy** | `users`, `consents` | User accounts (`home_region` aware) and DPDP consent audit trail. |
| **Master Profile** | `master_profiles`, `experience_chunks` | Canonical JSON profile and `pgvector` embeddings for RAG retrieval. |
| **Resumes & DAG** | `resumes`, `resume_versions` | Immutable version history with parent pointers and source references. |
| **Jobs & Async Tasks** | `job_descriptions`, `tasks`, `outbox_events` | JD storage, idempotent task lifecycle (`queued -> running -> succeeded`), and outbox relay. |
| **Partitioned Logs** | `score_runs`, `task_events`, `audit_log` | Monthly partitioned append-only tables for telemetry and scoring history. |

---

## 5. AI Model & Token Optimization (`packages/ai-skills/`)

* **Provider-Agnostic Gateway (`scoring/ports/llm.port.ts`)**: Unified `LLMProvider` interface (`generateStructured`, `embed`) supporting **Google Gemini**, **Groq**, **OpenAI**, **Anthropic**, and **Ollama** (`localhost:11434`).
* **Ephemeral BYOK Execution**: Client-supplied credentials (`apiKey`, `baseUrl`, `model`) are passed over TLS directly to the worker's memory for the duration of the task and immediately purged.
* **Token Optimization Pipeline**:
  * `schema-pruner.ts`: Strips empty fields and unselected sections before prompt construction.
  * `prompt-minifier.ts`: Uses concise, schema-anchored syntax and places static instructions at the head for provider prompt-caching.
  * `diff-patcher.ts`: Requests targeted JSON/bullet diffs instead of full-text rewrites (40–65% token savings).
  * `token-counter.ts`: Pre-flight BPE token estimation displayed in the UI before execution.

---

## 6. Agentic RAG Pipeline (Master Profile Memory)

Used when **Master Profile Memory** is enabled (ADR-0012) to curate a 1-page resume from a candidate's large career history:

1. **Chunking & Embedding (`profile/` + `packages/resume-schema`)**:
   * Experiences, projects, and skill contexts are broken into semantic blocks (`experience_chunk`, `project_chunk`, `skill_context_chunk`).
   * Embedded via the user's BYOK embedding endpoint and stored in Supabase `pgvector` (or browser `IndexedDB`).
2. **JD Requirement Extraction (`job-descriptions/`)**:
   * Parses the target JD into hard skills, domain requirements, and seniority signals.
3. **Top-$K$ Retrieval (`scoring/` / `tailoring/`)**:
   * Performs cosine similarity search against the user's Master Vault to pull only the highest-relevance verified chunks that fit the 1-page ATS line budget.
4. **Truthfulness Guard (`packages/ai-skills/truth-guard.ts`)**:
   * Every tailored bullet must reference a valid `source_ref` ID from the retrieved chunks.
   * Any entity, metric, or skill not traceable to the source data is rejected before reaching the user's Diff Review screen.

---

## 7. Quick Start (Local Web Development)

```bash
# 1. Install all workspace dependencies
npm install

# 2. Configure environment variables
# Copy .env.example to .env and add your Supabase DATABASE_URL & Upstash REDIS_URL
cp .env.example .env

# 3. Push database migrations to Supabase
npm run db:migrate --workspace=apps/api

# 4. Start Web, API, and Worker in parallel
npm run dev
```

### Useful Workspace Commands
* `npm run build` — Build all apps and packages via Turborepo.
* `npm run lint` — Run ESLint and `dependency-cruiser` boundary checks.
* `npm test` — Run unit tests across packages and API modules.
````[cite: 7, 8]