# AI-Powered Call Center Platform — Phased Development Plan

> Project: 487-ai-powered-call-center-platform · Created: 2026-05-31
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

This plan synthesises `research.md`, `features.md`, `standards.md`, `README.md`, and the four data-model suggestions. It adopts **Data Model Suggestion 3 (Hybrid Relational + JSONB on PostgreSQL)** as the persistence design: core operational entities are typed relational columns with foreign keys, while volatile AI-generated data (decision logs, extracted entities, escalation context, pipeline config) lives in JSONB columns that evolve without migrations. pgvector handles knowledge-base embeddings inside the same database, avoiding polyglot persistence in the MVP.

---

## Core Requirements (synthesis)

1. **What it does** — An open-source, self-hostable AI voice platform that answers inbound calls with an LLM-driven conversational agent (STT → LLM+RAG → TTS pipeline under ~800 ms turn latency), escalates complex calls to human agents with full context, runs compliant outbound campaigns, transcribes and QA-scores 100% of calls, and feeds knowledge gaps back into content improvement.
2. **Who uses it** — Contact-centre supervisors (dashboards, QA), human agents (assist copilot, WebRTC desktop), admins (AI config, knowledge base, compliance policy), and developers (REST API, webhooks, MCP). End callers interact only via voice.
3. **Key differentiators** — Only credible open-source/self-hostable option; transparent, auditable AI decision logs; multi-LLM routing per sub-task; SMB-accessible full stack (copilot + QA + outbound in one product); data-residency-friendly deployment.
4. **MVP** — Inbound natural-language voice agent, low-latency streaming pipeline, intelligent escalation with context transfer, SIP/PSTN via Twilio + WebRTC agent desktop, real-time diarised transcription, supervisor dashboard (queue depth, containment, AHT, CSAT), recording with PCI pause-resume and retention policy, REST API + webhooks with Salesforce/HubSpot connectors.
5. **Post-MVP** — Outbound campaign manager (DNC, TCPA/GDPR consent), automated post-call summaries + CRM auto-update, real-time agent assist copilot, 100% automated QA scoring, knowledge gap detection, multi-language, HIPAA mode.
6. **Backlog** — Voice cloning, multi-agent orchestration, multi-LLM routing UI, event-triggered proactive outbound, QA→fine-tuning loop.
7. **Deployment** — Self-hosted (Docker Compose / Kubernetes) primary; cloud and hybrid supported. Single PostgreSQL + Redis dependency for the MVP.
8. **Integration surface** — Telephony (Twilio Voice + Media Streams, Vonage), STT (Deepgram), TTS (Cartesia/ElevenLabs), LLM (Anthropic/OpenAI-compatible), CRM (Salesforce, HubSpot, Zendesk, ServiceNow), webhooks, MCP server.
9. **Standards** — SIP (RFC 3261), RTP (RFC 3550), WebRTC (RFC 8825), Opus (RFC 6716), SSML (W3C), STIR/SHAKEN (RFC 8226/8588) for outbound, OpenAPI 3.1 + AsyncAPI 3.0 for API docs, OAuth 2.0 + OIDC + PKCE for auth, PCI DSS 4.0, GDPR, TCPA 2025, HIPAA, OWASP API Top 10, EU AI Act caller-disclosure, MCP.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Primary language (services) | **Python 3.12** | The voice pipeline is LLM/STT/TTS-heavy; Deepgram, ElevenLabs, Anthropic, OpenAI Realtime all ship first-class Python SDKs. `asyncio` handles the concurrent streaming workloads (WebSocket audio in/out) cleanly. |
| API framework | **FastAPI** | Native async, automatic OpenAPI 3.1 generation (a standards requirement), Pydantic v2 validation, WebSocket support for media streams and live dashboards. |
| Real-time media bridge | **Twilio Media Streams (WebSocket) + `aiortc`** | Twilio Media Streams provides PSTN audio over WebSocket without running our own media server in the MVP; `aiortc` powers the browser WebRTC agent desktop (Opus, RFC 6716). FreeSWITCH is deferred to the self-hosted-SIP backlog. |
| STT | **Deepgram streaming (Nova-3)** | Sub-300 ms streaming, speaker diarisation, contact-centre models, PII redaction at the ASR layer — directly satisfies MVP transcription + diarisation needs. Provider is pluggable behind an interface. |
| TTS | **Cartesia Sonic (default), ElevenLabs Flash (alt)** | Streaming synthesis that begins audio before full text is ready; ~75–90 ms first-byte keeps turn latency under budget. SSML passthrough supported. |
| LLM | **Anthropic Claude (Sonnet for generation, Haiku for intent/routing)** via an OpenAI-compatible abstraction | Prompt caching for conversation context cuts per-turn cost and latency; multi-model routing is a first-class differentiator. Any OpenAI-compatible endpoint (self-hosted vLLM) can be swapped in for the data-residency tier. |
| Database | **PostgreSQL 16 + pgvector + pg_trgm** | Implements Data Model Suggestion 3. pgvector keeps RAG embeddings in-DB; JSONB absorbs evolving AI data; declarative partitioning handles transcript volume. |
| Session state | **Redis 7** | Durable mid-call conversation state (survives worker restart), pub/sub for live dashboard fan-out, and the task-queue broker. |
| Task queue | **Celery + Redis** (broker) | Async post-call work: summarisation, QA scoring, CRM sync, webhook delivery, campaign dialling, retention purges. Beat schedules retention and campaign jobs. |
| ORM / migrations | **SQLAlchemy 2.0 (async) + Alembic** | Async ORM matches FastAPI; Alembic gives the disciplined migration workflow the relational core needs. JSONB accessed via raw SQL / `JSONB` column type where needed. |
| Frontend | **Next.js 15 (React, TypeScript) + shadcn/ui + Tailwind** | Supervisor dashboard, agent desktop, admin console. WebRTC via `aiortc`-compatible browser APIs; live updates via WebSocket. |
| Auth | **Authlib (OAuth 2.0 / OIDC) + PKCE** | Enterprise SSO (Okta/Azure AD), CRM OAuth flows, PKCE for the browser agent desktop — all standards-mandated. JWT access tokens. |
| Containerisation | **Docker + docker-compose (dev/self-host); Helm chart (k8s)** | Self-hosted is the primary deployment mode. |
| Testing | **pytest + pytest-asyncio + respx (HTTP mocks) + testcontainers (real Postgres/Redis)** | Covers unit, mocked-integration, and real-dependency integration tiers. Playwright for frontend E2E. |
| Quality tooling | **ruff (lint+format), mypy (strict), pre-commit** | Single fast toolchain for lint/format; strict typing on the latency-critical pipeline. |
| Package manager | **uv** (Python), **pnpm** (frontend) | Fast, reproducible installs. |
| Observability | **OpenTelemetry + Prometheus + structured JSON logs** | Per-turn latency tracing is the defining UX metric; OTel spans across STT→LLM→TTS make latency regressions visible. |
| Object storage | **S3-compatible (MinIO for self-host)** | Call recordings; supports lifecycle/retention rules. |
| API docs | **OpenAPI 3.1 (FastAPI) + AsyncAPI 3.0 (hand-authored for webhooks/WS)** | Standards requirement; enables SDK generation and partner onboarding. |

