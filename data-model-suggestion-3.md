# Data Model Suggestion 3: Hybrid Relational + JSONB Model (PostgreSQL)

## Approach

A pragmatic middle ground that keeps core operational entities in traditional relational columns (with foreign keys, constraints, and typed indexes) while using PostgreSQL JSONB columns for data that is inherently variable, nested, or evolving rapidly. This recognizes that an AI call center platform has two kinds of data: stable operational data (calls, agents, queues, campaigns) and volatile AI-generated data (LLM chain-of-thought, multi-model routing decisions, variable entity extractions, flexible escalation rule configurations). The relational columns handle the first; JSONB handles the second.

## Why This Suits an AI Call Center Platform

AI-powered platforms face a fundamental tension: the operational call center domain is well-understood and relational (calls have agents, agents belong to queues, campaigns contain contacts), but the AI capabilities are rapidly evolving. New LLM features, new intent taxonomies, new escalation signal types, new QA scoring dimensions -- these change faster than schema migrations can keep up.

JSONB columns solve this by providing schema flexibility exactly where it is needed, while preserving relational integrity exactly where it matters. A call record still has a proper UUID primary key, a foreign key to its tenant, and indexed status and timestamp columns. But the AI decision log attached to that call -- which LLM was used, what prompt was sent, what confidence thresholds triggered escalation -- lives in a JSONB column that can evolve without migrations.

PostgreSQL's JSONB implementation is mature: GIN indexes support efficient querying, partial indexes can target specific JSON paths, and JSONB operations compose naturally with SQL joins and aggregates.

## Schema Definition

