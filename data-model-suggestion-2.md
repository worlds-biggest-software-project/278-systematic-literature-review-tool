# Data Model Suggestion 2: Event-Sourced / Audit-First (CQRS)

> Project: Systematic Literature Review Tool · Created: 2026-05-25

## Philosophy

This model treats every action in the systematic review process as an immutable event. Rather than storing the "current state" of a review in mutable rows, the system records a chronological stream of events (reference imported, screening decision made, extraction value entered, RoB judgement assigned) and derives the current state by replaying or projecting those events into materialised read models.

This approach is inspired by the way ASReview stores screening decisions as an ordered log with model state snapshots, and by the Cochrane requirement that systematic reviews be fully reproducible. In clinical and regulatory systematic reviews, the ability to answer "what was the state of this review on a specific date?" is not a nice-to-have — it is a methodological requirement. Event sourcing makes temporal queries trivial because the full history is the source of truth.

The architecture follows CQRS (Command Query Responsibility Segregation): write operations append events to the event store, while read operations query pre-built materialised views optimised for each use case (screening queue, PRISMA counts, GRADE tables). This separation allows the read side to be tuned independently for the specific query patterns of systematic review workflows.

**Best for:** Regulatory and clinical review environments where complete audit trails, temporal reproducibility, and the ability to replay review history are non-negotiable requirements.

**Trade-offs:**
- (+) Complete, immutable audit trail built into the architecture — not bolted on
- (+) Temporal queries are trivial: reconstruct the review state at any point in time
- (+) Full reproducibility of AI screening decisions and model retraining history
- (+) Natural fit for active learning workflows where each decision changes the model state
- (+) Event replay enables debugging, compliance audits, and methodology validation
- (-) Higher complexity: developers must understand event sourcing and projection patterns
- (-) Eventual consistency between event store and read models requires careful design
- (-) Storage requirements are higher (events are never deleted)
- (-) Simple CRUD operations require more code than traditional relational models
- (-) Schema evolution for events requires careful versioning (upcasting)

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| PRISMA 2020 | Flow diagram counts are materialised views computed from screening events; always consistent with actual decisions |
| Cochrane RoB 2 | RoB assessments are event streams; domain judgements derived from signalling question answer events |
| GRADE | Evidence certainty projections built from extraction and RoB events |
| PICO | PICO elements stored as review configuration events |
| PROSPERO | Protocol registration tracked as a lifecycle event |
| ASReview | Event log pattern inspired by ASReview's state storage (results.db with ordered decisions) |
| OCSF | Event schema influenced by Open Cybersecurity Schema Framework structured event patterns |

---

## Event Store (Core)

```sql
-- ============================================================
-- EVENT STORE — THE SINGLE SOURCE OF TRUTH
-- ============================================================

-- All state changes are recorded as immutable events.
-- The event_data JSONB column contains the full event payload.
-- Events are never updated or deleted.

CREATE TABLE event_store (
    event_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_id       UUID NOT NULL,          -- aggregate root ID (review, reference, etc.)
    stream_type     VARCHAR(100) NOT NULL,  -- 'review', 'reference', 'screening', 'extraction', 'rob', 'grade'
    event_type      VARCHAR(100) NOT NULL,  -- e.g. 'ReferenceImported', 'ScreeningDecisionMade'
    event_version   INTEGER NOT NULL,       -- monotonically increasing per stream
    event_data      JSONB NOT NULL,         -- full event payload
    metadata        JSONB,                  -- actor, IP, user_agent, correlation_id
    actor_id        UUID NOT NULL,          -- user who triggered the event
    organisation_id UUID NOT NULL,          -- tenant partition key
    review_id       UUID NOT NULL,          -- review partition key (for efficient queries)
    occurred_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (stream_id, event_version)       -- optimistic concurrency control
);

-- Immutability enforced at the database level
REVOKE UPDATE, DELETE ON event_store FROM PUBLIC;

-- Partition by review for large-scale deployments
-- CREATE TABLE event_store PARTITION BY HASH (review_id);

CREATE INDEX idx_event_stream ON event_store(stream_id, event_version);
CREATE INDEX idx_event_review ON event_store(review_id, occurred_at);
CREATE INDEX idx_event_type ON event_store(event_type, occurred_at);
CREATE INDEX idx_event_org ON event_store(organisation_id);
CREATE INDEX idx_event_actor ON event_store(actor_id, occurred_at);

-- ============================================================
-- SNAPSHOT STORE (for performance — avoids replaying all events)
-- ============================================================

CREATE TABLE event_snapshot (
    snapshot_id     UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_id       UUID NOT NULL,
    stream_type     VARCHAR(100) NOT NULL,
    snapshot_version INTEGER NOT NULL,      -- event_version at time of snapshot
    snapshot_data   JSONB NOT NULL,         -- serialised aggregate state
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (stream_id, snapshot_version)
);

CREATE INDEX idx_snapshot_stream ON event_snapshot(stream_id, snapshot_version DESC);
```

