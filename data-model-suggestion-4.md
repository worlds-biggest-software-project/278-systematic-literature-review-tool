# Data Model Suggestion 4: Graph-Relational Hybrid

> Project: Systematic Literature Review Tool · Created: 2026-05-25

## Philosophy

This model recognises that a systematic literature review is fundamentally a graph problem: papers cite other papers, authors collaborate across institutions, studies share interventions and outcomes, MeSH terms form hierarchies, and the relationships between these entities are as important as the entities themselves. A graph-relational hybrid stores operational CRUD data in relational tables but layers a property graph on top for relationship-heavy queries like citation network analysis, evidence gap mapping, author conflict-of-interest detection, and semantic study clustering.

The graph layer is implemented using a pair of generic `graph_node` and `graph_edge` tables in PostgreSQL (no external graph database required), with recursive CTEs for traversal queries. This approach is inspired by how academic search engines like OpenAlex and Semantic Scholar model the scholarly knowledge graph — works, authors, institutions, topics, and venues are all nodes connected by typed edges. For a systematic review tool, this graph enables powerful analytical features that are difficult or impossible with pure relational models: "show me all papers that cite any included study," "find author networks that span multiple included studies," "map the PICO overlap between studies visually."

The relational tables handle the systematic review workflow (screening, extraction, RoB, GRADE), while the graph handles the scholarly knowledge structure and cross-study relationships.

**Best for:** Platforms emphasising citation network analysis, evidence gap mapping, author network visualisation, and cross-review knowledge synthesis.

**Trade-offs:**
- (+) Enables citation chaining and snowball searching directly in the database
- (+) Evidence gap mapping across PICO dimensions is a natural graph query
- (+) Author and institution networks support conflict-of-interest detection
- (+) Semantic clustering of similar studies via graph community detection
- (+) Cross-review knowledge synthesis: link studies across multiple reviews
- (+) MeSH term hierarchy traversal without recursive CTE complexity
- (-) Graph tables add storage overhead for relationship metadata
- (-) Graph traversal queries require careful depth limiting to avoid runaway performance
- (-) Developers must understand both relational and graph query patterns
- (-) The generic graph layer loses some type safety compared to dedicated relational tables
- (-) Not as mature a pattern as pure relational — fewer off-the-shelf tools and ORMs

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| PRISMA 2020 | Flow diagram counts derived from screening tables; citation chasing (forward/backward) tracked as graph edges |
| Cochrane RoB 2 | Assessments stored in relational tables; inter-rater agreement analysable via graph queries |
| GRADE | Evidence profiles stored relationally; PICO overlap between studies mapped as graph edges |
| PICO | PICO elements modelled as graph nodes with typed edges to studies, enabling evidence gap matrices |
| OpenAlex | Reference metadata aligned with OpenAlex work object fields; graph structure mirrors OpenAlex knowledge graph |
| Semantic Scholar | Citation graph edges modelled after Semantic Scholar's citation intent classification |
| MeSH | MeSH hierarchy stored as a graph (tree_number paths encoded as graph edges) |
| DOI / PMID / ORCID | Standard identifiers used as node properties and for cross-system linking |

---

## Graph Layer

