# Data Model Suggestion 3: Hybrid Relational + JSONB

> Project: Systematic Literature Review Tool · Created: 2026-05-25

## Philosophy

This model uses a pragmatic hybrid approach: core entities and relationships are modelled in normalised relational tables with foreign keys, while variable-structure data — extraction fields, RoB signalling question responses, GRADE domain assessments, and domain-specific metadata — is stored in PostgreSQL JSONB columns. The result is a schema that is both structurally sound and highly flexible.

The motivation comes from the observation that systematic review tools must accommodate wildly different review types. A Cochrane clinical review uses RoB 2 with five fixed domains, but an environmental review uses CEE critical appraisal with different criteria. A drug efficacy review extracts sample size, dosage, and adverse events; a policy review extracts study design, setting, and stakeholder perspectives. Rather than creating separate tables for every possible extraction field or bias assessment structure, this model pushes variable data into validated JSONB columns while keeping the structural skeleton relational.

This pattern is widely used in modern SaaS platforms (Notion, Airtable, Linear) where the core data model is relational but user-defined fields live in JSONB. PostgreSQL's JSONB operators, GIN indexes, and containment queries make this approach performant even at scale.

**Best for:** Rapid MVP development and platforms that need to support multiple review methodologies (Cochrane, CEE, JBI, Campbell) without schema changes for each.

**Trade-offs:**
- (+) Dramatically fewer tables (~22 vs ~38) while retaining relational integrity for core entities
- (+) Extraction templates and RoB frameworks can be changed at runtime without migrations
- (+) Supports multiple review methodologies without schema modifications
- (+) JSONB GIN indexes provide efficient querying on variable-structure data
- (+) Faster to develop: less ORM mapping, fewer migration files
- (-) JSONB fields bypass column-level type checking — validation must be in the application layer
- (-) Complex aggregations across JSONB fields are slower than native column aggregations
- (-) JSON schema evolution requires careful application-level versioning
- (-) Harder to enforce referential integrity within JSONB structures
- (-) Reporting queries on JSONB fields are more verbose than simple column SELECTs

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| PRISMA 2020 | Checklist stored as JSONB array with 27 items; flow diagram derived from screening records |
| Cochrane RoB 2 | RoB template stored as JSONB schema; assessments store answers and judgements in JSONB |
| GRADE | Evidence profile stored as JSONB with structured domain assessments and effect estimates |
| PICO | PICO elements stored as a JSONB object on the review question |
| RIS/BibTeX/NBIB | Citation metadata normalised into relational columns; raw import preserved in JSONB |
| MeSH | MeSH terms stored as JSONB array on references, with GIN index for containment queries |
| OpenAlex/Semantic Scholar | External API metadata cached in JSONB `external_metadata` field |
| ISO 639-1 / ISO 3166-1 | Language and country codes as VARCHAR columns on relational tables |

---

## Organisation & Users

```sql
-- ============================================================
-- ORGANISATION & USERS
-- ============================================================

CREATE TABLE organisation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    settings        JSONB DEFAULT '{}',
    /*  Example:
        {
            "subscription_tier": "team",
            "default_screening_reviewers": 2,
            "ai_features_enabled": true,
            "sso_provider": "oidc",
            "sso_config": {"issuer": "https://idp.university.edu", "client_id": "..."}
        }
    */
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE app_user (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email           VARCHAR(255) NOT NULL UNIQUE,
    display_name    VARCHAR(255) NOT NULL,
    orcid_id        VARCHAR(20),
    password_hash   VARCHAR(255),
    auth_provider   VARCHAR(50) DEFAULT 'local',
    profile         JSONB DEFAULT '{}',
    /*  Example:
        {
            "institution": "University of Oxford",
            "department": "Centre for Evidence-Based Medicine",
            "country_code": "GB",
            "expertise": ["clinical_trials", "meta_analysis"],
            "notification_preferences": {"email": true, "in_app": true}
        }
    */
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE organisation_member (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    user_id         UUID NOT NULL REFERENCES app_user(id),
    role            VARCHAR(50) NOT NULL DEFAULT 'member',
    permissions     JSONB DEFAULT '[]',
    /*  Example: ["manage_reviews", "manage_members", "view_billing"]  */
    UNIQUE (organisation_id, user_id)
);

CREATE INDEX idx_org_member_org ON organisation_member(organisation_id);
```