### Project Structure

```
ai-call-center/
├── pyproject.toml                  # uv-managed, ruff/mypy/pytest config
├── docker-compose.yml              # postgres, redis, minio, api, worker, media-bridge, web
├── Dockerfile                      # multi-stage python service image
├── alembic.ini
├── deploy/
│   └── helm/                       # kubernetes chart
├── asyncapi.yaml                   # webhook + websocket event contracts
├── migrations/                     # alembic versions
├── src/
│   └── accp/                       # "AI Call Center Platform" package
│       ├── config.py               # pydantic-settings; all env/config
│       ├── main.py                 # FastAPI app factory, router mount, OTel init
│       ├── db/
│       │   ├── base.py             # async engine, session, Base
│       │   ├── models/             # SQLAlchemy models (one file per aggregate)
│       │   └── repositories/       # data-access layer
│       ├── schemas/                # Pydantic request/response + JSONB shape models
│       ├── auth/                   # OAuth2/OIDC, JWT, RBAC dependencies
│       ├── telephony/
│       │   ├── base.py             # TelephonyProvider protocol
│       │   ├── twilio.py           # Media Streams + Voice REST
│       │   └── webrtc.py           # aiortc agent-desktop signalling
│       ├── pipeline/
│       │   ├── orchestrator.py     # per-call turn loop, latency budget
│       │   ├── stt/                # STTProvider protocol + deepgram.py
│       │   ├── tts/                # TTSProvider protocol + cartesia.py, elevenlabs.py
│       │   ├── llm/                # LLMProvider protocol + anthropic.py, router.py
│       │   └── session.py          # Redis-backed ConversationSession
│       ├── ai/
│       │   ├── intent.py           # intent classification + entity extraction
│       │   ├── rag.py              # pgvector retrieval, citation enforcement
│       │   ├── escalation.py       # escalation signal evaluation
│       │   ├── summary.py          # post-call summarisation
│       │   ├── qa.py               # automated QA scoring
│       │   └── prompts/            # versioned prompt templates
│       ├── routing/                # queue/agent assignment engine
│       ├── campaigns/              # outbound scheduler, DNC, consent
│       ├── compliance/             # PCI pause/resume, consent, retention jobs
│       ├── integrations/
│       │   ├── crm/                # base.py + salesforce.py, hubspot.py, ...
│       │   └── mcp/                # MCP server exposing call data
│       ├── webhooks/               # signed delivery + retry
│       ├── api/                    # FastAPI routers (calls, agents, kb, campaigns, ...)
│       ├── realtime/               # dashboard WS hub, Redis pub/sub
│       └── tasks/                  # celery app + task modules
├── web/                            # Next.js app (dashboard, agent desktop, admin)
└── tests/
    ├── unit/
    ├── integration/
    ├── e2e/
    └── fixtures/                   # sample audio, transcripts, SARIF-style payloads
```

---

## Phase 1: Foundation, Data Model & Auth

### Purpose
Establish the project skeleton, the hybrid PostgreSQL schema (Suggestion 3), configuration, multi-tenant authentication/RBAC, and the CI/quality gate. After this phase the service boots, migrates the database, authenticates users, and enforces tenant isolation — the substrate every later phase builds on.

### Tasks

#### 1.1 — Project scaffold, config, and tooling

**What**: Create the `accp` package, `pyproject.toml`, Docker Compose stack, and quality tooling.