```sql
-- ============================================================
-- PROPERTY GRAPH LAYER
-- ============================================================

-- Generic graph nodes — each represents an entity in the knowledge graph
CREATE TABLE graph_node (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    node_type       VARCHAR(50) NOT NULL,
    /*  Node types:
        'reference'     — a paper/study in the review
        'author'        — a researcher
        'institution'   — a university, hospital, or organisation
        'journal'       — a publication venue
        'mesh_term'     — a MeSH descriptor
        'pico_element'  — a population, intervention, comparator, or outcome concept
        'topic'         — a research topic or theme (from OpenAlex or AI-generated)
        'review'        — a systematic review (enables cross-review queries)
    */
    external_id     VARCHAR(255),           -- DOI, PMID, ORCID, OpenAlex ID, MeSH UI, etc.
    label           TEXT NOT NULL,           -- display name
    properties      JSONB DEFAULT '{}',     -- node-type-specific properties
    /*  Example for node_type='reference':
        {
            "doi": "10.1001/jama.2026.1234",
            "pmid": "38456789",
            "publication_year": 2026,
            "journal": "JAMA",
            "document_type": "journal_article",
            "cited_by_count": 45
        }
        Example for node_type='author':
        {
            "orcid": "0000-0001-2345-6789",
            "affiliation": "University of Oxford",
            "h_index": 42
        }
        Example for node_type='mesh_term':
        {
            "descriptor_ui": "D060825",
            "tree_numbers": ["F04.096.180", "I02.783.200"],
            "is_major": false
        }
        Example for node_type='pico_element':
        {
            "element_type": "intervention",
            "canonical_name": "Cognitive Behavioural Therapy",
            "synonyms": ["CBT", "cognitive behavioral therapy"]
        }
    */
    review_id       UUID,                   -- NULL for global nodes (authors, institutions, MeSH)
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_graph_node_type ON graph_node(node_type);
CREATE INDEX idx_graph_node_external ON graph_node(external_id) WHERE external_id IS NOT NULL;
CREATE INDEX idx_graph_node_review ON graph_node(review_id) WHERE review_id IS NOT NULL;
CREATE INDEX idx_graph_node_properties ON graph_node USING GIN (properties jsonb_path_ops);
CREATE INDEX idx_graph_node_label ON graph_node USING GIN (to_tsvector('english', label));

-- Generic graph edges — typed, directed relationships between nodes
CREATE TABLE graph_edge (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    source_id       UUID NOT NULL REFERENCES graph_node(id) ON DELETE CASCADE,
    target_id       UUID NOT NULL REFERENCES graph_node(id) ON DELETE CASCADE,
    edge_type       VARCHAR(50) NOT NULL,
    /*  Edge types:
        'cites'             — reference A cites reference B
        'cited_by'          — inverse of cites (materialised for fast traversal)
        'authored_by'       — reference → author
        'affiliated_with'   — author → institution
        'published_in'      — reference → journal
        'tagged_with'       — reference → mesh_term
        'mesh_parent'       — mesh_term → parent mesh_term
        'studies_pico'      — reference → pico_element (with pico_role property)
        'related_to'        — semantic similarity between references
        'included_in'       — reference → review
        'same_as'           — deduplication link between references
        'co_authored'       — author → author (inferred from shared references)
    */
    properties      JSONB DEFAULT '{}',
    /*  Example for edge_type='cites':
        {
            "citation_context": "background",
            "intent": "background",  -- 'background', 'method', 'result_comparison'
            "section": "introduction"
        }
        Example for edge_type='studies_pico':
        {
            "pico_role": "intervention",
            "specifics": "CBT, 12 sessions, individual format"
        }
        Example for edge_type='related_to':
        {
            "similarity_score": 0.87,
            "method": "embedding_cosine",
            "model": "specter2"
        }
        Example for edge_type='authored_by':
        {
            "author_position": 1,
            "is_corresponding": true
        }
    */
    weight          NUMERIC(5,4),           -- optional edge weight for scoring/ranking
    review_id       UUID,                   -- scope to a review, or NULL for global
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_graph_edge_source ON graph_edge(source_id);
CREATE INDEX idx_graph_edge_target ON graph_edge(target_id);
CREATE INDEX idx_graph_edge_type ON graph_edge(edge_type);
CREATE INDEX idx_graph_edge_review ON graph_edge(review_id) WHERE review_id IS NOT NULL;
CREATE INDEX idx_graph_edge_source_type ON graph_edge(source_id, edge_type);
CREATE INDEX idx_graph_edge_target_type ON graph_edge(target_id, edge_type);
```

## Organisation & Users (Relational)

```sql
-- ============================================================
-- ORGANISATION & USERS (standard relational)
-- ============================================================

CREATE TABLE organisation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    settings        JSONB DEFAULT '{}',
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
    -- Link to graph node for author-network queries
    graph_node_id   UUID REFERENCES graph_node(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE organisation_member (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    user_id         UUID NOT NULL REFERENCES app_user(id),
    role            VARCHAR(50) NOT NULL DEFAULT 'member',
    UNIQUE (organisation_id, user_id)
);

CREATE INDEX idx_org_member_org ON organisation_member(organisation_id);
```