## Review & Protocol

```sql
-- ============================================================
-- REVIEW PROJECT
-- ============================================================

CREATE TABLE review (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    title           VARCHAR(500) NOT NULL,
    description     TEXT,
    review_type     VARCHAR(50) NOT NULL DEFAULT 'systematic_review',
    status          VARCHAR(50) NOT NULL DEFAULT 'protocol',
    created_by      UUID NOT NULL REFERENCES app_user(id),
    -- Protocol (all protocol fields in one JSONB column)
    protocol        JSONB DEFAULT '{}',
    /*  Example:
        {
            "prospero_id": "CRD42026000123",
            "registration_date": "2026-01-15",
            "questions": [
                {
                    "id": "q1",
                    "text": "Is CBT effective for GAD in adults?",
                    "pico": {
                        "population": "Adults ≥18 with GAD diagnosis",
                        "intervention": "Cognitive behavioural therapy (individual or group)",
                        "comparator": "Waitlist, TAU, or active control",
                        "outcome": "GAD-7 score change"
                    }
                }
            ],
            "eligibility": {
                "inclusion": ["RCTs", "Adults ≥18", "GAD diagnosis per DSM/ICD"],
                "exclusion": ["Case reports", "Non-English language", "Conference abstracts only"]
            },
            "methodology": "Cochrane Handbook Chapter 8 for RoB; GRADE for evidence certainty"
        }
    */
    -- PRISMA checklist (27 items tracked as JSONB)
    prisma_checklist JSONB DEFAULT '[]',
    /*  Example:
        [
            {"item": 1, "section": "Title", "topic": "Title", "completed": true, "location": "Page 1"},
            {"item": 2, "section": "Abstract", "topic": "Abstract", "completed": false, "location": null}
        ]
    */
    -- Review configuration
    config          JSONB DEFAULT '{}',
    /*  Example:
        {
            "screening": {
                "phases": ["title_abstract", "full_text"],
                "required_reviewers": 2,
                "conflict_resolution": "third_reviewer"
            },
            "extraction_template_id": "uuid",
            "rob_template_id": "uuid",
            "living_review": {
                "enabled": false,
                "interval_days": 30,
                "auto_screen": false
            },
            "exclusion_reasons": [
                {"code": "WRONG_POP", "label": "Wrong population", "phase": "both"},
                {"code": "WRONG_INT", "label": "Wrong intervention", "phase": "both"},
                {"code": "WRONG_DESIGN", "label": "Wrong study design", "phase": "title_abstract"},
                {"code": "NO_FULLTEXT", "label": "Full text not available", "phase": "full_text"}
            ]
        }
    */
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_review_org ON review(organisation_id);

CREATE TABLE review_team_member (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    review_id       UUID NOT NULL REFERENCES review(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES app_user(id),
    role            VARCHAR(50) NOT NULL DEFAULT 'reviewer',
    UNIQUE (review_id, user_id)
);
```

## References

