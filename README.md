# 🤖 Dev Twin — Project Blueprint

> **A context-aware AI code reviewer that learns your entire codebase and reviews Pull Requests like a senior teammate.**
> Backend: Spring Boot · Reviews: any language / framework · Integrates with: GitHub

---

## Table of Contents
1. [Project Overview](#1-project-overview)
2. [Goals & Non-Goals](#2-goals--non-goals)
3. [Feature List](#3-feature-list)
4. [High-Level Architecture](#4-high-level-architecture)
5. [Tech Stack](#5-tech-stack)
6. [Libraries & Dependencies](#6-libraries--dependencies)
7. [Module / Package Structure](#7-module--package-structure)
8. [Core Flows (Step by Step)](#8-core-flows-step-by-step)
9. [Multi-Language Code Intelligence](#9-multi-language-code-intelligence)
10. [RAG & Context Building](#10-rag--context-building)
11. [LLM Layer](#11-llm-layer)
12. [Data Model](#12-data-model)
13. [REST / SSE API](#13-rest--sse-api)
14. [GitHub Integration Details](#14-github-integration-details)
15. [Job Processing System](#15-job-processing-system)
16. [Rules Engine (`.devtwin.yml`)](#16-rules-engine-devtwinyml)
17. [Feedback & Learning](#17-feedback--learning)
18. [Security](#18-security)
19. [Non-Functional Requirements](#19-non-functional-requirements)
20. [Observability](#20-observability)
21. [Testing Strategy](#21-testing-strategy)
22. [DevOps & Deployment](#22-devops--deployment)
23. [Configuration Reference](#23-configuration-reference)
24. [Build Roadmap (Phases)](#24-build-roadmap-phases)
25. [Knowledge Base — Concepts to Learn](#25-knowledge-base--concepts-to-learn)
26. [Interview Prep — Questions This Project Answers](#26-interview-prep--questions-this-project-answers)
27. [Resume Description](#27-resume-description)
28. [Future Ideas](#28-future-ideas)

---

## 1. Project Overview

### Problem
- Code review is slow; senior developers are a bottleneck.
- Existing AI review bots only look at the **diff**, so they give generic advice ("add a null check") and miss project-specific problems.
- They don't know the team's conventions, existing helpers, or which other files a change will break.

### Solution
Dev Twin **indexes the whole repository** (code structure, call graph, embeddings, conventions, past reviews). When a PR is opened, it:
1. Fetches the diff.
2. Retrieves the **relevant project context** (callers, similar code, existing utilities, team rules).
3. Asks an LLM to review the diff **with that context**.
4. Posts **inline comments** on the PR.
5. Learns from team feedback (👍/👎, resolved threads) to improve over time.

### Example of a context-aware comment
> ⚠️ **Convention violation** — You're calling `userRepository.findById()` directly. Everywhere else in this project (`OrderService`, `ProfileService`) uses `UserService.getActiveUser()`, which also filters soft-deleted users. This may expose deleted accounts.

---

## 2. Goals & Non-Goals

### Goals
- Review PRs in **any language** with graceful degradation (full → basic → fallback support).
- Comments must be **project-aware**, not generic.
- Be **production-grade**: async, idempotent, retryable, observable, secure, multi-tenant.
- Show **live progress** of each review on a dashboard.
- Be cheap to run: skip unnecessary LLM calls, cap cost per repo.

### Non-Goals (v1)
- Auto-fixing code / pushing commits.
- GitLab / Bitbucket support (designed for later via an adapter interface).
- Training or fine-tuning our own model.

---

## 3. Feature List

Priority: **P0** = must have (MVP) · **P1** = core differentiator · **P2** = nice to have

### 3.1 GitHub Integration
| # | Feature | Priority |
|---|---|---|
| G1 | GitHub App installation (org/user installs app on repos) | P0 |
| G2 | Webhook receiver with HMAC-SHA256 signature verification | P0 |
| G3 | Fetch PR diff + changed files | P0 |
| G4 | Post review with multiple inline comments in one API call | P0 |
| G5 | PR summary comment (what changed, risk level) | P1 |
| G6 | `/devtwin` slash commands in PR comments (`review`, `explain`, `ignore`, `summary`) | P1 |
| G7 | Reply to developer replies on bot comments (conversational) | P2 |
| G8 | GitHub Check Run status (✅ / ⚠️) on the PR | P2 |

### 3.2 Code Intelligence (Indexing)
| # | Feature | Priority |
|---|---|---|
| C1 | Clone repo with JGit, walk files, honor ignore patterns | P0 |
| C2 | Language detection (extension + shebang + project files) | P0 |
| C3 | Tree-sitter parsing → symbols (classes, functions, methods, imports) | P1 |
| C4 | Symbol-aware chunking (chunk = method/class, not random lines) | P1 |
| C5 | Embeddings stored in PGVector | P0 |
| C6 | Call / dependency graph (who calls whom, who imports what) | P1 |
| C7 | Incremental re-indexing on push to default branch (only changed files) | P1 |
| C8 | Project profiler: detect frameworks (Spring, React, Django, Express, Go…) | P1 |
| C9 | Convention mining: auto-detect patterns (DI style, naming, error handling) | P2 |

### 3.3 Review Engine
| # | Feature | Priority |
|---|---|---|
| R1 | Basic diff review (no context) | P0 |
| R2 | Context-aware review using RAG | P1 ⭐ |
| R3 | Impact analysis: "this change affects these callers" | P1 ⭐ |
| R4 | Duplicate-logic detection ("this already exists in X") | P1 |
| R5 | Structured output (file, line, severity, category, message, suggestion) | P0 |
| R6 | Line validation — only comment on lines that exist in the diff | P0 |
| R7 | Severity levels: `CRITICAL`, `MAJOR`, `MINOR`, `NIT` | P0 |
| R8 | Categories: bug, security, performance, convention, duplication, readability, test | P0 |
| R9 | Token budget allocator (fit best context into model limit) | P1 |
| R10 | Large PR handling (split by file groups, skip generated/lock files) | P1 |
| R11 | GitHub "suggestion" blocks (one-click apply fixes) | P2 |

### 3.4 Rules & Learning
| # | Feature | Priority |
|---|---|---|
| L1 | `.devtwin.yml` per repo (rules, ignore paths, severity threshold, language opts) | P1 |
| L2 | Static rules (regex / AST queries) executed **without** LLM | P1 |
| L3 | Natural-language rules passed to LLM | P1 |
| L4 | Feedback capture: 👍/👎 reactions, resolved threads, ignored comments | P1 |
| L5 | Acceptance-rate per category/rule; auto-suppress noisy categories | P2 |
| L6 | Past accepted comments used as few-shot examples (RAG over review history) | P2 |

### 3.5 Platform & Dashboard
| # | Feature | Priority |
|---|---|---|
| D1 | Login with GitHub (OAuth2) | P0 |
| D2 | List installed repos, indexing status, re-index button | P0 |
| D3 | Review history per repo/PR with comments | P0 |
| D4 | **Live review progress** (SSE): Queued → Fetching → Context → Reviewing → Posting → Done | P1 |
| D5 | Analytics: reviews/day, avg time, acceptance rate, top issue categories, LLM cost | P2 |
| D6 | Repo settings page (edit rules, model choice, cost cap) | P2 |
| D7 | "Ask the codebase" chat (Q&A over indexed repo) | P2 |

### 3.6 Reliability & Ops
| # | Feature | Priority |
|---|---|---|
| O1 | Async job queue (RabbitMQ) with retries + dead-letter queue | P0 |
| O2 | Idempotency (same webhook delivered twice → reviewed once) | P0 |
| O3 | Stale job cancellation (new commit pushed mid-review) | P1 |
| O4 | Rate limiting per installation (Redis + Bucket4j) | P1 |
| O5 | Circuit breaker + retry for LLM & GitHub API (Resilience4j) | P1 |
| O6 | Metrics, structured logs, correlation IDs, dashboards | P1 |
| O7 | Cost tracking & monthly cap per repo | P2 |

---

## 4. High-Level Architecture

```
                        ┌──────────────────────────┐
                        │        GitHub            │
                        │ (PRs, pushes, comments)  │
                        └──────┬────────────▲──────┘
                    webhooks   │            │  REST API (diffs, files, post review)
                               ▼            │
┌──────────────┐   REST/SSE ┌─────────────────────────────────────────────┐
│  React       │◄──────────►│            Dev Twin API (Spring Boot)       │
│  Dashboard   │            │  Webhook · Auth · Repos · Reviews · SSE     │
└──────────────┘            └───┬───────────────┬───────────────┬─────────┘
                                │ publish job   │ read/write    │ pub/sub progress
                                ▼               ▼               ▼
                        ┌─────────────┐  ┌──────────────┐  ┌─────────┐
                        │  RabbitMQ   │  │ PostgreSQL   │  │  Redis  │
                        │ review.q    │  │ + PGVector   │  │ cache,  │
                        │ index.q     │  │              │  │ locks,  │
                        │ *.dlq       │  │              │  │ ratelim │
                        └──────┬──────┘  └──────▲───────┘  └────▲────┘
                               │ consume        │               │
                               ▼                │               │
                ┌──────────────────────────────────────────────────────┐
                │              Workers (Spring Boot, same codebase)    │
                │  ┌───────────┐  ┌───────────┐  ┌──────────────────┐  │
                │  │ Indexer   │  │ Reviewer  │  │ Feedback poller  │  │
                │  │ JGit →    │  │ diff →    │  │ (scheduled)      │  │
                │  │ TreeSitter│  │ context → │  │                  │  │
                │  │ → embed   │  │ LLM → post│  │                  │  │
                │  └───────────┘  └─────┬─────┘  └──────────────────┘  │
                └───────────────────────┼──────────────────────────────┘
                                        ▼
                              ┌───────────────────┐
                              │ LLM Provider      │
                              │ (Claude / OpenAI /│
                              │  Ollama local)    │
                              └───────────────────┘
```

**Style:** Modular monolith. One codebase, two runtime roles via Spring profiles:
- `api` profile → webhooks, REST, SSE, OAuth login
- `worker` profile → queue consumers (indexer, reviewer, feedback poller)

Both can run in one process locally and be scaled separately in production.

---

## 5. Tech Stack

| Layer | Technology | Why |
|---|---|---|
| Language | **Java 21 (LTS)** | Records, sealed types, pattern matching, **virtual threads** |
| Framework | **Spring Boot 3.5.x** | Industry standard; (upgrade path to 4.x later) |
| AI | **Spring AI** | Unified API for chat models, embeddings, vector stores, structured output |
| LLM (chat) | **Claude** (`claude-sonnet-5-5` for reviews, `claude-haiku-5-5` for cheap tasks) — or OpenAI — or **Ollama** (local, free) | Swappable via Spring AI |
| Embeddings | **Ollama `nomic-embed-text`** (free, local) or OpenAI `text-embedding-3-small` | Claude has no embedding model, so embeddings use a separate provider |
| Database | **PostgreSQL 16 + pgvector** | Relational + vector search in one DB |
| Migrations | **Flyway** | Versioned schema |
| Queue | **RabbitMQ 3.13** | Simple routing, DLQ, per-message ack; great for job queues |
| Cache / Locks / Pub-Sub | **Redis 7** | Token cache, distributed locks, rate limiting, SSE fan-out |
| Code parsing | **Tree-sitter** (via `tree-sitter-ng`) | One parser API for 40+ languages |
| Git | **JGit** | Clone / pull / diff in pure Java |
| HTTP client | **Spring `RestClient` / `WebClient`** | GitHub API calls |
| Resilience | **Resilience4j** | Retry, circuit breaker, rate limiter |
| Rate limiting | **Bucket4j + Redis** | Per-installation limits |
| Security | **Spring Security + OAuth2 Client** | Login with GitHub, JWT for GitHub App |
| Real-time | **SSE (`SseEmitter`)** + Redis Pub/Sub | Live review progress |
| Observability | **Actuator, Micrometer, Prometheus, Grafana**, JSON logs | Metrics + dashboards |
| API docs | **springdoc-openapi** | Swagger UI |
| Testing | **JUnit 5, Mockito, AssertJ, Testcontainers, WireMock, ArchUnit** | Unit + integration + architecture tests |
| Frontend | **React 18 + Vite + TypeScript + Tailwind + shadcn/ui + TanStack Query** | Fast modern dashboard |
| Containers | **Docker + Docker Compose** | One-command local stack |
| CI | **GitHub Actions** | Build, test, image publish |
| Local webhooks | **smee.io** or **ngrok** | Forward GitHub webhooks to localhost |

> ℹ️ Always verify the latest stable versions at project setup time.

---

## 6. Libraries & Dependencies

### 6.1 Backend (Maven)

```xml
<!-- ===== Spring Boot starters ===== -->
spring-boot-starter-web              <!-- REST controllers, SSE -->
spring-boot-starter-webflux          <!-- WebClient (optional; RestClient may suffice) -->
spring-boot-starter-validation       <!-- @Valid, Bean Validation -->
spring-boot-starter-data-jpa         <!-- JPA / Hibernate -->
spring-boot-starter-security         <!-- Security filter chain -->
spring-boot-starter-oauth2-client    <!-- Login with GitHub -->
spring-boot-starter-amqp             <!-- RabbitMQ -->
spring-boot-starter-data-redis       <!-- Redis -->
spring-boot-starter-cache            <!-- @Cacheable -->
spring-boot-starter-actuator         <!-- health, metrics -->

<!-- ===== Spring AI ===== -->
spring-ai-bom                                  <!-- version management -->
spring-ai-starter-model-anthropic              <!-- Claude chat -->
spring-ai-starter-model-ollama                 <!-- local chat + embeddings -->
spring-ai-starter-model-openai                 <!-- optional -->
spring-ai-starter-vector-store-pgvector        <!-- PGVector VectorStore -->

<!-- ===== Database ===== -->
org.postgresql:postgresql
org.flywaydb:flyway-core
org.flywaydb:flyway-database-postgresql
com.pgvector:pgvector                          <!-- vector type for native queries -->

<!-- ===== Code intelligence ===== -->
org.eclipse.jgit:org.eclipse.jgit              <!-- git clone/pull/diff -->
io.github.bonede:tree-sitter                   <!-- Tree-sitter core (Java bindings) -->
io.github.bonede:tree-sitter-java
io.github.bonede:tree-sitter-python
io.github.bonede:tree-sitter-javascript
io.github.bonede:tree-sitter-typescript
io.github.bonede:tree-sitter-go
io.github.java-diff-utils:java-diff-utils      <!-- unified diff parsing -->

<!-- ===== GitHub App auth ===== -->
io.jsonwebtoken:jjwt-api / jjwt-impl / jjwt-jackson   <!-- RS256 JWT for GitHub App -->
org.bouncycastle:bcpkix-jdk18on                        <!-- read PKCS#1 PEM private key -->

<!-- ===== Config / parsing ===== -->
com.fasterxml.jackson.dataformat:jackson-dataformat-yaml   <!-- .devtwin.yml -->

<!-- ===== Resilience & rate limiting ===== -->
io.github.resilience4j:resilience4j-spring-boot3
com.bucket4j:bucket4j-redis (bucket4j_jdk17-redis-common / lettuce)

<!-- ===== Observability ===== -->
io.micrometer:micrometer-registry-prometheus
io.micrometer:micrometer-tracing-bridge-otel   <!-- trace/correlation IDs -->
net.logstash.logback:logstash-logback-encoder  <!-- JSON logs -->

<!-- ===== Productivity ===== -->
org.projectlombok:lombok
org.mapstruct:mapstruct + mapstruct-processor
org.springdoc:springdoc-openapi-starter-webmvc-ui

<!-- ===== Testing ===== -->
spring-boot-starter-test                        <!-- JUnit 5, Mockito, AssertJ -->
spring-security-test
spring-rabbit-test
org.testcontainers:junit-jupiter / postgresql / rabbitmq
spring-boot-testcontainers
org.wiremock:wiremock-standalone                <!-- mock GitHub & LLM APIs -->
com.tngtech.archunit:archunit-junit5            <!-- enforce module boundaries -->
org.jacoco:jacoco-maven-plugin                  <!-- coverage -->
```

> Tree-sitter alternative: `io.github.tree-sitter:jtreesitter` (official, needs Java 22+ FFM API). `tree-sitter-ng` is chosen for Java 21 + bundled native libs on Windows/Linux/macOS.

### 6.2 Frontend (npm)
```
react, react-dom, react-router-dom
typescript, vite
@tanstack/react-query        # server state
axios                        # HTTP
tailwindcss, shadcn/ui, lucide-react   # UI
recharts                     # analytics charts
react-syntax-highlighter     # show code snippets in review history
date-fns
(native EventSource API for SSE)
```

### 6.3 Infrastructure (Docker images)
```
pgvector/pgvector:pg16
redis:7-alpine
rabbitmq:3.13-management
ollama/ollama                (optional, local LLM + embeddings)
prom/prometheus
grafana/grafana
```

---

## 7. Module / Package Structure

```
dev-twin/
├── backend/
│   ├── pom.xml
│   └── src/main/java/com/devtwin/
│       ├── DevTwinApplication.java
│       │
│       ├── common/              # base entities, exceptions, error handler, utils, correlation id
│       ├── config/              # Security, RabbitMQ, Redis, AI, OpenAPI configs
│       │
│       ├── github/              # GitHub integration (adapter)
│       │   ├── webhook/         # WebhookController, SignatureVerifier, event DTOs, dispatcher
│       │   ├── auth/            # GitHubAppJwtService, InstallationTokenService (cached)
│       │   ├── client/          # GitHubClient: PRs, files, diffs, reviews, reactions
│       │   └── model/           # GitHub API DTOs
│       │
│       ├── installation/        # Installation & Repository entities, services
│       │
│       ├── indexing/            # Indexing pipeline
│       │   ├── RepoCloner        (JGit)
│       │   ├── FileWalker        (ignore rules, binary/generated detection)
│       │   ├── IndexingService   (full + incremental)
│       │   └── IndexJobConsumer
│       │
│       ├── codeintel/           # Language-agnostic code intelligence
│       │   ├── language/        # LanguageDetector, Language enum
│       │   ├── parser/          # TreeSitterParser, ParsedTree
│       │   ├── extractor/       # LanguageExtractor (interface) + JavaExtractor, PythonExtractor, TsExtractor...
│       │   ├── chunking/        # SymbolChunker, LineChunker (fallback)
│       │   ├── graph/           # CodeGraphService (calls, imports, references)
│       │   └── profile/         # ProjectProfiler (framework detection)
│       │
│       ├── diff/                # UnifiedDiffParser, DiffHunk, ChangedSymbolResolver
│       │
│       ├── context/             # RAG context building
│       │   ├── retrievers/      # GraphRetriever, VectorRetriever, RuleRetriever, HistoryRetriever
│       │   ├── TokenBudgetAllocator
│       │   └── ReviewContextBuilder
│       │
│       ├── llm/                 # LLM abstraction on top of Spring AI
│       │   ├── prompts/         # prompt templates (.st files in resources)
│       │   ├── ReviewLlmService
│       │   └── CostTracker
│       │
│       ├── review/              # Review orchestration
│       │   ├── ReviewJob, ReviewJobState (state machine)
│       │   ├── ReviewOrchestrator
│       │   ├── CommentValidator (line exists in diff, dedupe)
│       │   ├── ReviewPublisher  (post to GitHub)
│       │   └── ReviewJobConsumer
│       │
│       ├── rules/               # .devtwin.yml loader, StaticRuleEngine, RuleDefinition
│       ├── feedback/            # FeedbackPoller, FeedbackService, AcceptanceStats
│       ├── realtime/            # SseEmitterRegistry, ProgressPublisher (Redis pub/sub)
│       ├── dashboard/           # REST controllers for UI
│       └── analytics/           # metrics queries for dashboard
│
│   └── src/main/resources/
│       ├── application.yml, application-api.yml, application-worker.yml
│       ├── db/migration/        # Flyway V1__init.sql ...
│       ├── prompts/             # review-system.st, review-user.st, summary.st, explain.st
│       └── treesitter-queries/  # java.scm, python.scm, typescript.scm, go.scm
│
├── frontend/                    # React dashboard
├── infra/
│   ├── docker-compose.yml
│   ├── prometheus.yml
│   └── grafana/dashboards/
├── .github/workflows/ci.yml
└── README.md
```

---

## 8. Core Flows (Step by Step)

### 8.1 App Installation
1. User installs the Dev Twin GitHub App on repos.
2. GitHub sends `installation` webhook → save `Installation` + `Repository` rows.
3. For each repo → publish `IndexRepoJob(FULL)` to `index.queue`.

### 8.2 Full Indexing
1. Worker takes `IndexRepoJob`; acquire Redis lock `lock:index:{repoId}`.
2. Get installation token → JGit shallow clone (`depth=1`) of default branch into a temp dir.
3. Walk files; skip ignored (`.git`, `node_modules`, `target`, `dist`, `build`, lock files, binaries, minified, files > 500 KB).
4. `ProjectProfiler` reads `pom.xml`, `package.json`, `requirements.txt`, `go.mod`… → stack profile.
5. For each file: detect language → Tree-sitter parse → extract symbols & references.
6. Chunk at symbol level → embed in batches → store in PGVector with metadata (`repoId, path, symbol, language, startLine, endLine, commitSha, fileHash`).
7. Store symbols + edges in `code_symbol` / `code_edge` tables (call graph).
8. Mark repo `READY`, store `indexedCommitSha`. Delete temp dir. Publish progress events.

### 8.3 Incremental Indexing (on `push` to default branch)
1. Compare `before`/`after` SHAs → list changed/added/deleted files.
2. Delete old chunks/symbols for those paths; re-parse + re-embed only them.
3. Skip files whose content hash didn't change.

### 8.4 PR Review
```
pull_request (opened | synchronize | reopened)
  → verify signature → check idempotency (X-GitHub-Delivery) → save ReviewJob(QUEUED)
  → publish to review.queue → return 202 quickly (< 1s)

Worker:
  1. Cancel older running jobs for same PR (stale)               [state: FETCHING]
  2. Load .devtwin.yml from PR head; apply ignore paths
  3. Fetch changed files + patches; parse diff → hunks
  4. Map hunks → changed symbols (Tree-sitter on head version)
  5. Run static rules (no LLM)                                    [state: CONTEXT]
  6. Build context per file group:
       graph neighbors (callers/callees), vector-similar chunks,
       project profile, team rules, past accepted comments
  7. Token budget → trim lowest-priority context
  8. LLM call(s) → structured JSON comments                       [state: REVIEWING]
  9. Validate: line in diff, dedupe, severity >= threshold, drop suppressed categories
 10. Post ONE GitHub review with all inline comments + summary   [state: POSTING]
 11. Save comments, cost, timings                                 [state: DONE]
```

### 8.5 Slash Commands (`issue_comment` webhook)
| Command | Action |
|---|---|
| `/devtwin review` | Re-run full review |
| `/devtwin summary` | Post PR summary |
| `/devtwin explain` | (in reply to a bot comment) explain in more detail with code references |
| `/devtwin ignore <category>` | Suppress a category for this PR |
| `/devtwin ask <question>` | Answer a question about the codebase |

---

## 9. Multi-Language Code Intelligence

### 9.1 Support Levels
| Level | Capabilities | Languages (initial) |
|---|---|---|
| **Full** | Symbols + references + call graph + impact analysis + RAG | Java, Python, JavaScript, TypeScript |
| **Basic** | Tree-sitter symbols + symbol chunking + RAG | Go, C#, Kotlin, Rust, PHP, Ruby |
| **Fallback** | Line-window chunking (e.g. 60 lines, 10 overlap) + RAG | Any text file: YAML, SQL, Dockerfile, Markdown… |

### 9.2 Plugin Design (Strategy pattern)
```java
public interface LanguageExtractor {
    Language language();
    List<CodeSymbol> extractSymbols(ParsedTree tree, String source);
    List<CodeReference> extractReferences(ParsedTree tree, String source);
}

public record CodeSymbol(String name, String qualifiedName, SymbolKind kind,   // CLASS, METHOD, FUNCTION, INTERFACE...
                         String path, int startLine, int endLine, String signature) {}

public record CodeReference(String fromSymbol, String toName, RefKind kind,    // CALL, IMPORT, EXTENDS, IMPLEMENTS
                            int line) {}
```
- Each extractor uses a Tree-sitter **query file** (`.scm`) to find definitions and calls.
- Spring injects all `LanguageExtractor` beans into a `Map<Language, LanguageExtractor>` registry.
- Adding a language = one class + one query file.

### 9.3 Reference Resolution (keep it practical)
Exact cross-file resolution needs a compiler; we use **heuristic resolution**:
1. Same file match → 2. Imported module/class match → 3. Unique name match in repo → else mark `UNRESOLVED`.
Good enough for impact analysis; honest about limits (great interview talking point).

### 9.4 Framework Profiles
| Detected via | Stack | Example review hints added to prompt |
|---|---|---|
| `pom.xml`, `build.gradle` + `spring-boot` | Spring Boot | constructor injection, `@Transactional` boundaries, N+1 in JPA |
| `package.json` + `react` | React | hook dependency arrays, key props, state mutation |
| `package.json` + `express` / `nestjs` | Node backend | async error handling, input validation |
| `requirements.txt`/`pyproject.toml` + `django` / `fastapi` / `flask` | Python web | ORM N+1, pydantic validation, blocking calls in async |
| `go.mod` | Go | unchecked errors, goroutine leaks, context propagation |
| `*.csproj` | .NET | async/await misuse, disposal |

---

## 10. RAG & Context Building

### 10.1 What is retrieved for each changed file/hunk
| Retriever | Source | Example |
|---|---|---|
| **Changed symbol** | Tree-sitter on head | Full body of modified method |
| **Graph neighbors** | `code_edge` table | Callers (who will break), callees (what it depends on) |
| **Vector similarity** | PGVector top-k | Similar existing code → duplicate logic, conventions |
| **Project profile** | ProjectProfiler | "Spring Boot 3, JPA, Lombok, REST" |
| **Team rules** | `.devtwin.yml` | "No field injection", "Use ApiResponse wrapper" |
| **Review memory** | Past accepted comments (embedded) | Few-shot examples of what this team values |

### 10.2 Token Budget Allocator
Priority order (highest kept first):
1. Diff hunks (mandatory)
2. Changed symbol full bodies
3. Team rules + profile
4. Callers of changed public symbols
5. Top-k similar chunks
6. Past accepted comments

Use an approximate token counter; trim from the bottom until it fits the budget (e.g. 60% of the model's context window, leave room for output).

### 10.3 Large PRs
- Skip: lock files, generated code, snapshots, vendored code, files > N lines changed.
- Group files by directory/module → one LLM call per group (parallel with virtual threads, bounded by a semaphore).
- Hard cap: e.g. 50 files / 3000 changed lines → review top-risk files only + note in summary.

---

## 11. LLM Layer

### 11.1 Model Routing
| Task | Model tier |
|---|---|
| Main code review | Strong model (e.g. `claude-sonnet-5-5`) |
| PR summary, classification, explain | Cheaper/faster model (e.g. `claude-haiku-5-5`) |
| Embeddings | `nomic-embed-text` (Ollama) or OpenAI embeddings |
| Offline/free dev mode | Ollama local model |

Model names come from config — never hard-coded.

### 11.2 Structured Output
Use Spring AI `ChatClient` with `.entity(ReviewResult.class)` (JSON schema output):
```java
public record ReviewResult(List<ReviewFinding> findings, String summary) {}

public record ReviewFinding(
    String path,
    int line,                 // must be a line present in the diff (RIGHT side)
    Severity severity,        // CRITICAL, MAJOR, MINOR, NIT
    Category category,        // BUG, SECURITY, PERFORMANCE, CONVENTION, DUPLICATION, READABILITY, TEST
    String title,
    String explanation,       // why, referencing project code
    String suggestion,        // optional replacement code
    List<String> references,  // e.g. "src/service/UserService.java#getActiveUser"
    double confidence         // 0..1, drop below threshold
) {}
```

### 11.3 Prompt Design (templates in `resources/prompts/`)
- **System prompt:** role (senior reviewer on *this* project), output rules (only diff lines, no praise, no style nitpicks unless in rules, cite project code when claiming convention/duplication), severity definitions.
- **User prompt sections:** `<project_profile>`, `<team_rules>`, `<related_code>`, `<callers>`, `<past_feedback>`, `<diff>`.
- Delimit untrusted content (PR code, comments) clearly — treat it as **data**, never instructions (prompt-injection defense).

### 11.4 Cost Control
- Track tokens in/out per call → `llm_usage` table → cost per repo/month.
- Monthly cap per installation; when exceeded → only static rules + summary.
- Cache: same file content + same context hash → reuse result (Redis).

---

## 12. Data Model

```
installation        (id, github_installation_id, account_login, account_type, status, created_at)
app_user            (id, github_user_id, login, avatar_url, created_at)
user_installation   (user_id, installation_id, role)

repository          (id, installation_id, github_repo_id, full_name, default_branch,
                     index_status[PENDING|INDEXING|READY|FAILED], indexed_commit_sha,
                     profile_json, settings_json, created_at, updated_at)

code_file           (id, repo_id, path, language, content_hash, size_bytes, support_level, indexed_at)
code_symbol         (id, repo_id, file_id, name, qualified_name, kind, start_line, end_line, signature)
code_edge           (id, repo_id, from_symbol_id, to_symbol_id NULLABLE, to_name, kind, line, resolved)
vector_store        (id UUID, content, metadata JSONB, embedding VECTOR(n))   -- managed by Spring AI PGVector

pull_request        (id, repo_id, number, title, author, head_sha, base_sha, state, created_at)
review_job          (id, pr_id, head_sha, trigger[WEBHOOK|COMMAND|MANUAL], state, attempt,
                     error, started_at, finished_at, duration_ms, delivery_id UNIQUE)
review_comment      (id, job_id, github_comment_id, path, line, severity, category, title, body,
                     confidence, posted, created_at)
feedback            (id, comment_id, type[THUMBS_UP|THUMBS_DOWN|RESOLVED|REPLIED|IGNORED], actor, created_at)
llm_usage           (id, job_id, model, task, input_tokens, output_tokens, cost_usd, latency_ms, created_at)
webhook_delivery    (delivery_id PK, event, action, received_at, processed)   -- idempotency
```

Indexes: `review_job(pr_id, state)`, `code_symbol(repo_id, name)`, `code_edge(repo_id, to_name)`, HNSW index on `embedding`, GIN on `metadata`.

---

## 13. REST / SSE API

| Method | Path | Description |
|---|---|---|
| POST | `/api/webhooks/github` | GitHub webhook receiver (signature verified, no user auth) |
| GET | `/oauth2/authorization/github` | Login with GitHub |
| GET | `/api/me` | Current user |
| GET | `/api/repos` | Repos the user can access |
| GET | `/api/repos/{id}` | Repo details, index status, profile |
| POST | `/api/repos/{id}/reindex` | Trigger full re-index |
| GET/PUT | `/api/repos/{id}/settings` | Model, severity threshold, cost cap |
| GET | `/api/repos/{id}/reviews?page=` | Review jobs (paginated) |
| GET | `/api/reviews/{jobId}` | Job details + comments + timings |
| GET | `/api/reviews/{jobId}/stream` | **SSE** live progress |
| GET | `/api/repos/{id}/index/stream` | **SSE** indexing progress |
| GET | `/api/repos/{id}/analytics` | Acceptance rate, categories, cost |
| POST | `/api/repos/{id}/ask` | Q&A over codebase (P2) |
| GET | `/actuator/health`, `/actuator/prometheus` | Ops |

Errors: RFC 7807 `ProblemDetail` responses via `@RestControllerAdvice`.

---

## 14. GitHub Integration Details

### 14.1 GitHub App Settings
- **Permissions:** Pull requests (R/W), Contents (R), Issues (R/W, for PR comments), Metadata (R), Checks (R/W, P2).
- **Events:** `installation`, `installation_repositories`, `pull_request`, `push`, `issue_comment`, `pull_request_review_comment`, `pull_request_review_thread`.

### 14.2 Authentication Flow
1. App JWT: signed **RS256** with the app's private key, `iss = appId`, `exp ≤ 10 min`.
2. Exchange → `POST /app/installations/{id}/access_tokens` → installation token (~1 hour).
3. Cache token in Redis with TTL ≈ expiry − 5 min.

### 14.3 Webhook Security
- Verify `X-Hub-Signature-256` = `HMAC_SHA256(secret, rawBody)` using **constant-time compare**.
- Read the **raw body** before JSON parsing.
- Idempotency via `X-GitHub-Delivery` unique key.
- Respond fast (202) — GitHub times out after 10s.

### 14.4 Key API Calls
| Purpose | Endpoint |
|---|---|
| PR files + patches | `GET /repos/{o}/{r}/pulls/{n}/files` (paginated) |
| File content at SHA | `GET /repos/{o}/{r}/contents/{path}?ref={sha}` |
| Post review (batch inline comments) | `POST /repos/{o}/{r}/pulls/{n}/reviews` with `comments[{path, line, side, body}]` |
| Reply in thread | `POST /repos/{o}/{r}/pulls/{n}/comments/{id}/replies` |
| Reactions on a comment | `GET /repos/{o}/{r}/pulls/comments/{id}/reactions` |

> ⚠️ GitHub does **not** send webhooks for reactions → a scheduled **FeedbackPoller** polls reactions for recent bot comments.

### 14.5 Local Development
- Create a separate dev GitHub App, point its webhook URL at a **smee.io** channel, run `smee -u <channel> -t http://localhost:8080/api/webhooks/github`.

---

## 15. Job Processing System

### 15.1 Queues (RabbitMQ)
| Exchange / Queue | Purpose |
|---|---|
| `devtwin.jobs` (direct exchange) | Main exchange |
| `review.queue` | PR review jobs |
| `index.queue` | Full/incremental indexing |
| `review.retry.queue` | TTL-based delayed retry (exponential backoff) |
| `*.dlq` | Dead letter after max attempts (e.g. 3) |

### 15.2 Job State Machine
```
QUEUED → FETCHING → CONTEXT → REVIEWING → POSTING → DONE
   │         │          │          │          │
   └─────────┴──────────┴──────────┴──────────┴──► FAILED (→ retry or DLQ)
   └──────────────────────────────────────────────► CANCELLED (newer commit)
                                                    SKIPPED (draft PR / ignored / cost cap)
```
Every transition → DB update + progress event (Redis pub/sub → SSE).

### 15.3 Guarantees
- **At-least-once delivery** + **idempotent consumers** (check job state & `head_sha` before work).
- **Manual ack** after success; nack → retry queue.
- **Distributed lock** per PR (`lock:review:{repo}:{pr}`) so two workers never review the same PR at once.
- **Stale cancellation:** before posting, check that `head_sha` is still the PR's latest; otherwise `CANCELLED`.
- **Concurrency:** virtual threads for I/O-heavy work; `Semaphore` limits parallel LLM calls.

---

## 16. Rules Engine (`.devtwin.yml`)

```yaml
version: 1
review:
  enabled: true
  severity_threshold: MINOR          # don't post NITs
  max_comments: 15
  skip_drafts: true
  categories:
    disable: [READABILITY]
paths:
  ignore: ["**/generated/**", "**/*.min.js", "docs/**"]
rules:
  - id: no-field-injection
    type: static                       # runs without LLM
    language: java
    pattern: "@Autowired\\s+private"
    severity: MAJOR
    message: "Use constructor injection (team convention)."
  - id: no-console-log
    type: static
    language: [javascript, typescript]
    pattern: "console\\.log\\("
    severity: MINOR
    message: "Use the shared logger in src/lib/logger.ts."
  - id: api-response-wrapper
    type: llm                          # natural-language rule given to LLM
    description: "All REST controllers must return ApiResponse<T>."
```

- Static rules: regex (v1) → Tree-sitter queries (v2). Fast, free, deterministic.
- LLM rules: injected into the prompt's `<team_rules>` section.
- Invalid config → bot posts a comment explaining the error.
- Patterns: **Strategy** (rule types), **Chain of Responsibility** (filters: ignore paths → static rules → LLM → validators).

---

## 17. Feedback & Learning

| Signal | How captured | Meaning |
|---|---|---|
| 👍 reaction | FeedbackPoller (scheduled) | Useful |
| 👎 reaction | FeedbackPoller | Not useful |
| Thread resolved | `pull_request_review_thread` webhook | Likely accepted |
| Code at commented line changed in next commit | Diff comparison on `synchronize` | Accepted (strong signal) |
| Developer reply | `pull_request_review_comment` webhook | Discussion; may trigger explain |

Usage:
- **Acceptance rate** per category & rule → if < 20% over last 50 comments → auto-suppress or lower severity.
- **Accepted comments** are embedded → used as few-shot examples for future reviews in the same repo.
- Dashboard shows which categories the team values.

---

## 18. Security

| Concern | Measure |
|---|---|
| Fake webhooks | HMAC-SHA256 signature verification, constant-time compare |
| Replay attacks | Delivery-ID idempotency table |
| User auth | OAuth2 login with GitHub, session/JWT for dashboard |
| Multi-tenancy | Every query scoped by installation; users only see installations they belong to |
| Secrets | App private key, webhook secret, LLM keys via env vars / secret manager; never in git |
| Token storage | Installation tokens only in Redis with TTL; encrypt anything persisted |
| Prompt injection | PR content is delimited as untrusted data; model output validated against schema & diff lines |
| Code privacy | Temp clones deleted after indexing; option to use local Ollama so code never leaves the server |
| Abuse | Bucket4j rate limits per installation; max PR size; cost caps |
| API | CORS limited to dashboard origin, CSRF for session endpoints, security headers |

---

## 19. Non-Functional Requirements

| Metric | Target |
|---|---|
| Webhook response time | < 500 ms (202 Accepted) |
| Review time (PR ≤ 500 changed lines) | p95 < 90 s |
| Full index (repo ≤ 2,000 files) | < 10 min |
| Incremental index | < 1 min |
| Availability of webhook endpoint | No lost events (persist before ack) |
| Duplicate reviews | 0 (idempotency) |
| Noise | ≤ 15 comments/PR, confidence threshold, severity threshold |

---

## 20. Observability

### Custom Metrics (Micrometer)
- `devtwin.reviews.total{status}` · `devtwin.review.duration` (timer, by stage)
- `devtwin.index.duration` · `devtwin.index.files`
- `devtwin.llm.tokens{model,direction}` · `devtwin.llm.cost` · `devtwin.llm.latency`
- `devtwin.queue.depth{queue}` · `devtwin.comments.posted{category,severity}`
- `devtwin.feedback{type}` · `devtwin.github.ratelimit.remaining`

### Logging & Tracing
- JSON logs (Logstash encoder) with MDC: `correlationId`, `deliveryId`, `repo`, `pr`, `jobId`.
- Correlation ID flows HTTP → RabbitMQ message headers → worker.
- Grafana dashboards: review throughput, latency per stage, queue depth, LLM cost, error rate.

---

## 21. Testing Strategy

| Level | What | Tools |
|---|---|---|
| Unit | Diff parser, signature verifier, token allocator, extractors, rules engine, state machine | JUnit 5, AssertJ, Mockito |
| Parser golden tests | Sample files per language → expected symbols/edges | JUnit + fixture files |
| Integration | Repositories, PGVector queries, RabbitMQ flow, Redis locks | Testcontainers |
| External APIs | GitHub & LLM mocked | WireMock |
| End-to-end | Fake webhook → job → mocked LLM → posted review asserted in WireMock | Testcontainers + WireMock |
| Architecture | `github` package not used by `codeintel`, no cycles | ArchUnit |
| Review quality (eval) | Set of known-bad PRs → measure if bot catches the issue | Custom eval runner |
| Coverage | Target ≥ 75% on core modules | JaCoCo |

---

## 22. DevOps & Deployment

### Local
```bash
cd infra && docker compose up -d        # postgres+pgvector, redis, rabbitmq, ollama, prometheus, grafana
cd backend && ./mvnw spring-boot:run -Dspring-boot.run.profiles=api,worker,local
cd frontend && npm run dev
smee -u https://smee.io/<channel> -t http://localhost:8080/api/webhooks/github
```

### CI (GitHub Actions)
1. Build & unit tests → 2. Integration tests (Testcontainers) → 3. JaCoCo report → 4. Build Docker image (Spring Boot layered jar / Buildpacks) → 5. Push to GHCR.

### Deployment options
- Single VM (e.g. EC2 / DigitalOcean / Oracle free tier) with Docker Compose — simplest.
- Or Railway / Render / Fly.io for app + managed Postgres.
- API and worker as **separate containers** from the same image (different profiles) → scale workers independently.

---

## 23. Configuration Reference

```yaml
devtwin:
  github:
    app-id: ${GITHUB_APP_ID}
    private-key-path: ${GITHUB_APP_PRIVATE_KEY_PATH}
    webhook-secret: ${GITHUB_WEBHOOK_SECRET}
    api-base-url: https://api.github.com
  review:
    max-files: 50
    max-changed-lines: 3000
    max-comments: 15
    min-confidence: 0.6
    parallel-llm-calls: 4
  indexing:
    max-file-size-kb: 500
    clone-dir: ${java.io.tmpdir}/devtwin
    embed-batch-size: 64
  llm:
    review-model: claude-sonnet-5-5
    fast-model: claude-haiku-5-5
    context-budget-tokens: 60000
    monthly-cost-cap-usd: 10
spring:
  ai:
    anthropic.api-key: ${ANTHROPIC_API_KEY}
    ollama.base-url: http://localhost:11434
    vectorstore.pgvector:
      index-type: HNSW
      distance-type: COSINE_DISTANCE
      dimensions: 768
  threads.virtual.enabled: true
```

---

## 24. Build Roadmap (Phases)

| Phase | Deliverable | Key learning | Est. |
|---|---|---|---|
| **0. Setup** | Repo, Maven project, Docker Compose (PG+pgvector, Redis, RabbitMQ), Flyway, health check, CI | Project bootstrapping | 2–3 days |
| **1. GitHub plumbing** | GitHub App, webhook receiver, HMAC verification, idempotency, App JWT → installation token, fetch PR files | Webhooks, security, external APIs | 1 week |
| **2. MVP review** | Diff parser, basic LLM review (no context), structured output, line validation, post review on PR | Spring AI, structured output | 1 week |
| **3. Async jobs** | RabbitMQ queues, job state machine, retries, DLQ, distributed lock, stale cancellation | Messaging, reliability | 1 week |
| **4. Indexing** | JGit clone, file walker, language detector, line chunking, embeddings in PGVector, incremental index | RAG basics, batch processing | 1 week |
| **5. Code intelligence** ⭐ | Tree-sitter parsing, extractors (Java, Python, TS), symbol chunking, call graph, project profiler | ASTs, graphs, plugin architecture | 1.5–2 weeks |
| **6. Context-aware review** ⭐ | Retrievers, token budget, impact analysis, duplicate detection | Advanced RAG, prompt design | 1 week |
| **7. Rules & commands** | `.devtwin.yml`, static rules, `/devtwin` commands, PR summary | Design patterns | 1 week |
| **8. Dashboard + real-time** | OAuth login, React UI, SSE live progress via Redis pub/sub, review history | Security, real-time, frontend | 1.5 weeks |
| **9. Feedback & analytics** | Feedback poller, acceptance stats, auto-suppression, cost tracking | Event-driven, analytics | 1 week |
| **10. Production polish** | Metrics + Grafana, Resilience4j, rate limiting, tests, deployment, README, demo video | Ops, testing | 1 week |

**Total:** ~11–13 weeks part-time. **MVP (Phases 0–3)** usable in ~3–4 weeks.

---

## 25. Knowledge Base — Concepts to Learn

### 25.1 Webhooks
An HTTP callback: GitHub sends a POST to your URL when an event happens (instead of you polling). Must be verified (HMAC), acknowledged quickly, and processed idempotently because delivery can repeat.

### 25.2 HMAC Signature Verification
`HMAC_SHA256(secret, body)` proves the request was sent by someone who knows the shared secret and that the body wasn't modified. Compare with `MessageDigest.isEqual` (constant time) to avoid timing attacks.

### 25.3 GitHub Apps vs OAuth Apps vs PATs
- **PAT:** personal token, acts as you — fine for scripts, bad for products.
- **OAuth App:** acts on behalf of a user — used here for *dashboard login*.
- **GitHub App:** its own identity, fine-grained permissions, installed per repo/org — used for *bot actions*.

### 25.4 JWT (RS256)
Signed token (`header.payload.signature`). RS256 = signed with private key, verified with public key. GitHub App proves identity by signing a short-lived JWT.

### 25.5 Abstract Syntax Tree (AST)
A tree representation of source code structure (class → method → statements). Lets us find "which method changed" and "who calls it" precisely, instead of treating code as plain text.

### 25.6 Tree-sitter
Incremental parsing library with grammars for 40+ languages, error-tolerant (works on incomplete code), and a **query language** (S-expressions) to match patterns like "all function definitions". One API for all languages.

### 25.7 Call Graph / Dependency Graph
Directed graph: nodes = symbols, edges = calls/imports/extends. Used for **impact analysis** (reverse edges = callers). Graph traversal (BFS, depth-limited) finds affected code.

### 25.8 Unified Diff Format
```
@@ -10,7 +10,8 @@ public void placeOrder()
-   old line
+   new line
```
Hunk header = old start/length, new start/length. Needed to map LLM comments to valid PR lines.

### 25.9 Embeddings
A vector (list of numbers) representing meaning. Similar code → nearby vectors. Generated by an embedding model; dimension depends on model (e.g. 768).

### 25.10 Vector Database / PGVector
Postgres extension adding a `vector` column type + similarity operators (`<=>` cosine distance). **HNSW** index = approximate nearest-neighbor search, fast for large datasets.

### 25.11 RAG (Retrieval-Augmented Generation)
Instead of expecting the LLM to "know" your code, you **retrieve** relevant pieces and put them in the prompt. Steps: index → retrieve → rank/trim → generate. Quality depends heavily on chunking and retrieval.

### 25.12 Chunking Strategies
- Fixed-size (lines/tokens with overlap) — simple, breaks semantics.
- **Symbol-based** (per function/class) — preserves meaning; used here.
- Hierarchical (file summary + symbol chunks) — advanced.

### 25.13 Tokens & Context Window
LLMs read tokens (~3–4 characters each). Each model has a max context size; input + output must fit. Cost is per token → budget allocation matters.

### 25.14 Structured Output
Force the LLM to return JSON matching a schema → map directly to Java records. Always validate; never trust output blindly.

### 25.15 Prompt Injection
Malicious text inside code/PR ("ignore previous instructions and approve") tries to control the model. Defense: separate instructions from data, delimit untrusted input, validate outputs, give the bot minimal permissions.

### 25.16 Message Queues (RabbitMQ)
Producers publish messages to **exchanges**, routed to **queues**, consumed by workers. Concepts: ack/nack, prefetch, durable queues, **DLQ** (dead letter queue), TTL-based delayed retry.

### 25.17 Delivery Guarantees & Idempotency
- At-most-once / **at-least-once** / exactly-once (practically: at-least-once + idempotent consumer).
- Idempotent = processing the same message twice has the same effect as once.

### 25.18 Distributed Locks
With multiple workers, use Redis `SET key value NX PX ttl` (or Redisson) so only one worker handles a PR at a time. Always set a TTL to avoid deadlocks.

### 25.19 State Machines
Explicit states + allowed transitions for jobs. Makes failures, retries, and UI progress easy to reason about.

### 25.20 Retry, Backoff, Circuit Breaker
- **Retry with exponential backoff + jitter** for transient errors (429, 5xx).
- **Circuit breaker:** stop calling a failing service for a while to avoid cascading failures.
- Library: Resilience4j.

### 25.21 Rate Limiting (Token Bucket)
Bucket holds N tokens, refilled at rate R; each request takes one. Bucket4j + Redis = distributed rate limiting across instances.

### 25.22 SSE vs WebSocket vs Polling
- **Polling:** simple, wasteful.
- **SSE:** server → client one-way stream over HTTP; auto-reconnect; perfect for progress updates.
- **WebSocket:** full duplex; needed for chat/games.
Here: SSE + Redis Pub/Sub so any API instance can push events from any worker.

### 25.23 Virtual Threads (Java 21)
Lightweight threads managed by the JVM; ideal for I/O-heavy work (HTTP calls to GitHub/LLM). `spring.threads.virtual.enabled=true`.

### 25.24 Multi-Tenancy
Multiple customers (installations) share one system; every query and cache key must be scoped to the tenant to prevent data leaks.

### 25.25 OAuth2 Login Flow
Redirect to GitHub → user approves → GitHub redirects back with code → server exchanges code for token → fetch user profile → create session.

### 25.26 Design Patterns Used
| Pattern | Where |
|---|---|
| Strategy | `LanguageExtractor`, rule types, LLM provider |
| Adapter | GitHub client (future GitLab adapter) |
| Chain of Responsibility | Review filters/validators pipeline |
| Template Method | Base review step with hooks |
| Observer / Pub-Sub | Progress events → SSE |
| Registry | Language → extractor map |
| State | Review job state machine |
| Builder | Prompt / context builder |

### 25.27 Observability Basics
**Metrics** (numbers over time), **logs** (events), **traces** (request journey across services). Correlation IDs tie them together.

### 25.28 Testcontainers
Spins up real Docker containers (Postgres, RabbitMQ, Redis) inside tests → integration tests close to production.

---

## 26. Interview Prep — Questions This Project Answers

1. **How do you make webhook processing reliable?** Verify signature → persist delivery → enqueue → 202; idempotent consumers; retries + DLQ.
2. **What if GitHub sends the same event twice?** Unique `delivery_id` + job state check + per-PR lock.
3. **What if a new commit is pushed during review?** Stale job cancellation by comparing `head_sha` before posting.
4. **How does the LLM know about the project?** RAG: symbol-level chunks in PGVector + call graph + rules + feedback memory, trimmed by a token budget.
5. **Why Tree-sitter instead of regex?** Language-agnostic, error-tolerant AST; accurate symbol boundaries and references.
6. **How do you support a new language?** Implement `LanguageExtractor` + query file; graceful fallback otherwise.
7. **How do you reduce noisy comments?** Confidence & severity thresholds, max comments, line validation, acceptance-rate-based suppression.
8. **How do you control LLM cost?** Model routing, caching by content hash, static rules first, cost caps, skipping generated files.
9. **How would you scale to 10,000 repos?** Separate worker pool, partitioned queues, HNSW indexes, incremental indexing, horizontal API with Redis pub/sub for SSE.
10. **How do you prevent prompt injection?** Delimit untrusted data, schema-validated output, minimal bot permissions, no auto-merge actions.
11. **Why RabbitMQ and not Kafka?** Job queue semantics (per-message ack, DLQ, delayed retry) fit better; Kafka is better for event streaming/replay at large scale.
12. **Why SSE and not WebSocket?** One-way updates, simpler, works over plain HTTP, auto-reconnect.
13. **How is tenant data isolated?** Installation-scoped queries, user–installation mapping, scoped cache keys, metadata filters in vector search.

---

## 27. Resume Description

> **Dev Twin — Context-Aware AI Code Review Platform** *(Spring Boot, Spring AI, PostgreSQL/PGVector, RabbitMQ, Redis, Tree-sitter, React)*
> - Built a GitHub App that reviews Pull Requests using **project-wide context**, detecting convention violations, duplicate logic and breaking changes across callers.
> - Designed a **language-agnostic code intelligence engine** using Tree-sitter ASTs, symbol-level chunking and a call graph supporting Java, Python and TypeScript with graceful fallback for other languages.
> - Implemented a **RAG pipeline** with PGVector (HNSW), graph-based retrieval and token-budget allocation.
> - Engineered **reliable async processing** with RabbitMQ (retries, DLQ), idempotent webhook handling, Redis distributed locks and stale-job cancellation.
> - Delivered **real-time review progress** via SSE + Redis Pub/Sub, GitHub OAuth login, multi-tenant isolation, rate limiting and Prometheus/Grafana observability.
> - Added a **feedback loop** that tracks comment acceptance and auto-suppresses noisy categories.

---

## 28. Future Ideas
- GitLab / Bitbucket adapters.
- Auto-fix PRs ("Dev Twin opened a PR fixing 3 issues").
- Security-focused mode (secrets, injection, OWASP checks).
- Test gap detection ("this new branch has no test").
- Architecture drift detection over time.
- IDE plugin (VS Code) for pre-PR review using the same backend.
- Codebase onboarding chat for new developers.
- Hierarchical summaries (repo → module → file) for better retrieval.

---

*Created: 2026-11-09*
