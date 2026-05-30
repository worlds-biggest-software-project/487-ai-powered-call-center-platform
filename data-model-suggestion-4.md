# Data Model Suggestion 4: Graph Database Model (Neo4j)

## Approach

A graph-native data model using Neo4j, where core domain entities -- agents, callers, calls, queues, skills, knowledge articles, campaigns, and escalation paths -- are nodes, and the rich relationships between them are first-class edges with properties. This approach treats the call center as a network of relationships: routing decisions traverse skill graphs, escalation paths follow organizational hierarchies, knowledge retrieval walks topic taxonomies, and caller history forms a temporal interaction graph.

## Why a Graph Database Suits This Domain

1. **Skill-based routing is graph traversal.** Matching an inbound call to the best available agent requires traversing intent-to-skill-to-agent-to-queue relationships. In a relational model this needs 4-6 table joins; in a graph it is a single traversal.

2. **Escalation paths are directed graphs.** Routing decisions consider team hierarchies, supervisor availability, skill overlap, and prior interaction history -- naturally modeled as weighted edges.

3. **Knowledge base retrieval benefits from graph structure.** Articles relate to intents, products, and resolution procedures through a taxonomy DAG. Graph traversal enables "related article" discovery and gap detection more naturally than relational joins.

4. **Caller interaction history is a temporal graph.** A caller's journey across calls, channels, and agents creates a rich interaction graph enabling context-aware routing ("route to the same agent who handled this caller's previous call").

5. **Agent assist recommendations.** Suggesting responses, articles, and next-best actions based on call context is a recommendation problem well-served by graph-based collaborative filtering.

### Where This Model Is Less Ideal

- **High-throughput writes.** Millions of daily transcript utterances may stress Neo4j write capacity. Mitigated by buffering in Redis and batch-inserting.
- **Aggregation queries.** Dashboard metrics (AHT, containment rate, queue depth) are better served by time-series or columnar stores.
- **Operational complexity.** Neo4j clustering adds overhead compared to managed PostgreSQL. Fewer developers have Cypher experience.

---

## Schema Definition (Cypher DDL)

### Constraints and Indexes

```cypher
// Uniqueness constraints (implicitly create indexes)
CREATE CONSTRAINT tenant_id_unique IF NOT EXISTS FOR (t:Tenant) REQUIRE t.tenant_id IS UNIQUE;
CREATE CONSTRAINT agent_id_unique IF NOT EXISTS FOR (a:Agent) REQUIRE a.agent_id IS UNIQUE;
CREATE CONSTRAINT caller_id_unique IF NOT EXISTS FOR (c:Caller) REQUIRE c.caller_id IS UNIQUE;
CREATE CONSTRAINT call_id_unique IF NOT EXISTS FOR (c:Call) REQUIRE c.call_id IS UNIQUE;
CREATE CONSTRAINT queue_id_unique IF NOT EXISTS FOR (q:Queue) REQUIRE q.queue_id IS UNIQUE;
CREATE CONSTRAINT skill_id_unique IF NOT EXISTS FOR (s:Skill) REQUIRE s.skill_id IS UNIQUE;
CREATE CONSTRAINT campaign_id_unique IF NOT EXISTS FOR (c:Campaign) REQUIRE c.campaign_id IS UNIQUE;
CREATE CONSTRAINT article_id_unique IF NOT EXISTS FOR (a:KnowledgeArticle) REQUIRE a.article_id IS UNIQUE;
CREATE CONSTRAINT intent_id_unique IF NOT EXISTS FOR (i:Intent) REQUIRE i.intent_id IS UNIQUE;
CREATE CONSTRAINT utterance_id_unique IF NOT EXISTS FOR (u:Utterance) REQUIRE u.utterance_id IS UNIQUE;
CREATE CONSTRAINT qa_score_id_unique IF NOT EXISTS FOR (q:QAScore) REQUIRE q.score_id IS UNIQUE;
CREATE CONSTRAINT dnc_phone_unique IF NOT EXISTS FOR (d:DNCEntry) REQUIRE d.phone_number IS UNIQUE;
CREATE CONSTRAINT consent_id_unique IF NOT EXISTS FOR (cr:ConsentRecord) REQUIRE cr.consent_id IS UNIQUE;
CREATE CONSTRAINT coaching_id_unique IF NOT EXISTS FOR (cs:CoachingSession) REQUIRE cs.session_id IS UNIQUE;

// Performance indexes
CREATE INDEX call_start_time IF NOT EXISTS FOR (c:Call) ON (c.started_at);
CREATE INDEX call_status IF NOT EXISTS FOR (c:Call) ON (c.status);
CREATE INDEX agent_availability IF NOT EXISTS FOR (a:Agent) ON (a.status, a.available_since);
CREATE INDEX campaign_status IF NOT EXISTS FOR (c:Campaign) ON (c.status, c.scheduled_start);
CREATE INDEX utterance_timestamp IF NOT EXISTS FOR (u:Utterance) ON (u.timestamp);
CREATE INDEX article_category IF NOT EXISTS FOR (a:KnowledgeArticle) ON (a.category, a.status);
CREATE INDEX qa_score_timestamp IF NOT EXISTS FOR (q:QAScore) ON (q.scored_at);

// Full-text indexes for search
CREATE FULLTEXT INDEX article_content_ft IF NOT EXISTS
  FOR (a:KnowledgeArticle) ON EACH [a.title, a.body, a.summary];
CREATE FULLTEXT INDEX utterance_text_ft IF NOT EXISTS
  FOR (u:Utterance) ON EACH [u.text];
```