```sql
-- ============================================================
-- REFERENCES
-- ============================================================

CREATE TABLE reference (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    review_id       UUID NOT NULL REFERENCES review(id) ON DELETE CASCADE,
    -- Core bibliographic fields (relational for fast filtering)
    doi             VARCHAR(255),
    pmid            VARCHAR(20),
    title           TEXT NOT NULL,
    abstract        TEXT,
    authors         TEXT,
    journal         VARCHAR(500),
    publication_year INTEGER,
    language_code   VARCHAR(10),
    document_type   VARCHAR(50),
    -- Extended metadata in JSONB (varies by source)
    metadata        JSONB DEFAULT '{}',
    /*  Example:
        {
            "pmcid": "PMC12345678",
            "openalex_id": "W1234567890",
            "arxiv_id": "2401.12345",
            "volume": "386",
            "issue": "12",
            "pages": "1123-1134",
            "publisher": "Elsevier",
            "publication_date": "2026-03-15",
            "url": "https://doi.org/10.1016/...",
            "authors_structured": [
                {"full_name": "Smith J", "family": "Smith", "given": "Jane", "orcid": "0000-0001-2345-6789", "affiliation": "University of Oxford", "is_corresponding": true},
                {"full_name": "Jones B", "family": "Jones", "given": "Bob"}
            ],
            "keywords": ["cognitive behavioural therapy", "anxiety", "randomised controlled trial"],
            "mesh_terms": [
                {"ui": "D060825", "term": "Cognitive Behavioral Therapy", "major": true},
                {"ui": "D001008", "term": "Anxiety Disorders", "major": false}
            ],
            "fwci": 2.34,
            "cited_by_count": 45
        }
    */
    -- Import tracking
    import_source   VARCHAR(100),
    import_batch_id UUID,
    import_format   VARCHAR(20),
    raw_import      JSONB,                  -- original RIS/BibTeX record preserved as-is
    -- Deduplication
    dedup_cluster_id UUID,
    is_duplicate    BOOLEAN DEFAULT FALSE,
    master_ref_id   UUID REFERENCES reference(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_reference_review ON reference(review_id);
CREATE INDEX idx_reference_doi ON reference(doi) WHERE doi IS NOT NULL;
CREATE INDEX idx_reference_pmid ON reference(pmid) WHERE pmid IS NOT NULL;
CREATE INDEX idx_reference_year ON reference(review_id, publication_year);
CREATE INDEX idx_reference_dedup ON reference(dedup_cluster_id) WHERE dedup_cluster_id IS NOT NULL;

-- GIN index for JSONB containment queries on metadata
CREATE INDEX idx_reference_metadata ON reference USING GIN (metadata jsonb_path_ops);

-- Example query: find references with a specific MeSH term
-- SELECT * FROM reference
-- WHERE review_id = 'uuid'
--   AND metadata @> '{"mesh_terms": [{"ui": "D060825"}]}';

-- Example query: find references by a specific author ORCID
-- SELECT * FROM reference
-- WHERE review_id = 'uuid'
--   AND metadata @> '{"authors_structured": [{"orcid": "0000-0001-2345-6789"}]}';
```

## Screening

```sql
-- ============================================================
-- SCREENING
-- ============================================================

CREATE TABLE screening_decision (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    review_id       UUID NOT NULL REFERENCES review(id) ON DELETE CASCADE,
    reference_id    UUID NOT NULL REFERENCES reference(id) ON DELETE CASCADE,
    phase           VARCHAR(20) NOT NULL,   -- 'title_abstract', 'full_text'
    reviewer_id     UUID NOT NULL REFERENCES app_user(id),
    decision        VARCHAR(20) NOT NULL,   -- 'include', 'exclude', 'maybe'
    exclusion_reason VARCHAR(50),           -- code from review.config.exclusion_reasons
    notes           TEXT,
    decided_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (review_id, reference_id, phase, reviewer_id)
);

CREATE INDEX idx_screening_review_phase ON screening_decision(review_id, phase);
CREATE INDEX idx_screening_ref ON screening_decision(reference_id);

-- Final screening outcome per reference (resolved view)
CREATE TABLE screening_outcome (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    review_id       UUID NOT NULL REFERENCES review(id) ON DELETE CASCADE,
    reference_id    UUID NOT NULL REFERENCES reference(id) ON DELETE CASCADE,
    phase           VARCHAR(20) NOT NULL,
    final_decision  VARCHAR(20) NOT NULL,   -- 'include', 'exclude'
    exclusion_reason VARCHAR(50),
    had_conflict    BOOLEAN DEFAULT FALSE,
    resolved_by     UUID REFERENCES app_user(id),
    resolved_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (review_id, reference_id, phase)
);

CREATE INDEX idx_screening_outcome_review ON screening_outcome(review_id, phase, final_decision);

-- AI screening predictions
CREATE TABLE screening_prediction (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    review_id       UUID NOT NULL REFERENCES review(id) ON DELETE CASCADE,
    reference_id    UUID NOT NULL REFERENCES reference(id) ON DELETE CASCADE,
    phase           VARCHAR(20) NOT NULL,
    model_info      JSONB NOT NULL,
    /*  Example:
        {
            "model_version": "active_learning_v3.2",
            "classifier": "nb_svm_ensemble",
            "feature_extractor": "tfidf_256",
            "training_labels": 156,
            "relevance_score": 0.8734,
            "uncertainty": 0.12,
            "features_hash": "sha256:abc123"
        }
    */
    predicted_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_screening_pred_review ON screening_prediction(review_id, phase);
CREATE INDEX idx_screening_pred_score ON screening_prediction(
    review_id,
    ((model_info->>'relevance_score')::NUMERIC) DESC
);
```

