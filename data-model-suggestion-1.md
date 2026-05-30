# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: Systematic Literature Review Tool · Created: 2026-05-25

## Philosophy

This model follows classical relational database design principles: every distinct concept receives its own table, relationships are enforced through foreign keys, and data integrity is maintained through constraints and normalisation. The schema closely mirrors the real-world workflow of a systematic review — from protocol registration through search execution, screening, data extraction, risk of bias assessment, and evidence synthesis.

The design draws heavily on the PRISMA 2020 workflow phases, the Cochrane RoB 2 domain structure, and the GRADE evidence assessment framework as first-class entities rather than bolted-on features. Each phase of the review pipeline has dedicated tables, ensuring that the arithmetic of the PRISMA flow diagram (records in = records included + records excluded at each stage) can be enforced at the database level.

This approach is well-suited for teams building a production-grade platform where data integrity, complex cross-entity queries, and regulatory compliance are paramount. It maps naturally to REST APIs (one resource per table) and supports standard ORM patterns.

**Best for:** Institutional deployments where data integrity, Cochrane compatibility, and complex reporting queries are the primary requirements.

**Trade-offs:**
- (+) Maximum data integrity via foreign keys and constraints
- (+) Natural mapping to REST API resources and ORM patterns
- (+) Clear PRISMA flow tracking with enforceable arithmetic
- (+) Easy to generate audit reports with standard SQL joins
- (-) High table count (~45-55 tables) increases schema migration complexity
- (-) Adding jurisdiction-specific or domain-specific extraction fields requires schema changes
- (-) Many-to-many junction tables add query complexity for simple lookups
- (-) Performance may suffer for very large reviews (50,000+ records) without careful indexing

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| PRISMA 2020 | Dedicated `prisma_flowchart` and `prisma_checklist_item` tables track the 27-item checklist and 4-phase flow diagram counts |
| Cochrane RoB 2 | `rob_assessment`, `rob_domain`, and `rob_signalling_question` tables mirror the 5-domain, signalling-question hierarchy |
| GRADE | `grade_assessment` and `grade_outcome` tables model Summary of Findings and Evidence Profile structures |
| PICO | `review_question` table has explicit P/I/C/O columns; `extraction_field` references PICO elements |
| PROSPERO | `review_protocol` stores registration ID and links to the PROSPERO record |
| RIS/BibTeX/NBIB | `citation_import` records raw file imports; `reference` stores normalised citation data with format-agnostic fields |
| MeSH | `mesh_term` table with tree hierarchy; `reference_mesh_term` junction for tagging references |
| ISO 639-1 | `reference.language_code` stores detected language using the standard two-letter code |
| ISO 3166-1 | `institution.country_code` and `search_source.jurisdiction` use standard country codes |
| OpenAPI 3.1 | Schema designed to map directly to OpenAPI resource definitions |

---

## Organisation & Tenancy

```sql
-- ============================================================
-- ORGANISATION & USER MANAGEMENT
-- ============================================================

CREATE TABLE organisation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    subscription_tier VARCHAR(50) DEFAULT 'free',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE app_user (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email           VARCHAR(255) NOT NULL UNIQUE,
    display_name    VARCHAR(255) NOT NULL,
    orcid_id        VARCHAR(20),          -- ORCID identifier for researchers
    password_hash   VARCHAR(255),
    auth_provider   VARCHAR(50) DEFAULT 'local',  -- 'local', 'oidc', 'saml'
    auth_subject    VARCHAR(255),         -- external IdP subject ID
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE organisation_member (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    user_id         UUID NOT NULL REFERENCES app_user(id),
    role            VARCHAR(50) NOT NULL DEFAULT 'member',  -- 'owner', 'admin', 'member'
    invited_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    accepted_at     TIMESTAMPTZ,
    UNIQUE (organisation_id, user_id)
);

CREATE INDEX idx_org_member_org ON organisation_member(organisation_id);
CREATE INDEX idx_org_member_user ON organisation_member(user_id);
```