**Design**:
- `pyproject.toml` declares deps (fastapi, uvicorn, sqlalchemy[asyncio], asyncpg, alembic, pydantic-settings, authlib, celery, redis, pgvector, deepgram-sdk, anthropic, cartesia, twilio, boto3, opentelemetry-*). Dev deps: pytest, pytest-asyncio, respx, testcontainers, ruff, mypy.
- `src/accp/config.py` using `pydantic-settings`:
```python
class Settings(BaseSettings):
    database_url: PostgresDsn
    redis_url: RedisDsn
    s3_endpoint: str; s3_bucket: str = "recordings"
    jwt_secret: SecretStr; jwt_algorithm: str = "HS256"
    access_token_ttl_seconds: int = 3600
    deepgram_api_key: SecretStr | None = None
    anthropic_api_key: SecretStr | None = None
    cartesia_api_key: SecretStr | None = None
    twilio_account_sid: str | None = None
    twilio_auth_token: SecretStr | None = None
    turn_latency_budget_ms: int = 800
    ai_caller_disclosure: str = "You are speaking with an AI assistant."  # EU AI Act
    model_config = SettingsConfigDict(env_prefix="ACCP_", env_file=".env")
```
- `docker-compose.yml`: services `postgres` (pgvector/pgvector:pg16), `redis`, `minio`, `api`, `worker`, `media-bridge`, `web`.
- `.pre-commit-config.yaml`: ruff (lint+format), mypy.

**Testing**:
- `Unit: Settings loads from env with ACCP_ prefix → correct typed values, secrets masked in repr`.
- `Unit: missing required DATABASE_URL → ValidationError naming the field`.
- `Integration (real): docker-compose up postgres redis minio → all healthchecks pass`.
- `CI: ruff check, ruff format --check, mypy --strict all pass on empty package`.

#### 1.2 — Database models & Alembic migrations (Suggestion 3 schema)

**What**: Implement the hybrid relational+JSONB schema from Data Model Suggestion 3 as SQLAlchemy 2.0 models with an initial Alembic migration.

**Design**:
- Enable extensions in the first migration: `pgcrypto`, `vector`, `pg_trgm`.
- One model module per aggregate: `tenant.py`, `user.py` (`users`, `agent_profiles`), `queue.py`, `ai_config.py`, `call.py` (`calls`, `call_transcripts`, `call_summaries`), `knowledge.py` (`knowledge_articles`, `knowledge_gaps`), `campaign.py`, `compliance.py` (`dnc_list`, `consent_records`, `retention_policies`), `qa.py`, `integration.py` (`crm_connections`, `webhooks`).
- JSONB columns typed with `Mapped[dict]`/`Mapped[list]` and a Pydantic shape model in `schemas/` for application-level validation (compensating for no DB-enforced JSONB schema). Example shape:
```python
class AiDecision(BaseModel):
    timestamp: datetime
    type: Literal["intent_detected","entity_extracted","knowledge_retrieved",
                  "escalation_triggered","response_generated"]
    model: str
    latency_ms: int
    payload: dict[str, Any]
```
- `embedding_vector VECTOR(1536)` via `pgvector.sqlalchemy.Vector`.
- Range-partition `calls` and `call_transcripts` by month on `started_at` (partitioned table + monthly child creation in a Celery beat task).
- All GIN/ivfflat/partial indexes from Suggestion 3 included.

**Testing**:
- `Integration (testcontainers Postgres): run alembic upgrade head → all tables, extensions, indexes exist (query pg_indexes / pg_extension)`.
- `Unit: AiDecision rejects unknown type → ValidationError`.
- `Integration: insert call with extracted_entities JSONB, query with @> containment → row returned`.
- `Integration: insert knowledge_article with embedding, ivfflat cosine search → ordered by distance`.
- `Integration: alembic downgrade base then upgrade head → idempotent, no errors`.

#### 1.3 — Auth, RBAC, and tenant isolation

**What**: OAuth 2.0 / OIDC login, JWT issuance, PKCE for browser clients, and a tenant-scoped RBAC dependency.

**Design**:
- `auth/` issues JWTs `{sub, tenant_id, role, exp}`; roles `admin|supervisor|agent|readonly` (from `users.role`).
- FastAPI dependencies: `current_user()`, `require_role(*roles)`, `tenant_scope()` that injects `tenant_id` into every repository call. Every query filters by `tenant_id` — the single most important isolation control (OWASP API1 broken object-level authz).
- OIDC discovery + PKCE authorization-code flow endpoints: `GET /auth/login`, `GET /auth/callback`, `POST /auth/token`, `POST /auth/refresh`, `POST /auth/revoke` (RFC 7009).
- Local password fallback (argon2) for self-host without an IdP.

**Testing**:
- `Unit: JWT round-trip encode/decode → claims preserved; expired token → 401`.
- `Unit: require_role("supervisor") with agent token → 403`.
- `Integration: PKCE flow with mismatched code_verifier → token endpoint 400`.
- `Integration: tenant A user requests tenant B call_id → 404 (not 403, no existence leak)`.
- `Integration (mocked OIDC): valid callback → JWT minted with mapped role`.

---

## Phase 2: Telephony Bridge & Media Transport

### Purpose
Connect the platform to the PSTN and the browser. This phase establishes inbound call answering via Twilio Media Streams (WebSocket audio frames), a pluggable `TelephonyProvider` interface, and the WebRTC agent-desktop signalling path (Opus). After this phase a real phone call can reach the platform and stream bidirectional audio — without any AI yet, audio is echoed/recorded to prove the transport.

### Tasks

