# ATS Resume Platform

Pre-production architecture scaffold. Decisions live in [`docs/adr`](docs/adr/README.md). Read those first.

## What is here

| Path | Purpose |
|---|---|
| `docs/adr/` | 13 ADRs: modular monolith, monorepo, mobile, async pipeline, resume schema & truthfulness guard, LLM gateway, India privacy, scaling path, India product constraints, BYOK & desktop apps, token optimization & telemetry, optional RAG memory, and LaTeX dual rendering |
| `packages/resume-schema/` | `resume.schema.json` (source of truth for resume and master profile data) plus samples that validate against it |
| `packages/contracts/openapi.yaml` | `/v1` API contract. TypeScript and Rust clients are generated from it |
| `packages/resume-templates/` | Standard, ATS-safe LaTeX templates (`template.tex`) and character sanitization filters |
| `packages/ai-skills/` | Token estimators, prompt minifiers, diff-only patch schemas, and local RAG retrieval primitives |
| `apps/api/db/migrations/0001_init.sql` | Postgres schema (identity, consents, profile, job descriptions, resumes and versions, async tasks, outbox, partitioned score and audit tables, pgvector embeddings) |

## Target layout (not yet created)


```
apps/web      TypeScript (React). Mobile-web first, PWA
apps/desktop  Tauri desktop/system app (Rust + shared web UI). Offline LaTeX compile & local BYOK/Ollama 
apps/api      Modular monolith, TypeScript. Modules: identity, profile, resume,
              job-descriptions, scoring, tailoring, rendering, billing
apps/worker   Queue consumers, same codebase as api, scaled independently (ephemeral BYOK execution)
apps/mobile   Rust core (UniFFI) + Jetpack Compose and SwiftUI shells. Android first
packages/     contracts, resume-schema, ai-skills, resume-templates, config
infra/        Terraform, Dockerfiles
evals/        Golden resumes and job descriptions for AI regression tests
```

Rules enforced in CI: apps import only from `packages/*`; modules inside `apps/api` import only each other's public interface; generated clients are never hand-edited; OpenAPI breaking changes fail the build.

## Suggested build order

1. Repo skeleton: pnpm and Turborepo, Cargo workspace for mobile and desktop, lint and boundary rules, CI with path filters.
2. Contract tooling: generate TS and Rust clients and types from `openapi.yaml` and `resume.schema.json`.
3. `identity` and `profile` modules, then `resume` (versions are immutable).
4. Task and outbox plumbing with a fake worker. Prove the async lifecycle before adding any AI.
5. Deterministic scoring (format checks, skill coverage) behind the `ScoringStrategy` port.
6. LLM gateway, BYOK headers, token telemetry hooks, diff-only tailoring with the truthfulness guard, and the eval suite in CI.
7. Rendering engine: LaTeX templating via Tera/Handlebars, character escaping, embedded Tectonic engine in desktop, containerized Tectonic CLI in workers for dual export (`.tex` and `.pdf`).
8. BYOK status UI, token estimation skills (`packages/ai-skills`), and optional local/pgvector RAG Master Profile memory.
9. Mobile core and Android shell, Tauri desktop build, web polish, and launch hardening.

## Open items before writing production code

- Confirm default local embedding model (e.g. `all-MiniLM-L6-v2` or `bge-small-en-v1.5`) for desktop RAG and set dimension in migrations.
- Pick the managed auth provider. It must support phone OTP and Google sign-in.
- Verify BYOK client-vault integration (native OS Keychain/SecretService for Tauri desktop, Keystore for Android, session storage for web).
- Confirm zero-cost onboarding defaults and documentation for free-tier BYOK keys (Google AI Studio Gemini Flash, Groq, Ollama).
- Legal review of consent text, retention periods, and DPDP obligations.
- Replace the `example.com` domains in the OpenAPI servers and the schema `$id`.

