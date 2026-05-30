# Data Model Suggestion 1: Normalized Relational Model (PostgreSQL)

## Approach

A traditional third-normal-form (3NF+) relational schema in PostgreSQL. Every entity has its own table with strict foreign key constraints, proper indexes, and well-defined cardinalities. This approach maximizes data integrity, leverages decades of relational tooling, and maps cleanly onto the operational entities of a call center platform: tenants, agents, queues, calls, transcripts, campaigns, knowledge articles, and QA scores.

## Why This Suits an AI Call Center Platform

Call center operations are fundamentally transactional: a call starts, events happen in sequence, the call ends, and downstream processes (QA scoring, CRM sync, billing) consume the result. Relational databases excel at this kind of structured, transactional workload. Strict foreign keys ensure referential integrity across the complex web of relationships (agent-to-queue, call-to-campaign, transcript-to-call, QA-score-to-call). PostgreSQL's MVCC model supports the concurrent read/write patterns of live supervisory dashboards reading while calls are being written.

The main trade-off is schema rigidity: adding new entity types or attributes requires migrations. For a platform still evolving its AI capabilities (new intent types, new escalation signals, new analytics dimensions), this friction is real but manageable with a disciplined migration workflow.

## Schema Definition

```sql
-- ============================================================
-- TENANT / ORGANIZATION
-- ============================================================
CREATE TABLE tenants (
    tenant_id       UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    plan_tier       VARCHAR(50) NOT NULL DEFAULT 'free',
    settings        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- USERS AND AGENTS
-- ============================================================
CREATE TABLE users (
    user_id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(tenant_id),
    email           VARCHAR(320) NOT NULL,
    display_name    VARCHAR(255) NOT NULL,
    role            VARCHAR(50) NOT NULL CHECK (role IN ('admin', 'supervisor', 'agent', 'readonly')),
    password_hash   TEXT,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, email)
);

CREATE TABLE agent_profiles (
    agent_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES users(user_id),
    tenant_id       UUID NOT NULL REFERENCES tenants(tenant_id),
    status          VARCHAR(30) NOT NULL DEFAULT 'offline'
                    CHECK (status IN ('online', 'offline', 'busy', 'away', 'wrap_up')),
    max_concurrent  INTEGER NOT NULL DEFAULT 1,
    skills          TEXT[] NOT NULL DEFAULT '{}',
    last_active_at  TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_agent_profiles_tenant_status ON agent_profiles(tenant_id, status);
CREATE INDEX idx_agent_profiles_skills ON agent_profiles USING GIN(skills);

-- ============================================================
-- QUEUES AND ROUTING
-- ============================================================
CREATE TABLE queues (
    queue_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(tenant_id),
    name            VARCHAR(255) NOT NULL,
    priority        INTEGER NOT NULL DEFAULT 0,
    routing_strategy VARCHAR(50) NOT NULL DEFAULT 'round_robin'
                    CHECK (routing_strategy IN ('round_robin', 'least_recent', 'skills_based', 'priority')),
    max_wait_seconds INTEGER NOT NULL DEFAULT 300,
    overflow_queue_id UUID REFERENCES queues(queue_id),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE queue_agents (
    queue_id        UUID NOT NULL REFERENCES queues(queue_id),
    agent_id        UUID NOT NULL REFERENCES agent_profiles(agent_id),
    priority        INTEGER NOT NULL DEFAULT 0,
    assigned_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (queue_id, agent_id)
);

-- ============================================================
-- AI VOICE AGENT CONFIGURATION
-- ============================================================
CREATE TABLE ai_agent_configs (
    config_id       UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(tenant_id),
    name            VARCHAR(255) NOT NULL,
    system_prompt   TEXT NOT NULL,
    voice_id        VARCHAR(100) NOT NULL,
    stt_provider    VARCHAR(50) NOT NULL DEFAULT 'deepgram',
    tts_provider    VARCHAR(50) NOT NULL DEFAULT 'cartesia',
    llm_provider    VARCHAR(50) NOT NULL DEFAULT 'anthropic',
    llm_model       VARCHAR(100) NOT NULL DEFAULT 'claude-sonnet',
    temperature     NUMERIC(3,2) NOT NULL DEFAULT 0.3,
    max_turn_tokens INTEGER NOT NULL DEFAULT 300,
    escalation_rules JSONB NOT NULL DEFAULT '[]',
    is_active       BOOLEAN NOT NULL DEFAULT true,
    version         INTEGER NOT NULL DEFAULT 1,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- CALLS
-- ============================================================
CREATE TABLE calls (
    call_id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(tenant_id),
    direction       VARCHAR(10) NOT NULL CHECK (direction IN ('inbound', 'outbound')),
    status          VARCHAR(30) NOT NULL DEFAULT 'ringing'
                    CHECK (status IN ('ringing', 'in_progress', 'on_hold', 'transferring',
                                      'completed', 'abandoned', 'failed', 'voicemail')),
    caller_number   VARCHAR(50),
    callee_number   VARCHAR(50),
    queue_id        UUID REFERENCES queues(queue_id),
    ai_config_id    UUID REFERENCES ai_agent_configs(config_id),
    assigned_agent_id UUID REFERENCES agent_profiles(agent_id),
    campaign_id     UUID,  -- FK added after campaigns table
    started_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    answered_at     TIMESTAMPTZ,
    ended_at        TIMESTAMPTZ,
    duration_seconds INTEGER,
    hold_time_seconds INTEGER DEFAULT 0,
    wrap_up_seconds INTEGER DEFAULT 0,
    end_reason      VARCHAR(50),
    ai_contained    BOOLEAN NOT NULL DEFAULT false,
    escalated       BOOLEAN NOT NULL DEFAULT false,
    escalation_reason VARCHAR(255),
    recording_url   TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_calls_tenant_started ON calls(tenant_id, started_at DESC);
CREATE INDEX idx_calls_status ON calls(tenant_id, status) WHERE status NOT IN ('completed', 'abandoned', 'failed');
CREATE INDEX idx_calls_agent ON calls(assigned_agent_id, started_at DESC);
CREATE INDEX idx_calls_campaign ON calls(campaign_id) WHERE campaign_id IS NOT NULL;

-- ============================================================
-- CALL TRANSCRIPTS
-- ============================================================
CREATE TABLE call_transcripts (
    transcript_id   UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    call_id         UUID NOT NULL REFERENCES calls(call_id),
    sequence_num    INTEGER NOT NULL,
    speaker         VARCHAR(20) NOT NULL CHECK (speaker IN ('caller', 'ai_agent', 'human_agent', 'system')),
    content         TEXT NOT NULL,
    confidence      NUMERIC(4,3),
    started_at      TIMESTAMPTZ NOT NULL,
    duration_ms     INTEGER,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (call_id, sequence_num)
);

CREATE INDEX idx_transcripts_call ON call_transcripts(call_id, sequence_num);
CREATE INDEX idx_transcripts_content_search ON call_transcripts USING GIN(to_tsvector('english', content));

-- ============================================================
-- INTENTS AND ENTITIES EXTRACTED FROM CALLS
-- ============================================================
CREATE TABLE call_intents (
    intent_id       UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    call_id         UUID NOT NULL REFERENCES calls(call_id),
    intent_name     VARCHAR(255) NOT NULL,
    confidence      NUMERIC(4,3) NOT NULL,
    detected_at     TIMESTAMPTZ NOT NULL,
    resolved        BOOLEAN NOT NULL DEFAULT false
);

CREATE INDEX idx_call_intents_call ON call_intents(call_id);

CREATE TABLE call_entities (
    entity_id       UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    call_id         UUID NOT NULL REFERENCES calls(call_id),
    entity_type     VARCHAR(100) NOT NULL,  -- e.g. 'account_number', 'order_id', 'email'
    entity_value    TEXT NOT NULL,
    confidence      NUMERIC(4,3) NOT NULL,
    transcript_id   UUID REFERENCES call_transcripts(transcript_id),
    detected_at     TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_call_entities_call ON call_entities(call_id);

-- ============================================================
-- CALL SUMMARIES (AI-generated post-call)
-- ============================================================
CREATE TABLE call_summaries (
    summary_id      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    call_id         UUID NOT NULL REFERENCES calls(call_id) UNIQUE,
    summary_text    TEXT NOT NULL,
    action_items    TEXT[],
    resolution_status VARCHAR(50) CHECK (resolution_status IN ('resolved', 'unresolved', 'follow_up', 'escalated')),
    crm_synced      BOOLEAN NOT NULL DEFAULT false,
    crm_sync_at     TIMESTAMPTZ,
    generated_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- KNOWLEDGE BASE
-- ============================================================
CREATE TABLE knowledge_articles (
    article_id      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(tenant_id),
    title           VARCHAR(500) NOT NULL,
    content         TEXT NOT NULL,
    category        VARCHAR(255),
    tags            TEXT[] DEFAULT '{}',
    embedding_vector VECTOR(1536),  -- pgvector extension
    is_published    BOOLEAN NOT NULL DEFAULT false,
    version         INTEGER NOT NULL DEFAULT 1,
    created_by      UUID REFERENCES users(user_id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_knowledge_tenant_cat ON knowledge_articles(tenant_id, category);
CREATE INDEX idx_knowledge_tags ON knowledge_articles USING GIN(tags);
CREATE INDEX idx_knowledge_embedding ON knowledge_articles USING ivfflat(embedding_vector vector_cosine_ops) WITH (lists = 100);
CREATE INDEX idx_knowledge_content_search ON knowledge_articles USING GIN(to_tsvector('english', content));

CREATE TABLE knowledge_gaps (
    gap_id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(tenant_id),
    question_text   TEXT NOT NULL,
    frequency       INTEGER NOT NULL DEFAULT 1,
    sample_call_ids UUID[] DEFAULT '{}',
    status          VARCHAR(30) NOT NULL DEFAULT 'open'
                    CHECK (status IN ('open', 'in_progress', 'resolved', 'dismissed')),
    resolved_article_id UUID REFERENCES knowledge_articles(article_id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- OUTBOUND CAMPAIGNS
-- ============================================================
CREATE TABLE campaigns (
    campaign_id     UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(tenant_id),
    name            VARCHAR(255) NOT NULL,
    type            VARCHAR(50) NOT NULL CHECK (type IN ('appointment_reminder', 'payment_collection',
                                                        'survey', 'notification', 'custom')),
    status          VARCHAR(30) NOT NULL DEFAULT 'draft'
                    CHECK (status IN ('draft', 'scheduled', 'running', 'paused', 'completed', 'cancelled')),
    ai_config_id    UUID REFERENCES ai_agent_configs(config_id),
    scheduled_start TIMESTAMPTZ,
    scheduled_end   TIMESTAMPTZ,
    timezone_rules  JSONB NOT NULL DEFAULT '{}',
    total_contacts  INTEGER NOT NULL DEFAULT 0,
    completed_calls INTEGER NOT NULL DEFAULT 0,
    created_by      UUID REFERENCES users(user_id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

ALTER TABLE calls ADD CONSTRAINT fk_calls_campaign FOREIGN KEY (campaign_id) REFERENCES campaigns(campaign_id);

CREATE TABLE campaign_contacts (
    contact_id      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    campaign_id     UUID NOT NULL REFERENCES campaigns(campaign_id),
    phone_number    VARCHAR(50) NOT NULL,
    contact_name    VARCHAR(255),
    custom_data     JSONB DEFAULT '{}',
    status          VARCHAR(30) NOT NULL DEFAULT 'pending'
                    CHECK (status IN ('pending', 'scheduled', 'calling', 'completed',
                                      'failed', 'dnc_blocked', 'skipped')),
    attempt_count   INTEGER NOT NULL DEFAULT 0,
    last_attempt_at TIMESTAMPTZ,
    call_id         UUID REFERENCES calls(call_id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_campaign_contacts_status ON campaign_contacts(campaign_id, status);

-- ============================================================
-- DO NOT CALL (DNC) LIST
-- ============================================================
CREATE TABLE dnc_list (
    dnc_id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(tenant_id),
    phone_number    VARCHAR(50) NOT NULL,
    reason          VARCHAR(100),
    source          VARCHAR(50) NOT NULL CHECK (source IN ('customer_request', 'regulatory', 'manual', 'system')),
    added_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
    expires_at      TIMESTAMPTZ,
    UNIQUE (tenant_id, phone_number)
);

CREATE INDEX idx_dnc_lookup ON dnc_list(tenant_id, phone_number);

-- ============================================================
-- CONSENT TRACKING (TCPA / GDPR)
-- ============================================================
CREATE TABLE consent_records (
    consent_id      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(tenant_id),
    phone_number    VARCHAR(50) NOT NULL,
    consent_type    VARCHAR(50) NOT NULL CHECK (consent_type IN ('tcpa_express', 'tcpa_prior_express_written',
                                                                 'gdpr_legitimate_interest', 'gdpr_consent',
                                                                 'recording_consent')),
    granted         BOOLEAN NOT NULL,
    granted_at      TIMESTAMPTZ NOT NULL,
    revoked_at      TIMESTAMPTZ,
    evidence_type   VARCHAR(50),  -- 'voice_recording', 'web_form', 'written'
    evidence_url    TEXT,
    call_id         UUID REFERENCES calls(call_id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_consent_lookup ON consent_records(tenant_id, phone_number, consent_type);

-- ============================================================
-- QA SCORING
-- ============================================================
CREATE TABLE qa_scores (
    score_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    call_id         UUID NOT NULL REFERENCES calls(call_id),
    tenant_id       UUID NOT NULL REFERENCES tenants(tenant_id),
    overall_score   NUMERIC(5,2) NOT NULL,
    sentiment_score NUMERIC(5,2),
    compliance_score NUMERIC(5,2),
    resolution_score NUMERIC(5,2),
    empathy_score   NUMERIC(5,2),
    scored_by       VARCHAR(20) NOT NULL CHECK (scored_by IN ('ai', 'supervisor', 'both')),
    flags           TEXT[] DEFAULT '{}',
    coaching_triggered BOOLEAN NOT NULL DEFAULT false,
    scored_at       TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_qa_scores_call ON qa_scores(call_id);
CREATE INDEX idx_qa_scores_tenant_date ON qa_scores(tenant_id, scored_at DESC);
CREATE INDEX idx_qa_scores_low ON qa_scores(tenant_id, overall_score) WHERE overall_score < 60;

-- ============================================================
-- CRM INTEGRATION MAPPINGS
-- ============================================================
CREATE TABLE crm_connections (
    connection_id   UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(tenant_id),
    crm_type        VARCHAR(50) NOT NULL CHECK (crm_type IN ('salesforce', 'hubspot', 'zendesk', 'servicenow', 'custom')),
    api_endpoint    TEXT NOT NULL,
    auth_encrypted  BYTEA NOT NULL,
    field_mappings  JSONB NOT NULL DEFAULT '{}',
    sync_enabled    BOOLEAN NOT NULL DEFAULT true,
    last_sync_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- COMPLIANCE: CALL RETENTION POLICIES
-- ============================================================
CREATE TABLE retention_policies (
    policy_id       UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(tenant_id),
    recording_retention_days INTEGER NOT NULL DEFAULT 90,
    transcript_retention_days INTEGER NOT NULL DEFAULT 365,
    pci_pause_enabled BOOLEAN NOT NULL DEFAULT false,
    gdpr_auto_delete BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- WEBHOOKS
-- ============================================================
CREATE TABLE webhooks (
    webhook_id      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(tenant_id),
    url             TEXT NOT NULL,
    events          TEXT[] NOT NULL,  -- e.g. {'call.completed', 'call.escalated', 'qa.scored'}
    secret_hash     TEXT NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_webhooks_tenant ON webhooks(tenant_id, is_active);
```