#### 2.1 — TelephonyProvider abstraction & Twilio inbound

**What**: Define the provider protocol and implement Twilio inbound answering + Media Streams.

**Design**:
```python
class TelephonyProvider(Protocol):
    async def answer(self, call_ctx: CallContext) -> TwiMLOrNCCO: ...
    async def transfer(self, call_id: str, target: str) -> None: ...
    async def hangup(self, call_id: str) -> None: ...
    async def play(self, call_id: str, audio: AsyncIterator[bytes]) -> None: ...
```
- `POST /telephony/twilio/incoming` returns TwiML `<Connect><Stream url="wss://.../telephony/twilio/media/{call_id}"/>`. Inserts a `calls` row (`direction=inbound`, `status=ringing`, `caller_number`).
- `WS /telephony/twilio/media/{call_id}`: parses Twilio Media Stream JSON frames (`start`, `media` base64 μ-law 8 kHz, `stop`); resamples to PCM16 16 kHz for STT; queues outbound audio back as μ-law.
- Validate Twilio request signatures (`X-Twilio-Signature`) on the HTTP webhook (OWASP API authn).

**Testing**:
- `Unit: TwiML builder → well-formed XML with correct wss stream URL`.
- `Unit: μ-law↔PCM16 resampler round-trips within tolerance on a sine fixture`.
- `Integration (mocked WS): replay recorded Twilio media frames → frames decoded, call row transitions ringing→in_progress`.
- `Integration: webhook with invalid X-Twilio-Signature → 403, no call row created`.

#### 2.2 — WebRTC agent desktop signalling

**What**: `aiortc`-based offer/answer signalling so a browser agent can join a call audio bridge using Opus.

**Design**:
- `POST /telephony/webrtc/offer` (agent JWT) accepts SDP offer, returns SDP answer; negotiates Opus (RFC 6716). ICE/STUN config from settings (TURN optional).
- A `MediaBridge` mixes caller RTP and agent RTP for escalations (conference). MVP: 1:1 caller↔agent or caller↔AI.

**Testing**:
- `Unit: SDP answer offers Opus and rejects unsupported codecs`.
- `Integration (aiortc loopback): offer/answer → DTLS-SRTP connected, audio frames flow`.
- `E2E (Playwright + fake media): agent page connects, mic indicator active`.

#### 2.3 — Recording with S3 lifecycle

**What**: Capture call audio to S3/MinIO and store `recording_url` on the call.

**Design**:
- Recording writer subscribes to the mixed audio stream, writes Opus/WAV to S3 keyed `{tenant}/{yyyy}/{mm}/{call_id}.wav`.
- Designed with a `pause()/resume()` hook (used by PCI in Phase 8).

**Testing**:
- `Integration (MinIO): record 5 s fixture stream → object exists, playable, call.recording_url set`.
- `Unit: recording key path matches tenant/date/call_id pattern`.

---

## Phase 3: Real-Time Voice AI Pipeline (Core Value)

### Purpose
The heart of the product: the STT → LLM(+RAG) → TTS turn loop with a hard latency budget, durable Redis-backed conversation state, and a transparent AI decision log. After this phase the platform can hold a natural multi-turn voice conversation with a caller and answer from a knowledge base.

### Tasks

#### 3.1 — Streaming STT provider (Deepgram)

**What**: Stream caller audio to Deepgram, emit interim + final transcripts with diarisation.

**Design**:
```python
class STTProvider(Protocol):
    async def stream(self, audio: AsyncIterator[bytes]) -> AsyncIterator[Transcript]: ...

class Transcript(BaseModel):
    text: str; is_final: bool; speaker: int
    confidence: float; start_ms: int; duration_ms: int
```
- Deepgram WS with `model=nova-3, diarize=true, interim_results=true, endpointing=300, redact=[pci,pii]`.
- Final transcripts persisted to `call_transcripts` (speaker mapped caller/agent), interim used to drive turn-taking.

**Testing**:
- `Integration (mocked Deepgram WS): feed fixture audio → ordered finals with monotonic sequence_num persisted`.
- `Unit: interim result does not write a transcript row; final does`.
- `Integration (real, opt-in marker): short WAV → non-empty transcript`.

#### 3.2 — Conversation session & multi-LLM routing

**What**: Redis-backed durable session plus an LLM router that sends sub-tasks to the cheapest capable model.

**Design**:
```python
class ConversationSession(BaseModel):
    call_id: str; tenant_id: str
    turns: list[Turn]; entities: dict[str, str]
    state: Literal["greeting","listening","thinking","speaking","escalating","ended"]
# stored at redis key call:{call_id}:session, TTL 24h, written every turn
class LLMProvider(Protocol):
    async def complete(self, msgs, *, model, max_tokens, stream=True) -> AsyncIterator[str]: ...
```
- `llm/router.py` maps `intent_detection`/`knowledge_retrieval` → Haiku, `response_generation` → Sonnet, configurable per `ai_agent_configs.pipeline_config.multi_model_routing`.
- Anthropic prompt caching on the system prompt + KB context to cut per-turn latency/cost.
- Every model call appends an `AiDecision` to `calls.ai_decisions` (transparent/auditable log — a stated differentiator).

**Testing**:
- `Unit: router selects Haiku for intent_detection, Sonnet for response_generation per config`.
- `Integration (redis testcontainer): write session, simulate worker restart, reload → context intact`.
- `Unit: each completion appends exactly one AiDecision with latency_ms and model`.