## Event Type Catalogue

```sql
-- ============================================================
-- EVENT TYPE REGISTRY (documentation, not enforcement)
-- ============================================================

-- This table documents the event types used in the system.
-- It is NOT used for validation (events are schemaless JSONB)
-- but serves as a living data dictionary.

CREATE TABLE event_type_registry (
    event_type      VARCHAR(100) PRIMARY KEY,
    stream_type     VARCHAR(100) NOT NULL,
    description     TEXT NOT NULL,
    schema_version  INTEGER NOT NULL DEFAULT 1,
    example_payload JSONB,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Example event types and their payloads:

/*
Event Type: ReviewCreated
Stream: review
Payload: {
    "title": "Efficacy of CBT for anxiety disorders",
    "review_type": "systematic_review",
    "organisation_id": "uuid",
    "created_by": "uuid",
    "protocol": {
        "prospero_id": "CRD42026000123",
        "pico": {
            "population": "Adults with generalised anxiety disorder",
            "intervention": "Cognitive behavioural therapy",
            "comparator": "Waitlist control or treatment as usual",
            "outcome": "Anxiety symptom severity (GAD-7 or equivalent)"
        }
    }
}

Event Type: SearchExecuted
Stream: review
Payload: {
    "source": "PubMed",
    "query_string": "(CBT OR \"cognitive behav*\") AND (anxiety)",
    "results_count": 2847,
    "executed_by": "uuid"
}

Event Type: ReferencesImported
Stream: review
Payload: {
    "import_batch_id": "uuid",
    "source": "PubMed",
    "file_format": "nbib",
    "records_total": 2847,
    "records_imported": 2831,
    "records_duplicate": 16,
    "references": ["uuid1", "uuid2", ...]
}

Event Type: ReferenceCreated
Stream: reference
Payload: {
    "doi": "10.1001/jama.2026.1234",
    "pmid": "38456789",
    "title": "A randomised controlled trial of...",
    "authors": [{"full_name": "Smith J", "orcid": "0000-0001-2345-6789"}],
    "journal": "JAMA",
    "publication_year": 2025,
    "language_code": "en"
}

Event Type: DuplicateDetected
Stream: reference
Payload: {
    "duplicate_of": "uuid-master",
    "detection_method": "semantic",  -- 'doi_match', 'pmid_match', 'semantic', 'manual'
    "confidence": 0.97
}

Event Type: ScreeningDecisionMade
Stream: screening
Payload: {
    "reference_id": "uuid",
    "phase": "title_abstract",
    "decision": "include",
    "exclusion_reason": null,
    "reviewer_id": "uuid",
    "time_spent_seconds": 45
}

Event Type: ScreeningConflictDetected
Stream: screening
Payload: {
    "reference_id": "uuid",
    "phase": "title_abstract",
    "decisions": [
        {"reviewer_id": "uuid1", "decision": "include"},
        {"reviewer_id": "uuid2", "decision": "exclude"}
    ]
}

Event Type: ScreeningConflictResolved
Stream: screening
Payload: {
    "reference_id": "uuid",
    "resolved_by": "uuid",
    "resolution": "include",
    "method": "third_reviewer"
}

Event Type: AIScreeningPredictionGenerated
Stream: screening
Payload: {
    "reference_id": "uuid",
    "model_version": "active_learning_v3.2",
    "relevance_score": 0.8734,
    "training_labels_count": 156,
    "features_hash": "sha256:abc123"
}

Event Type: ExtractionValueRecorded
Stream: extraction
Payload: {
    "reference_id": "uuid",
    "template_id": "uuid",
    "field_name": "sample_size",
    "field_type": "number",
    "value": 245,
    "pico_element": "population",
    "ai_extracted": true,
    "ai_confidence": 0.92,
    "source_page": 3,
    "source_quote": "245 participants were randomised..."
}

Event Type: RoBAnswerRecorded
Stream: rob
Payload: {
    "reference_id": "uuid",
    "template": "rob2_individual_rct",
    "domain": 1,
    "domain_name": "Bias arising from the randomization process",
    "question_number": "1.1",
    "answer": "yes",
    "support_text": "Central computerised randomisation was used"
}

Event Type: RoBDomainJudged
Stream: rob
Payload: {
    "reference_id": "uuid",
    "domain": 1,
    "judgement": "low",
    "assessor_id": "uuid"
}

Event Type: GRADEAssessmentUpdated
Stream: grade
Payload: {
    "outcome_name": "Anxiety symptoms (GAD-7)",
    "outcome_type": "continuous",
    "certainty": "moderate",
    "downgraded_for": ["imprecision"],
    "num_studies": 12,
    "effect_estimate": {"SMD": -0.45, "ci_lower": -0.67, "ci_upper": -0.23}
}
*/
```