```sql
-- ============================================================
-- EXTENSIONS
-- ============================================================
CREATE EXTENSION IF NOT EXISTS pgcrypto;      -- gen_random_uuid()
CREATE EXTENSION IF NOT EXISTS vector;         -- pgvector for embeddings
CREATE EXTENSION IF NOT EXISTS pg_trgm;        -- trigram similarity for fuzzy search

-- ============================================================
-- TENANT / ORGANIZATION
-- ============================================================
CREATE TABLE tenants (
    tenant_id       UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    plan_tier       VARCHAR(50) NOT NULL DEFAULT 'free',
    -- JSONB: feature flags, branding, notification preferences, custom fields
    -- Avoids a separate settings table and schema changes for every new config option
    config          JSONB NOT NULL DEFAULT '{
        "features": {},
        "branding": {},
        "notifications": {},
        "custom_fields": {}
    }',
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
    -- JSONB: notification preferences, UI layout, custom fields per tenant
    preferences     JSONB NOT NULL DEFAULT '{}',
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
    -- JSONB: skills with proficiency levels, language capabilities, certifications
    -- More flexible than a TEXT[] column -- allows structured skill metadata
    skills          JSONB NOT NULL DEFAULT '[]',
    -- Example: [{"name": "billing", "level": "expert"}, {"name": "spanish", "level": "fluent"}]
    last_active_at  TIMESTAMPTZ,
    -- JSONB: shift schedule, break preferences, routing weight overrides
    schedule_config JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_agent_profiles_tenant_status ON agent_profiles(tenant_id, status);
CREATE INDEX idx_agent_skills ON agent_profiles USING GIN(skills jsonb_path_ops);

-- ============================================================
-- QUEUES AND ROUTING
-- ============================================================
CREATE TABLE queues (
    queue_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(tenant_id),
    name            VARCHAR(255) NOT NULL,
    priority        INTEGER NOT NULL DEFAULT 0,
    -- JSONB: routing rules are complex and vary by tenant
    -- May include skill requirements, time-of-day rules, load balancing weights
    routing_config  JSONB NOT NULL DEFAULT '{
        "strategy": "round_robin",
        "skill_requirements": [],
        "max_wait_seconds": 300,
        "overflow_queue_id": null,
        "priority_boost_rules": []
    }',
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
    is_active       BOOLEAN NOT NULL DEFAULT true,
    version         INTEGER NOT NULL DEFAULT 1,
    -- JSONB: the entire AI pipeline configuration
    -- This changes frequently as models, providers, and prompts evolve
    pipeline_config JSONB NOT NULL DEFAULT '{
        "system_prompt": "",
        "voice_id": "",
        "stt": {"provider": "deepgram", "model": "nova-2", "language": "en"},
        "tts": {"provider": "cartesia", "model": "sonic", "voice_id": ""},
        "llm": {"provider": "anthropic", "model": "claude-sonnet", "temperature": 0.3, "max_tokens": 300},
        "multi_model_routing": {
            "intent_detection": {"provider": "anthropic", "model": "claude-haiku"},
            "knowledge_retrieval": {"provider": "anthropic", "model": "claude-haiku"},
            "response_generation": {"provider": "anthropic", "model": "claude-sonnet"}
        }
    }',
    -- JSONB: escalation rules are highly variable per tenant and use case
    escalation_rules JSONB NOT NULL DEFAULT '[
        {"trigger": "frustration_score", "threshold": 0.8, "action": "escalate"},
        {"trigger": "explicit_request", "action": "escalate"},
        {"trigger": "max_turns_exceeded", "threshold": 10, "action": "escalate"}
    ]',
    -- JSONB: guardrails, compliance rules, response constraints
    guardrails      JSONB NOT NULL DEFAULT '{
        "max_response_length": 200,
        "prohibited_topics": [],
        "required_disclaimers": [],
        "hallucination_threshold": 0.7
    }',
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
    end_reason      VARCHAR(50),
    ai_contained    BOOLEAN NOT NULL DEFAULT false,
    escalated       BOOLEAN NOT NULL DEFAULT false,
    -- JSONB: timing breakdown (hold time, wrap time, IVR time, etc.)
    timing          JSONB NOT NULL DEFAULT '{
        "ring_seconds": 0,
        "hold_seconds": 0,
        "talk_seconds": 0,
        "wrap_up_seconds": 0,
        "ai_processing_ms": 0
    }',
    -- JSONB: all AI decisions made during this call
    -- Grows during the call; final state is the complete decision log
    ai_decisions    JSONB NOT NULL DEFAULT '[]',
    -- Example entry: {"timestamp": "...", "type": "intent_detected", "intent": "billing_inquiry",
    --   "confidence": 0.94, "model": "claude-haiku", "latency_ms": 45}
    -- JSONB: entities extracted during the call
    extracted_entities JSONB NOT NULL DEFAULT '{}',
    -- Example: {"account_number": "12345", "order_id": "ORD-789", "email": "user@example.com"}
    -- JSONB: escalation context when escalated
    escalation_context JSONB,
    -- Example: {"reason": "customer_frustrated", "frustration_score": 0.85,
    --   "summary": "Customer upset about billing error", "recommended_resolution": "Issue credit"}
    recording_url   TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_calls_tenant_started ON calls(tenant_id, started_at DESC);
CREATE INDEX idx_calls_active ON calls(tenant_id, status) WHERE status NOT IN ('completed', 'abandoned', 'failed');
CREATE INDEX idx_calls_agent ON calls(assigned_agent_id, started_at DESC);
CREATE INDEX idx_calls_campaign ON calls(campaign_id) WHERE campaign_id IS NOT NULL;
-- GIN index on ai_decisions for querying specific decision types
CREATE INDEX idx_calls_ai_decisions ON calls USING GIN(ai_decisions jsonb_path_ops);
-- GIN index on extracted entities for searching by entity values
CREATE INDEX idx_calls_entities ON calls USING GIN(extracted_entities jsonb_path_ops);

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
    -- JSONB: per-utterance analytics (sentiment, keywords, compliance flags)
    analytics       JSONB NOT NULL DEFAULT '{
        "sentiment": null,
        "keywords": [],
        "compliance_flags": [],
        "language_detected": null
    }',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (call_id, sequence_num)
);

CREATE INDEX idx_transcripts_call ON call_transcripts(call_id, sequence_num);
CREATE INDEX idx_transcripts_content ON call_transcripts USING GIN(to_tsvector('english', content));
CREATE INDEX idx_transcripts_analytics ON call_transcripts USING GIN(analytics jsonb_path_ops);

-- ============================================================
-- CALL SUMMARIES (AI-generated post-call)
-- ============================================================
CREATE TABLE call_summaries (
    summary_id      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    call_id         UUID NOT NULL REFERENCES calls(call_id) UNIQUE,
    summary_text    TEXT NOT NULL,
    resolution_status VARCHAR(50) CHECK (resolution_status IN ('resolved', 'unresolved', 'follow_up', 'escalated')),
    -- JSONB: structured summary fields that may vary by tenant or call type
    structured_summary JSONB NOT NULL DEFAULT '{
        "action_items": [],
        "key_topics": [],
        "customer_sentiment_arc": [],
        "compliance_issues": [],
        "follow_up_required": false,
        "follow_up_details": null
    }',
    -- JSONB: CRM sync status and field mapping
    crm_sync        JSONB NOT NULL DEFAULT '{
        "synced": false,
        "synced_at": null,
        "crm_record_id": null,
        "fields_updated": [],
        "errors": []
    }',
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
    embedding_vector VECTOR(1536),
    is_published    BOOLEAN NOT NULL DEFAULT false,
    version         INTEGER NOT NULL DEFAULT 1,
    -- JSONB: article metadata that varies by content type
    metadata        JSONB NOT NULL DEFAULT '{
        "content_type": "article",
        "source": "manual",
        "language": "en",
        "audience": "all",
        "related_intents": [],
        "effectiveness_stats": {
            "times_retrieved": 0,
            "times_helpful": 0,
            "times_insufficient": 0
        }
    }',
    created_by      UUID REFERENCES users(user_id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_knowledge_tenant_cat ON knowledge_articles(tenant_id, category);
CREATE INDEX idx_knowledge_tags ON knowledge_articles USING GIN(tags);
CREATE INDEX idx_knowledge_embedding ON knowledge_articles USING ivfflat(embedding_vector vector_cosine_ops) WITH (lists = 100);
CREATE INDEX idx_knowledge_content ON knowledge_articles USING GIN(to_tsvector('english', content));
CREATE INDEX idx_knowledge_metadata ON knowledge_articles USING GIN(metadata jsonb_path_ops);

CREATE TABLE knowledge_gaps (
    gap_id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(tenant_id),
    question_text   TEXT NOT NULL,
    frequency       INTEGER NOT NULL DEFAULT 1,
    sample_call_ids UUID[] DEFAULT '{}',
    status          VARCHAR(30) NOT NULL DEFAULT 'open'
                    CHECK (status IN ('open', 'in_progress', 'resolved', 'dismissed')),
    resolved_article_id UUID REFERENCES knowledge_articles(article_id),
    -- JSONB: analysis of what queries were attempted and why they failed
    analysis        JSONB NOT NULL DEFAULT '{
        "attempted_queries": [],
        "closest_articles": [],
        "suggested_content": null
    }',
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
    -- JSONB: timezone rules, retry policies, pacing configuration
    execution_config JSONB NOT NULL DEFAULT '{
        "timezone_rules": {},
        "retry_policy": {"max_attempts": 3, "delay_minutes": 60},
        "pacing": {"calls_per_minute": 10, "max_concurrent": 5},
        "business_hours_only": true
    }',
    -- JSONB: live campaign metrics (updated by background workers)
    metrics         JSONB NOT NULL DEFAULT '{
        "total_contacts": 0,
        "dialed": 0,
        "connected": 0,
        "completed": 0,
        "failed": 0,
        "dnc_blocked": 0,
        "avg_duration_seconds": 0,
        "success_rate": 0
    }',
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
    status          VARCHAR(30) NOT NULL DEFAULT 'pending'
                    CHECK (status IN ('pending', 'scheduled', 'calling', 'completed',
                                      'failed', 'dnc_blocked', 'skipped')),
    attempt_count   INTEGER NOT NULL DEFAULT 0,
    last_attempt_at TIMESTAMPTZ,
    call_id         UUID REFERENCES calls(call_id),
    -- JSONB: contact-specific data for personalised outreach
    -- e.g. appointment details, payment amounts, survey questions
    custom_data     JSONB NOT NULL DEFAULT '{}',
    -- JSONB: per-contact outcome details
    outcome         JSONB,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_campaign_contacts_status ON campaign_contacts(campaign_id, status);
CREATE INDEX idx_campaign_contacts_custom ON campaign_contacts USING GIN(custom_data jsonb_path_ops);

-- ============================================================
-- DO NOT CALL (DNC) LIST
-- ============================================================
CREATE TABLE dnc_list (
    dnc_id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(tenant_id),
    phone_number    VARCHAR(50) NOT NULL,
    reason          VARCHAR(100),
    source          VARCHAR(50) NOT NULL CHECK (source IN ('customer_request', 'regulatory', 'manual', 'system')),
    -- JSONB: regulatory details, source documentation
    details         JSONB NOT NULL DEFAULT '{}',
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
    consent_type    VARCHAR(50) NOT NULL,
    granted         BOOLEAN NOT NULL,
    granted_at      TIMESTAMPTZ NOT NULL,
    revoked_at      TIMESTAMPTZ,
    -- JSONB: evidence chain with flexible structure
    evidence        JSONB NOT NULL DEFAULT '{
        "type": null,
        "url": null,
        "transcript_segment": null,
        "ip_address": null,
        "user_agent": null
    }',
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
    scored_by       VARCHAR(20) NOT NULL CHECK (scored_by IN ('ai', 'supervisor', 'both')),
    -- JSONB: flexible scoring dimensions
    -- Allows tenants to define custom QA criteria without schema changes
    dimensions      JSONB NOT NULL DEFAULT '{
        "sentiment": null,
        "compliance": null,
        "resolution": null,
        "empathy": null,
        "accuracy": null,
        "professionalism": null
    }',
    -- JSONB: detailed scoring rationale from the AI scorer
    rationale       JSONB NOT NULL DEFAULT '{
        "flags": [],
        "coaching_triggered": false,
        "coaching_reason": null,
        "notable_moments": [],
        "improvement_suggestions": []
    }',
    scored_at       TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_qa_scores_call ON qa_scores(call_id);
CREATE INDEX idx_qa_scores_tenant_date ON qa_scores(tenant_id, scored_at DESC);
CREATE INDEX idx_qa_scores_low ON qa_scores(tenant_id, overall_score) WHERE overall_score < 60;
CREATE INDEX idx_qa_dimensions ON qa_scores USING GIN(dimensions jsonb_path_ops);

-- ============================================================
-- CRM CONNECTIONS
-- ============================================================
CREATE TABLE crm_connections (
    connection_id   UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(tenant_id),
    crm_type        VARCHAR(50) NOT NULL CHECK (crm_type IN ('salesforce', 'hubspot', 'zendesk', 'servicenow', 'custom')),
    api_endpoint    TEXT NOT NULL,
    auth_encrypted  BYTEA NOT NULL,
    -- JSONB: field mappings between call data and CRM fields
    -- Highly tenant-specific and changes frequently
    field_mappings  JSONB NOT NULL DEFAULT '{
        "call_summary": null,
        "resolution_status": null,
        "entities": {},
        "custom_fields": {}
    }',
    -- JSONB: sync configuration and status
    sync_config     JSONB NOT NULL DEFAULT '{
        "enabled": true,
        "sync_on_call_end": true,
        "sync_on_qa_score": false,
        "retry_on_failure": true,
        "last_sync_at": null,
        "last_error": null
    }',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- WEBHOOKS
-- ============================================================
CREATE TABLE webhooks (
    webhook_id      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(tenant_id),
    url             TEXT NOT NULL,
    events          TEXT[] NOT NULL,
    secret_hash     TEXT NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    -- JSONB: delivery status, retry config, filtering rules
    config          JSONB NOT NULL DEFAULT '{
        "retry_policy": {"max_attempts": 5, "backoff_seconds": [1, 5, 30, 120, 600]},
        "filters": {},
        "headers": {},
        "last_delivery": null,
        "failure_count": 0
    }',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- RETENTION POLICIES
-- ============================================================
CREATE TABLE retention_policies (
    policy_id       UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(tenant_id) UNIQUE,
    -- JSONB: flexible retention rules per data type
    rules           JSONB NOT NULL DEFAULT '{
        "recordings": {"retention_days": 90, "auto_delete": true},
        "transcripts": {"retention_days": 365, "auto_delete": true},
        "call_metadata": {"retention_days": 730, "auto_delete": false},
        "pci_data": {"retention_days": 0, "auto_delete": true, "immediate_purge": true},
        "consent_records": {"retention_days": 2555, "auto_delete": false},
        "gdpr_auto_delete_on_request": true
    }',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## Key JSONB Query Patterns

```sql
-- Find calls where the AI detected a specific intent with high confidence
SELECT call_id, started_at, ai_decisions
FROM calls
WHERE tenant_id = :tenant_id
  AND ai_decisions @> '[{"type": "intent_detected", "intent": "billing_inquiry"}]'
  AND started_at > now() - INTERVAL '7 days';