### Node Definitions

```cypher
// Tenant — multi-tenancy root
// tenant_id (UUID), name, plan_tier (free|starter|professional|enterprise),
// data_residency (us-east|eu-west|ap-southeast|self-hosted), created_at, settings (Map)

// Agent
// agent_id (UUID), email, display_name, role (agent|supervisor|admin),
// status (available|on_call|wrap_up|offline|break), available_since (DateTime),
// max_concurrent (Int), timezone, created_at, deactivated_at (nullable)

// Caller
// caller_id (UUID), phone_number (E.164), email, display_name, preferred_lang (BCP 47),
// crm_contact_id, first_seen_at, last_contact_at, sentiment_trend (Float -1..1), metadata (Map)

// Call
// call_id (UUID), direction (inbound|outbound), channel (pstn|webrtc|sip),
// status (ringing|ai_handling|queued|agent_handling|on_hold|conferenced|wrap_up|completed|abandoned),
// started_at, answered_at, ended_at, duration_ms, ai_handle_ms, wait_time_ms,
// disposition (resolved_by_ai|resolved_by_agent|abandoned|voicemail|callback_scheduled),
// ai_containment (Bool), recording_url, pci_paused (Bool), summary,
// sip_call_id, stir_shaken (A|B|C|none)

// Queue
// queue_id (UUID), name, priority (1-10), sla_target_ms, max_wait_ms,
// overflow_action (voicemail|callback|overflow_queue), is_active (Bool)

// Skill
// skill_id (UUID), name, category (language|product|technical|compliance), description

// Intent
// intent_id (UUID), name, category (billing|support|sales|general),
// confidence_threshold (Float), requires_auth (Bool), typical_handle_ms

// Utterance — transcript segment
// utterance_id (UUID), speaker (caller|ai_agent|human_agent), text, timestamp,
// duration_ms, confidence (Float), sentiment (Float -1..1), language (BCP 47),
// entities (List<Map>), redacted_text

// KnowledgeArticle
// article_id (UUID), title, body, summary, category, status (draft|published|archived),
// version (Int), embedding_vector (List<Float>), created_at, updated_at,
// view_count, resolution_rate (Float)

// Campaign — outbound
// campaign_id (UUID), name, type (reminder|collection|survey|notification),
// status (draft|scheduled|running|paused|completed), scheduled_start, scheduled_end,
// call_window_start (Time), call_window_end (Time), total_contacts, completed_calls,
// success_rate (Float), tcpa_compliant (Bool), gdpr_compliant (Bool), script_template

// QAScore
// score_id (UUID), overall_score (0-100), sentiment_score (-1..1),
// compliance_score (0-100), resolution_score (0-100), empathy_score (0-100),
// scored_at, model_version, flags (List<String>), notes

// DNCEntry — Do-Not-Call
// phone_number (E.164), source (federal|state|internal|customer_request), added_at, expires_at

// ConsentRecord — immutable audit trail
// consent_id (UUID), consent_type (recording|outbound_contact|data_processing|marketing),
// granted (Bool), captured_at, capture_method (verbal_ivr|verbal_ai|web_form|api),
// evidence_url, regulation (tcpa|gdpr|ccpa|pipeda), revoked_at (nullable)

// CoachingSession
// session_id (UUID), status (scheduled|completed|cancelled), scheduled_at,
// completed_at, focus_areas (List<String>), supervisor_notes, outcome

// EscalationEvent
// event_id (UUID), reason (frustration_detected|complexity_threshold|customer_request|
// auth_required|compliance_risk), ai_confidence (Float), frustration_score (Float 0-1),
// context_summary, recommended_action, occurred_at

// AIDecisionLog — explainability/audit
// decision_id (UUID), decision_type (routing|escalation|response_selection|
// knowledge_retrieval|intent_classification), model_used, input_summary,
// reasoning, output, latency_ms, token_count, decided_at
```