## Read Models (Materialised Views)

```sql
-- ============================================================
-- READ MODELS — MATERIALISED FROM EVENTS
-- ============================================================

-- These tables are projections of the event store, built
-- by event handlers that process each event as it's appended.
-- They can be rebuilt at any time by replaying all events.

-- ============================================================
-- ORGANISATION & USER (read model)
-- ============================================================

CREATE TABLE rm_organisation (
    id              UUID PRIMARY KEY,
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    subscription_tier VARCHAR(50) DEFAULT 'free',
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE rm_user (
    id              UUID PRIMARY KEY,
    email           VARCHAR(255) NOT NULL UNIQUE,
    display_name    VARCHAR(255) NOT NULL,
    orcid_id        VARCHAR(20),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- REVIEW SUMMARY (read model)
-- ============================================================

CREATE TABLE rm_review (
    id              UUID PRIMARY KEY,
    organisation_id UUID NOT NULL,
    title           VARCHAR(500) NOT NULL,
    review_type     VARCHAR(50) NOT NULL,
    status          VARCHAR(50) NOT NULL,
    prospero_id     VARCHAR(50),
    total_references INTEGER DEFAULT 0,
    total_duplicates INTEGER DEFAULT 0,
    screening_ta_total INTEGER DEFAULT 0,
    screening_ta_included INTEGER DEFAULT 0,
    screening_ta_excluded INTEGER DEFAULT 0,
    screening_ta_pending INTEGER DEFAULT 0,
    screening_ft_total INTEGER DEFAULT 0,
    screening_ft_included INTEGER DEFAULT 0,
    screening_ft_excluded INTEGER DEFAULT 0,
    screening_ft_pending INTEGER DEFAULT 0,
    extraction_completed INTEGER DEFAULT 0,
    rob_completed   INTEGER DEFAULT 0,
    team_member_count INTEGER DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rm_review_org ON rm_review(organisation_id);

-- ============================================================
-- REFERENCE (read model)
-- ============================================================

CREATE TABLE rm_reference (
    id              UUID PRIMARY KEY,
    review_id       UUID NOT NULL,
    doi             VARCHAR(255),
    pmid            VARCHAR(20),
    title           TEXT NOT NULL,
    abstract        TEXT,
    authors         TEXT,
    journal         VARCHAR(500),
    publication_year INTEGER,
    language_code   VARCHAR(10),
    document_type   VARCHAR(50),
    is_duplicate    BOOLEAN DEFAULT FALSE,
    master_ref_id   UUID,
    -- Screening state (derived from events)
    ta_screening_status VARCHAR(20) DEFAULT 'pending',
        -- 'pending', 'included', 'excluded', 'conflicted'
    ft_screening_status VARCHAR(20) DEFAULT 'pending',
    ta_decisions    JSONB DEFAULT '[]',     -- array of {reviewer_id, decision, decided_at}
    ft_decisions    JSONB DEFAULT '[]',
    ai_relevance_score NUMERIC(5,4),
    -- Extraction state
    extraction_status VARCHAR(20) DEFAULT 'not_started',
    -- RoB state
    rob_overall     VARCHAR(20),
    -- Timestamps
    imported_at     TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rm_reference_review ON rm_reference(review_id);
CREATE INDEX idx_rm_reference_ta ON rm_reference(review_id, ta_screening_status);
CREATE INDEX idx_rm_reference_ft ON rm_reference(review_id, ft_screening_status);
CREATE INDEX idx_rm_reference_ai ON rm_reference(review_id, ai_relevance_score DESC NULLS LAST);

-- ============================================================
-- SCREENING QUEUE (read model — optimised for active learning)
-- ============================================================

CREATE TABLE rm_screening_queue (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    review_id       UUID NOT NULL,
    reference_id    UUID NOT NULL,
    phase           VARCHAR(20) NOT NULL,   -- 'title_abstract', 'full_text'
    -- Pre-computed display data
    title           TEXT NOT NULL,
    abstract        TEXT,
    authors         TEXT,
    journal         VARCHAR(500),
    -- Priority for active learning
    ai_relevance_score NUMERIC(5,4),
    model_uncertainty NUMERIC(5,4),         -- higher = model less confident = more valuable to label
    priority_rank   INTEGER,                -- computed rank for reviewer queue
    -- Assignment
    assigned_to     UUID,
    assigned_at     TIMESTAMPTZ,
    -- State
    decisions_count INTEGER DEFAULT 0,
    required_decisions INTEGER DEFAULT 2,
    is_resolved     BOOLEAN DEFAULT FALSE,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rm_queue_review ON rm_screening_queue(review_id, phase, is_resolved, priority_rank);
CREATE INDEX idx_rm_queue_assigned ON rm_screening_queue(assigned_to, is_resolved);

-- ============================================================
-- PRISMA FLOW (read model — auto-calculated from events)
-- ============================================================

CREATE TABLE rm_prisma_flow (
    review_id       UUID PRIMARY KEY,
    -- These counts are always derived from events — never manually set
    records_from_databases INTEGER DEFAULT 0,
    records_from_registers INTEGER DEFAULT 0,
    records_from_other INTEGER DEFAULT 0,
    duplicates_removed INTEGER DEFAULT 0,
    records_screened INTEGER DEFAULT 0,
    records_excluded_screening INTEGER DEFAULT 0,
    reports_sought_for_retrieval INTEGER DEFAULT 0,
    reports_not_retrieved INTEGER DEFAULT 0,
    reports_assessed_for_eligibility INTEGER DEFAULT 0,
    reports_excluded_with_reasons INTEGER DEFAULT 0,
    studies_included INTEGER DEFAULT 0,
    reports_included INTEGER DEFAULT 0,
    -- Exclusion reasons breakdown
    exclusion_reasons JSONB DEFAULT '{}',   -- {"reason_code": count, ...}
    last_calculated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- EXTRACTION DATA (read model)
-- ============================================================

CREATE TABLE rm_extraction (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    review_id       UUID NOT NULL,
    reference_id    UUID NOT NULL,
    template_id     UUID NOT NULL,
    extractor_id    UUID NOT NULL,
    status          VARCHAR(50) DEFAULT 'in_progress',
    -- Denormalised extraction data as JSONB for fast reads
    field_values    JSONB NOT NULL DEFAULT '{}',
    /*  Example:
        {
            "sample_size": {"value": 245, "type": "number", "ai_extracted": true, "verified": true},
            "intervention": {"value": "CBT 12 sessions", "type": "text", "pico": "intervention"},
            "country": {"value": "UK", "type": "select"}
        }
    */
    completed_at    TIMESTAMPTZ,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rm_extraction_review ON rm_extraction(review_id);
CREATE INDEX idx_rm_extraction_ref ON rm_extraction(reference_id);

-- ============================================================
-- ROB SUMMARY (read model)
-- ============================================================

CREATE TABLE rm_rob_summary (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    review_id       UUID NOT NULL,
    reference_id    UUID NOT NULL,
    assessor_id     UUID NOT NULL,
    template_name   VARCHAR(255) NOT NULL,
    domain_judgements JSONB NOT NULL DEFAULT '{}',
    /*  Example:
        {
            "1": {"name": "Randomization process", "judgement": "low"},
            "2": {"name": "Deviations from interventions", "judgement": "some_concerns"},
            "3": {"name": "Missing outcome data", "judgement": "low"},
            "4": {"name": "Measurement of the outcome", "judgement": "low"},
            "5": {"name": "Selection of the reported result", "judgement": "low"}
        }
    */
    overall_judgement VARCHAR(20),
    status          VARCHAR(50) DEFAULT 'in_progress',
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rm_rob_review ON rm_rob_summary(review_id);

-- ============================================================
-- GRADE SUMMARY (read model)
-- ============================================================

CREATE TABLE rm_grade_summary (
    review_id       UUID NOT NULL,
    outcome_name    VARCHAR(255) NOT NULL,
    outcome_type    VARCHAR(50) NOT NULL,
    importance      INTEGER,
    overall_certainty VARCHAR(20),
    domains         JSONB NOT NULL DEFAULT '{}',
    /*  Example:
        {
            "risk_of_bias": "not_serious",
            "inconsistency": "not_serious",
            "indirectness": "not_serious",
            "imprecision": "serious",
            "publication_bias": "undetected"
        }
    */
    effect_estimate JSONB,
    /*  Example:
        {
            "num_studies": 12,
            "num_participants": 1456,
            "relative": {"type": "SMD", "estimate": -0.45, "ci": [-0.67, -0.23]},
            "absolute": null
        }
    */
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (review_id, outcome_name)
);
```

