# Data Model Suggestion 2: Event-Sourced / CQRS Model

## Approach

An event-sourcing architecture where the canonical data store is an append-only log of domain events. Every state change -- call initiated, utterance transcribed, intent detected, agent assigned, call escalated, QA scored -- is recorded as an immutable event. The current state of any aggregate (a call, a campaign, an agent shift) is reconstructed by replaying its event stream. Separate read-side projections (materialized views) are built from these events to serve dashboards, search, and analytics queries. Commands flow through command handlers that validate business rules before emitting events.

## Why This Suits an AI Call Center Platform

Call center operations are inherently event-driven. A call is not a static record; it is a sequence of things that happened: the phone rang, the AI greeted the caller, the caller stated their intent, the knowledge base was queried, the AI responded, frustration was detected, the call was escalated, the human agent resolved it, the call ended, QA scored it. Event sourcing captures this temporal reality directly rather than flattening it into mutable rows.

Key advantages for this domain:

- **Complete audit trail.** Compliance regulations (TCPA, GDPR, PCI DSS) demand knowing exactly what happened and when. The event log IS the audit trail -- no separate audit table needed.
- **Temporal queries.** "What was the state of this call 30 seconds before escalation?" is trivially answered by replaying events up to that timestamp.
- **AI decision explainability.** Every AI decision (intent classification, escalation trigger, knowledge retrieval) can be an event with full context, making the AI's reasoning chain auditable.
- **Decoupled read models.** The supervisor dashboard, QA scoring engine, and CRM sync service can each maintain their own optimized projection without coupling to each other.

The main trade-off is operational complexity: event stores require careful schema versioning, projection rebuilds need infrastructure, and developers must think in terms of events rather than CRUD.

## Event Store Schema

```sql
-- ============================================================
-- CORE EVENT STORE (PostgreSQL as event store)
-- ============================================================
CREATE TABLE event_store (
    event_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    aggregate_type  VARCHAR(50) NOT NULL,   -- 'Call', 'Campaign', 'AgentShift', 'KnowledgeBase'
    aggregate_id    UUID NOT NULL,
    event_type      VARCHAR(100) NOT NULL,  -- 'CallInitiated', 'UtteranceTranscribed', etc.
    event_version   INTEGER NOT NULL,       -- schema version for this event type
    sequence_num    BIGINT NOT NULL,         -- per-aggregate ordering
    payload         JSONB NOT NULL,          -- event-specific data
    metadata        JSONB NOT NULL DEFAULT '{}',  -- correlation_id, causation_id, actor_id, tenant_id
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (aggregate_id, sequence_num)
);

CREATE INDEX idx_events_aggregate ON event_store(aggregate_id, sequence_num);
CREATE INDEX idx_events_type ON event_store(event_type, created_at);
CREATE INDEX idx_events_tenant ON event_store((metadata->>'tenant_id'), created_at);
CREATE INDEX idx_events_correlation ON event_store((metadata->>'correlation_id'));

-- Optimistic concurrency: before inserting, check that sequence_num = last + 1
-- This prevents conflicting writes to the same aggregate.

-- ============================================================
-- SNAPSHOTS (optional, for long-lived aggregates)
-- ============================================================
CREATE TABLE aggregate_snapshots (
    aggregate_id    UUID NOT NULL,
    aggregate_type  VARCHAR(50) NOT NULL,
    sequence_num    BIGINT NOT NULL,
    state           JSONB NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (aggregate_id, sequence_num)
);
```

## Domain Events Catalog