## Review & References (Relational)

```sql
-- ============================================================
-- REVIEW PROJECT (relational)
-- ============================================================

CREATE TABLE review (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    title           VARCHAR(500) NOT NULL,
    description     TEXT,
    review_type     VARCHAR(50) NOT NULL DEFAULT 'systematic_review',
    status          VARCHAR(50) NOT NULL DEFAULT 'protocol',
    protocol        JSONB DEFAULT '{}',
    config          JSONB DEFAULT '{}',
    -- Link to graph node for cross-review queries
    graph_node_id   UUID REFERENCES graph_node(id),
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
    UNIQUE (review_id, user_id)
);

-- ============================================================
-- REFERENCES (relational, with graph_node_id link)
-- ============================================================

CREATE TABLE reference (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    review_id       UUID NOT NULL REFERENCES review(id) ON DELETE CASCADE,
    doi             VARCHAR(255),
    pmid            VARCHAR(20),
    title           TEXT NOT NULL,
    abstract        TEXT,
    authors         TEXT,
    journal         VARCHAR(500),
    publication_year INTEGER,
    language_code   VARCHAR(10),
    document_type   VARCHAR(50),
    metadata        JSONB DEFAULT '{}',
    -- Link to graph node for graph traversal
    graph_node_id   UUID REFERENCES graph_node(id),
    -- Deduplication
    is_duplicate    BOOLEAN DEFAULT FALSE,
    master_ref_id   UUID REFERENCES reference(id),
    -- Import tracking
    import_source   VARCHAR(100),
    import_batch_id UUID,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_reference_review ON reference(review_id);
CREATE INDEX idx_reference_doi ON reference(doi) WHERE doi IS NOT NULL;
CREATE INDEX idx_reference_pmid ON reference(pmid) WHERE pmid IS NOT NULL;
CREATE INDEX idx_reference_graph ON reference(graph_node_id) WHERE graph_node_id IS NOT NULL;
```

## Screening (Relational)

```sql
-- ============================================================
-- SCREENING (relational — same as hybrid model)
-- ============================================================

CREATE TABLE screening_decision (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    review_id       UUID NOT NULL REFERENCES review(id) ON DELETE CASCADE,
    reference_id    UUID NOT NULL REFERENCES reference(id) ON DELETE CASCADE,
    phase           VARCHAR(20) NOT NULL,
    reviewer_id     UUID NOT NULL REFERENCES app_user(id),
    decision        VARCHAR(20) NOT NULL,
    exclusion_reason VARCHAR(50),
    notes           TEXT,
    decided_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (review_id, reference_id, phase, reviewer_id)
);

CREATE INDEX idx_screening_review_phase ON screening_decision(review_id, phase);

CREATE TABLE screening_outcome (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    review_id       UUID NOT NULL REFERENCES review(id) ON DELETE CASCADE,
    reference_id    UUID NOT NULL REFERENCES reference(id) ON DELETE CASCADE,
    phase           VARCHAR(20) NOT NULL,
    final_decision  VARCHAR(20) NOT NULL,
    exclusion_reason VARCHAR(50),
    had_conflict    BOOLEAN DEFAULT FALSE,
    resolved_by     UUID REFERENCES app_user(id),
    resolved_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (review_id, reference_id, phase)
);

CREATE TABLE screening_prediction (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    review_id       UUID NOT NULL REFERENCES review(id) ON DELETE CASCADE,
    reference_id    UUID NOT NULL REFERENCES reference(id) ON DELETE CASCADE,
    phase           VARCHAR(20) NOT NULL,
    relevance_score NUMERIC(5,4) NOT NULL,
    model_info      JSONB NOT NULL,
    predicted_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_screening_pred_score ON screening_prediction(review_id, relevance_score DESC);
```

## Documents, Extraction, RoB, GRADE (Relational)