### Relationship Definitions

```cypher
// --- Tenant membership ---
// (Agent)-[:BELONGS_TO]->(Tenant)
// (Queue)-[:BELONGS_TO]->(Tenant)
// (Campaign)-[:BELONGS_TO]->(Tenant)
// (KnowledgeArticle)-[:BELONGS_TO]->(Tenant)
// (Intent)-[:BELONGS_TO]->(Tenant)

// --- Agent skills and hierarchy ---
// (Agent)-[:HAS_SKILL {proficiency: Float, certified_at: DateTime, expires_at: DateTime}]->(Skill)
// (Agent)-[:MEMBER_OF {priority: Int, joined_at: DateTime}]->(Queue)
// (Agent)-[:REPORTS_TO]->(Agent)  -- org hierarchy for escalation traversal

// --- Queue configuration ---
// (Queue)-[:REQUIRES_SKILL {min_proficiency: Float}]->(Skill)
// (Queue)-[:OVERFLOWS_TO {after_ms: Int}]->(Queue)

// --- Call lifecycle ---
// (Caller)-[:MADE_CALL]->(Call)
// (Call)-[:ENTERED_QUEUE {entered_at: DateTime, position: Int}]->(Queue)
// (Call)-[:HANDLED_BY {started_at, ended_at, role: "primary"|"conferenced"|"supervisor_monitor"}]->(Agent)
// (Call)-[:DETECTED_INTENT {confidence: Float, detected_at: DateTime}]->(Intent)
// (Call)-[:CONTAINS_UTTERANCE {sequence: Int}]->(Utterance)
// (Call)-[:SCORED_BY]->(QAScore)
// (Call)-[:TRIGGERED_ESCALATION]->(EscalationEvent)
// (Call)-[:PART_OF_CAMPAIGN]->(Campaign)

// --- Knowledge graph ---
// (KnowledgeArticle)-[:RESOLVES_INTENT {effectiveness: Float}]->(Intent)
// (KnowledgeArticle)-[:RELATED_TO {similarity: Float}]->(KnowledgeArticle)
// (KnowledgeArticle)-[:CHILD_OF]->(KnowledgeArticle)  -- taxonomy hierarchy
// (Call)-[:REFERENCED_ARTICLE {surfaced_at: DateTime, was_helpful: Bool}]->(KnowledgeArticle)
// (Intent)-[:REQUIRES_SKILL]->(Skill)

// --- Consent and compliance ---
// (Caller)-[:GAVE_CONSENT]->(ConsentRecord)
// (ConsentRecord)-[:APPLIES_TO_CALL]->(Call)
// (Caller)-[:ON_DNC_LIST]->(DNCEntry)

// --- Coaching and QA ---
// (QAScore)-[:TRIGGERED_COACHING]->(CoachingSession)
// (CoachingSession)-[:COACH]->(Agent)     -- supervisor
// (CoachingSession)-[:COACHEE]->(Agent)   -- agent being coached
// (CoachingSession)-[:ABOUT_CALL]->(Call)

// --- AI decision audit trail ---
// (Call)-[:AI_DECISION]->(AIDecisionLog)
// (AIDecisionLog)-[:SELECTED_ARTICLE]->(KnowledgeArticle)
// (AIDecisionLog)-[:ROUTED_TO]->(Agent)
// (AIDecisionLog)-[:CLASSIFIED_AS]->(Intent)

// --- Caller history ---
// (Caller)-[:PREVIOUSLY_HANDLED_BY {last_call_id, last_contact: DateTime}]->(Agent)
// (Caller)-[:HAS_OPEN_ISSUE {issue_type, since: DateTime}]->(Intent)
```