```yaml
# ============================================================
# CALL AGGREGATE EVENTS
# ============================================================
CallInitiated:
  aggregate: Call
  payload:
    tenant_id: uuid
    direction: "inbound" | "outbound"
    caller_number: string
    callee_number: string
    queue_id: uuid | null
    ai_config_id: uuid
    campaign_id: uuid | null
    sip_call_id: string

CallAnswered:
  aggregate: Call
  payload:
    answered_by: "ai_agent" | "human_agent"
    agent_id: uuid | null

UtteranceTranscribed:
  aggregate: Call
  payload:
    speaker: "caller" | "ai_agent" | "human_agent"
    content: string
    confidence: float
    stt_provider: string
    duration_ms: integer
    timestamp_offset_ms: integer

IntentDetected:
  aggregate: Call
  payload:
    intent_name: string
    confidence: float
    entities: [{type: string, value: string, confidence: float}]
    llm_model: string
    prompt_tokens: integer
    completion_tokens: integer

KnowledgeRetrieved:
  aggregate: Call
  payload:
    query_text: string
    articles_returned: [{article_id: uuid, relevance_score: float}]
    retrieval_latency_ms: integer

KnowledgeGapDetected:
  aggregate: Call
  payload:
    question_text: string
    attempted_queries: [string]
    fallback_response_given: boolean

AIResponseGenerated:
  aggregate: Call
  payload:
    response_text: string
    llm_model: string
    prompt_tokens: integer
    completion_tokens: integer
    latency_ms: integer
    tts_provider: string

FrustrationDetected:
  aggregate: Call
  payload:
    frustration_score: float
    signals: [string]  # e.g. ["raised_voice", "repeated_request", "explicit_complaint"]
    transcript_segment_id: uuid

EscalationTriggered:
  aggregate: Call
  payload:
    reason: string
    frustration_score: float | null
    complexity_score: float | null
    customer_requested: boolean
    context_summary: string
    recommended_resolution: string

AgentAssigned:
  aggregate: Call
  payload:
    agent_id: uuid
    queue_id: uuid
    wait_time_seconds: integer
    routing_strategy: string

CallPlacedOnHold:
  aggregate: Call
  payload:
    placed_by: "ai_agent" | "human_agent"
    reason: string | null

CallResumedFromHold:
  aggregate: Call
  payload:
    hold_duration_seconds: integer

PCIPauseActivated:
  aggregate: Call
  payload:
    reason: "card_data_collection"
    recording_paused: boolean

PCIPauseDeactivated:
  aggregate: Call
  payload:
    pause_duration_seconds: integer
    recording_resumed: boolean

ConsentCaptured:
  aggregate: Call
  payload:
    consent_type: string
    granted: boolean
    evidence_type: string
    transcript_segment_id: uuid | null

CallEnded:
  aggregate: Call
  payload:
    end_reason: "completed" | "abandoned" | "failed" | "voicemail" | "transferred"
    duration_seconds: integer
    ai_contained: boolean
    total_hold_seconds: integer

CallSummarized:
  aggregate: Call
  payload:
    summary_text: string
    action_items: [string]
    resolution_status: string
    llm_model: string

QAScoreAssigned:
  aggregate: Call
  payload:
    overall_score: float
    sentiment_score: float
    compliance_score: float
    resolution_score: float
    empathy_score: float
    scored_by: "ai" | "supervisor"
    flags: [string]
    coaching_triggered: boolean

CRMSynced:
  aggregate: Call
  payload:
    crm_type: string
    crm_record_id: string
    fields_updated: [string]
    sync_latency_ms: integer

# ============================================================
# CAMPAIGN AGGREGATE EVENTS
# ============================================================
CampaignCreated:
  aggregate: Campaign
  payload:
    tenant_id: uuid
    name: string
    type: string
    ai_config_id: uuid
    contact_count: integer
    scheduled_start: timestamp
    scheduled_end: timestamp

CampaignStarted:
  aggregate: Campaign
  payload:
    actual_start: timestamp

ContactDialed:
  aggregate: Campaign
  payload:
    contact_id: uuid
    phone_number: string
    attempt_number: integer

ContactDNCBlocked:
  aggregate: Campaign
  payload:
    contact_id: uuid
    phone_number: string
    dnc_reason: string

CampaignCompleted:
  aggregate: Campaign
  payload:
    total_dialed: integer
    total_connected: integer
    total_dnc_blocked: integer
    actual_end: timestamp

# ============================================================
# AGENT SHIFT AGGREGATE EVENTS
# ============================================================
AgentLoggedIn:
  aggregate: AgentShift
  payload:
    agent_id: uuid
    tenant_id: uuid

AgentStatusChanged:
  aggregate: AgentShift
  payload:
    previous_status: string
    new_status: string
    reason: string | null

AgentLoggedOut:
  aggregate: AgentShift
  payload:
    total_active_seconds: integer
    calls_handled: integer
```

## Command Handlers (pseudocode)