## Event Processing Infrastructure

```sql
-- ============================================================
-- EVENT PROCESSING INFRASTRUCTURE
-- ============================================================

-- Tracks which projections have processed which events
CREATE TABLE projection_checkpoint (
    projection_name VARCHAR(100) PRIMARY KEY,
    last_event_id   UUID NOT NULL,
    last_event_version INTEGER NOT NULL,
    processed_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Dead letter queue for events that failed to project
CREATE TABLE projection_dead_letter (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    projection_name VARCHAR(100) NOT NULL,
    event_id        UUID NOT NULL REFERENCES event_store(event_id),
    error_message   TEXT NOT NULL,
    retry_count     INTEGER DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- CONFIGURATION TABLES (not event-sourced — reference data)
-- ============================================================

CREATE TABLE search_source (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    source_type     VARCHAR(50) NOT NULL,
    api_base_url    VARCHAR(500),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE rob_template (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    version         VARCHAR(20) NOT NULL,
    domains         JSONB NOT NULL,
    /*  Example:
        [
            {
                "number": 1,
                "name": "Bias arising from the randomization process",
                "questions": [
                    {"number": "1.1", "text": "Was the allocation sequence random?"},
                    {"number": "1.2", "text": "Was the allocation sequence concealed...?"}
                ]
            }
        ]
    */
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE extraction_template (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    review_id       UUID,
    name            VARCHAR(255) NOT NULL,
    fields          JSONB NOT NULL,
    /*  Example:
        [
            {"name": "sample_size", "label": "Sample Size", "type": "number", "pico": "population", "required": true},
            {"name": "intervention_desc", "label": "Intervention Description", "type": "textarea", "pico": "intervention"},
            {"name": "outcome_measure", "label": "Outcome Measure", "type": "select", "options": ["GAD-7", "HADS", "BAI"]}
        ]
    */
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE exclusion_reason_config (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    review_id       UUID NOT NULL,
    code            VARCHAR(50) NOT NULL,
    label           VARCHAR(255) NOT NULL,
    phase           VARCHAR(50) NOT NULL,
    sort_order      INTEGER DEFAULT 0,
    UNIQUE (review_id, code)
);
```