## Review Project & Protocol

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
        -- 'systematic_review', 'scoping_review', 'rapid_review', 'meta_analysis', 'systematic_map'
    status          VARCHAR(50) NOT NULL DEFAULT 'protocol',
        -- 'protocol', 'searching', 'screening', 'extraction', 'synthesis', 'reporting', 'published', 'archived'
    created_by      UUID NOT NULL REFERENCES app_user(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_review_org ON review(organisation_id);

CREATE TABLE review_team_member (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    review_id       UUID NOT NULL REFERENCES review(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES app_user(id),
    role            VARCHAR(50) NOT NULL DEFAULT 'reviewer',
        -- 'lead', 'reviewer', 'screener', 'extractor', 'statistician', 'observer'
    assigned_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (review_id, user_id)
);

CREATE INDEX idx_review_team_review ON review_team_member(review_id);

CREATE TABLE review_protocol (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    review_id       UUID NOT NULL REFERENCES review(id) ON DELETE CASCADE UNIQUE,
    prospero_id     VARCHAR(50),          -- PROSPERO registration number e.g. CRD42026000001
    registration_date DATE,
    background      TEXT,
    objectives      TEXT,
    eligibility_criteria TEXT,
    information_sources TEXT,
    search_strategy TEXT,
    study_selection TEXT,
    data_collection TEXT,
    risk_of_bias_method TEXT,
    synthesis_method TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- PICO FRAMEWORK
-- ============================================================

CREATE TABLE review_question (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    review_id       UUID NOT NULL REFERENCES review(id) ON DELETE CASCADE,
    question_text   TEXT NOT NULL,
    population      TEXT,                 -- P: target population description
    intervention    TEXT,                 -- I: intervention or exposure
    comparator      TEXT,                 -- C: comparator or control
    outcome         TEXT,                 -- O: primary outcome(s)
    study_design    TEXT,                 -- optional: study design filter
    sort_order      INTEGER NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_review_question_review ON review_question(review_id);
```

## Search & References

```sql
-- ============================================================
-- SEARCH MANAGEMENT
-- ============================================================

CREATE TABLE search_source (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,  -- 'PubMed', 'Scopus', 'Web of Science', etc.
    source_type     VARCHAR(50) NOT NULL,   -- 'database', 'registry', 'grey_literature', 'hand_search', 'citation_chasing'
    api_base_url    VARCHAR(500),
    jurisdiction    VARCHAR(10),            -- ISO 3166-1 alpha-2 if region-specific
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE search_execution (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    review_id       UUID NOT NULL REFERENCES review(id) ON DELETE CASCADE,
    source_id       UUID NOT NULL REFERENCES search_source(id),
    query_string    TEXT NOT NULL,
    executed_at     TIMESTAMPTZ NOT NULL,
    executed_by     UUID NOT NULL REFERENCES app_user(id),
    results_count   INTEGER,
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_search_exec_review ON search_execution(review_id);

-- ============================================================
-- REFERENCES (CITATIONS)
-- ============================================================

CREATE TABLE reference (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    review_id       UUID NOT NULL REFERENCES review(id) ON DELETE CASCADE,
    -- Identifiers (standards-aligned)
    doi             VARCHAR(255),         -- Digital Object Identifier
    pmid            VARCHAR(20),          -- PubMed ID
    pmcid           VARCHAR(20),          -- PubMed Central ID
    openalex_id     VARCHAR(100),         -- OpenAlex ID
    semantic_scholar_id VARCHAR(100),
    arxiv_id        VARCHAR(50),
    -- Bibliographic fields (RIS/BibTeX superset)
    title           TEXT NOT NULL,
    abstract        TEXT,
    authors         TEXT,                 -- semicolon-separated full author list
    journal         VARCHAR(500),
    volume          VARCHAR(50),
    issue           VARCHAR(50),
    pages           VARCHAR(50),
    publication_year INTEGER,
    publication_date DATE,
    publisher       VARCHAR(255),
    language_code   VARCHAR(10),          -- ISO 639-1
    document_type   VARCHAR(50),          -- 'journal_article', 'conference_paper', 'book_chapter', etc.
    url             VARCHAR(1000),
    -- Import tracking
    import_source_id UUID REFERENCES search_source(id),
    import_batch_id  UUID,
    -- Deduplication
    dedup_cluster_id UUID,                -- references in the same cluster are potential duplicates
    is_duplicate    BOOLEAN DEFAULT FALSE,
    master_ref_id   UUID REFERENCES reference(id),  -- if duplicate, points to the master record
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_reference_review ON reference(review_id);
CREATE INDEX idx_reference_doi ON reference(doi) WHERE doi IS NOT NULL;
CREATE INDEX idx_reference_pmid ON reference(pmid) WHERE pmid IS NOT NULL;
CREATE INDEX idx_reference_dedup ON reference(dedup_cluster_id) WHERE dedup_cluster_id IS NOT NULL;

CREATE TABLE reference_author (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    reference_id    UUID NOT NULL REFERENCES reference(id) ON DELETE CASCADE,
    full_name       VARCHAR(255) NOT NULL,
    family_name     VARCHAR(255),
    given_name      VARCHAR(255),
    orcid_id        VARCHAR(20),
    affiliation     VARCHAR(500),
    author_position INTEGER NOT NULL,     -- 1-based ordering
    is_corresponding BOOLEAN DEFAULT FALSE
);

CREATE INDEX idx_ref_author_ref ON reference_author(reference_id);

CREATE TABLE reference_keyword (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    reference_id    UUID NOT NULL REFERENCES reference(id) ON DELETE CASCADE,
    keyword         VARCHAR(255) NOT NULL,
    keyword_source  VARCHAR(50) DEFAULT 'author',  -- 'author', 'mesh', 'openalex_concept', 'ai_generated'
);

CREATE INDEX idx_ref_keyword_ref ON reference_keyword(reference_id);

-- ============================================================
-- MeSH TERMS
-- ============================================================

CREATE TABLE mesh_term (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    descriptor_ui   VARCHAR(20) NOT NULL UNIQUE,  -- e.g. 'D000086382'
    term            VARCHAR(255) NOT NULL,
    tree_number     VARCHAR(100),                 -- e.g. 'C08.381.495.108'
    parent_ui       VARCHAR(20) REFERENCES mesh_term(descriptor_ui)
);

CREATE INDEX idx_mesh_term_tree ON mesh_term(tree_number);

CREATE TABLE reference_mesh_term (
    reference_id    UUID NOT NULL REFERENCES reference(id) ON DELETE CASCADE,
    mesh_term_id    UUID NOT NULL REFERENCES mesh_term(id),
    is_major_topic  BOOLEAN DEFAULT FALSE,
    PRIMARY KEY (reference_id, mesh_term_id)
);

-- ============================================================
-- CITATION IMPORT BATCHES
-- ============================================================

CREATE TABLE citation_import (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    review_id       UUID NOT NULL REFERENCES review(id) ON DELETE CASCADE,
    source_id       UUID REFERENCES search_source(id),
    file_name       VARCHAR(255) NOT NULL,
    file_format     VARCHAR(20) NOT NULL,   -- 'ris', 'bibtex', 'nbib', 'csv', 'endnote_xml'
    records_total   INTEGER,
    records_imported INTEGER,
    records_duplicate INTEGER,
    imported_by     UUID NOT NULL REFERENCES app_user(id),
    imported_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## Screening

```sql
-- ============================================================
-- SCREENING WORKFLOW
-- ============================================================

CREATE TABLE screening_phase (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    review_id       UUID NOT NULL REFERENCES review(id) ON DELETE CASCADE,
    phase_type      VARCHAR(50) NOT NULL,   -- 'title_abstract', 'full_text'
    required_reviewers INTEGER NOT NULL DEFAULT 2,
    conflict_resolution VARCHAR(50) DEFAULT 'consensus',  -- 'consensus', 'third_reviewer', 'lead_decides'
    status          VARCHAR(50) NOT NULL DEFAULT 'not_started',
        -- 'not_started', 'in_progress', 'completed'
    started_at      TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_screening_phase_review ON screening_phase(review_id);

CREATE TABLE screening_decision (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    screening_phase_id UUID NOT NULL REFERENCES screening_phase(id) ON DELETE CASCADE,
    reference_id    UUID NOT NULL REFERENCES reference(id) ON DELETE CASCADE,
    reviewer_id     UUID NOT NULL REFERENCES app_user(id),
    decision        VARCHAR(20) NOT NULL,   -- 'include', 'exclude', 'maybe'
    exclusion_reason_id UUID,               -- FK added after exclusion_reason table
    notes           TEXT,
    decided_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (screening_phase_id, reference_id, reviewer_id)
);

CREATE INDEX idx_screening_decision_phase ON screening_decision(screening_phase_id);
CREATE INDEX idx_screening_decision_ref ON screening_decision(reference_id);

CREATE TABLE exclusion_reason (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    review_id       UUID NOT NULL REFERENCES review(id) ON DELETE CASCADE,
    code            VARCHAR(50) NOT NULL,
    label           VARCHAR(255) NOT NULL,
    description     TEXT,
    phase_type      VARCHAR(50) NOT NULL,   -- applicable to 'title_abstract', 'full_text', or 'both'
    sort_order      INTEGER DEFAULT 0,
    UNIQUE (review_id, code)
);

ALTER TABLE screening_decision
    ADD CONSTRAINT fk_screening_exclusion_reason
    FOREIGN KEY (exclusion_reason_id) REFERENCES exclusion_reason(id);

CREATE TABLE screening_conflict (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    screening_phase_id UUID NOT NULL REFERENCES screening_phase(id) ON DELETE CASCADE,
    reference_id    UUID NOT NULL REFERENCES reference(id) ON DELETE CASCADE,
    status          VARCHAR(50) NOT NULL DEFAULT 'unresolved',  -- 'unresolved', 'resolved'
    resolved_by     UUID REFERENCES app_user(id),
    resolution      VARCHAR(20),            -- 'include', 'exclude'
    resolved_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_screening_conflict_phase ON screening_conflict(screening_phase_id);

-- ============================================================
-- AI SCREENING PREDICTIONS
-- ============================================================

CREATE TABLE screening_prediction (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    screening_phase_id UUID NOT NULL REFERENCES screening_phase(id) ON DELETE CASCADE,
    reference_id    UUID NOT NULL REFERENCES reference(id) ON DELETE CASCADE,
    model_version   VARCHAR(100) NOT NULL,
    relevance_score NUMERIC(5,4) NOT NULL,  -- 0.0000 to 1.0000
    predicted_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    training_labels_count INTEGER           -- number of human labels the model was trained on
);

CREATE INDEX idx_screening_pred_phase ON screening_prediction(screening_phase_id);
CREATE INDEX idx_screening_pred_score ON screening_prediction(relevance_score DESC);
```

## Full-Text Documents

```sql
-- ============================================================
-- DOCUMENT MANAGEMENT
-- ============================================================

CREATE TABLE document (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    reference_id    UUID NOT NULL REFERENCES reference(id) ON DELETE CASCADE,
    file_name       VARCHAR(255) NOT NULL,
    file_type       VARCHAR(20) NOT NULL,   -- 'pdf', 'html', 'docx', 'xml'
    file_size_bytes BIGINT,
    storage_path    VARCHAR(1000) NOT NULL,  -- object storage key
    content_hash    VARCHAR(64),            -- SHA-256 for deduplication
    full_text       TEXT,                   -- extracted plain text
    uploaded_by     UUID NOT NULL REFERENCES app_user(id),
    uploaded_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_document_ref ON document(reference_id);

CREATE TABLE document_annotation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    document_id     UUID NOT NULL REFERENCES document(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES app_user(id),
    page_number     INTEGER,
    x_position      NUMERIC(8,2),
    y_position      NUMERIC(8,2),
    width           NUMERIC(8,2),
    height          NUMERIC(8,2),
    annotation_type VARCHAR(50) NOT NULL,   -- 'highlight', 'note', 'extraction_link'
    colour          VARCHAR(7),             -- hex colour
    content         TEXT,
    linked_extraction_field_id UUID,        -- FK to extraction_field_value
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

CREATE TABLE extraction_template (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    review_id       UUID NOT NULL REFERENCES review(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    version         INTEGER NOT NULL DEFAULT 1,
    is_active       BOOLEAN DEFAULT TRUE,
    created_by      UUID NOT NULL REFERENCES app_user(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_extraction_tpl_review ON extraction_template(review_id);

CREATE TABLE extraction_field (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    template_id     UUID NOT NULL REFERENCES extraction_template(id) ON DELETE CASCADE,
    field_name      VARCHAR(255) NOT NULL,
    field_label     VARCHAR(255) NOT NULL,
    field_type      VARCHAR(50) NOT NULL,
        -- 'text', 'number', 'date', 'select', 'multi_select', 'boolean', 'textarea'
    pico_element    VARCHAR(20),            -- 'population', 'intervention', 'comparator', 'outcome', null
    is_required     BOOLEAN DEFAULT FALSE,
    options         TEXT[],                 -- for select/multi_select fields
    validation_rule VARCHAR(255),
    help_text       TEXT,
    section_name    VARCHAR(255),           -- grouping within the template
    sort_order      INTEGER NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_extraction_field_tpl ON extraction_field(template_id);

CREATE TABLE extraction_record (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    reference_id    UUID NOT NULL REFERENCES reference(id) ON DELETE CASCADE,
    template_id     UUID NOT NULL REFERENCES extraction_template(id),
    extractor_id    UUID NOT NULL REFERENCES app_user(id),
    status          VARCHAR(50) NOT NULL DEFAULT 'in_progress',
        -- 'in_progress', 'completed', 'verified'
    completed_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (reference_id, template_id, extractor_id)
);

CREATE INDEX idx_extraction_record_ref ON extraction_record(reference_id);

CREATE TABLE extraction_field_value (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    extraction_record_id UUID NOT NULL REFERENCES extraction_record(id) ON DELETE CASCADE,
    field_id        UUID NOT NULL REFERENCES extraction_field(id),
    value_text      TEXT,
    value_number    NUMERIC,
    value_date      DATE,
    value_boolean   BOOLEAN,
    ai_extracted    BOOLEAN DEFAULT FALSE,
    ai_confidence   NUMERIC(5,4),
    human_verified  BOOLEAN DEFAULT FALSE,
    source_page     INTEGER,               -- page number in the PDF where this was extracted
    source_quote    TEXT,                   -- supporting text from the document
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (extraction_record_id, field_id)
);

CREATE INDEX idx_extraction_value_record ON extraction_field_value(extraction_record_id);
```

## Risk of Bias (RoB 2)

```sql
-- ============================================================
-- RISK OF BIAS ASSESSMENT (Cochrane RoB 2)
-- ============================================================

CREATE TABLE rob_template (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,  -- 'RoB 2 (individual RCTs)', 'RoB 2 (cluster RCTs)', 'ROBINS-I'
    version         VARCHAR(20) NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE rob_domain (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    template_id     UUID NOT NULL REFERENCES rob_template(id) ON DELETE CASCADE,
    domain_number   INTEGER NOT NULL,       -- 1-5 for RoB 2
    domain_name     VARCHAR(255) NOT NULL,
        -- e.g. 'Bias arising from the randomization process'
    sort_order      INTEGER NOT NULL
);

CREATE TABLE rob_signalling_question (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    domain_id       UUID NOT NULL REFERENCES rob_domain(id) ON DELETE CASCADE,
    question_number VARCHAR(10) NOT NULL,   -- e.g. '1.1', '1.2'
    question_text   TEXT NOT NULL,
    help_text       TEXT,
    sort_order      INTEGER NOT NULL
);

CREATE TABLE rob_assessment (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    review_id       UUID NOT NULL REFERENCES review(id) ON DELETE CASCADE,
    reference_id    UUID NOT NULL REFERENCES reference(id) ON DELETE CASCADE,
    template_id     UUID NOT NULL REFERENCES rob_template(id),
    assessor_id     UUID NOT NULL REFERENCES app_user(id),
    overall_judgement VARCHAR(20),          -- 'low', 'some_concerns', 'high'
    overall_notes   TEXT,
    status          VARCHAR(50) NOT NULL DEFAULT 'in_progress',
    completed_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (review_id, reference_id, template_id, assessor_id)
);

CREATE INDEX idx_rob_assessment_review ON rob_assessment(review_id);

CREATE TABLE rob_answer (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    assessment_id   UUID NOT NULL REFERENCES rob_assessment(id) ON DELETE CASCADE,
    question_id     UUID NOT NULL REFERENCES rob_signalling_question(id),
    answer          VARCHAR(20) NOT NULL,   -- 'yes', 'probably_yes', 'no', 'probably_no', 'no_information'
    domain_judgement VARCHAR(20),           -- 'low', 'some_concerns', 'high' (set at domain level)
    support_text    TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (assessment_id, question_id)
);

CREATE INDEX idx_rob_answer_assessment ON rob_answer(assessment_id);
```

## GRADE & Evidence Synthesis

```sql
-- ============================================================
-- GRADE EVIDENCE ASSESSMENT
-- ============================================================

CREATE TABLE grade_outcome (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    review_id       UUID NOT NULL REFERENCES review(id) ON DELETE CASCADE,
    outcome_name    VARCHAR(255) NOT NULL,
    outcome_type    VARCHAR(50) NOT NULL,   -- 'dichotomous', 'continuous', 'time_to_event'
    importance      INTEGER CHECK (importance BETWEEN 1 AND 9),
        -- 1-3: not important, 4-6: important, 7-9: critical
    sort_order      INTEGER DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE grade_assessment (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    outcome_id      UUID NOT NULL REFERENCES grade_outcome(id) ON DELETE CASCADE,
    -- Quality assessment domains
    risk_of_bias    VARCHAR(20) DEFAULT 'not_serious',
        -- 'not_serious', 'serious', 'very_serious'
    inconsistency   VARCHAR(20) DEFAULT 'not_serious',
    indirectness    VARCHAR(20) DEFAULT 'not_serious',
    imprecision     VARCHAR(20) DEFAULT 'not_serious',
    publication_bias VARCHAR(20) DEFAULT 'undetected',
        -- 'undetected', 'strongly_suspected'
    -- Upgrade factors
    large_effect    BOOLEAN DEFAULT FALSE,
    dose_response   BOOLEAN DEFAULT FALSE,
    plausible_confounding BOOLEAN DEFAULT FALSE,
    -- Summary
    overall_certainty VARCHAR(20) NOT NULL DEFAULT 'high',
        -- 'high', 'moderate', 'low', 'very_low'
    -- Effect estimates
    num_studies     INTEGER,
    num_participants INTEGER,
    relative_effect_type VARCHAR(20),       -- 'RR', 'OR', 'HR'
    relative_effect_estimate NUMERIC(8,4),
    relative_effect_ci_lower NUMERIC(8,4),
    relative_effect_ci_upper NUMERIC(8,4),
    absolute_effect_estimate NUMERIC(8,4),
    absolute_effect_ci_lower NUMERIC(8,4),
    absolute_effect_ci_upper NUMERIC(8,4),
    narrative_summary TEXT,
    assessor_id     UUID NOT NULL REFERENCES app_user(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_grade_assessment_outcome ON grade_assessment(outcome_id);
```

## PRISMA Tracking

```sql
-- ============================================================
-- PRISMA FLOW DIAGRAM
-- ============================================================

CREATE TABLE prisma_flowchart (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    review_id       UUID NOT NULL REFERENCES review(id) ON DELETE CASCADE UNIQUE,
    -- Identification
    records_from_databases INTEGER DEFAULT 0,
    records_from_registers INTEGER DEFAULT 0,
    records_from_other INTEGER DEFAULT 0,
    records_removed_before_screening INTEGER DEFAULT 0,
    duplicates_removed INTEGER DEFAULT 0,
    -- Screening
    records_screened INTEGER DEFAULT 0,
    records_excluded_screening INTEGER DEFAULT 0,
    -- Eligibility
    reports_sought_for_retrieval INTEGER DEFAULT 0,
    reports_not_retrieved INTEGER DEFAULT 0,
    reports_assessed_for_eligibility INTEGER DEFAULT 0,
    reports_excluded_with_reasons INTEGER DEFAULT 0,
    -- Included
    studies_included INTEGER DEFAULT 0,
    reports_included INTEGER DEFAULT 0,
    -- Metadata
    auto_calculated BOOLEAN DEFAULT TRUE,  -- whether counts are derived from screening decisions
    last_calculated_at TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE prisma_checklist_item (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    review_id       UUID NOT NULL REFERENCES review(id) ON DELETE CASCADE,
    item_number     INTEGER NOT NULL,       -- 1-27
    section         VARCHAR(100) NOT NULL,  -- 'Title', 'Abstract', 'Introduction', etc.
    topic           VARCHAR(255) NOT NULL,
    completed       BOOLEAN DEFAULT FALSE,
    location_in_report TEXT,               -- where in the manuscript this is addressed
    notes           TEXT,
    UNIQUE (review_id, item_number)
);

CREATE INDEX idx_prisma_checklist_review ON prisma_checklist_item(review_id);
```

## Meta-Analysis & Synthesis

```sql
-- ============================================================
-- META-ANALYSIS
-- ============================================================

CREATE TABLE meta_analysis (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    review_id       UUID NOT NULL REFERENCES review(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    analysis_type   VARCHAR(50) NOT NULL,   -- 'pairwise', 'network', 'subgroup', 'sensitivity'
    effect_measure  VARCHAR(20) NOT NULL,   -- 'RR', 'OR', 'HR', 'MD', 'SMD', 'RD'
    statistical_model VARCHAR(20) NOT NULL, -- 'fixed_effect', 'random_effects'
    heterogeneity_i2 NUMERIC(5,2),
    heterogeneity_tau2 NUMERIC(10,6),
    pooled_estimate NUMERIC(10,6),
    pooled_ci_lower NUMERIC(10,6),
    pooled_ci_upper NUMERIC(10,6),
    pooled_p_value  NUMERIC(10,8),
    created_by      UUID NOT NULL REFERENCES app_user(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE meta_analysis_study (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    meta_analysis_id UUID NOT NULL REFERENCES meta_analysis(id) ON DELETE CASCADE,
    reference_id    UUID NOT NULL REFERENCES reference(id),
    effect_estimate NUMERIC(10,6) NOT NULL,
    ci_lower        NUMERIC(10,6),
    ci_upper        NUMERIC(10,6),
    se              NUMERIC(10,6),         -- standard error
    weight          NUMERIC(8,4),          -- percentage weight in the meta-analysis
    -- Raw data (for recalculation)
    events_intervention INTEGER,
    total_intervention INTEGER,
    events_control  INTEGER,
    total_control   INTEGER,
    mean_intervention NUMERIC(10,4),
    sd_intervention NUMERIC(10,4),
    mean_control    NUMERIC(10,4),
    sd_control      NUMERIC(10,4),
    subgroup        VARCHAR(255),
    UNIQUE (meta_analysis_id, reference_id)
);

CREATE INDEX idx_meta_study_analysis ON meta_analysis_study(meta_analysis_id);
```

## Living Review & Notifications

```sql
-- ============================================================
-- LIVING REVIEW
-- ============================================================

CREATE TABLE living_review_config (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    review_id       UUID NOT NULL REFERENCES review(id) ON DELETE CASCADE UNIQUE,
    is_active       BOOLEAN DEFAULT FALSE,
    search_interval_days INTEGER DEFAULT 30,
    last_run_at     TIMESTAMPTZ,
    next_run_at     TIMESTAMPTZ,
    notify_on_new_records BOOLEAN DEFAULT TRUE,
    auto_screen     BOOLEAN DEFAULT FALSE,  -- use AI to auto-screen new records
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE living_review_run (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    config_id       UUID NOT NULL REFERENCES living_review_config(id) ON DELETE CASCADE,
    run_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    new_records_found INTEGER DEFAULT 0,
    records_auto_included INTEGER DEFAULT 0,
    records_auto_excluded INTEGER DEFAULT 0,
    records_pending_review INTEGER DEFAULT 0,
    status          VARCHAR(50) NOT NULL DEFAULT 'completed'
);

-- ============================================================
-- AUDIT LOG
-- ============================================================

CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    review_id       UUID REFERENCES review(id),
    user_id         UUID REFERENCES app_user(id),
    action          VARCHAR(100) NOT NULL,
        -- 'reference.imported', 'screening.decided', 'extraction.saved', 'rob.assessed', etc.
    entity_type     VARCHAR(100) NOT NULL,
    entity_id       UUID NOT NULL,
    old_value       TEXT,
    new_value       TEXT,
    ip_address      INET,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_log_review ON audit_log(review_id);
CREATE INDEX idx_audit_log_created ON audit_log(created_at);
CREATE INDEX idx_audit_log_entity ON audit_log(entity_type, entity_id);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Organisation & Users | 3 | Multi-tenant with RBAC |
| Review Project & Protocol | 4 | Including PICO questions |
| Search & References | 7 | Full bibliographic model with MeSH, authors, keywords |
| Screening | 4 | Dual-phase with AI predictions and conflict resolution |
| Documents | 2 | PDF storage and annotation |
| Data Extraction | 4 | Template-driven with AI-assisted fields |
| Risk of Bias | 5 | Full RoB 2 hierarchy (template, domain, signalling question, assessment, answer) |
| GRADE & Synthesis | 2 | Summary of Findings table structure |
| PRISMA | 2 | Flow diagram counts and 27-item checklist |
| Meta-Analysis | 2 | Pairwise/network with study-level data |
| Living Review | 2 | Scheduled search monitoring |
| Audit | 1 | Append-only event log |
| **Total** | **38** | |

---

## Key Design Decisions

1. **Separate `reference_author` table** rather than a flat author string — enables ORCID linking, institution mapping, and deduplication by author identity across references.

2. **Dual screening phase model** — `screening_phase` and `screening_decision` support the standard two-stage (title/abstract then full-text) workflow with configurable reviewer requirements and conflict resolution strategies.

3. **AI predictions stored alongside human decisions** — `screening_prediction` records model scores with version tracking so that the active learning history is fully auditable and reproducible.

4. **Template-driven extraction with typed values** — `extraction_field_value` stores values in type-appropriate columns (`value_text`, `value_number`, `value_date`, `value_boolean`) rather than everything as text, enabling type-safe queries and aggregation.

5. **Full RoB 2 hierarchy** — five tables mirror the Cochrane RoB 2 structure exactly (template → domain → signalling question → assessment → answer), supporting both RoB 2 and ROBINS-I through different template definitions.

6. **PRISMA flowchart as a materialised view** — the `prisma_flowchart` table can be auto-calculated from screening decisions or manually overridden, supporting both automated and manual PRISMA reporting.

7. **Deduplication modelled at the reference level** — `dedup_cluster_id` and `master_ref_id` support both automated (semantic) and manual deduplication with a clear master-duplicate relationship.

8. **GRADE assessment as a first-class entity** — separate from extraction and RoB, with all six downgrade/upgrade domains as discrete columns for structured queries and automated Summary of Findings generation.

9. **Living review as a scheduled job** — `living_review_config` and `living_review_run` support continuously updated reviews without disrupting the main screening workflow.

10. **Append-only audit log** — captures every significant action with before/after values, supporting regulatory compliance and reproducibility requirements.