```python
class InitiateCallHandler:
    def handle(self, cmd: InitiateCall) -> list[Event]:
        # Validate: tenant exists, AI config exists, not on DNC list
        dnc_check = self.dnc_projection.is_blocked(cmd.tenant_id, cmd.caller_number)
        if dnc_check:
            raise DNCBlockedError(cmd.caller_number)
        
        consent = self.consent_projection.has_consent(cmd.tenant_id, cmd.caller_number, 'recording_consent')
        
        return [CallInitiated(
            tenant_id=cmd.tenant_id,
            direction=cmd.direction,
            caller_number=cmd.caller_number,
            callee_number=cmd.callee_number,
            queue_id=cmd.queue_id,
            ai_config_id=cmd.ai_config_id,
            campaign_id=cmd.campaign_id,
            sip_call_id=cmd.sip_call_id,
        )]

class EscalateCallHandler:
    def handle(self, cmd: EscalateCall, call_state: CallAggregate) -> list[Event]:
        if call_state.status != 'in_progress':
            raise InvalidStateError("Can only escalate in-progress calls")
        if call_state.already_escalated:
            raise InvalidStateError("Call already escalated")
        
        return [EscalationTriggered(
            reason=cmd.reason,
            frustration_score=cmd.frustration_score,
            complexity_score=cmd.complexity_score,
            customer_requested=cmd.customer_requested,
            context_summary=self.build_context(call_state),
            recommended_resolution=cmd.recommended_resolution,
        )]
```

## Read-Side Projections

```sql
-- ============================================================
-- PROJECTION: Live Call Board (for supervisor dashboard)
-- ============================================================
CREATE TABLE proj_live_calls (
    call_id         UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    direction       VARCHAR(10) NOT NULL,
    status          VARCHAR(30) NOT NULL,
    caller_number   VARCHAR(50),
    queue_id        UUID,
    assigned_agent_id UUID,
    ai_contained    BOOLEAN NOT NULL DEFAULT true,
    started_at      TIMESTAMPTZ NOT NULL,
    current_duration_seconds INTEGER,
    last_utterance  TEXT,
    frustration_level NUMERIC(4,3) DEFAULT 0,
    last_updated    TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_proj_live_calls_tenant ON proj_live_calls(tenant_id, status);

-- ============================================================
-- PROJECTION: Call History (for search and reporting)
-- ============================================================
CREATE TABLE proj_call_history (
    call_id         UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    direction       VARCHAR(10) NOT NULL,
    caller_number   VARCHAR(50),
    callee_number   VARCHAR(50),
    queue_name      VARCHAR(255),
    agent_name      VARCHAR(255),
    campaign_name   VARCHAR(255),
    started_at      TIMESTAMPTZ NOT NULL,
    ended_at        TIMESTAMPTZ,
    duration_seconds INTEGER,
    ai_contained    BOOLEAN,
    escalated       BOOLEAN,
    escalation_reason VARCHAR(255),
    qa_overall_score NUMERIC(5,2),
    summary_text    TEXT,
    resolution_status VARCHAR(50),
    intent_names    TEXT[],
    search_vector   TSVECTOR  -- for full-text search across summary + transcript highlights
);

CREATE INDEX idx_proj_history_tenant_date ON proj_call_history(tenant_id, started_at DESC);
CREATE INDEX idx_proj_history_search ON proj_call_history USING GIN(search_vector);

-- ============================================================
-- PROJECTION: Agent Performance (for coaching and scheduling)
-- ============================================================
CREATE TABLE proj_agent_performance (
    agent_id        UUID NOT NULL,
    tenant_id       UUID NOT NULL,
    period_date     DATE NOT NULL,
    calls_handled   INTEGER NOT NULL DEFAULT 0,
    avg_handle_time_seconds NUMERIC(8,2),
    avg_qa_score    NUMERIC(5,2),
    escalation_rate NUMERIC(5,4),
    total_active_seconds INTEGER NOT NULL DEFAULT 0,
    coaching_flags  INTEGER NOT NULL DEFAULT 0,
    PRIMARY KEY (agent_id, period_date)
);

-- ============================================================
-- PROJECTION: Campaign Progress (for campaign manager)
-- ============================================================
CREATE TABLE proj_campaign_progress (
    campaign_id     UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    name            VARCHAR(255) NOT NULL,
    status          VARCHAR(30) NOT NULL,
    total_contacts  INTEGER NOT NULL,
    dialed          INTEGER NOT NULL DEFAULT 0,
    connected       INTEGER NOT NULL DEFAULT 0,
    dnc_blocked     INTEGER NOT NULL DEFAULT 0,
    completed       INTEGER NOT NULL DEFAULT 0,
    success_rate    NUMERIC(5,4),
    started_at      TIMESTAMPTZ,
    last_updated    TIMESTAMPTZ
);

-- ============================================================
-- PROJECTION: Consent Ledger (for compliance queries)
-- ============================================================
CREATE TABLE proj_consent_ledger (
    tenant_id       UUID NOT NULL,
    phone_number    VARCHAR(50) NOT NULL,
    consent_type    VARCHAR(50) NOT NULL,
    current_status  BOOLEAN NOT NULL,
    granted_at      TIMESTAMPTZ,
    revoked_at      TIMESTAMPTZ,
    last_evidence_event_id UUID,
    PRIMARY KEY (tenant_id, phone_number, consent_type)
);

-- ============================================================
-- PROJECTION: DNC Registry (for real-time outbound checking)
-- ============================================================
CREATE TABLE proj_dnc_registry (
    tenant_id       UUID NOT NULL,
    phone_number    VARCHAR(50) NOT NULL,
    reason          VARCHAR(100),
    source          VARCHAR(50),
    blocked_since   TIMESTAMPTZ NOT NULL,
    expires_at      TIMESTAMPTZ,
    PRIMARY KEY (tenant_id, phone_number)
);

-- ============================================================
-- PROJECTION: Knowledge Gap Tracker
-- ============================================================
CREATE TABLE proj_knowledge_gaps (
    gap_id          UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    question_text   TEXT NOT NULL,
    frequency       INTEGER NOT NULL DEFAULT 1,
    first_seen_at   TIMESTAMPTZ NOT NULL,
    last_seen_at    TIMESTAMPTZ NOT NULL,
    sample_call_ids UUID[],
    status          VARCHAR(30) NOT NULL DEFAULT 'open'
);

CREATE INDEX idx_proj_gaps_tenant_freq ON proj_knowledge_gaps(tenant_id, frequency DESC);
```