```sql
-- ============================================================
-- DOCUMENTS
-- ============================================================

CREATE TABLE document (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    reference_id    UUID NOT NULL REFERENCES reference(id) ON DELETE CASCADE,
    file_name       VARCHAR(255) NOT NULL,
    file_type       VARCHAR(20) NOT NULL,
    storage_path    VARCHAR(1000) NOT NULL,
    full_text       TEXT,
    uploaded_by     UUID NOT NULL REFERENCES app_user(id),
    uploaded_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_document_ref ON document(reference_id);

-- ============================================================
-- DATA EXTRACTION (JSONB hybrid)
-- ============================================================

CREATE TABLE extraction_template (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    review_id       UUID REFERENCES review(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    schema          JSONB NOT NULL,
    created_by      UUID NOT NULL REFERENCES app_user(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE extraction_record (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    review_id       UUID NOT NULL REFERENCES review(id) ON DELETE CASCADE,
    reference_id    UUID NOT NULL REFERENCES reference(id) ON DELETE CASCADE,
    template_id     UUID NOT NULL REFERENCES extraction_template(id),
    extractor_id    UUID NOT NULL REFERENCES app_user(id),
    status          VARCHAR(50) NOT NULL DEFAULT 'in_progress',
    values          JSONB NOT NULL DEFAULT '{}',
    completed_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (review_id, reference_id, template_id, extractor_id)
);

CREATE INDEX idx_extraction_review ON extraction_record(review_id);

-- ============================================================
-- RISK OF BIAS (JSONB hybrid)
-- ============================================================

CREATE TABLE rob_template (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    version         VARCHAR(20) NOT NULL,
    schema          JSONB NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE rob_assessment (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    review_id       UUID NOT NULL REFERENCES review(id) ON DELETE CASCADE,
    reference_id    UUID NOT NULL REFERENCES reference(id) ON DELETE CASCADE,
    template_id     UUID NOT NULL REFERENCES rob_template(id),
    assessor_id     UUID NOT NULL REFERENCES app_user(id),
    overall_judgement VARCHAR(20),
    assessment      JSONB NOT NULL DEFAULT '{}',
    status          VARCHAR(50) NOT NULL DEFAULT 'in_progress',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (review_id, reference_id, template_id, assessor_id)
);

-- ============================================================
-- GRADE & META-ANALYSIS (JSONB)
-- ============================================================

CREATE TABLE grade_assessment (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    review_id       UUID NOT NULL REFERENCES review(id) ON DELETE CASCADE,
    assessment      JSONB NOT NULL,
    assessor_id     UUID NOT NULL REFERENCES app_user(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE meta_analysis (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    review_id       UUID NOT NULL REFERENCES review(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    config          JSONB NOT NULL,
    results         JSONB,
    created_by      UUID NOT NULL REFERENCES app_user(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

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
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_review ON audit_log(review_id, created_at);
```

## Graph Query Examples