-- Find calls with a specific extracted entity
SELECT call_id, extracted_entities->>'account_number' AS account
FROM calls
WHERE tenant_id = :tenant_id
  AND extracted_entities ? 'account_number';

-- Find agents with a specific skill at expert level
SELECT agent_id, user_id, skills
FROM agent_profiles
WHERE tenant_id = :tenant_id
  AND skills @> '[{"name": "billing", "level": "expert"}]';

-- QA scores with specific compliance flags
SELECT score_id, call_id, overall_score, rationale->'flags' AS flags
FROM qa_scores
WHERE tenant_id = :tenant_id
  AND rationale->'flags' ? 'missing_disclosure'
  AND scored_at > now() - INTERVAL '30 days';

-- Campaign contacts with specific custom data
SELECT contact_id, phone_number, custom_data->>'appointment_date' AS appt_date
FROM campaign_contacts
WHERE campaign_id = :campaign_id
  AND custom_data->>'appointment_date' = '2026-05-27';
```

## Trade-offs

**Strengths:**
- Best of both worlds: relational integrity for core entities, schema flexibility for AI and configuration data.
- Single database technology (PostgreSQL) reduces operational burden compared to polyglot persistence.
- JSONB columns evolve without migrations -- critical for rapidly iterating AI features.
- GIN indexes on JSONB provide efficient querying without sacrificing flexibility.
- Natural fit for multi-tenant SaaS where each tenant may have different custom fields, scoring criteria, and CRM mappings.
- pgvector integration for knowledge base embeddings keeps everything in one database.

**Weaknesses:**
- JSONB data lacks database-enforced schema validation -- application-level validation is required (use JSON Schema or Zod).
- Complex JSONB queries can be slower than equivalent queries on typed columns, especially with deeply nested paths.
- ORMs may not fully support JSONB querying, requiring raw SQL for advanced JSONB operations.
- Risk of "schema drift" where different records in the same table have different JSONB shapes without discipline.
- JSONB columns cannot participate in foreign key constraints -- referential integrity within JSONB must be enforced by the application.

## Scalability Considerations

- **JSONB size monitoring:** Set alerts if any JSONB column (especially `ai_decisions` on calls) grows beyond 100KB per row. Consider extracting to a separate table if this happens.
- **GIN index maintenance:** GIN indexes on large JSONB columns add write overhead. Use partial GIN indexes (e.g., only on active calls) to reduce this.
- **Table partitioning:** Partition `calls` and `call_transcripts` by month on `started_at`, same as the normalized model.
- **TOAST compression:** PostgreSQL automatically compresses large JSONB values via TOAST, but monitoring TOAST table size is important for capacity planning.
- **Read replicas:** Same strategy as the normalized model -- analytics queries hit replicas.

## Migration Path

This is the most natural evolution from the normalized model (Suggestion 1). Start with strict relational columns for everything. As flexibility needs emerge (new AI features, tenant-specific configurations), add JSONB columns to existing tables rather than creating new tables or running schema migrations. If a JSONB field's structure stabilizes and performance becomes critical, promote it to typed relational columns. This approach also serves as a natural stepping stone toward the event-sourced model (Suggestion 2) -- the `ai_decisions` JSONB array on calls is structurally similar to an event stream.