#### 3.3 — Intent classification & entity extraction

**What**: Classify caller intent and extract entities (account number, order ID, email) per turn.

**Design**:
- Single Haiku call returning structured JSON (Pydantic-validated): `{intent, confidence, entities:{...}}`. Prompt template versioned in `ai/prompts/intent_v1.txt`.
- Extracted entities merged into `calls.extracted_entities` and `ConversationSession.entities`.

**Testing**:
- `Unit (mocked LLM): "my order 12345 is late" → intent=order_status, entities.order_id=12345`.
- `Unit: low-confidence (<0.5) intent flagged for clarification turn`.
- `Fixture: 20 labelled utterances → intent accuracy ≥ threshold recorded as regression baseline`.

#### 3.4 — Knowledge-base RAG with citation/hallucination constraint

**What**: Retrieve KB articles via pgvector and constrain the LLM to cite retrieved content.

**Design**:
- Embed query, `ORDER BY embedding_vector <=> :q LIMIT k` filtered by tenant + `is_published`.
- Response prompt requires inline `[article_id]` citations; if top similarity < `guardrails.hallucination_threshold`, return "I don't have that information" and emit a `knowledge_gap` candidate (Phase 7).
- Bump `metadata.effectiveness_stats.times_retrieved` on used articles.

**Testing**:
- `Integration: seed 3 articles, query semantically matching one → that article top-ranked, citation present in response`.
- `Unit: all similarities below threshold → fallback response, no fabricated citation, gap candidate emitted`.
- `Unit: response containing a citation id not in retrieved set → rejected/regenerated`.

#### 3.5 — Streaming TTS & turn orchestrator (latency budget)

**What**: Stream LLM tokens into TTS and back to the caller, enforcing the ~800 ms turn budget with OTel spans.

**Design**:
```python
class TTSProvider(Protocol):
    async def synthesize(self, text: AsyncIterator[str]) -> AsyncIterator[bytes]: ...
```
- `pipeline/orchestrator.py` turn loop: detect end-of-utterance (Deepgram endpointing) → route LLM → first sentence boundary kicks TTS → audio streamed to telephony `play()`. OTel span per stage; if cumulative > `turn_latency_budget_ms`, log a `latency_breach` event and (config) play a brief filler.
- EU AI Act caller disclosure (`settings.ai_caller_disclosure`) spoken on first turn.

**Testing**:
- `Integration (mocked STT/LLM/TTS, simulated delays): turn completes; OTel spans recorded for stt/llm/tts`.
- `Unit: cumulative latency over budget → latency_breach event emitted`.
- `Unit: first synthesized chunk dispatched before full LLM text available (streaming proven)`.
- `Unit: first AI turn includes the disclosure phrase`.

---

## Phase 4: Routing, Escalation & Human Agent Handoff

### Purpose
Turn the AI agent into part of a contact centre: detect when to escalate, route to the right human agent with full context, and bridge the call. After this phase a caller can be transferred from AI to a human who sees the transcript, entities, and a recommended resolution.

### Tasks

#### 4.1 — Escalation signal engine

**What**: Evaluate `ai_agent_configs.escalation_rules` against live signals (frustration, max-turns, explicit request, low confidence).

**Design**:
```python
class EscalationSignal(BaseModel):
    trigger: Literal["frustration_score","explicit_request","max_turns_exceeded",
                     "low_confidence","compliance_flag"]
    value: float | None; should_escalate: bool
```
- Frustration via per-turn sentiment (Haiku); explicit-request via intent. On escalation, write `calls.escalation_context` (`reason`, score, `summary`, `recommended_resolution`) and set `escalated=true`, `ai_contained=false`.

**Testing**:
- `Unit: frustration_score 0.85 > threshold 0.8 → should_escalate true, context populated`.
- `Unit: explicit "talk to a person" intent → immediate escalation regardless of score`.
- `Unit: contained call (no triggers) → ai_contained true at end`.

#### 4.2 — Queue & agent routing engine

**What**: Assign escalated calls to an available agent per the queue routing strategy.

**Design**:
- `routing/engine.py` implements `round_robin|least_recent|skills_based|priority` from `queues.routing_config`. Skills-based matches `agent_profiles.skills` (JSONB `@>`) to call intent skill tags. Respects `max_concurrent` and `status=online`. Overflow to `overflow_queue_id` after `max_wait_seconds`.
- Atomic assignment via `SELECT ... FOR UPDATE SKIP LOCKED` to avoid double-assignment.

**Testing**:
- `Integration: 2 online agents, round_robin, 4 calls → 2/2 distribution`.
- `Integration: skills_based, only one agent has "billing" expert → routed there`.
- `Integration (concurrent): two routers, one free agent → exactly one assignment (SKIP LOCKED)`.
- `Unit: no available agent within max_wait → overflow queue selected`.

#### 4.3 — Context transfer & call bridge

**What**: Bridge caller↔agent and deliver full context to the agent desktop.

**Design**:
- On assignment: `calls.status=transferring`, telephony `transfer()` bridges caller to agent's WebRTC session (Phase 2.2 MediaBridge). Push `escalation_context` + transcript + entities over the realtime hub (Phase 6) to the agent UI.
- `WS /realtime/agent/{agent_id}` receives `call.assigned` with the full context payload.

**Testing**:
- `Integration (mocked telephony): escalation → transfer() called with agent target, status=transferring then in_progress`.
- `Integration: agent WS receives call.assigned containing transcript, entities, recommended_resolution`.
- `E2E: AI call → "agent please" → agent desktop shows context panel`.

