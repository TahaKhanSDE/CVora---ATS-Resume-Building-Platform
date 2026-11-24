# Architecture Decision Records

One file per significant decision. Never edit an accepted ADR to change history. Write a new ADR that supersedes it.

| # | Title | Status |
|---|---|---|
| 0001 | Modular monolith with separate async workers | Accepted |
| 0002 | Single TypeScript monorepo, independence through contracts | Accepted |
| 0003 | Web-first PWA with optional Tauri desktop wrapper | Accepted |
| 0004 | AI work is asynchronous, idempotent and observable (BYOK-Compatible) | Accepted |
| 0005 | Structured resume JSON is source of truth; AI may not invent facts | Accepted |
| 0006 | Provider-agnostic LLM gateway, hybrid scoring, and token-metered evals | Accepted |
| 0007 | India-first hosting and privacy, residency-ready | Accepted |
| 0008 | Scaling path: configuration first, database is the known limit | Accepted |
| 0009 | Product and platform constraints for tier 1 and tier 2 India (BYOK Edition) | Accepted |
| 0010 | BYOK Architecture and Client Secret Vault | Accepted |
| 0011 | Token Optimization Skills, Pre-flight Minification, and BYOK Status Telemetry | Accepted |
| 0012 | Optional Client-Side RAG Memory for Master Experience Curation | Accepted |

Template: Context, Decision, Consequences, Revisit when.

---

# ADR-0001: Modular monolith with separate async workers

- Status: Accepted
- Date: 2026-11-21

## Context
We need a backend that serves a TypeScript web application, handles slow and unpredictable AI work, and scales without a rewrite. The team is small. Microservices on day one would add network failure modes, distributed transactions, and operational cost before we have traffic.

## Decision
- One deployable API codebase (`apps/api`) organised into **modules** with enforced boundaries: `identity`, `profile`, `resume`, `job-descriptions`, `scoring`, `tailoring`, `rendering`.
- Each module is internally **hexagonal** (domain, application, ports, adapters). Modules talk to each other only through their public interface or domain events, never through another module's tables.
- AI and rendering work runs in **workers** (`apps/worker`), built from the same codebase but deployed and scaled independently.
- Boundaries are enforced in CI (`dependency-cruiser` or ESLint boundaries rule). Violations fail the build.

## Consequences
- Simple deploys, simple local dev, one transaction boundary per module.
- Any module can be extracted into its own service later by swapping the in-process call for a network call.
- Discipline is required. The boundary lint rule is not optional.

## Revisit when
A module needs a different runtime (for example a Python model server), a different scaling profile that workers cannot cover, or a separate team owns it.

---

# ADR-0002: Single TypeScript monorepo, independence through contracts

- Status: Accepted
- Date: 2026-11-21

## Context
The frontend, API, and async workers are all written in TypeScript. They must be independent in release cadence while always agreeing on API contracts and the resume data shape.

## Decision
One pure TypeScript monorepo. Independence is enforced by **contracts**, not by repo boundaries.

```text
apps/
  web/        # Next.js / React (mobile-first PWA)
  api/        # Modular monolith (Node.js / TypeScript)
  worker/     # Async AI & LaTeX/PDF rendering workers
  desktop/    # (Optional) Thin Tauri webview wrapper around apps/web
packages/
  contracts/      # Shared TS API types / OpenAPI spec
  resume-schema/  # Single Zod schema + JSON Schema
  ai-skills/      # Schema pruner, prompt minifier, diff-patcher, JS token counter
  config/         # Shared ESLint, TSConfig, Tailwind presets
infra/  docs/adr/  evals/
```

- `packages/resume-schema` is the single resume shape, defined in Zod and exported as JSON Schema.
- Apps may import only from `packages/*`, never from each other. Enforced in CI.
- Tooling: `pnpm` workspaces plus Turborepo.
- CI runs per path. Web changes do not rebuild workers, and the reverse.

## Consequences
- Zero cross-language codegen overhead. API changes and client updates land in one atomic PR.
- Splitting into multiple repos later is cheap because apps already depend only on `packages/*`.

## Revisit when
CI times grow beyond what path filtering and caching can fix, or a non-TypeScript service is introduced.

---

# ADR-0003: Web-first PWA with optional Tauri desktop wrapper

- Status: Accepted
- Date: 2026-11-21

## Context
Maintaining native Android/iOS shells, a Rust core, and a mandatory desktop application introduces heavy build, CI, and maintenance overhead before product-market fit. Our target users in tier 1 and tier 2 Indian cities can be reached immediately through a fast, mobile-responsive web app.