### Example Queries

```cypher
// 1. Skill-based routing: find best available agent for a call
MATCH (call:Call {call_id: $callId})-[:DETECTED_INTENT]->(intent:Intent)-[:REQUIRES_SKILL]->(skill:Skill)
MATCH (agent:Agent)-[hs:HAS_SKILL]->(skill)
WHERE agent.status = 'available'
  AND hs.proficiency >= 0.7
  AND (hs.expires_at IS NULL OR hs.expires_at > datetime())
MATCH (agent)-[:BELONGS_TO]->(tenant:Tenant {tenant_id: $tenantId})
OPTIONAL MATCH (caller:Caller)-[:MADE_CALL]->(call)
OPTIONAL MATCH (caller)-[prev:PREVIOUSLY_HANDLED_BY]->(agent)
WITH agent, avg(hs.proficiency) + CASE WHEN prev IS NOT NULL THEN 0.2 ELSE 0.0 END AS score
RETURN agent.agent_id, agent.display_name, score ORDER BY score DESC LIMIT 5;

// 2. Escalation path: find nearest available supervisor
MATCH (agent:Agent {agent_id: $agentId})
MATCH path = (agent)-[:REPORTS_TO*1..3]->(sup:Agent)
WHERE sup.status = 'available'
RETURN [n IN nodes(path) | n.display_name] AS chain, sup.agent_id
ORDER BY length(path) LIMIT 1;

// 3. Knowledge gap detection: high-volume intents with no effective articles
MATCH (intent:Intent)-[:BELONGS_TO]->(t:Tenant {tenant_id: $tenantId})
OPTIONAL MATCH (a:KnowledgeArticle)-[r:RESOLVES_INTENT]->(intent)
WHERE a.status = 'published' AND r.effectiveness > 0.5
WITH intent, count(a) AS effective WHERE effective = 0
MATCH (call:Call)-[:DETECTED_INTENT]->(intent)
WHERE call.started_at > datetime() - duration('P30D')
RETURN intent.name, intent.category, count(call) AS monthly_volume
ORDER BY monthly_volume DESC;

// 4. AI decision audit trail for a single call
MATCH (call:Call {call_id: $callId})-[:AI_DECISION]->(d:AIDecisionLog)
OPTIONAL MATCH (d)-[:CLASSIFIED_AS]->(i:Intent)
OPTIONAL MATCH (d)-[:SELECTED_ARTICLE]->(a:KnowledgeArticle)
OPTIONAL MATCH (d)-[:ROUTED_TO]->(ag:Agent)
RETURN d.decision_type, d.reasoning, d.model_used, d.latency_ms,
       i.name AS intent, a.title AS article, ag.display_name AS routed_to
ORDER BY d.decided_at;
```

---

## Trade-offs

| Dimension | Strength/Weakness | Assessment |
| --- | --- | --- |
| Routing queries | Strength | Graph traversal for skill-based routing is natural -- no multi-table joins. Finding the best agent across skill, queue, availability, and caller history is a single Cypher query. |
| Explainability | Strength | AI decision audit trails follow edges from call to decision to intent/article/agent, rather than joining on foreign keys. |
| Knowledge management | Strength | Article taxonomies, intent-to-article mappings, and gap detection are graph-native pattern matches. |
| Caller context | Strength | Full caller history -- every call, agent, intent, outcome -- is a single traversal from the caller node. |
| Schema evolution | Strength | Adding node labels, relationship types, or properties needs no migrations. |
| Write throughput | Weakness | Thousands of concurrent calls producing utterances every few seconds may challenge Neo4j. Mitigate by buffering in Redis and batch-inserting, or storing utterances in a separate append-optimized store. |
| Analytics/OLAP | Weakness | AHT percentiles, hourly containment rates, and campaign funnels require scanning many nodes. Neo4j is not optimized for this. Project metrics to TimescaleDB or ClickHouse. |
| Transactions | Weakness | Complex multi-node atomic writes (call + utterances + decisions) are slower than equivalent relational batch inserts. |
| Ecosystem | Weakness | ORM support, migration tooling, and monitoring are less mature than PostgreSQL. Cypher expertise is less common than SQL. |
| Cost | Weakness | Neo4j Enterprise (required for clustering) has significant licensing costs. Community edition limits clustering and database count. |