---

## Phase 5: REST API, Webhooks & CRM Integration

### Purpose
Expose the platform to developers and connect it to systems of record. After this phase external systems can read calls/transcripts/summaries via a documented OpenAPI 3.1 surface, subscribe to events via signed webhooks (AsyncAPI 3.0), and sync call outcomes to Salesforce and HubSpot.

### Tasks

#### 5.1 — Public REST API (OpenAPI 3.1)

**What**: CRUD + query endpoints for calls, transcripts, summaries, agents, queues, knowledge articles, AI configs.

**Design**:
- Resource routers under `/api/v1/...`; cursor pagination; tenant-scoped; OAuth2 bearer. Examples: `GET /api/v1/calls?status=&from=&to=&cursor=`, `GET /api/v1/calls/{id}`, `GET /api/v1/calls/{id}/transcript`, `GET /api/v1/calls/{id}/summary`, `POST /api/v1/knowledge/articles`.
- FastAPI emits OpenAPI 3.1 at `/openapi.json`. Validate against OWASP API Top 10 (object-level authz tests, rate limiting, no excessive exposure — secrets never serialised).

**Testing**:
- `Integration: GET /api/v1/calls cross-tenant id → 404`.
- `Integration: pagination cursor returns stable, non-overlapping pages`.
- `Contract: generated openapi.json validates against OpenAPI 3.1 schema`.
- `Security: response models never include password_hash / auth_encrypted (excessive-exposure check)`.

#### 5.2 — Signed webhooks (AsyncAPI 3.0)

**What**: Deliver `call.started|completed|escalated|qa.scored|summary.ready|knowledge.gap` events with HMAC signatures and retries.

**Design**:
- `webhooks/dispatcher.py` Celery task; HMAC-SHA256 over body with `webhooks.secret_hash`, `X-ACCP-Signature` header. Exponential backoff `[1,5,30,120,600]s` from `webhooks.config.retry_policy`; failure_count tracked; auto-disable after N failures.
- Event payloads documented in `asyncapi.yaml`.

**Testing**:
- `Integration (mock receiver): call.completed → POST with valid HMAC within event filter`.
- `Unit: receiver 500 → retried per backoff schedule; 5th failure disables webhook`.
- `Unit: event not in webhook.events filter → not delivered`.

#### 5.3 — CRM connector framework + Salesforce & HubSpot

**What**: Bi-directional connectors with encrypted credentials and field mapping.

**Design**:
```python
class CRMConnector(Protocol):
    async def upsert_contact(self, phone: str, fields: dict) -> str: ...
    async def create_activity(self, contact_id: str, call: CallRecord) -> str: ...
    async def lookup(self, phone: str) -> dict | None: ...
```
- Credentials in `crm_connections.auth_encrypted` (AES-256-GCM via app key). OAuth 2.0 for Salesforce (JWT bearer) and HubSpot. Field mapping from `crm_connections.field_mappings`. Inbound: caller lookup at call start enriches the AI/agent. Outbound: on `call.completed`/`summary.ready`, upsert contact + create activity.

**Testing**:
- `Unit: encrypt→decrypt credentials round-trips; ciphertext != plaintext`.
- `Integration (respx-mocked Salesforce): summary.ready → activity created with mapped fields`.
- `Integration: caller lookup hit → CRM fields injected into ConversationSession`.
- `Unit: CRM 401 → connection flagged, sync_config.last_error set, retried`.

---

## Phase 6: Supervisor Dashboard & Agent Desktop (Frontend)

### Purpose
Give supervisors live visibility and agents their working surface. After this phase supervisors see real-time queue depth, containment, AHT and CSAT, and agents have a desktop that rings, shows context, and lets them take over from the AI.

### Tasks

#### 6.1 — Realtime hub (WebSocket + Redis pub/sub)

**What**: Fan-out live events to dashboard and agent clients.

**Design**:
- `realtime/hub.py`: clients subscribe to tenant channels; backend publishes to Redis `tenant:{id}:events`; hub relays to subscribed WS connections. Event types: `metrics.tick` (every 2 s aggregate), `call.assigned`, `call.status`, `sentiment.alert`.
- Auth on WS via JWT query param validated at connect; tenant-scoped subscription only.

**Testing**:
- `Integration: publish metrics.tick → all subscribed dashboard sockets receive it; other tenant sockets do not`.
- `Unit: WS connect with expired JWT → closed 4401`.

#### 6.2 — Supervisor dashboard (Next.js)

**What**: Live metrics dashboard meeting ISO 18295-1 KPI reporting.

**Design**:
- Pages: Live (queue depth, active calls, agent states, AI containment rate, AHT, CSAT gauges), Calls (searchable list → detail with transcript + AI decision log + recording player), Agents.
- Metrics computed by a Celery beat aggregator into a `metrics` materialized view / Redis cache, pushed via `metrics.tick`.
- KPIs: AHT, FCR, CSAT, abandon rate, containment rate (ISO 18295-1).

**Testing**:
- `E2E (Playwright): seed active calls → Live page shows correct queue depth and containment %`.
- `E2E: open call detail → transcript and AI decision log render in order`.
- `Unit: AHT/containment aggregation SQL matches hand-computed fixture`.

#### 6.3 — Agent desktop