## Documents

```sql
-- ============================================================
-- DOCUMENTS
-- ============================================================

CREATE TABLE document (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    reference_id    UUID NOT NULL REFERENCES reference(id) ON DELETE CASCADE,
    file_name       VARCHAR(255) NOT NULL,
    file_type       VARCHAR(20) NOT NULL,
    file_size_bytes BIGINT,
    storage_path    VARCHAR(1000) NOT NULL,
    content_hash    VARCHAR(64),
    full_text       TEXT,
    uploaded_by     UUID NOT NULL REFERENCES app_user(id),
    uploaded_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_document_ref ON document(reference_id);

CREATE TABLE document_annotation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    document_id     UUID NOT NULL REFERENCES document(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES app_user(id),
    annotation      JSONB NOT NULL,
    /*  Example:
        {
            "type": "highlight",
            "page": 3,
            "rect": {"x": 100, "y": 200, "width": 300, "height": 20},
            "colour": "#FFFF00",
            "text": "245 participants were randomised...",
            "note": "Sample size for extraction",
            "linked_extraction_field": "sample_size"
        }
    */
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_doc_annotation_doc ON document_annotation(document_id);
```

## Data Extraction

```sql
-- ============================================================
-- DATA EXTRACTION
-- ============================================================

-- Templates are stored as JSONB — runtime-configurable without migrations
CREATE TABLE extraction_template (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    review_id       UUID REFERENCES review(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    version         INTEGER NOT NULL DEFAULT 1,
    is_active       BOOLEAN DEFAULT TRUE,
    schema          JSONB NOT NULL,
    /*  Example:
        {
            "sections": [
                {
                    "name": "Study Characteristics",
                    "fields": [
                        {"name": "study_design", "label": "Study Design", "type": "select",
                         "options": ["RCT", "Cluster RCT", "Quasi-experimental"], "required": true},
                        {"name": "country", "label": "Country", "type": "text", "pico": null},
                        {"name": "sample_size", "label": "Sample Size", "type": "number", "pico": "population", "required": true},
                        {"name": "age_range", "label": "Age Range", "type": "text", "pico": "population"}
                    ]
                },
                {
                    "name": "Intervention Details",
                    "fields": [
                        {"name": "intervention_name", "label": "Intervention", "type": "text", "pico": "intervention", "required": true},
                        {"name": "intervention_duration", "label": "Duration (weeks)", "type": "number", "pico": "intervention"},
                        {"name": "comparator_name", "label": "Comparator", "type": "text", "pico": "comparator"}
                    ]
                },
                {
                    "name": "Outcomes",
                    "fields": [
                        {"name": "primary_outcome", "label": "Primary Outcome Measure", "type": "text", "pico": "outcome", "required": true},
                        {"name": "outcome_timepoint", "label": "Assessment Timepoint", "type": "text"},
                        {"name": "mean_intervention", "label": "Mean (Intervention)", "type": "number"},
                        {"name": "sd_intervention", "label": "SD (Intervention)", "type": "number"},
                        {"name": "mean_control", "label": "Mean (Control)", "type": "number"},
                        {"name": "sd_control", "label": "SD (Control)", "type": "number"}
                    ]
                }
            ]
        }
    */
    created_by      UUID NOT NULL REFERENCES app_user(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Extraction records: one per reference per extractor
CREATE TABLE extraction_record (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    review_id       UUID NOT NULL REFERENCES review(id) ON DELETE CASCADE,
    reference_id    UUID NOT NULL REFERENCES reference(id) ON DELETE CASCADE,
    template_id     UUID NOT NULL REFERENCES extraction_template(id),
    extractor_id    UUID NOT NULL REFERENCES app_user(id),
    status          VARCHAR(50) NOT NULL DEFAULT 'in_progress',
    -- All extracted values in a single JSONB object
    values          JSONB NOT NULL DEFAULT '{}',
    /*  Example:
        {
            "study_design": {"value": "RCT", "ai_extracted": false, "verified": true},
            "sample_size": {"value": 245, "ai_extracted": true, "ai_confidence": 0.95, "verified": true, "source_page": 3, "source_quote": "245 participants..."},
            "intervention_name": {"value": "Individual CBT (12 sessions)", "ai_extracted": true, "ai_confidence": 0.88, "verified": false},
            "primary_outcome": {"value": "GAD-7", "ai_extracted": true, "ai_confidence": 0.97, "verified": true},
            "mean_intervention": {"value": 8.3, "ai_extracted": true, "ai_confidence": 0.72, "verified": false, "source_page": 7},
            "sd_intervention": {"value": 3.2, "ai_extracted": true, "ai_confidence": 0.68, "verified": false, "source_page": 7}
        }
    */
    completed_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (review_id, reference_id, template_id, extractor_id)
);

CREATE INDEX idx_extraction_review ON extraction_record(review_id);
CREATE INDEX idx_extraction_ref ON extraction_record(reference_id);
CREATE INDEX idx_extraction_values ON extraction_record USING GIN (values jsonb_path_ops);

-- Example query: find all references where sample size > 100
-- SELECT r.title, e.values->'sample_size'->>'value' AS sample_size
-- FROM extraction_record e
-- JOIN reference r ON r.id = e.reference_id
-- WHERE e.review_id = 'uuid'
--   AND (e.values->'sample_size'->>'value')::INT > 100;
```