## Decision
- **Primary Target (`apps/web`):** A responsive, mobile-first Progressive Web App (PWA) built with Next.js/React. It works across mobile browsers and desktop browsers from a single codebase.
- **No Native Mobile App:** Native Android/iOS apps are out of scope.
- **Optional Desktop Wrapper (`apps/desktop`):** A Tauri desktop build is strictly optional and deferred. If added later, it acts solely as a thin webview shell wrapping `apps/web`—with no custom Rust business logic.

## Consequences
- Single codebase, faster iteration, and zero app-store review delays.
- Users on desktop who want local-first convenience or direct `localhost` Ollama access can optionally use the Tauri wrapper later without requiring any backend changes.

## Revisit when
Web usage demonstrates a clear, validated need for native mobile distribution or OS-level desktop integrations.

---

# ADR-0004: AI work is asynchronous, idempotent and observable (BYOK-Compatible)

- Status: Accepted
- Date: 2026-10-30

## Context
LLM calls take seconds to minutes, fail unpredictably, and consume user-supplied API quotas. They must never run on request threads, and a retry must never duplicate results or waste user tokens.

## Decision
- API endpoints that trigger AI work return **202 Accepted** with a `task_id`. Clients poll `GET /v1/tasks/{id}` or subscribe to server-sent events (SSE).
- Task lifecycle: `queued -> running -> succeeded | failed | cancelled`, with `retrying` as a transient state. Every transition is persisted.
- **Transactional outbox:** the task row and its "enqueue" event are written in one DB transaction. A relay publishes to the queue. Tasks are never lost between the DB and the queue.
- **Idempotency keys** are required on task creation. The key is a hash of (user, resume version, JD hash, task type). A repeated request returns the existing task or cached result.
- Queue: BullMQ on Redis to start. Queue access sits behind a `TaskQueue` port so SQS or Cloud Tasks can replace it without touching domain code.
- Workers use bounded concurrency, retry with exponential backoff and jitter, and a dead-letter queue.
- Every task carries a trace id. OpenTelemetry spans cover API, queue, and worker.

## Consequences
- The API stays fast under load. Workers scale on queue depth.
- Clients must handle async UX (progress states).

## Revisit when
Queue throughput or latency needs exceed Redis-backed queues.

---

# ADR-0005: Structured resume JSON is the source of truth; AI may not invent facts

- Status: Accepted
- Date: 2026-10-30

## Context
Scoring, tailoring, and rendering all need to reason about resume content. Operating on PDFs or free text makes every one of them fragile. Separately, an AI that rewrites resumes can be pushed into fabricating experience, which harms users and the product.

## Decision
1. **Canonical data** is the structured document defined in `packages/resume-schema`. Files (PDF, DOCX) are only inputs (parsed into the schema) or outputs (rendered from it).
2. **Versioning:** every edit creates an immutable `resume_version`. Tailored resumes are new versions derived from a base version and a job description. Nothing is overwritten.
3. **Truthfulness guard:** the tailoring step may only rephrase, reorder, select, and emphasise facts already present in the user's profile. It may not add skills, employers, titles, dates, degrees, or metrics. Enforcement:
   - Tailoring output must be schema-valid and reference source item ids (`source_ref`).
   - A validator rejects any output skill, entity, or number that cannot be traced to the source data.
   - The user sees a **diff** and must approve before a tailored version becomes final.
4. **Relevance selection:** profile items (skills, projects, experience bullets) are ranked against the job description, and only the top items that fit the page budget are included.
5. **Rendering** is server-side (JSON -> LaTeX/HTML template -> PDF) so output is identical everywhere. ATS-safe templates: single column, standard section headings, real text layer, no layout tables, no images, standard fonts.

## Consequences
- Scoring and tailoring are testable on data, not on documents.
- Full audit trail and easy undo.
- Missing skills are surfaced as an honest gap report rather than fabricated.

## Revisit when
We add new section types via additive, versioned schema changes.

---

# ADR-0006: Provider-agnostic LLM gateway, hybrid scoring, and token-metered evals

- Status: Accepted
- Date: 2026-11-21

## Context
LLM vendors, prices, and capabilities change quickly. Under a BYOK architecture, the gateway must handle client-supplied credentials, custom base URLs (e.g., local Ollama/OpenAI-compatible endpoints), and strict token economy constraints.