## Example Temporal Queries

```sql
-- ============================================================
-- EXAMPLE QUERIES: THE POWER OF EVENT SOURCING
-- ============================================================

-- 1. Reconstruct the state of screening at a specific date
-- "What were the screening decisions as of 2026-03-15?"
SELECT
    (event_data->>'reference_id')::UUID AS reference_id,
    event_data->>'decision' AS decision,
    event_data->>'phase' AS phase,
    actor_id AS reviewer_id,
    occurred_at
FROM event_store
WHERE review_id = 'review-uuid'
  AND event_type = 'ScreeningDecisionMade'
  AND occurred_at <= '2026-03-15T23:59:59Z'
ORDER BY occurred_at;

-- 2. Active learning performance over time
-- "How did the AI model's accuracy improve as more labels were added?"
SELECT
    (event_data->>'training_labels_count')::INTEGER AS labels,
    AVG((event_data->>'relevance_score')::NUMERIC) AS avg_score,
    event_data->>'model_version' AS model,
    DATE_TRUNC('day', occurred_at) AS day
FROM event_store
WHERE review_id = 'review-uuid'
  AND event_type = 'AIScreeningPredictionGenerated'
GROUP BY labels, model, day
ORDER BY labels;

-- 3. Reviewer workload and throughput
-- "How many decisions did each reviewer make per day?"
SELECT
    actor_id,
    DATE_TRUNC('day', occurred_at) AS day,
    COUNT(*) AS decisions,
    AVG(EXTRACT(EPOCH FROM
        (occurred_at - LAG(occurred_at) OVER (PARTITION BY actor_id ORDER BY occurred_at))
    )) AS avg_seconds_between_decisions
FROM event_store
WHERE review_id = 'review-uuid'
  AND event_type = 'ScreeningDecisionMade'
GROUP BY actor_id, day
ORDER BY day;

-- 4. Conflict detection audit
-- "Show all references where reviewers initially disagreed"
SELECT
    (event_data->>'reference_id')::UUID AS reference_id,
    event_data->'decisions' AS conflicting_decisions,
    occurred_at AS detected_at
FROM event_store
WHERE review_id = 'review-uuid'
  AND event_type = 'ScreeningConflictDetected'
ORDER BY occurred_at;

-- 5. Full event timeline for a single reference
-- "Show everything that happened to reference X"
SELECT
    event_type,
    event_data,
    actor_id,
    occurred_at
FROM event_store
WHERE stream_id = 'reference-uuid'
ORDER BY event_version;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Store (write side) | 2 | Event store + snapshots |
| Event Infrastructure | 3 | Type registry, checkpoint, dead letter queue |
| Read Models — Core | 3 | Organisation, user, review summary |
| Read Models — References | 2 | Reference list + screening queue |
| Read Models — PRISMA | 1 | Auto-calculated flow diagram |
| Read Models — Extraction | 1 | Denormalised extraction data |
| Read Models — RoB | 1 | Domain judgement summary |
| Read Models — GRADE | 1 | Summary of Findings projection |
| Configuration (reference data) | 4 | Search sources, RoB templates, extraction templates, exclusion reasons |
| **Total** | **18** | Read models are rebuildable from events |

---

## Key Design Decisions

1. **Single event store table** — all events across all aggregate types live in one table, partitioned by `review_id`. This simplifies backup, replication, and cross-aggregate queries (e.g. "show me everything that happened in this review").

2. **JSONB event payloads** — events are schemaless by design. The `event_type_registry` documents expected structures but does not enforce them, allowing event schema evolution without database migrations.

3. **Optimistic concurrency via `(stream_id, event_version)` uniqueness** — prevents race conditions when two reviewers try to modify the same aggregate simultaneously.

4. **Immutability enforced at the database level** — `REVOKE UPDATE, DELETE ON event_store FROM PUBLIC` ensures events cannot be tampered with, even by application bugs.

5. **Read models are disposable and rebuildable** — all `rm_*` tables can be dropped and rebuilt by replaying the event store. This enables schema changes to read models without data migration.

6. **Screening queue as a dedicated read model** — optimised for the active learning loop with pre-computed priority ranks, model uncertainty scores, and assignment tracking.

7. **Denormalised extraction data in JSONB** — the read model stores extraction field values as a single JSONB object per record, enabling fast reads for summary tables and evidence synthesis without joining through template/field/value hierarchies.

8. **Snapshot store for performance** — long-lived aggregates (reviews with 50,000+ references) can use periodic snapshots to avoid replaying the entire event history.

9. **Configuration tables are not event-sourced** — reference data like RoB templates, search sources, and exclusion reason catalogues are stored in traditional relational tables because they change infrequently and don't benefit from event history.

10. **Projection checkpoints and dead letter queue** — production infrastructure for reliable event processing with exactly-once semantics and error recovery.