## Risk of Bias

```sql
-- ============================================================
-- RISK OF BIAS
-- ============================================================

-- RoB templates stored as JSONB — supports RoB 2, ROBINS-I, CEE CAT, etc.
CREATE TABLE rob_template (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    version         VARCHAR(20) NOT NULL,
    schema          JSONB NOT NULL,
    /*  Example (RoB 2):
        {
            "domains": [
                {
                    "number": 1,
                    "name": "Bias arising from the randomization process",
                    "questions": [
                        {"id": "1.1", "text": "Was the allocation sequence random?", "help": "..."},
                        {"id": "1.2", "text": "Was the allocation sequence concealed until participants were enrolled and assigned to interventions?"},
                        {"id": "1.3", "text": "Did baseline differences between intervention groups suggest a problem with the randomization process?"}
                    ]
                },
                {
                    "number": 2,
                    "name": "Bias due to deviations from intended interventions",
                    "questions": [
                        {"id": "2.1", "text": "Were participants aware of their assigned intervention during the trial?"},
                        {"id": "2.2", "text": "Were carers and people delivering the interventions aware of participants' assigned intervention during the trial?"}
                    ]
                }
            ],
            "judgement_options": ["low", "some_concerns", "high"],
            "answer_options": ["yes", "probably_yes", "no", "probably_no", "no_information"]
        }
    */
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- RoB assessments: answers and judgements all in JSONB
CREATE TABLE rob_assessment (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    review_id       UUID NOT NULL REFERENCES review(id) ON DELETE CASCADE,
    reference_id    UUID NOT NULL REFERENCES reference(id) ON DELETE CASCADE,
    template_id     UUID NOT NULL REFERENCES rob_template(id),
    assessor_id     UUID NOT NULL REFERENCES app_user(id),
    status          VARCHAR(50) NOT NULL DEFAULT 'in_progress',
    overall_judgement VARCHAR(20),
    -- All answers and domain judgements in JSONB
    assessment      JSONB NOT NULL DEFAULT '{}',
    /*  Example:
        {
            "domains": {
                "1": {
                    "answers": {
                        "1.1": {"answer": "yes", "support": "Central computerised randomisation"},
                        "1.2": {"answer": "yes", "support": "Sealed opaque envelopes"},
                        "1.3": {"answer": "no", "support": "Table 1 shows balanced groups"}
                    },
                    "judgement": "low",
                    "notes": "Adequate randomisation and concealment"
                },
                "2": {
                    "answers": {
                        "2.1": {"answer": "probably_yes", "support": "Blinding described but unclear"},
                        "2.2": {"answer": "no_information", "support": ""}
                    },
                    "judgement": "some_concerns",
                    "notes": "Insufficient blinding information"
                }
            },
            "overall_notes": "Some concerns due to domain 2"
        }
    */
    completed_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (review_id, reference_id, template_id, assessor_id)
);

CREATE INDEX idx_rob_review ON rob_assessment(review_id);
CREATE INDEX idx_rob_ref ON rob_assessment(reference_id);
```