## Decision
- **Gateway:** All model calls execute through an `LLMProvider` port (`generateStructured`, `embed`). Adapters exist for OpenAI, Anthropic, Gemini, Groq, and custom OpenAI-compatible endpoints (Ollama). The gateway accepts dynamic client-supplied runtime config (`apiKey`, `baseUrl`, `model`, `timeoutMs`).
- **Token Telemetry Hook:** Every call emits usage metadata (`prompt_tokens`, `completion_tokens`, `cached_tokens`) and active provider rate-limit headers (`x-ratelimit-*` where provided) back to the UI.
- **Structured Output & Diff Engine:**
  - LLM outputs are validated against strict Zod schemas.
  - Tailoring operations request delta/diff instructions (targeted replacement objects or JSON patches) rather than re-streaming entire full-text resumes, cutting completion token overhead by 40–60%.
- **Hybrid Score:**
  - Skill and keyword coverage using a normalized skill taxonomy + aliases (deterministic, explainable).
  - Semantic similarity between job description and resume sections (embeddings in `pgvector` or browser IndexedDB).
  - LLM rubric judgment with fixed criteria (seniority fit, impact, relevance).
  - Format and parseability checks (ATS-readiness).
- **Prompts are versioned code:** Covered by regression evals (`/evals`).

## Consequences
- Works seamlessly across cloud providers and local OpenAI-compatible endpoints.
- Users have full transparency into exact token burn per request.
- Zero centralized server LLM inference cost.

## Revisit when
Eval results show deterministic checks or small models match commercial LLM tailoring quality for common tech roles.

---

# ADR-0007: India-first hosting and privacy, residency-ready for global growth

- Status: Accepted
- Date: 2026-10-30

## Context
Resumes are sensitive personal data. We launch in India and intend to grow globally. India's Digital Personal Data Protection Act, 2023 (DPDP) governs processing.

## Decision
- **Primary region: India** (Mumbai, with Hyderabad as a secondary region) for compute, database, and object storage.
- **Region-aware from day one:** every user row carries a `home_region` column, and object-storage keys are prefixed by region.
- **Consent as data:** a `consents` table records purpose, policy version, timestamp, and withdrawal.
- **User rights as features:** data export (JSON + PDF) and account deletion with a hard-delete job built in the first release.
- **Encryption:** TLS everywhere, encryption at rest, application-level encryption for contact fields (phone, email, address).
- **Logging hygiene:** no resume content or contact details in logs, traces, or analytics events.
- **Retention:** uploaded originals and rendered PDFs expire on a configurable schedule.

## Consequences
- Compliance posture is built in from day one rather than retrofitted.

## Revisit when
Legal review changes requirements, or we enter a market with strict localisation rules.

---

# ADR-0008: Scaling path: configuration first, database is the known limit

- Status: Accepted
- Date: 2026-10-30

## Context
The goal is to absorb user growth by upgrading provider plans, not by rewriting code.

## Decision
- **Stateless tiers (`apps/api`, `apps/worker`):** No local state, no sticky sessions, config via environment, health/readiness endpoints. Scale horizontally.
- **Database (Postgres):**
  - UUIDv7 primary keys.
  - `user_id` indexed on every user-owned table.
  - No cross-module joins or foreign keys across module boundaries.
  - Cursor pagination only.
  - Heavy append-only tables (`score_runs`, `task_events`, `audit_log`) partitioned by month.
  - Connection pooling (PgBouncer or managed pooler).
- **Cache and limits:** Redis for caching (keyed by content hash), rate limiting, and queue backing.
- **Files:** Object storage + CDN with signed URLs.

## Revisit when
Primary DB CPU or write latency stays above SLO thresholds for a sustained period.

---

# ADR-0009: Product and platform constraints for tier 1 and tier 2 India (BYOK Edition)

- Status: Accepted
- Date: 2026-11-21

## Context
Target users are job seekers in tier 1 and tier 2 Indian cities: heavily mobile-browser users, variable networks, price-sensitive, and comfortable in English for work while often preferring Hindi or regional languages for UI navigation.

## Decision
- **Mobile-web first:** Responsive from the first page, lightweight bundles, installable PWA behaviour.
- **Offline-tolerant and resumable:** Drafts save locally in browser storage (`IndexedDB`/`localStorage`) and sync when online. Long AI tasks survive tab refreshes or backgrounding (ADR-0004).
- **Auth:** Phone-number OTP and Google sign-in as first-class options via a managed identity provider (OIDC).
- **i18n from day one:** All UI strings externalised, UTF-8 and Indic font support verified. Resume content is English at launch.
- **Resume conventions for India:** Schema supports optional fields common in Indian resumes (LinkedIn, GitHub, notice period, current/expected CTC marked **private and never rendered unless the user opts in**). Photo and date of birth are off by default in ATS templates.

## Consequences
- Fast, accessible web experience on mid-range and low-end mobile devices without requiring app-store downloads.