---

## Scalability Considerations

**Data volume estimates (mid-size center, 500 agents, 50K calls/day):** ~50K Call nodes, ~500K Utterance nodes, ~200K AIDecisionLog nodes, ~50K QAScore nodes per day. At 365 days: ~18M Call nodes, ~180M Utterance nodes, ~900M relationships per year.

**Scaling strategy:**

1. **Shard by tenant.** Neo4j 5.x supports multiple databases per instance. Large tenants get dedicated databases; smaller tenants share with label-based filtering.
2. **Hot/warm/cold tiering.** Keep 90 days in the primary graph; archive older data to read-only instances or Parquet exports.
3. **Hybrid utterance storage.** Store raw utterances in an append-optimized store (Kafka + object storage). Keep only summary nodes in the graph with external pointers -- reduces graph size ~70%.
4. **Read replicas.** Neo4j causal clustering supports replicas for scaling read-heavy dashboard/search queries.
5. **Caching.** Cache agent-skill subgraphs and queue configs in Redis -- read on every routing decision but rarely mutated.

**Performance targets:**

| Query | Target | Notes |
| --- | --- | --- |
| Agent routing | < 50 ms | Traverses skill/queue subgraph with cached availability |
| Caller history | < 100 ms | Bounded by per-caller relationship count |
| Knowledge retrieval | < 80 ms | Traversal + optional vector similarity plugin |
| Call audit trail | < 200 ms | All decision nodes for a single call |
| Dashboard aggregations | N/A | Served from projected analytics store |

---

## Migration Path

### From Relational (Suggestions 1 or 3)

1. **Phase 1 -- Parallel read.** Deploy Neo4j alongside PostgreSQL. ETL core entities (agents, skills, queues, articles, intents) into the graph. Route routing-decision queries to Neo4j; PostgreSQL remains system of record.
2. **Phase 2 -- Dual write via CDC.** Use Debezium to stream changes from PostgreSQL to Neo4j. Neo4j serves relationship-heavy reads; PostgreSQL handles transactional writes and aggregation.
3. **Phase 3 -- Graph as primary for operations.** Migrate write paths for calls, intents, and agent assignments to Neo4j. PostgreSQL retains campaign management, billing, and analytics.
4. **Phase 4 -- Full graph (optional).** Migrate remaining entities if the team has sufficient expertise. Most organizations find Phase 3 hybrid optimal.

### Recommended Hybrid Architecture

```
                  +------------------+
 Inbound Call --> | Voice Pipeline   |
                  +--------+---------+
                           |
            +--------------+--------------+
            |                             |
  +---------v----------+     +-----------v-----------+
  |   Neo4j (Graph)    |     | TimescaleDB / CK      |
  |                    |     | (Time-Series/Columnar) |
  | - Agent routing    |     | - Utterance storage    |
  | - Escalation paths |     | - Call metrics / AHT   |
  | - Knowledge graph  |     | - QA score trends      |
  | - Caller history   |     | - Dashboard aggs       |
  | - AI decision log  |     | - Campaign analytics   |
  | - Consent graph    |     |                        |
  +--------------------+     +------------------------+
            |                             |
            +-------------+---------------+
                          |
                 +--------v--------+
                 |   Redis         |
                 | - Session state |
                 | - Agent status  |
                 | - Queue depth   |
                 +-----------------+
```

Neo4j serves as the "brain" for intelligent routing and explainability where relationships matter most. The time-series store serves as the "memory" for historical analysis and OLAP dashboards. Redis provides the real-time state layer for sub-second routing decisions. This hybrid captures the best of the graph paradigm while mitigating its weaknesses in write throughput and aggregation.