```sql
-- ============================================================
-- GRAPH QUERY EXAMPLES
-- ============================================================

-- 1. CITATION CHAINING: Forward snowball search
-- "Find all papers that cite any of our included studies"
WITH included_studies AS (
    SELECT r.graph_node_id
    FROM reference r
    JOIN screening_outcome so ON so.reference_id = r.id
    WHERE so.review_id = 'review-uuid'
      AND so.phase = 'full_text'
      AND so.final_decision = 'include'
)
SELECT DISTINCT
    gn.label AS citing_paper,
    gn.properties->>'doi' AS doi,
    gn.properties->>'publication_year' AS year
FROM graph_edge ge
JOIN included_studies inc ON ge.target_id = inc.graph_node_id
JOIN graph_node gn ON gn.id = ge.source_id
WHERE ge.edge_type = 'cites'
  AND gn.node_type = 'reference';

-- 2. CITATION CHAINING: Backward snowball search
-- "Find all papers cited by our included studies"
WITH included_studies AS (
    SELECT r.graph_node_id
    FROM reference r
    JOIN screening_outcome so ON so.reference_id = r.id
    WHERE so.review_id = 'review-uuid'
      AND so.phase = 'full_text'
      AND so.final_decision = 'include'
)
SELECT DISTINCT
    gn.label AS cited_paper,
    gn.properties->>'doi' AS doi,
    COUNT(*) AS times_cited_by_included
FROM graph_edge ge
JOIN included_studies inc ON ge.source_id = inc.graph_node_id
JOIN graph_node gn ON gn.id = ge.target_id
WHERE ge.edge_type = 'cites'
  AND gn.node_type = 'reference'
GROUP BY gn.id, gn.label, gn.properties->>'doi'
ORDER BY times_cited_by_included DESC;

-- 3. EVIDENCE GAP MAP
-- "Show the PICO coverage matrix: which intervention-outcome pairs have studies?"
SELECT
    interv.label AS intervention,
    outcome.label AS outcome,
    COUNT(DISTINCT ref_node.id) AS num_studies
FROM graph_edge e_interv
JOIN graph_node interv ON interv.id = e_interv.target_id
    AND interv.node_type = 'pico_element'
    AND interv.properties->>'element_type' = 'intervention'
JOIN graph_node ref_node ON ref_node.id = e_interv.source_id
    AND ref_node.node_type = 'reference'
JOIN graph_edge e_outcome ON e_outcome.source_id = ref_node.id
    AND e_outcome.edge_type = 'studies_pico'
JOIN graph_node outcome ON outcome.id = e_outcome.target_id
    AND outcome.node_type = 'pico_element'
    AND outcome.properties->>'element_type' = 'outcome'
WHERE e_interv.edge_type = 'studies_pico'
  AND e_interv.review_id = 'review-uuid'
GROUP BY interv.label, outcome.label
ORDER BY num_studies DESC;

-- 4. AUTHOR NETWORK
-- "Find authors who appear in multiple included studies"
WITH included_refs AS (
    SELECT r.graph_node_id
    FROM reference r
    JOIN screening_outcome so ON so.reference_id = r.id
    WHERE so.review_id = 'review-uuid'
      AND so.phase = 'full_text'
      AND so.final_decision = 'include'
)
SELECT
    author.label AS author_name,
    author.properties->>'orcid' AS orcid,
    COUNT(DISTINCT ir.graph_node_id) AS papers_in_review,
    ARRAY_AGG(DISTINCT (ref_node.properties->>'publication_year')) AS years
FROM graph_edge ge
JOIN included_refs ir ON ge.source_id = ir.graph_node_id
JOIN graph_node ref_node ON ref_node.id = ge.source_id
JOIN graph_node author ON author.id = ge.target_id
    AND author.node_type = 'author'
WHERE ge.edge_type = 'authored_by'
GROUP BY author.id, author.label, author.properties->>'orcid'
HAVING COUNT(DISTINCT ir.graph_node_id) > 1
ORDER BY papers_in_review DESC;

-- 5. MeSH HIERARCHY TRAVERSAL
-- "Find all references tagged with any child of 'Anxiety Disorders' (D001008)"
WITH RECURSIVE mesh_descendants AS (
    -- Base case: the root MeSH term
    SELECT id FROM graph_node
    WHERE node_type = 'mesh_term'
      AND properties->>'descriptor_ui' = 'D001008'
    UNION ALL
    -- Recursive case: children in the MeSH tree
    SELECT gn.id
    FROM graph_edge ge
    JOIN graph_node gn ON gn.id = ge.source_id
    JOIN mesh_descendants md ON ge.target_id = md.id
    WHERE ge.edge_type = 'mesh_parent'
      AND gn.node_type = 'mesh_term'
)
SELECT DISTINCT
    ref_node.label AS reference_title,
    ref_node.properties->>'doi' AS doi
FROM graph_edge ge
JOIN graph_node ref_node ON ref_node.id = ge.source_id
    AND ref_node.node_type = 'reference'
JOIN mesh_descendants md ON ge.target_id = md.id
WHERE ge.edge_type = 'tagged_with';

-- 6. SEMANTIC SIMILARITY CLUSTERS
-- "Find clusters of similar studies based on embedding similarity"
SELECT
    gn1.label AS study_a,
    gn2.label AS study_b,
    (ge.properties->>'similarity_score')::NUMERIC AS similarity
FROM graph_edge ge
JOIN graph_node gn1 ON gn1.id = ge.source_id
JOIN graph_node gn2 ON gn2.id = ge.target_id
WHERE ge.edge_type = 'related_to'
  AND ge.review_id = 'review-uuid'
  AND (ge.properties->>'similarity_score')::NUMERIC > 0.8
ORDER BY similarity DESC;

-- 7. CONFLICT OF INTEREST DETECTION
-- "Find reviewers who are also authors of included studies"
SELECT
    u.display_name AS reviewer,
    author.label AS author_name,
    ref_node.label AS paper_title
FROM app_user u
JOIN graph_node author ON author.id = u.graph_node_id
JOIN graph_edge ge ON ge.target_id = author.id AND ge.edge_type = 'authored_by'
JOIN graph_node ref_node ON ref_node.id = ge.source_id AND ref_node.node_type = 'reference'
JOIN reference r ON r.graph_node_id = ref_node.id
JOIN review_team_member rtm ON rtm.user_id = u.id AND rtm.review_id = r.review_id
WHERE r.review_id = 'review-uuid';
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Graph Layer | 2 | Generic node + edge tables with JSONB properties |
| Organisation & Users | 3 | Standard relational with graph_node_id links |
| Review & References | 3 | Relational with graph_node_id for graph traversal |
| Screening | 3 | Decisions, outcomes, AI predictions |
| Documents | 1 | PDF storage |
| Extraction | 2 | Template + record with JSONB values |
| Risk of Bias | 2 | Template + assessment with JSONB |
| GRADE & Meta-Analysis | 2 | JSONB-based evidence profiles |
| Audit | 1 | Change tracking |
| **Total** | **19** | Plus graph with potentially millions of nodes/edges |

---

## Key Design Decisions

1. **Graph as a separate layer, not a replacement** — the relational tables handle the systematic review workflow (screening, extraction, RoB, GRADE) with standard SQL patterns. The graph layer adds analytical power for citation networks, PICO mapping, and author analysis without complicating the core workflow.

2. **Bridge columns (`graph_node_id`)** — relational entities (`reference`, `review`, `app_user`) link to their graph representations via a `graph_node_id` foreign key, enabling seamless joins between the two layers.

3. **Global vs. review-scoped nodes** — author, institution, journal, and MeSH term nodes are global (shared across reviews), while reference and PICO element nodes can be scoped to a review. This enables cross-review knowledge synthesis while maintaining review-level isolation.

4. **Typed edges with JSONB properties** — edge types (`cites`, `authored_by`, `studies_pico`, `related_to`) provide structure, while JSONB properties store edge-specific metadata (citation intent, author position, similarity score) without separate tables for each relationship type.

5. **Citation intent classification** — edges of type `cites` include a `properties.intent` field (background, method, result_comparison) aligned with Semantic Scholar's citation intent classification, enabling more nuanced citation analysis than simple citation counts.

6. **Evidence gap mapping as a graph query** — PICO elements as graph nodes connected to references via `studies_pico` edges makes evidence gap mapping a straightforward aggregation query rather than a complex join across extraction records.

7. **MeSH hierarchy as a graph** — rather than storing tree numbers as strings and parsing them, MeSH parent-child relationships are explicit `mesh_parent` edges, making recursive traversal clean and efficient.

8. **Semantic similarity edges** — AI-computed embedding similarities between references are stored as weighted `related_to` edges, enabling cluster detection and visual maps of study relationships.

9. **Conflict-of-interest detection** — linking `app_user` to `graph_node` (author) enables a simple join to detect when a review team member is also an author of an included study — a compliance requirement for Cochrane reviews.

10. **PostgreSQL-native** — the entire graph layer runs in PostgreSQL using standard tables, indexes, and recursive CTEs. No external graph database (Neo4j, etc.) is required, simplifying deployment and backup while still enabling powerful graph queries.