## Revisit when
Usage data shows different dominant devices or language needs.

---

# ADR-0010: BYOK Architecture and Client Secret Vault

- Status: Accepted
- Date: 2026-11-21

## Context
To make the platform free and accessible without incurring prohibitive inference bills, the system adopts a Bring Your Own Key (BYOK) model. Users supply their own API keys (OpenAI, Anthropic, Google Gemini, Groq, or custom OpenAI-compatible endpoints).

## Decision
1. **Zero Centralized AI Billing:** Core scoring, tailoring, and ATS parsing run using the user's BYOK credentials. The platform charges no markups or subscriptions for AI inference.
2. **Client-Side Secret Vault (`apps/web`):**
   - Keys are persisted in browser `localStorage` or `sessionStorage` (optionally encrypted with a user-supplied session passphrase), or held strictly in memory during the active session.
   - Keys are **never** written to the server's database or permanent storage; they are passed ephemerally over TLS when workers execute async tasks.
3. **Optional Local Runtime Support:**
   - Support for custom base URLs (e.g., `http://localhost:11434` for Ollama) when running locally or via the optional Tauri desktop wrapper.

## Consequences
- Users control their own token usage, rate limits, and provider bills.
- Zero server-side secret storage liability.

## Revisit when
A privacy-preserving, cost-effective centralized inference tier becomes viable at scale.

---

# ADR-0011: Token Optimization Skills, Pre-flight Minification, and BYOK Status Telemetry

- Status: Accepted
- Date: 2026-11-21

## Context
BYOK users pay per token or run on strict free-tier rate limits (RPM/TPM). Feeding entire multi-page resumes and verbose system prompts burns unnecessary tokens and exhausts context windows.

## Decision
1. **Token Optimization Pipeline (`packages/ai-skills`):**
   - **Schema Pruner:** Strips empty fields, unselected optional sections, and irrelevant profile items before generating prompts.
   - **Prompt Minifier:** Replaces natural-language structural instructions with concise, schema-anchored prompt syntax.
   - **Diff-Only Patch Generation:** Instructs the LLM to output targeted modifications (replacement bullets or JSON patches) rather than rewriting unchanged sections.
   - **Prompt Caching Layout:** Places static instructions at the head and dynamic JD/resume data at the tail to maximize provider prompt-caching discounts.
2. **Pre-flight Token Estimation:**
   - Before executing an AI action, the UI computes estimated input/output tokens using JS/TS `tiktoken` (or lightweight BPE estimation) in `packages/ai-skills` and displays estimated cost.
3. **BYOK Status & Quota Telemetry:**
   - **Provider Probe:** Pings the provider's `/models` or lightweight health endpoint to confirm key validity and detect quota exhaustion (HTTP 401/429).
   - **UI Telemetry Widget:** Displays active provider status (e.g., `Gemini 2.0 Flash | Est. Context: 840 tokens | Session: 4.2k tokens`).

## Consequences
- Reduces token burn by 40% to 65% across tailoring operations and prevents surprise bills or rate-limit errors.

## Revisit when
Context windows become virtually free and prompt caching is ubiquitous.

---

# ADR-0012: Optional Client-Side RAG Memory for Master Experience Curation

- Status: Accepted
- Date: 2026-11-21

## Context
A candidate's full experience often contains far more projects, metrics, and achievement variants than can fit onto a 1-page or 2-page ATS resume. Users should be able to curate from a "Master Profile" without manually copy-pasting bullets.

## Decision
1. **Optional Feature:** RAG Memory is completely optional. Users can run direct resume tailoring or enable "Master Profile Memory".
2. **Master Experience Vault:** Extends `packages/resume-schema` to support an unconstrained **Master Experience Repository** structured into semantic blocks (`experience_chunk`, `project_chunk`, `skill_context_chunk`).
3. **Web Embeddings & Storage:**
   - Embeddings are generated using the user's BYOK embedding endpoint (e.g., OpenAI `text-embedding-3-small` or Gemini embeddings).
   - Stored in user-scoped `pgvector` tables (or browser `IndexedDB` for pure client-side mode).
4. **Retrieval & Curation Pipeline:**
   - Extracts key requirements from the JD, retrieves the top-$K$ most relevant verified chunks from the Master Vault, and passes them to the tailoring engine subject to the **Truthfulness Guard (ADR-0005)**.

## Consequences
- Candidates with deep experience generate hyper-relevant resumes for specific JDs while respecting the 1-page ATS budget.
- Zero cross-user pooling or shared training.

## Revisit when
Context windows are large and cheap enough that full-profile prompts outperform vector retrieval in quality and cost.