**What**: Browser agent surface: presence toggle, incoming-call ring, WebRTC audio, context panel.

**Design**:
- Presence sets `agent_profiles.status`. On `call.assigned`, ring + show `escalation_context`, transcript, entities, recommended resolution. WebRTC via Phase 2.2 signalling. Wrap-up timer writes `timing.wrap_up_seconds`.

**Testing**:
- `E2E: agent online → simulated assignment rings → accept → audio connects, context shown`.
- `E2E: end call → wrap-up timer recorded, agent returns to online`.

---

## Phase 7: Speech Analytics, QA Scoring & Knowledge Gaps

### Purpose
Deliver the AI-augmentation differentiators: 100% automated QA scoring (vs 2–5% manual sampling), automated post-call summaries with CRM auto-update, and knowledge-gap detection feeding content improvement. These run as async post-call work.

### Tasks

#### 7.1 — Automated post-call summary

**What**: Generate a structured summary on call end and trigger CRM sync.

**Design**:
- Celery task on `call.completed`: Sonnet summarises transcript → `call_summaries` (`summary_text`, `structured_summary.action_items|key_topics|follow_up_*`, `resolution_status`). Emits `summary.ready` (→ webhooks + CRM sync in Phase 5.3).

**Testing**:
- `Integration (mocked LLM): completed call → summary row with action_items, resolution_status valid enum`.
- `Unit: empty transcript → summary task no-ops gracefully, logs`.

#### 7.2 — 100% automated QA scoring

**What**: Score every call across sentiment, compliance, resolution, empathy.

**Design**:
- Celery task scores each call → `qa_scores` (`overall_score`, `dimensions` JSONB, `rationale.flags|notable_moments|improvement_suggestions`). Tenant-defined custom dimensions read from JSONB without migration. `overall_score < 60` (partial index) triggers `coaching_triggered` and a `sentiment.alert`/coaching workflow.

**Testing**:
- `Integration (mocked LLM): scored call → all dimensions populated, overall in [0,100]`.
- `Unit: score < 60 → coaching_triggered true, alert emitted`.
- `Integration: query qa_scores GIN dimensions for "missing_disclosure" flag → returns flagged calls`.

#### 7.3 — Knowledge gap detection & dashboard

**What**: Aggregate unanswered questions (from Phase 3.4 fallbacks) into actionable gaps.

**Design**:
- Gap candidates clustered by embedding similarity; near-duplicates increment `knowledge_gaps.frequency` and append `sample_call_ids`; `analysis.closest_articles` stored. Admin UI lists open gaps ranked by frequency; resolving links a new `knowledge_articles` row (`resolved_article_id`).

**Testing**:
- `Unit: two semantically similar unanswered questions → single gap, frequency=2`.
- `Integration: resolve gap → status=resolved, resolved_article_id set, article published`.
- `E2E: gaps dashboard ranks by frequency desc`.

---

## Phase 8: Outbound Campaigns & Compliance Automation

### Purpose
Add compliant outbound calling and the regulatory controls (PCI, TCPA, GDPR, retention) required for production and regulated industries. After this phase the platform can run appointment-reminder/survey/collection campaigns lawfully and handle card data and data-subject rights.

### Tasks

#### 8.1 — Outbound campaign scheduler & dialler

**What**: Schedule and pace outbound batches via Celery, dialling through the telephony provider.

**Design**:
- `campaigns/scheduler.py`: beat task dequeues `campaign_contacts` (status=pending) respecting `execution_config.pacing` (`calls_per_minute`, `max_concurrent`), `timezone_rules`, `business_hours_only`, and `retry_policy`. Each attempt updates `metrics` JSONB. Connected calls run the Phase 3 pipeline with `custom_data` injected for personalisation.
- Outbound calls routed through a STIR/SHAKEN-compliant trunk (Twilio) with registered caller ID (RFC 8226/8588).

**Testing**:
- `Integration: campaign of 5 contacts, pacing 10/min → calls placed within window, metrics updated`.
- `Unit: outside business hours → contact deferred, not dialled`.
- `Unit: retry_policy max_attempts reached → contact status=failed`.

#### 8.2 — DNC checking & TCPA/GDPR consent

**What**: Block calls without consent or on DNC; log consent immutably (TCPA 2025, GDPR).

**Design**:
- Pre-dial gate: check `dnc_list` (tenant + number) and a valid `consent_records` row (`tcpa_prior_express_written` for marketing). No consent/DNC hit → status `dnc_blocked`, never dialled, audited. Consent revocation honoured within 10 business days; consent records append-only (immutable evidence chain in `evidence` JSONB).

**Testing**:
- `Unit: number on DNC → dnc_blocked, no dial`.
- `Unit: marketing campaign, no tcpa_prior_express_written consent → blocked`.
- `Integration: revoke consent → subsequent attempts blocked`.
- `Unit: consent record update attempt → rejected (append-only)`.

#### 8.3 — PCI pause/resume recording

**What**: Auto-pause recording (and divert DTMF away from STT/LLM) when card entry is detected (PCI DSS 4.0).

**Design**:
- DTMF/payment-intent detection triggers `recording.pause()` (Phase 2.3) and routes digits through a secure path that never reaches STT, LLM, transcripts, or recordings. Resume after capture. A `compliance_flag` records pause/resume timestamps (not the digits).

**Testing**:
- `Integration: simulated card-entry trigger → recording paused, no transcript rows during window, resumes after`.
- `Unit: DTMF digits during pause never appear in call_transcripts or recording object`.

