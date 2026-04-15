# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

AI zero-code application generation platform built on **Spring Boot 3 + LangChain4j + LangGraph4j**. Users describe their needs in natural language; the system routes to an AI agent workflow that collects images, generates code, checks quality, builds the project, and deploys a live web app.

The repo contains three sub-projects:
- **Root** (`src/`) — monolithic Spring Boot backend
- **`yu-ai-code-mother-frontend/`** — Vue 3 + Vite frontend
- **`yu-ai-code-mother-microservice/`** — Spring Cloud Alibaba microservice version (modules: ai, app, client, common, model, screenshot, user)

---

## Common Commands

### Backend (monolith)
```bash
# Build and package
./mvnw clean package -DskipTests

# Run
./mvnw spring-boot:run

# Run a single test
./mvnw test -Dtest=SomeTestClass#methodName
```

### Frontend
```bash
cd yu-ai-code-mother-frontend

npm install          # install deps
npm run dev          # dev server (proxies /api to backend)
npm run build        # type-check + build
npm run lint         # ESLint --fix
npm run openapi2ts   # regenerate API types from backend OpenAPI spec
```

---

## Backend Architecture

**Package root:** `com.yupi.yuaicodemother`

### Key Layers

| Package | Purpose |
|---|---|
| `controller/` | REST endpoints; `WorkflowSseController` handles SSE streaming for the AI workflow |
| `service/` | Business logic; `AiCodeGeneratorServiceFactory` selects the right generator strategy |
| `ai/` | LangChain4j AI service interfaces, guardrail (prompt safety check), model wrappers |
| `langgraph4j/` | LangGraph4j workflow definition and nodes |
| `core/` | Code file builder, parser, saver — handles multi-file/Vue/HTML output types |
| `ratelimter/` | Custom `@RateLimit` annotation + `RateLimitAspect` (Redisson `RRateLimiter`, token bucket, user/IP/endpoint granularity) |
| `aop/` | `AuthInterceptor` — `@AuthCheck` annotation for role-based access control |

### AI Workflow (LangGraph4j)

`CodeGenWorkflow` wires these nodes into a directed graph:

1. **RouterNode** — classifies prompt into `codeGenType` (HTML / multi-file / Vue)
2. **PromptEnhancerNode** — rewrites user prompt for better generation
3. **ImagePlanNode** — plans what images are needed
4. **Concurrent image collection** (parallel):
   - `ContentImageCollectorNode`, `DiagramCollectorNode`, `IllustrationCollectorNode`, `LogoCollectorNode`
4. **ImageAggregatorNode** — merges parallel results
5. **CodeGeneratorNode** — calls LLM with tool use to produce code
6. **CodeQualityCheckNode** — validates generated code
7. **ProjectBuilderNode** — compiles/packages the project

The workflow state flows through `langgraph4j/state/`. The Studio debug UI is wired via `LangGraphStudioSampleConfig`.

### AI Model Strategy

Three distinct models configured in `application.yml`:
- **`chat-model` / `streaming-chat-model`** — DeepSeek Chat (primary generation)
- **`reasoning-streaming-chat-model`** — DeepSeek Reasoner (complex tasks, 32k tokens)
- **`routing-chat-model`** — Qwen Turbo (cheap router classification, 100 tokens max)

`AiCodeGenTypeRoutingServiceFactory` and `AiCodeGeneratorServiceFactory` select the appropriate service at runtime.

### Cross-Cutting Concerns

- **Session**: Spring Session stored in Redis (30-day TTL)
- **Chat memory**: LangChain4j community Redis chat memory store; keyed per user+app
- **Caching**: Caffeine local cache in front of Redis
- **Rate limiting**: `@RateLimit` via AOP + Redisson `RRateLimiter`
- **Auth**: `@AuthCheck(mustRole = "admin")` via AOP
- **Guardrails**: LangChain4j input guard in `ai/guardrail/` for prompt safety
- **Screenshot/deploy**: Selenium + WebDriverManager for capturing previews; Tencent COS for file storage
- **Monitoring**: Actuator + Micrometer → Prometheus; Grafana dashboard in `grafana/`
- **Virtual threads**: I/O-heavy tasks (build, screenshot, upload) run on virtual threads

### Database

MySQL with three tables: `user`, `app`, `chat_history`. ORM via MyBatis-Flex. Init SQL in `sql/create_table.sql`. Database name: `yu_ai_code_mother`.

---

## Frontend Architecture

**Stack:** Vue 3 + Vite + Pinia + Vue Router + Ant Design Vue + TypeScript

- `src/api/` — auto-generated from backend OpenAPI via `openapi2ts` (run `npm run openapi2ts` after backend changes)
- `src/pages/` — views split into `user/`, `admin/`, `app/`
- `src/stores/` — Pinia stores (user state)
- `src/router/index.ts` — route definitions including admin-guarded routes
- Dev proxy: Vite forwards `/api/*` to `http://localhost:8123`

---

## Configuration

Copy `src/main/resources/application.yml` and fill in:
- MySQL credentials and database name `yu_ai_code_mother`
- Redis host/port
- `langchain4j.open-ai.chat-model.api-key` (DeepSeek)
- `langchain4j.open-ai.routing-chat-model.api-key` (Qwen/DashScope)
- `cos.client.*` (Tencent COS)
- `pexels.api-key`
- `dashscope.api-key`