## GRADE & Meta-Analysis

```sql
-- ============================================================
-- GRADE EVIDENCE ASSESSMENT
-- ============================================================

CREATE TABLE grade_assessment (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    review_id       UUID NOT NULL REFERENCES review(id) ON DELETE CASCADE,
    -- All GRADE data in a single structured JSONB
    assessment      JSONB NOT NULL,
    /*  Example:
        {
            "outcomes": [
                {
                    "name": "Anxiety symptoms (GAD-7)",
                    "type": "continuous",
                    "importance": 8,
                    "quality_assessment": {
                        "risk_of_bias": "not_serious",
                        "inconsistency": "not_serious",
                        "indirectness": "not_serious",
                        "imprecision": "serious",
                        "publication_bias": "undetected",
                        "large_effect": false,
                        "dose_response": false,
                        "plausible_confounding": false
                    },
                    "overall_certainty": "moderate",
                    "summary_of_findings": {
                        "num_studies": 12,
                        "num_participants": 1456,
                        "relative_effect": null,
                        "absolute_effect": {"SMD": -0.45, "ci_lower": -0.67, "ci_upper": -0.23},
                        "narrative": "CBT probably reduces anxiety symptoms moderately compared to control"
                    },
                    "footnotes": [
                        "Downgraded one level for imprecision: 95% CI includes trivial effect"
                    ]
                }
            ]
        }
    */
    assessor_id     UUID NOT NULL REFERENCES app_user(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- META-ANALYSIS
-- ============================================================

CREATE TABLE meta_analysis (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    review_id       UUID NOT NULL REFERENCES review(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    config          JSONB NOT NULL,
    /*  Example:
        {
            "analysis_type": "pairwise",
            "effect_measure": "SMD",
            "statistical_model": "random_effects",
            "subgroup_variable": null
        }
    */
    results         JSONB,
    /*  Example:
        {
            "pooled_estimate": -0.45,
            "ci_lower": -0.67,
            "ci_upper": -0.23,
            "p_value": 0.0001,
            "heterogeneity": {"i2": 34.2, "tau2": 0.023, "q": 16.7, "df": 11, "p": 0.12},
            "studies": [
                {"reference_id": "uuid", "label": "Smith 2024", "estimate": -0.52, "ci": [-0.89, -0.15], "weight": 12.3},
                {"reference_id": "uuid", "label": "Jones 2025", "estimate": -0.38, "ci": [-0.71, -0.05], "weight": 14.1}
            ]
        }
    */
    created_by      UUID NOT NULL REFERENCES app_user(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_meta_analysis_review ON meta_analysis(review_id);
```

## Audit & Activity Log