## Projection Updater (pseudocode)

```python
class LiveCallProjectionUpdater:
    """Subscribes to call events and maintains the proj_live_calls table."""
    
    def on_event(self, event: Event):
        match event.event_type:
            case "CallInitiated":
                self.db.upsert("proj_live_calls", {
                    "call_id": event.aggregate_id,
                    "tenant_id": event.payload["tenant_id"],
                    "direction": event.payload["direction"],
                    "status": "ringing",
                    "caller_number": event.payload["caller_number"],
                    "queue_id": event.payload["queue_id"],
                    "started_at": event.created_at,
                    "last_updated": event.created_at,
                })
            case "CallAnswered":
                self.db.update("proj_live_calls", event.aggregate_id, {
                    "status": "in_progress",
                    "assigned_agent_id": event.payload.get("agent_id"),
                    "last_updated": event.created_at,
                })
            case "FrustrationDetected":
                self.db.update("proj_live_calls", event.aggregate_id, {
                    "frustration_level": event.payload["frustration_score"],
                    "last_updated": event.created_at,
                })
            case "CallEnded":
                self.db.delete("proj_live_calls", event.aggregate_id)
```

## Trade-offs

**Strengths:**
- Perfect audit trail for compliance (TCPA, GDPR, PCI DSS) -- events are immutable and timestamped.
- AI decision explainability is built-in: every classification, retrieval, and escalation decision is a discrete event with full context.
- Temporal queries ("replay call state at timestamp T") are natural.
- Independent read projections allow the supervisor dashboard, QA engine, and CRM sync to scale independently.
- Easy to add new projections (e.g., a new analytics dimension) without modifying the write side.

**Weaknesses:**
- Operational complexity: projection rebuilds, event versioning/upcasting, and eventual consistency between write and read sides.
- Queries across aggregates (e.g., "all calls for agent X in campaign Y with QA score below 60") require well-designed projections.
- Higher storage cost: the event store plus multiple projections duplicates data.
- Steeper learning curve for developers accustomed to CRUD patterns.
- Eventual consistency means the supervisor dashboard may lag a few hundred milliseconds behind reality (acceptable for most use cases but requires careful UX).

## Scalability Considerations

- **Event store partitioning:** Partition `event_store` by `created_at` (monthly). Old partitions can be archived to cold storage.
- **Event streaming:** Use Kafka or NATS as the event bus for real-time projection updates. The PostgreSQL event store serves as the durable source of truth; the message broker handles fan-out to projections.
- **Projection parallelism:** Each projection updater runs as an independent consumer, allowing horizontal scaling.
- **Snapshotting:** For long-lived aggregates (campaigns with thousands of contacts), periodic snapshots prevent slow aggregate reconstruction.

## Migration Path

The event-sourced model can be introduced incrementally. Start with the normalized relational model (Suggestion 1) for core CRUD operations, then introduce event sourcing for the call lifecycle aggregate where the audit trail and temporal query benefits are most valuable. The existing relational tables become read-side projections fed by the event stream. Other aggregates (campaigns, knowledge base) can migrate to event sourcing later if the benefits justify the complexity.