## Trade-offs

**Strengths:**
- Referential integrity is enforced at the database level, preventing orphaned records across the complex call-to-transcript-to-QA-score chain.
- Mature ecosystem: every ORM, migration tool, backup strategy, and monitoring tool works out of the box.
- Strong transactional guarantees for compliance-critical operations (consent recording, DNC checking, PCI pause/resume).
- PostgreSQL's pgvector extension handles knowledge base embedding search without a separate vector database.
- Full-text search on transcripts via built-in GIN indexes avoids an external search engine for basic use cases.

**Weaknesses:**
- Schema migrations required for every new entity or attribute -- friction when rapidly iterating on AI features.
- High-throughput real-time transcript ingestion (hundreds of concurrent calls, each producing utterances every few seconds) may hit write contention on the transcripts table. Partitioning by tenant or time range mitigates this.
- Deeply nested or variable-structure data (escalation rules, LLM chain-of-thought logs, multi-model routing decisions) fits awkwardly into fixed columns.
- Star-schema analytics queries (average handle time by queue by hour) require materialized views or a separate analytics store.

## Scalability Considerations

- **Partitioning:** The `calls` and `call_transcripts` tables should be range-partitioned by `started_at` (monthly or weekly) to keep query performance manageable as data grows. PostgreSQL's declarative partitioning handles this natively.
- **Read replicas:** Supervisor dashboards and analytics queries should read from streaming replicas to avoid impacting call-path writes.
- **Connection pooling:** PgBouncer or Supavisor in front of PostgreSQL to handle the high connection count from concurrent call workers.
- **Archival:** A background job should move calls older than the retention policy to cold storage (S3 + Parquet) and delete from the primary database.

## Migration Path

This normalized schema serves well as a starting point. If flexibility becomes a bottleneck, individual tables can evolve toward the hybrid model (Suggestion 3) by adding JSONB columns for extensible attributes without abandoning the relational core. If event-driven patterns emerge as dominant, the call lifecycle tables can be fronted with an event store (Suggestion 2) while keeping this schema as the read-side projection.