#### 8.4 — Retention & GDPR erasure jobs

**What**: Enforce `retention_policies` and honour erasure requests.

**Design**:
- Beat task deletes recordings/transcripts past `rules.*.retention_days` (S3 + DB), archiving call metadata to cold storage where `auto_delete=false`. `DELETE /api/v1/data-subjects/{phone}` cascades erasure of calls/transcripts/recordings/summaries for that number, preserving append-only consent/audit records (legal basis).

**Testing**:
- `Integration: transcript older than retention → deleted; recording object removed`.
- `Integration: erasure request → caller's PII removed, consent audit retained`.
- `Unit: pci_data immediate_purge → no residual rows`.

---

## Phase 9: Packaging, Deployment, MCP & Observability

### Purpose
Make the platform deployable, observable, and agent-accessible. After this phase the stack runs via Docker Compose (self-host) or Helm (k8s), exposes latency/health telemetry, and offers an MCP server so external AI agents can query call data.

### Tasks

#### 9.1 — Deployment artifacts (Compose + Helm) & HIPAA mode

**What**: Production-ready containers and a self-host/k8s deployment path with an optional HIPAA-eligible profile.

**Design**:
- Multi-stage Dockerfile; Compose stack with healthchecks; Helm chart (configurable replicas for api/worker/media-bridge, external Postgres/Redis/S3). HIPAA profile: TLS 1.2+ enforced, AES-256 at rest, 6-year audit retention, RBAC, BAA-vendor toggle, self-hosted LLM endpoint option (data residency tier).

**Testing**:
- `Integration: docker-compose up → /health green across services, sample call works E2E`.
- `Unit: helm template lints, renders valid manifests`.
- `Unit: HIPAA profile sets retention=6y and forbids non-TLS endpoints`.

#### 9.2 — Observability (OTel, Prometheus, audit log)

**What**: Per-turn latency tracing, metrics, and an auditable AI decision/event log.

**Design**:
- OTel spans across STT/LLM/TTS (turn-latency histogram, p50/p95/p99 by tenant). Prometheus metrics: active_calls, containment_rate, queue_depth, webhook_failures, latency_breaches. Structured JSON logs. The `calls.ai_decisions` log is the user-facing transparency feature (queryable + exportable).

**Testing**:
- `Integration: drive a call → turn-latency histogram populated; latency_breach increments on injected delay`.
- `Unit: every AI decision is both logged (OTel) and persisted (ai_decisions)`.

#### 9.3 — MCP server

**What**: Expose call transcripts, summaries, and customer history as MCP tools for external AI agents.

**Design**:
- `integrations/mcp/server.py` (MCP spec) tools: `search_calls`, `get_transcript`, `get_summary`, `customer_history(phone)`. Tenant-scoped via an MCP auth token mapped to tenant+role; read-only.

**Testing**:
- `Integration (MCP client): search_calls returns tenant-scoped results only`.
- `Unit: MCP token for tenant A cannot read tenant B transcript`.
- `Contract: tool schemas validate against MCP specification`.

---

## Phase Summary & Dependencies

```
Phase 1: Foundation, Data Model & Auth      ─── required by everything
    │
Phase 2: Telephony Bridge & Media           ─── requires P1
    │
Phase 3: Real-Time Voice AI Pipeline        ─── requires P1, P2   (CORE VALUE)
    │
    ├── Phase 4: Routing, Escalation, Handoff      ─── requires P3
    │       │
    │       └── Phase 6: Dashboard & Agent Desktop ─── requires P4 (+ P2 WebRTC)
    │
    ├── Phase 5: REST API, Webhooks, CRM           ─── requires P3 (parallel with P4)
    │
    └── Phase 7: Analytics, QA, Knowledge Gaps     ─── requires P3 (+ P5 for CRM sync)
            │
Phase 8: Outbound Campaigns & Compliance    ─── requires P3, P5 (consent/DNC), P2.3 (PCI pause)
    │
Phase 9: Packaging, Deployment, MCP, Obs.   ─── requires all prior (cross-cutting; start early, finish last)
```

**Parallelism opportunities**
- After Phase 3: Phases **4, 5, and 7** can be developed concurrently (distinct subsystems sharing only the call/transcript models).
- Phase **6** (frontend) can begin against mocked realtime/API once Phase 4 and 5 contracts are fixed.
- Phase **9** observability (9.2) and packaging (9.1) can be started incrementally from Phase 1 and hardened at the end.

---

## Definition of Done (per phase)

1. All tasks in the phase implemented.
2. All unit and mocked-integration tests pass; real-dependency integration tests pass under testcontainers (or are explicitly skipped behind a marker).
3. `ruff check` and `ruff format --check` pass.
4. `mypy --strict` passes on changed packages.
5. Docker image builds; `docker-compose up` reaches healthy state for affected services.
6. The phase's headline capability works end-to-end (demonstrable via an E2E or integration test).
7. New configuration options documented in `config.py` and `.env.example`.
8. New REST endpoints appear in the generated OpenAPI 3.1 spec; new events documented in `asyncapi.yaml`.
9. Alembic migration created and reversible where the schema changed.
10. New JSONB shapes have a corresponding Pydantic validator (application-level schema enforcement).
11. Tenant isolation verified for any new data-access path (cross-tenant access test present).
12. Latency-sensitive paths (Phase 3+) have OTel spans and a latency assertion in tests.
```