```sql
-- ============================================================
-- AUDIT LOG
-- ============================================================

CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    review_id       UUID REFERENCES review(id),
    user_id         UUID REFERENCES app_user(id),
    action          VARCHAR(100) NOT NULL,
    entity_type     VARCHAR(100) NOT NULL,
    entity_id       UUID NOT NULL,
    changes         JSONB,
    /*  Example:
        {
            "field": "screening_decision",
            "old": null,
            "new": "include",
            "phase": "title_abstract",
            "reference_title": "A randomised controlled trial..."
        }
    */
    ip_address      INET,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_review ON audit_log(review_id, created_at);
CREATE INDEX idx_audit_entity ON audit_log(entity_type, entity_id);

-- ============================================================
-- LIVING REVIEW
-- ============================================================

CREATE TABLE living_review_run (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    review_id       UUID NOT NULL REFERENCES review(id) ON DELETE CASCADE,
    config          JSONB NOT NULL,         -- snapshot of config at run time
    results         JSONB NOT NULL,
    /*  Example:
        {
            "new_records_found": 23,
            "auto_included": 0,
            "auto_excluded": 15,
            "pending_review": 8,
            "sources_searched": ["PubMed", "Scopus"]
        }
    */
    run_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_living_run_review ON living_review_run(review_id);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Organisation & Users | 3 | Settings and profile in JSONB |
| Review & Protocol | 2 | Protocol, PRISMA checklist, config all in JSONB columns |
| References | 1 | Extended metadata in JSONB; core fields relational |
| Screening | 3 | Decisions, outcomes, AI predictions |
| Documents | 2 | Annotations as JSONB |
| Extraction | 2 | Template schema and values both JSONB |
| Risk of Bias | 2 | Template schema and assessments both JSONB |
| GRADE | 1 | Full evidence profile in one JSONB document |
| Meta-Analysis | 1 | Config and results in JSONB |
| Audit & Living Review | 2 | Change tracking and scheduled runs |
| **Total** | **19** | |

---

## Key Design Decisions

1. **Core bibliographic fields are relational; extended metadata is JSONB** — `doi`, `pmid`, `title`, `publication_year`, and `journal` are columns because they are filtered and sorted frequently. Author details, MeSH terms, keywords, and source-specific fields live in `metadata` JSONB, queried via GIN index when needed.

2. **Protocol, PRISMA checklist, and review configuration in JSONB columns on `review`** — these are write-once, read-often structures that vary by review type. Storing them as JSONB avoids needing separate tables for protocol sections, PICO questions, and exclusion reason catalogues.

3. **Extraction templates as JSONB schema definitions** — templates define field structures at runtime without database migrations. The `extraction_record.values` JSONB stores all extracted data for a reference as a single document, enabling fast reads for summary tables.

4. **RoB assessments as nested JSONB** — the `rob_assessment.assessment` column stores all domain answers and judgements in a structure that mirrors the RoB 2 hierarchy. This supports RoB 2, ROBINS-I, and any future assessment framework without schema changes.

5. **GRADE as a single JSONB document** — the entire Summary of Findings table for a review is stored as one JSONB object. This matches how GRADE data is typically consumed (as a complete table) and simplifies export to GRADEpro GDT.

6. **Screening outcome table separates individual decisions from resolved outcomes** — `screening_decision` records each reviewer's vote; `screening_outcome` records the final include/exclude decision after conflict resolution. This avoids complex queries to determine the current state.

7. **AI prediction metadata in JSONB** — model version, classifier type, feature extractor, training label count, and uncertainty are all captured in `model_info` JSONB, accommodating different model architectures without schema changes.

8. **Raw import preserved** — `reference.raw_import` stores the original RIS/BibTeX/NBIB record as JSONB, enabling lossless round-trip import/export and debugging of parsing issues.

9. **GIN indexes on JSONB columns** — `jsonb_path_ops` GIN indexes on `reference.metadata`, `extraction_record.values`, and other JSONB columns enable efficient containment queries (`@>` operator) without scanning every row.

10. **19 tables total** — roughly half the table count of the fully normalised model, with the same functional coverage. The trade-off is that validation and type safety for JSONB fields must be enforced in the application layer.
