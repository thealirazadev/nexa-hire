# NexaHire — Implementation Guide

> **A learn-by-building blueprint for an AI-powered, multi-tenant recruitment SaaS — engineered to expose you to every concept a senior (15+ yrs) Laravel engineer is expected to know.**

- **Project codename:** NexaHire
- **Target framework:** Laravel **13.x** (released March 2026)
- **Target PHP:** **8.3+** (8.4 recommended)
- **Approach:** Modular monolith → service-oriented → cloud-native deploy
- **Total duration (full-time):** 12 weeks. Part-time: 5–6 months.

---

## Table of Contents

1. [How to use this document](#1-how-to-use-this-document)
2. [Tech stack (locked-in versions)](#2-tech-stack)
3. [Architecture overview](#3-architecture-overview)
4. [Domain model & ERD](#4-domain-model--erd)
5. [Workstation setup (Phase 0)](#5-phase-0--workstation-setup)
6. [Phase 1 — Foundations](#6-phase-1--foundations-laravel-fundamentals)
7. [Phase 2 — Authn / Authz](#7-phase-2--authentication--authorization)
8. [Phase 3 — Multi-tenancy & billing](#8-phase-3--multi-tenancy--billing)
9. [Phase 4 — Queues, events, real-time](#9-phase-4--queues-events-real-time)
10. [Phase 5 — Search, files, notifications](#10-phase-5--search-files-notifications)
11. [Phase 6 — AI layer](#11-phase-6--ai-layer)
12. [Phase 7 — Public API, GraphQL, webhooks](#12-phase-7--public-api-graphql-webhooks)
13. [Phase 8 — Testing & quality](#13-phase-8--testing--quality)
14. [Phase 9 — Performance engineering](#14-phase-9--performance-engineering)
15. [Phase 10 — Docker & local devops](#15-phase-10--docker--local-devops)
16. [Phase 11 — CI/CD & GitOps](#16-phase-11--cicd--gitops)
17. [Phase 12 — Cloud infrastructure (Terraform)](#17-phase-12--cloud-infrastructure-terraform)
18. [Phase 13 — Kubernetes deploy](#18-phase-13--kubernetes-deploy)
19. [Phase 14 — Observability & SRE](#19-phase-14--observability--sre)
20. [Phase 15 — Security hardening](#20-phase-15--security-hardening)
21. [Phase 16 — Compliance, polish, launch](#21-phase-16--compliance-polish-launch)
22. [Concept coverage matrix](#22-concept-coverage-matrix)
23. [Daily learning rituals](#23-daily-learning-rituals)
24. [Reading list & references](#24-reading-list--references)

---

## 1. How to use this document

This is **not** a tutorial that you copy-paste. It is a **curriculum**:

- Each **Phase** has three blocks:
  - **Learn** → concepts to study before coding
  - **Build** → the feature/PR to ship
  - **Reflect** → questions you must be able to answer before moving on
- Treat every phase as if it were a **real PR**. Open a feature branch, write tests, write a CHANGELOG entry, and merge via PR (even if you're solo). This trains the muscle.
- Keep an **engineering journal** (`/docs/journal/`) — one markdown file per day. Senior engineers all keep notes; this is non-negotiable.
- Use **GitHub Issues** for every task. Link the PR to the issue. Use labels: `phase-1`, `bug`, `chore`, `security`, `perf`, etc.

**Rule of thumb:** if you cannot explain *why* you wrote a line of code, you haven't learned yet — go back to "Learn".

---

## 2. Tech stack

> Versions chosen are the latest stable as of **May 2026**. Don't downgrade unless you hit a real blocker.

### Core
| Layer | Choice | Why |
|---|---|---|
| Language | **PHP 8.4** | Latest features (property hooks, asymmetric visibility) |
| Framework | **Laravel 13.x** | Latest, with first-party AI SDK & vector search |
| Web server | **FrankenPHP** (fallback: Nginx + PHP-FPM) | Modern, HTTP/3, embeddable |
| App server | **Laravel Octane** (FrankenPHP / Swoole) | Persistent workers, big perf gains |
| DB primary | **PostgreSQL 17** (with `pgvector`, `pg_partman`) | JSONB, FTS, vectors, partitioning |
| Cache / queue / lock | **Redis 7.4** (or DragonflyDB) | The Swiss-army knife |
| Search | **Meilisearch 1.x** (alt: Typesense / OpenSearch) | Fast, simple |
| Object storage | **MinIO** locally → **AWS S3** in prod | Same S3 API |
| Mail dev | **Mailpit** | Beats Mailhog |
| Reverse proxy local | **Traefik** | Auto-HTTPS, dashboards |

### Frontend
| | |
|---|---|
| **Inertia 2 + Vue 3** (or Livewire 3 + Volt) | Pick **one path**; this guide uses Inertia + Vue + TypeScript |
| **Vite 6**, **Tailwind 4** | Build + styling |
| **shadcn-vue** | Component library |

### DevOps
| | |
|---|---|
| **Docker 26 + Compose v2** | Local + CI |
| **GitHub Actions** | CI/CD |
| **Terraform 1.9** | IaC |
| **Kubernetes** (EKS or k3s for learning) | Orchestration |
| **Helm 3** + **ArgoCD** | GitOps deploy |
| **Prometheus + Grafana + Loki + Tempo** | Observability stack |
| **Sentry** | Errors |
| **OpenTelemetry** | Tracing standard |

### Key Composer packages
```
laravel/framework             ^13.0
laravel/octane                ^3.0
laravel/sanctum               ^5.0
laravel/passport              ^13.0
laravel/horizon               ^6.0
laravel/pulse                 ^2.0
laravel/reverb                ^2.0
laravel/scout                 ^11.0
laravel/cashier-stripe        ^16.0
laravel/pennant               ^2.0
laravel/telescope             ^6.0   # dev only
laravel/socialite             ^6.0
laravel/ai                    ^1.0   # Laravel 13 first-party AI SDK
spatie/laravel-permission     ^7.0
spatie/laravel-medialibrary   ^12.0
spatie/laravel-activitylog    ^5.0
spatie/laravel-data           ^5.0
spatie/laravel-query-builder  ^6.0
stancl/tenancy                ^4.0
nuwave/lighthouse             ^6.0   # GraphQL
nwidart/laravel-modules       ^12.0
bezhansalleh/filament-shield  ^4.0   # admin
filament/filament             ^4.0
pestphp/pest                  ^3.0
pestphp/pest-plugin-laravel   ^3.0
larastan/larastan             ^3.0
rector/rector                 ^2.0
laravel/pint                  ^1.20
```

### Frontend npm
```
vue@^3.5  @inertiajs/vue3@^2  vite@^6  tailwindcss@^4
typescript@^5.6  vue-tsc  @vueuse/core  pinia@^3
```

---

## 3. Architecture overview

```
┌──────────────────────────────────────────────────────────────────────────┐
│                            CLIENTS                                       │
│  Marketing site │ Career pages │ Recruiter SPA │ Mobile │ 3rd-party API  │
└──────────────────────────────────┬───────────────────────────────────────┘
                                   │   (TLS, mTLS for partners)
                ┌──────────────────▼──────────────────┐
                │     Edge (CloudFront + WAF)         │
                └──────────────────┬──────────────────┘
                                   │
                ┌──────────────────▼──────────────────┐
                │   Ingress (Traefik / ALB → K8s)     │
                └──┬──────────┬──────────┬────────────┘
                   │          │          │
        ┌──────────▼─┐  ┌─────▼─────┐ ┌──▼────────┐
        │ Web (HTTP) │  │ API (REST │ │ WebSocket │
        │  Inertia   │  │ + GraphQL)│ │ Reverb    │
        │  Octane    │  │  Octane   │ │           │
        └──────┬─────┘  └─────┬─────┘ └─────┬─────┘
               │              │             │
               └──────┬───────┴─────────────┘
                      │
        ┌─────────────▼─────────────┐
        │      Domain modules       │
        │  Hiring │ Billing │ AI │  │
        │  Identity │ Comms │ Audit │
        └─────────────┬─────────────┘
                      │
        ┌─────────────┼────────────────┬───────────────┬──────────────┐
        │             │                │               │              │
   ┌────▼─────┐ ┌─────▼────┐    ┌──────▼─────┐  ┌──────▼─────┐ ┌──────▼─────┐
   │ Postgres │ │  Redis   │    │ Meilisearch│  │ S3 / MinIO │ │  pgvector  │
   │ (RDS HA) │ │ Cluster  │    │            │  │            │ │            │
   └──────────┘ └──────────┘    └────────────┘  └────────────┘ └────────────┘
                      │
                ┌─────▼──────┐
                │  Workers   │ ── Horizon supervisors (default, ai, mail, billing)
                │  (Octane)  │
                └─────┬──────┘
                      │
            ┌─────────▼─────────┐
            │ External services │
            │ Stripe │ Twilio │ │
            │ OpenAI │ AWS SES │
            │ Calendar APIs    │
            └───────────────────┘
```

### Architectural principles
- **Modular monolith first.** Every "module" lives under `app/Modules/<Module>` with its own routes, models, services, and tests. Boundaries are enforced via `deptrac` static analysis.
- **DDD-lite.** Entities, value objects, services, actions, repositories. Don't over-DDD; pragmatism wins.
- **CQRS-lite.** Heavy reads (analytics) hit denormalized read models; writes go through command handlers (single-responsibility action classes).
- **Event-driven where it pays off.** Domain events trigger side effects via async listeners. Use the **Outbox pattern** for events that must reach external systems.
- **Tenancy-aware everything.** Every query, every job, every cache key carries the `tenant_id`.
- **12-Factor.** Config via env, stateless processes, logs to stdout, etc.

---

## 4. Domain model & ERD

> Names are simplified. Real schema will have more columns (timestamps, UUIDs, soft deletes, audit cols).

```
┌─────────────┐ 1   N ┌───────────────┐ 1  N ┌──────────────┐
│  Tenant     │──────▶│  User         │─────▶│ TeamMember   │
│  (company)  │       │ (recruiter,   │      │              │
│             │       │  candidate,   │      │              │
│             │       │  admin)       │      │              │
└──────┬──────┘       └───────┬───────┘      └──────────────┘
       │                      │
       │1                     │N
       ▼N                     ▼
┌─────────────┐ 1     N ┌───────────────┐
│  Job        │────────▶│  Application  │◀── Candidate (User)
│  (posting)  │         │  (state mach.)│
└──────┬──────┘         └───────┬───────┘
       │1                       │1
       │N                       │N
       ▼                        ▼
┌─────────────┐           ┌───────────────┐
│ Pipeline    │           │  Interview    │
│ Stage       │           │  Assessment   │
│             │           │  Note, Score  │
└─────────────┘           └───────────────┘

   Cross-cutting:
   ─────────────────────────────────────────────
   AuditLog      → polymorphic, every write
   Notification  → polymorphic
   Activity      → activity log
   Attachment    → media library
   Embedding     → pgvector, polymorphic
   FeatureFlag   → Pennant
   Event (outbox)→ outbox pattern
```

### Core tables (first cut)
- `tenants` (id, name, slug, plan_id, custom_domain, settings JSONB)
- `users` (id, tenant_id?, email, password, type[recruiter|candidate|admin])
- `roles`, `permissions`, `role_user`, `permission_role` (Spatie)
- `teams`, `team_user`
- `jobs` (id, tenant_id, title, description, requirements JSONB, status, salary_range, location, remote)
- `pipelines`, `pipeline_stages`
- `candidates` (profile data, parsed resume JSONB)
- `applications` (id, tenant_id, job_id, candidate_id, stage_id, score, status)
- `interviews` (scheduled_at, type, video_url, transcript, ai_score)
- `assessments`, `assessment_responses`
- `offers` (terms JSONB, signed_at, signature_meta)
- `messages` (email/sms/whatsapp, threadable)
- `audit_logs` (causer, subject, event, properties JSONB, ip, ua)
- `embeddings` (subject_id, subject_type, vector(1536), model, created_at)
- `events_outbox` (event_id, type, payload, status, attempts)

---

## 5. Phase 0 — Workstation setup

**Goal:** every command works on day 1 without yak-shaving later.

### Learn
- Difference between **Docker Desktop**, **Colima**, **OrbStack** — pick one (OrbStack is fastest on macOS).
- How `composer`, `npm`, `node`, `php` versions are managed (Herd, asdf, mise).
- Why **per-project versions** matter (use `mise.toml` or `.tool-versions`).

### Build
1. Install **OrbStack** (or Docker Desktop).
2. Install **mise**: `curl https://mise.run | sh`.
3. Create `~/Projects/nexa-hire` (already done).
4. Add `mise.toml`:
   ```toml
   [tools]
   php = "8.4"
   node = "22"
   composer = "2"
   ```
5. Install GitHub CLI: `brew install gh && gh auth login`.
6. Install **Lazygit**, **k9s**, **kubectx**, **stern** for later.
7. Install **Postgres.app** *only* for the GUI client (`psql`, `pg_dump`); we'll run real Postgres in Docker.
8. Install IDE plugins: Laravel Idea (PhpStorm) or Laravel + PHP Intelephense (VS Code/Cursor).
9. Configure **commit signing** with GPG or SSH key (`git config --global commit.gpgsign true`).

### Reflect
- Why do we run services in Docker but the IDE on the host?
- What's the difference between **Composer's** `require` and `require-dev`?

---

## 6. Phase 1 — Foundations (Laravel fundamentals)

**Duration:** week 1
**Goal:** ship a vertical slice — companies can post a job, candidates can apply.

### Learn
- The request lifecycle: `public/index.php` → kernel → middleware → router → controller → response.
- Service container, service providers, deferred providers.
- Facades vs DI vs helpers — when to use which.
- Configuration & env: `config/`, `.env`, `config:cache`.
- Routing: web, api, route model binding, fallback, prefixes, sub-domain.
- Eloquent: relationships (1:1, 1:N, N:N, polymorphic, has-many-through), scopes, accessors/mutators, casts (incl. custom casts), observers, model events.
- Migrations & seeders, factories, model states.
- Form requests, validation rules (incl. `Rule` objects), custom rules.
- API resources & resource collections.
- Blade vs Inertia: pick **Inertia + Vue + TS**.
- Pint (formatter), Larastan (static analysis), Pest (tests).

### Build (PRs in this order)
1. **PR-001 `chore: bootstrap project`** — `laravel new nexa-hire --using=laravel/laravel:^13`, set up git, branch protection, conventional commits.
2. **PR-002 `chore: tooling`** — add Pint, Larastan (level max), Rector, Pest, GitHub Actions skeleton.
3. **PR-003 `feat: auth scaffolding`** — Inertia + Vue starter kit, register/login.
4. **PR-004 `feat: jobs CRUD`** — companies post jobs (title, JD, requirements, salary range).
5. **PR-005 `feat: applications`** — candidates apply, list-my-applications page.
6. **PR-006 `feat: pipeline kanban`** — drag-and-drop applications across stages (Vue + dnd-kit).
7. **PR-007 `chore: Larastan level 9 clean`** — fix all static analysis issues.
8. **PR-008 `test: pest coverage 70%+`** — feature + unit tests.

### Reflect
- Trace one request from URL to JSON output. Where would you add a metric?
- Why is `protected $fillable` important security-wise?
- Difference between `whereHas` and `with`?
- When does Eloquent N+1?

---

## 7. Phase 2 — Authentication & Authorization

**Duration:** week 2

### Learn
- Sessions vs tokens vs cookies — when each is appropriate.
- **Sanctum** for SPA + first-party API tokens.
- **Passport** as an OAuth2 server — for *third-party* integrations into your platform.
- **Socialite** for Google/GitHub/LinkedIn login.
- **Magic links** & **passkeys (WebAuthn)** — modern passwordless.
- **TOTP 2FA** (Google Authenticator).
- **Spatie Permission** — roles & permissions, *team-scoped* (per tenant).
- Policies, Gates, `authorize()` in form requests.
- **Rate limiting** with `RateLimiter` facade.

### Build
- **PR-009 `feat: sanctum auth`** — SPA cookie-based auth, CSRF flow.
- **PR-010 `feat: 2FA + recovery codes`**.
- **PR-011 `feat: passkeys (WebAuthn)`** using `web-auth/webauthn-lib`.
- **PR-012 `feat: socialite providers`** — Google, GitHub, LinkedIn.
- **PR-013 `feat: roles & permissions`** — recruiter, hiring-manager, interviewer, candidate; team-scoped.
- **PR-014 `feat: passport oauth server`** — clients can request tokens.
- **PR-015 `feat: rate limiting`** — per IP, per user, per tenant.

### Reflect
- Why is Sanctum's CSRF cookie endpoint needed?
- Difference between *Authentication* and *Authorization* (be precise).
- How would you revoke ALL of a user's sessions on password change?

---

## 8. Phase 3 — Multi-tenancy & billing

**Duration:** week 3
**Goal:** turn the app into a real SaaS.

### Learn
- Tenancy strategies: **single DB / shared schema** vs **DB-per-tenant** vs **schema-per-tenant**. Trade-offs of each.
- Sub-domain routing (`acme.nexahire.test`).
- Tenant-aware queues, cache, storage.
- **Stripe** concepts: Customer, Subscription, Invoice, PaymentIntent, Webhook idempotency.
- **Cashier** for recurring billing.
- **Laravel Pennant** for feature flags (per-plan features).

### Build
- **PR-016 `feat: tenancy via stancl/tenancy`** — DB-per-tenant, sub-domain routing, tenant context middleware.
- **PR-017 `feat: tenant onboarding wizard`** — sign up → create tenant → seed defaults.
- **PR-018 `feat: cashier + plans`** — Free / Pro / Business plans, Stripe integration, hosted checkout.
- **PR-019 `feat: feature flags`** — Pennant gating premium features (e.g., AI screening).
- **PR-020 `feat: usage metering`** — count seats, jobs, AI tokens; enforce plan limits.
- **PR-021 `feat: custom domains`** — let tenants point `careers.acme.com` at us; SSL via Let's Encrypt automation.

### Reflect
- What happens to a request when the sub-domain doesn't match a tenant?
- How do you keep migrations in sync across tenant DBs?
- How do you do a "global" search across tenants for a super-admin (without breaking isolation)?

---

## 9. Phase 4 — Queues, events, real-time

**Duration:** week 4

### Learn
- Queue drivers: sync, database, Redis, SQS — pros/cons.
- **Horizon** for monitoring Redis queues; supervisor configuration.
- Job patterns: **batched**, **chained**, **rate-limited**, `ShouldBeUnique`, `ShouldQueueAfterCommit`.
- Events & Listeners (sync vs queued listeners).
- **Broadcasting** with **Reverb** (Laravel's first-party WebSockets server).
- **Domain events** vs **integration events**.
- **Outbox pattern** for guaranteed event delivery to external systems.

### Build
- **PR-022 `chore: redis + horizon`** — Horizon dashboard at `/horizon`, multiple supervisors (default, ai, mail, billing, low).
- **PR-023 `feat: events + listeners`** — `ApplicationSubmitted`, `StageChanged`, `OfferSigned`, etc.
- **PR-024 `feat: broadcasting via Reverb`** — live kanban updates when a teammate moves a card.
- **PR-025 `feat: outbox pattern`** — persisted events table, dispatcher job, dead-letter handling.
- **PR-026 `feat: idempotent webhook receiver`** — Stripe webhooks with idempotency keys.

### Reflect
- What's the failure mode if Redis dies mid-job? How does Horizon recover?
- Why use the outbox pattern rather than just publishing inside the transaction?
- When should a listener be queued vs sync?

---

## 10. Phase 5 — Search, files, notifications

**Duration:** week 5

### Learn
- **Scout** drivers; how indexing works on save/delete.
- **Meilisearch** filterable & sortable attributes; faceting.
- **Postgres FTS** (alternative for cheap setups).
- **Filesystem** abstraction; S3 vs MinIO; **signed URLs**, **temporary URLs**, **chunked uploads**.
- **Spatie Media Library** for file conversions/thumbnails.
- **Notifications** — multi-channel (mail, database, broadcast, Slack, SMS via Twilio, WhatsApp via Twilio).
- **Mailables**, **Markdown mail**, dynamic templates.

### Build
- **PR-027 `feat: scout + meilisearch`** — index jobs, candidates, applications.
- **PR-028 `feat: faceted search UI`** — recruiter search by skills, location, status.
- **PR-029 `feat: chunked resume uploads`** — large PDFs to S3 via signed URLs, virus scan with ClamAV side-car.
- **PR-030 `feat: media conversions`** — generate avatar thumbnails, resume previews.
- **PR-031 `feat: notifications`** — multi-channel; "your application moved to Interview" email + in-app + push.
- **PR-032 `feat: mail templating`** — tenants can edit templates with variables.

### Reflect
- How would you re-index 10M records with zero downtime?
- Why store files in S3 vs the DB? When *would* DB be acceptable?
- What's the diff between presigned PUT and POST?

---

## 11. Phase 6 — AI layer

**Duration:** weeks 6–7
**Goal:** real AI features, not toys.

### Learn
- **Laravel AI SDK** (Laravel 13 first-party) — text gen, embeddings, tool-calling, vector stores.
- **Embeddings** (OpenAI `text-embedding-3-large`, or local via Ollama for dev).
- **pgvector** — `vector` columns, HNSW indexes, cosine similarity queries.
- **RAG** (retrieval-augmented generation): chunking, retrieval, context window.
- **Function calling / tools** — give the LLM safe tools to query your DB.
- **Prompt engineering**: system prompts, few-shot, chain-of-thought, output format constraints (JSON schema).
- **Guardrails**: output validation, content filters, PII redaction, prompt injection defense.
- **Streaming** responses via SSE.
- **Cost & latency** — caching, smaller models for cheap tasks.

### Build
- **PR-033 `feat: AI service abstraction`** — `App\Modules\AI\Contracts\AiClient`, swappable provider.
- **PR-034 `feat: resume parser`** — PDF → text (Tika sidecar) → structured JSON via LLM with strict schema.
- **PR-035 `feat: embeddings + pgvector`** — embed JDs and resumes; nightly re-embed job.
- **PR-036 `feat: semantic candidate search`** — recruiter searches "senior php devs in berlin who built billing systems"; vector + filters.
- **PR-037 `feat: JD-CV match score`** — hybrid (BM25 + vector), explainability surface.
- **PR-038 `feat: AI screening pipeline`** — Laravel Pipeline with stages: parse → enrich → score → flag → notify.
- **PR-039 `feat: AI interviewer (async)`** — candidate records video, Whisper transcribes, LLM scores against rubric.
- **PR-040 `feat: AI assistant (agent)`** — recruiter chat with tool-calling: `findCandidates`, `draftEmail`, `scheduleInterview`.
- **PR-041 `feat: bias detection in JDs`** — scan postings for biased language pre-publish.
- **PR-042 `chore: AI cost guardrails`** — per-tenant budgets, token usage logged, hard stops.

### Reflect
- What happens when the LLM returns invalid JSON? (Hint: schema validators + retries + repair prompts.)
- How do you protect against prompt injection from a candidate's resume?
- Why HNSW and not IVFFlat for our use case?

---

## 12. Phase 7 — Public API, GraphQL, webhooks

**Duration:** week 8

### Learn
- **REST design**: resource modeling, status codes, pagination styles (cursor vs offset), filtering, sparse fieldsets.
- **JSON:API** (Laravel 13 has first-party support).
- **API versioning** — URL (`/v1`) vs header (`Accept: application/vnd.nexahire+json; version=1`).
- **OpenAPI 3.1**, auto-generated docs, SDK generation.
- **GraphQL** with Lighthouse — schema-first, N+1 with dataloaders, complexity limits.
- **Webhooks**: signing (HMAC), retries with exponential backoff, idempotency.
- **API keys** vs **OAuth** for third-party access.

### Build
- **PR-043 `feat: public REST API v1`** — Passport-secured, JSON:API formatted.
- **PR-044 `feat: openapi spec + docs`** — auto-generated from controllers, served at `/docs`.
- **PR-045 `feat: graphql endpoint`** — Lighthouse, queries + mutations, complexity & depth limits.
- **PR-046 `feat: outbound webhooks`** — tenant configures URLs, events delivered with HMAC, retries, dead-letter.
- **PR-047 `feat: SDKs`** — generate PHP + TS SDKs from OpenAPI, publish to packagist/npm.

### Reflect
- Why HMAC signing on webhooks? How do you prevent replay attacks?
- When is GraphQL a *bad* idea?
- How would you deprecate a v1 endpoint?

---

## 13. Phase 8 — Testing & quality

**Duration:** week 9 (but you've been writing tests since day 1)

### Learn
- Test pyramid: unit → integration → feature → E2E.
- **Pest 3** — datasets, higher-order tests, architecture tests (`arch()->expect()`).
- **Browser testing** with **Dusk** or **Playwright** (Playwright preferred for modern stacks).
- **Mutation testing** with **Infection** — measures real test quality.
- **Contract tests** for external APIs (record/replay with **Pact** or VCR-style).
- **Static analysis**: Larastan level 9 → max.
- **Mutation diffing** in CI.
- **Code coverage** as a *signal*, not a goal. Aim 80%+ on services, 60%+ overall.

### Build
- **PR-048 `test: full pest coverage to 80%`**.
- **PR-049 `test: arch tests`** — enforce module boundaries, no cross-module model imports.
- **PR-050 `test: e2e via playwright`** — happy paths for recruiter and candidate.
- **PR-051 `test: mutation testing`** — Infection in CI on changed files only.
- **PR-052 `chore: contract tests for stripe/openai`**.

### Reflect
- What's the difference between a unit and integration test in Laravel land?
- Why does 100% coverage mean nothing without mutation testing?
- How do you test code that calls an LLM?

---

## 14. Phase 9 — Performance engineering

**Duration:** week 10

### Learn
- **Octane** — workers, memory leaks, stateful gotchas.
- **N+1** detection (`barryvdh/laravel-debugbar`, `beyondcode/laravel-query-detector`).
- **Database**: indexes (B-tree, GIN, HNSW), `EXPLAIN ANALYZE`, partial indexes, table partitioning.
- **Read replicas** (Laravel's `read`/`write` config).
- **Cache strategies**: cache-aside, write-through, **stampede prevention** (`Cache::lock`), **tagged caches**.
- **HTTP caching**: ETag, 304s, CDN.
- **Profilers**: **Clockwork**, **Blackfire**, **XHProf**.
- **Pulse** for prod-grade visibility.

### Build
- **PR-053 `chore: octane (frankenphp)`** — boot under Octane, fix any state leaks.
- **PR-054 `perf: kill all N+1`** — `query-detector` in CI fails the build.
- **PR-055 `perf: indexes audit`** — every slow query has an index, every index is justified.
- **PR-056 `perf: cache hot endpoints`** — tenant settings, plan limits.
- **PR-057 `perf: read replica routing`** — analytics queries hit replicas.
- **PR-058 `perf: pulse + slow query log`** — dashboards.

### Reflect
- What's the danger of using static properties under Octane?
- When would adding an index *hurt* performance?
- What's the cache stampede problem and how do you solve it?

---

## 15. Phase 10 — Docker & local devops

**Duration:** week 10 (parallel)

### Learn
- Dockerfile best practices: multi-stage, distroless/alpine, non-root user, `.dockerignore`, BuildKit cache mounts.
- Compose: profiles, healthchecks, depends_on, networks, named volumes.
- **Sail** vs custom Compose — we'll write a custom one (more learning).
- Local TLS via **mkcert** + Traefik.

### Build
**Files to create:**
- `docker/php/Dockerfile` (multi-stage: composer-deps → npm-build → runtime)
- `docker/nginx/default.conf` (or use FrankenPHP, no Nginx needed)
- `compose.yaml` with services:
  - `app` (FrankenPHP + Octane)
  - `worker` (Horizon)
  - `scheduler` (runs `schedule:work`)
  - `reverb` (WebSockets)
  - `postgres` (with `pgvector` extension)
  - `redis`
  - `meilisearch`
  - `minio` + `minio-init`
  - `mailpit`
  - `traefik`
- `compose.override.yaml` for local-only mounts/dev overrides.
- `Makefile` with `make up`, `make down`, `make test`, `make shell`, `make migrate`, etc.

**PRs:**
- **PR-059 `chore: docker compose dev env`**.
- **PR-060 `chore: makefile + dx scripts`**.
- **PR-061 `chore: traefik + mkcert`** — `https://nexahire.test`.
- **PR-062 `chore: postgres init with pgvector`**.

### Reflect
- Why multi-stage builds?
- What's the difference between `COPY` and `ADD`?
- Why run as non-root inside the container?

---

## 16. Phase 11 — CI/CD & GitOps

**Duration:** week 11 (parallel with 12)

### Learn
- GitHub Actions: jobs, matrices, caching, reusable workflows, OIDC to AWS (no static keys).
- **Trunk-based development** vs Git Flow — pick **trunk-based** with short-lived feature branches.
- Conventional commits, semantic versioning, automated changelogs.
- **PR previews** — ephemeral environments per PR.
- **Container registry**: GHCR or ECR.
- **GitOps**: ArgoCD watches a repo of manifests; commits = deploys.

### Build
**`.github/workflows/`:**
- `ci.yaml` — on PR: lint (Pint), static analysis (Larastan), tests (Pest, parallel), security scan (`composer audit`, `npm audit`, Snyk), build Docker image, push to GHCR.
- `deploy-staging.yaml` — on merge to `main`: deploy to staging via ArgoCD bump.
- `deploy-prod.yaml` — on tag: deploy to prod with manual approval.
- `preview.yaml` — per PR: spin up a preview env in K8s, comment URL on PR.

**PRs:**
- **PR-063 `ci: full pipeline`**.
- **PR-064 `ci: security scanning`** — Trivy on images, gitleaks for secrets, OWASP dep-check.
- **PR-065 `ci: preview environments`**.
- **PR-066 `ci: release-please`** — auto-changelog & version bump.
- **PR-067 `chore: argocd setup`**.

### Reflect
- Why OIDC instead of long-lived AWS keys in CI?
- What's the blast radius of a leaked deploy key vs an OIDC role?
- Why is trunk-based easier for CD than Git Flow?

---

## 17. Phase 12 — Cloud infrastructure (Terraform)

**Duration:** week 11

### Learn
- **Terraform** basics: state, backends, modules, workspaces, plan/apply.
- Remote state in S3 with DynamoDB locking.
- AWS networking: VPC, subnets (public/private/intra), NAT, security groups.
- Managed services: **RDS Aurora Postgres**, **ElastiCache Redis**, **OpenSearch** or self-hosted Meili, **S3**, **CloudFront**, **Route53**, **ACM**, **SES**, **Secrets Manager**.
- IAM least privilege, OIDC providers.
- **EKS** essentials (or k3s for cheap learning) — node groups, addons.

### Build
**`infra/terraform/` layout:**
```
infra/terraform/
  modules/
    network/
    database/
    cache/
    storage/
    cdn/
    eks/
    secrets/
  envs/
    staging/
    production/
```
- **PR-068 `infra: vpc + subnets`**.
- **PR-069 `infra: rds aurora postgres`** with read replica + automated backups.
- **PR-070 `infra: elasticache redis`**.
- **PR-071 `infra: s3 + cloudfront`**.
- **PR-072 `infra: route53 + acm`**.
- **PR-073 `infra: eks cluster`** + managed node group, irsa for service accounts.
- **PR-074 `infra: secrets manager`**.

### Reflect
- Why is Terraform state sensitive? What happens if it's lost?
- Difference between Security Groups and NACLs?
- IRSA vs node IAM roles — which and why?

---

## 18. Phase 13 — Kubernetes deploy

**Duration:** week 11–12

### Learn
- Kubernetes objects: Deployment, Service, Ingress, ConfigMap, Secret, HPA, PDB, NetworkPolicy, ServiceAccount.
- **Helm 3** — charts, values, templates, hooks.
- Probes: liveness, readiness, startup. Why each matters.
- Rolling updates vs blue-green vs canary (use **Argo Rollouts**).
- Horizontal Pod Autoscaler — scale on CPU + custom metrics (queue depth via KEDA).
- **KEDA** for event-driven autoscaling (scale workers on Redis queue depth).
- **External Secrets Operator** to sync from AWS Secrets Manager.

### Build
**`infra/helm/nexahire/`** with sub-charts/templates:
- `app` (web Octane), `worker`, `scheduler`, `reverb`, `migrations` (Job).
- `keda-scaledobject.yaml` for workers.
- `argocd/applications.yaml` for GitOps.

**PRs:**
- **PR-075 `deploy: helm chart`**.
- **PR-076 `deploy: argocd app of apps`**.
- **PR-077 `deploy: keda autoscaling for workers`**.
- **PR-078 `deploy: argo rollouts canary for web`**.
- **PR-079 `deploy: external-secrets`**.
- **PR-080 `deploy: cert-manager + letsencrypt`**.

### Reflect
- Why is the readiness probe critical during deploys?
- What happens in a rolling update if migrations are required? (Hint: pre-deploy hook job, **expand-contract** schema changes.)
- How do you do a zero-downtime DB migration on a 100M-row table?

---

## 19. Phase 14 — Observability & SRE

**Duration:** week 12

### Learn
- **Three pillars**: metrics (Prometheus), logs (Loki), traces (Tempo). Plus **events** as a 4th pillar.
- **OpenTelemetry** as the vendor-neutral standard. Auto + manual instrumentation in PHP.
- **RED** method (Rate, Errors, Duration) for services; **USE** (Utilization, Saturation, Errors) for resources.
- **SLI / SLO / SLA / error budgets**.
- **Alert design**: page only on user-impact symptoms, not causes.
- **Runbooks** for every alert.

### Build
- **PR-081 `obs: opentelemetry instrumentation`** — HTTP, DB, Redis, queue jobs.
- **PR-082 `obs: prometheus + grafana stack`** in K8s; expose `/metrics`.
- **PR-083 `obs: loki for logs`** — JSON structured logs.
- **PR-084 `obs: tempo for traces`** — distributed traces across web → queue → AI.
- **PR-085 `obs: sentry`** — errors with release tracking, source maps.
- **PR-086 `obs: dashboards + alerts`** — RED dashboards, on-call alerts to Slack.
- **PR-087 `obs: SLOs`** — define + track availability and latency SLOs with burn-rate alerts.
- **PR-088 `chore: runbooks`** in `/docs/runbooks/`.

### Reflect
- Why is logging at INFO too noisy in prod?
- What's the difference between a metric and a log?
- Burn-rate alerts: why two windows?

---

## 20. Phase 15 — Security hardening

**Duration:** week 12

### Learn
- **OWASP Top 10** — drill each one against your code.
- **CSP**, **HSTS**, **Subresource Integrity**, **SameSite cookies**, **Permissions-Policy**.
- **CSRF** in Laravel — how Sanctum's flow works.
- **IDOR** — every controller checks ownership/policy.
- **SSRF** — outbound HTTP from server-side never trusts user input.
- **Mass assignment** — `$fillable` audit.
- **Secrets**: never in code, never in `.env` in prod, always in Secrets Manager / Vault.
- **Encryption at rest**: per-field via custom casts (`EncryptedCast`); KMS for key management.
- **Least privilege** for IAM, DB users, container users.
- **Supply chain**: pin versions, signed commits, SBOM (Syft), image signing (Cosign).
- **Dependency scanning** (Snyk / Dependabot / `composer audit`).
- **Pen-test mindset** — try to break your own app.

### Build
- **PR-089 `sec: full headers (CSP, HSTS, etc.)`**.
- **PR-090 `sec: encrypted casts for PII`** (SSN, salary, DOB).
- **PR-091 `sec: audit log polymorphic`** — every read/write on sensitive entities.
- **PR-092 `sec: HIBP password check`** on signup/change.
- **PR-093 `sec: sast in CI`** — Psalm + Larastan + Semgrep + Snyk.
- **PR-094 `sec: container hardening`** — distroless, read-only rootfs, no-new-privileges, drop all caps.
- **PR-095 `sec: network policies`** in K8s — default deny.
- **PR-096 `sec: SBOM + cosign`** — sign images, verify in admission controller.
- **PR-097 `sec: pen-test session`** — log every finding & fix.

### Reflect
- How would you discover IDOR in your own app?
- Why is `Strict-Transport-Security` not enough alone?
- What's the difference between encryption at rest in the DB vs at the field level?

---

## 21. Phase 16 — Compliance, polish, launch

**Duration:** week 12 (final stretch)

### Learn
- **GDPR** essentials: lawful basis, data export, right-to-be-forgotten, data residency.
- **SOC2-style controls**: change management, access reviews, backup tests.
- **Data retention** policies.
- **Disaster recovery**: RPO / RTO; back up + *restore drills*.

### Build
- **PR-098 `feat: GDPR data export`** — user requests ZIP of their data.
- **PR-099 `feat: GDPR right-to-be-forgotten`** — anonymization workflow.
- **PR-100 `feat: data residency by region`** — EU tenants in EU cluster.
- **PR-101 `chore: backup + restore drill`** — quarterly DR exercise.
- **PR-102 `docs: architecture decision records`** in `/docs/adr/`.
- **PR-103 `docs: public docs site`** (VitePress / Mintlify).
- **PR-104 `chore: launch checklist`** — close every box.

### Reflect
- Can you restore your DB to 17:32 yesterday? Prove it.
- Where does PII live in your system? Map every column.

---

## 22. Concept coverage matrix

> Tick each as you complete it. If anything is unticked at the end, you haven't finished.

### Laravel core
- [ ] Routing (web/api/sub-domain/fallback)
- [ ] Middleware (global/route/group)
- [ ] Controllers (resource/single-action/invokable)
- [ ] Service container & providers (incl. deferred)
- [ ] Facades, helpers, contracts
- [ ] Configuration & env
- [ ] Eloquent (all relationship types)
- [ ] Polymorphic relations
- [ ] Custom casts
- [ ] Observers & model events
- [ ] Migrations & seeders & factories
- [ ] Form requests & custom validation rules
- [ ] API resources & JSON:API
- [ ] Localization (10+ languages)
- [ ] Mail (mailables, markdown, dynamic templates)
- [ ] Notifications (multi-channel)
- [ ] Events & listeners
- [ ] Broadcasting (Reverb)
- [ ] Queues (Horizon, batches, chains, unique, rate-limited)
- [ ] Scheduler
- [ ] File storage (local/S3, signed URLs, chunked)
- [ ] Sanctum
- [ ] Passport (OAuth2 server)
- [ ] Socialite
- [ ] Pennant feature flags
- [ ] Cashier (Stripe)
- [ ] Pulse, Telescope, Horizon dashboards
- [ ] Octane (FrankenPHP/Swoole)
- [ ] Scout + Meilisearch
- [ ] Spatie Permission (team-scoped)
- [ ] Custom artisan commands
- [ ] Pipelines (Laravel's Pipeline class)
- [ ] Macros & macroable
- [ ] Custom service providers
- [ ] Package development (publish a private package)

### Laravel 13 specifics
- [ ] Laravel AI SDK (text, embeddings, tools)
- [ ] PHP Attributes across the framework
- [ ] JSON:API resources (first-party)
- [ ] Vector queries on pgvector
- [ ] `#[DebounceFor]` queue debouncing
- [ ] Queue routing by class
- [ ] JsonFormatter for log context

### Architecture
- [ ] Modular monolith (boundaries enforced via deptrac)
- [ ] DDD-lite (entities, VOs, services, actions, repos)
- [ ] CQRS-lite (separate read models)
- [ ] Event sourcing (audit trail)
- [ ] Outbox pattern
- [ ] Saga (cross-module workflow)
- [ ] Idempotency keys
- [ ] API versioning strategy
- [ ] OpenAPI 3.1 + SDK gen
- [ ] GraphQL (Lighthouse)
- [ ] Webhooks (HMAC + retries)
- [ ] Multi-tenancy (DB-per-tenant)
- [ ] Data residency

### Database
- [ ] PostgreSQL deep dive (JSONB, FTS, partitioning)
- [ ] pgvector with HNSW
- [ ] Read replicas
- [ ] Zero-downtime migrations (expand/contract)
- [ ] Indexes (B-tree, GIN, partial, HNSW)
- [ ] EXPLAIN ANALYZE habit

### AI
- [ ] LLM text generation
- [ ] Tool-calling agents
- [ ] Embeddings + vector search
- [ ] RAG
- [ ] Streaming (SSE)
- [ ] Output schema enforcement
- [ ] Prompt-injection defense
- [ ] Cost guardrails

### Testing
- [ ] Pest (unit + feature)
- [ ] Architecture tests
- [ ] E2E via Playwright
- [ ] Mutation testing
- [ ] Contract tests
- [ ] Static analysis level max (Larastan)
- [ ] Rector automated refactors
- [ ] 80%+ coverage on services

### DevOps
- [ ] Docker multi-stage
- [ ] Compose v2 with profiles
- [ ] Makefile DX
- [ ] GitHub Actions CI
- [ ] Trunk-based dev + conventional commits
- [ ] Semantic releases
- [ ] PR preview envs
- [ ] Container scanning (Trivy)
- [ ] SBOM (Syft) + Cosign signing
- [ ] OIDC to AWS (no static keys)

### Cloud
- [ ] Terraform modules + remote state
- [ ] AWS VPC, subnets, SGs
- [ ] RDS Aurora Postgres + replicas
- [ ] ElastiCache Redis
- [ ] S3 + CloudFront + Route53 + ACM
- [ ] SES for mail
- [ ] Secrets Manager
- [ ] EKS + IRSA
- [ ] cert-manager + Let's Encrypt
- [ ] External Secrets

### Kubernetes
- [ ] Deployments, Services, Ingress
- [ ] HPA + KEDA (queue-depth scaling)
- [ ] PDB, NetworkPolicy
- [ ] Helm chart
- [ ] ArgoCD GitOps
- [ ] Argo Rollouts canary
- [ ] Pre-deploy migration jobs
- [ ] Probes (liveness/readiness/startup)
- [ ] Resource requests/limits

### Observability
- [ ] OpenTelemetry instrumentation
- [ ] Prometheus + Grafana
- [ ] Loki for logs
- [ ] Tempo for traces
- [ ] Sentry for errors
- [ ] SLOs + burn-rate alerts
- [ ] Runbooks

### Security
- [ ] OWASP Top 10 audit
- [ ] CSP / HSTS / SameSite / Permissions-Policy
- [ ] CSRF correctness
- [ ] IDOR defense (policy on every read)
- [ ] SSRF defense
- [ ] Mass-assignment audit
- [ ] Encryption at rest (field-level)
- [ ] HIBP password breach check
- [ ] 2FA + passkeys
- [ ] Audit log
- [ ] Container hardening (distroless, non-root, RO rootfs)
- [ ] NetworkPolicy default-deny
- [ ] Pen-test of own app
- [ ] Image signing + admission policy
- [ ] Dependency scanning gates

### Compliance & SRE
- [ ] GDPR data export & erasure
- [ ] Data residency
- [ ] Backup + tested restore (DR drill)
- [ ] ADRs
- [ ] Public docs site
- [ ] Launch checklist

---

## 23. Daily learning rituals

- **Morning (15 min):** read the Laravel docs for one feature you'll touch today.
- **Coding (focused 90-min blocks):** Pomodoro 4 × 25 with breaks; no Discord.
- **Evening (15 min):** write 1–3 paragraphs in `/docs/journal/YYYY-MM-DD.md`:
  - What did I build?
  - What broke and why?
  - What concept clicked today?
  - What's still confusing?
- **Weekly (1 hr Sunday):** review the past week's PRs as a *reviewer*. Leave comments on your own code as if you were a senior reviewing a junior.

### Forcing-functions to lock in learning
- **Teach it:** post a Twitter/blog/dev.to writeup at the end of every phase.
- **Open-source it:** push to GitHub publicly. Public commit history is itself a portfolio.
- **Talk about it:** present one phase to a friend or in a meetup.

---

## 24. Reading list & references

### Books (read these alongside building)
- *Refactoring*, Martin Fowler — language-agnostic but you'll feel it in every PR.
- *Domain-Driven Design Distilled*, Vaughn Vernon — DDD basics in 150 pages.
- *Designing Data-Intensive Applications*, Martin Kleppmann — the bible for systems.
- *The Twelve-Factor App* — short, free, essential.
- *Database Internals*, Alex Petrov — how DBs really work.
- *Site Reliability Engineering* (Google, free online) — SRE fundamentals.

### Laravel-specific
- Official docs (read 100% of them — yes, all)
- Spatie blog
- Laravel News
- *"Laravel Beyond CRUD"* by Spatie (free online)

### DevOps / Cloud
- AWS Well-Architected Framework
- Kubernetes the Hard Way (Kelsey Hightower)
- *"Production-Grade Container Orchestration with Kubernetes"* — official docs

### Security
- OWASP Top 10
- OWASP Application Security Verification Standard (ASVS)
- *"The Tangled Web"* by Michal Zalewski

### AI
- OpenAI cookbook
- Anthropic prompting guide
- "RAG patterns" (LangChain docs are useful even if you don't use LangChain)

---

## Final word

You will be tempted to skip phases. Don't. The point is *not* to ship NexaHire — the point is that **building NexaHire forces you to learn every concept a senior engineer is expected to know**. The product is the by-product.

When you finish:
- You'll have one of the most impressive personal portfolios on GitHub.
- You'll be able to walk into any senior Laravel interview and answer *every* system-design question with concrete experience.
- You'll have written ~50–100k lines across the stack.

**Now make the first commit. Today. Before you close this file.**

```bash
cd ~/Projects/nexa-hire
git init
git add IMPLEMENTATION.md
git commit -m "docs: add 16-phase implementation roadmap"
gh repo create nexa-hire --public --source=. --push
```
