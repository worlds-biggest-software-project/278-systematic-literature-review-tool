# Standards & API Reference

> Project: Systematic Literature Review Tool · Generated: 2026-05-03

## Industry Standards & Specifications

### Reporting & Methodology Standards

**PRISMA 2020 (Preferred Reporting Items for Systematic reviews and Meta-Analyses)**
- URL: https://www.prisma-statement.org/prisma-2020
- The gold-standard reporting guideline for systematic reviews, comprising a 27-item checklist and standardised flow diagram templates. Any tool that generates review outputs or audits completeness should support PRISMA 2020 checklist validation and automated flowchart generation.

**PRISMA Abstract Checklist**
- URL: https://www.prisma-statement.org/prisma-2020-checklist
- A 12-item checklist for systematic review abstracts, companion to the main PRISMA 2020 statement. Tools generating structured abstracts or review summaries should validate against this checklist. An interactive Shiny-based checker is available at https://prisma.shinyapps.io/checklist/

**EQUATOR Network Reporting Guidelines**
- URL: https://www.equator-network.org/reporting-guidelines/prisma/
- Umbrella network of evidence reporting standards including CONSORT (randomised trials), STROBE (observational studies), and dozens more. A systematic review tool should be aware of these standards when classifying included study types and guiding data extraction template design.

**Cochrane Handbook for Systematic Reviews of Interventions**
- URL: https://training.cochrane.org/handbook
- Definitive methodological reference for clinical systematic reviews, maintained by the Cochrane Collaboration. Defines standard phases (search, screening, extraction, synthesis) that tools must support to be Cochrane-compatible.

**GRADE (Grading of Recommendations Assessment, Development and Evaluation)**
- URL: https://www.gradeworkinggroup.org/
- Framework for rating the certainty of a body of evidence across six domains (risk of bias, inconsistency, indirectness, imprecision, publication bias, and large effects). Tools that support evidence synthesis should produce GRADE Summary of Findings tables and Evidence Profiles, using GRADEpro GDT (https://www.gradepro.org/) as the reference implementation.

**AMSTAR-2**
- URL: https://amstar.ca/Amstar_Checklist.php
- A 16-item critical appraisal checklist for systematic reviews of healthcare interventions. Tools that assess or quality-check systematic reviews for inclusion in meta-reviews should implement AMSTAR-2 rating workflows.

**CEE Guidelines for Systematic Reviews in Environmental Management**
- URL: https://www.environmentalevidence.org/guidelines-and-policies/systematic-review-and-map-guidelines
- Collaboration for Environmental Evidence standards for non-clinical systematic reviews and maps. Relevant for tools targeting environmental, social policy, or educational evidence synthesis domains.

**PROSPERO International Prospective Register**
- URL: https://www.crd.york.ac.uk/prospero/
- International registry for systematic review protocols, maintained by the University of York Centre for Reviews and Dissemination. Tools should support PROSPERO registration number tracking and protocol export to enable pre-registration, a requirement for many journals and funders.

### Citation & Data Format Standards

**RIS (Research Information Systems) File Format**
- URL: https://en.wikipedia.org/wiki/RIS_(file_format)
- The de facto standard tagged format for exchanging bibliographic citations between databases (Web of Science, Scopus, IEEE Xplore, PubMed via export) and review management platforms. All systematic review tools must support RIS import and export.

**BibTeX Format**
- URL: https://www.bibtex.com/
- Plain-text bibliography format used by LaTeX and widely adopted in academic tooling. Required for interoperability with Zotero, Mendeley, and citation managers commonly used by systematic reviewers.

**NBIB (PubMed/NCBI Format)**
- URL: https://www.ncbi.nlm.nih.gov/books/NBK25499/
- NCBI's native export format for PubMed citations, using two-letter field tags (e.g., FAU for Full Author). Tools sourcing records from PubMed should parse NBIB and convert to RIS or internal formats.

**MeSH (Medical Subject Headings) Controlled Vocabulary**
- URL: https://www.nlm.nih.gov/mesh/meshhome.html
- Hierarchical thesaurus of ~30,000 biomedical terms maintained by the US National Library of Medicine. Systematic review tools targeting clinical or biomedical literature should support MeSH term expansion and tree traversal to ensure comprehensive search coverage.

### Security & Authentication Standards

**OAuth 2.0 (RFC 6749)**
- URL: https://datatracker.ietf.org/doc/html/rfc6749
- The standard protocol for delegated authorisation, enabling secure third-party integrations with databases (e.g., institutional Scopus, Web of Science access). Required for any tool that allows users to connect their institutional database subscriptions or reference manager accounts.

**OpenID Connect (OIDC)**
- URL: https://openid.net/connect/
- Identity layer on top of OAuth 2.0, providing standardised user authentication for institutional SSO (SAML federations, university identity providers). Enterprise and academic tools commonly require OIDC-based login.

**GDPR (General Data Protection Regulation)**
- URL: https://gdpr.eu/
- EU regulation governing personal data; applicable to systematic review platforms storing researcher data, author metadata from papers, and institutional usage logs. Tools operating in Europe must implement data minimisation, right-to-erasure, and lawful basis for processing.

**HIPAA (Health Insurance Portability and Accountability Act)**
- URL: https://www.hhs.gov/hipaa/index.html
- US federal regulation for protected health information. Relevant if the platform is used for clinical trial evidence synthesis involving patient-level data or is deployed within healthcare organisations.

### Data Model & API Specifications

**JSON-LD (JSON for Linked Data)**
- URL: https://json-ld.org/
- Serialisation format for structured linked data; used by GRADEpro GDT for exporting Summary of Findings tables and evidence profiles. An interoperable systematic review tool should support JSON-LD import/export for GRADE data.

**OpenAPI Specification (OAS 3.1)**
- URL: https://spec.openapis.org/oas/v3.1.0
- Industry standard for describing RESTful APIs; both Elicit and DistillerSR expose REST APIs documented using OpenAPI-compatible conventions. New tools should publish an OpenAPI 3.1 schema for all programmatic interfaces.

**PICO Data Model**
- URL: https://www.cochranelibrary.com/about/pico-search
- Structured Population–Intervention–Comparator–Outcome framework for framing clinical research questions. Tools that support structured extraction templates should model PICO as a first-class data entity to enable consistent cross-study evidence mapping.

---

## Similar Products — Developer Documentation & APIs

### Elicit
- **Description:** AI-native systematic review platform with semantic and Boolean search across 138 million+ papers, plus data extraction and evidence synthesis.
- **API Documentation:** https://docs.elicit.com/
- **API Launch:** March 2026; supports searching the full Elicit paper index or PubMed; returns paper metadata and generated reports.
- **SDKs/Libraries:** REST API with JSON responses; no official SDK yet (as of May 2026).
- **Developer Guide:** https://agentsapis.com/elicit-api/
- **Standards:** REST/JSON; semantic query rewriting via LLM; Boolean query support.
- **Authentication:** API key.

### DistillerSR
- **Description:** Evidence Partners' systematic review management platform used by pharma, HTA, and clinical teams for full-workflow evidence synthesis.
- **API Documentation:** https://apidocs.evidencepartners.com/
- **SDKs/Libraries:** REST API; DistillerSR API documentation on the Evidence Partners developer portal.
- **Developer Guide:** https://help.distillersr.com/hc/en-us/articles/19822095909389-DistillerSR-API-Documentation
- **Standards:** REST/JSON; supports programmatic reference import/export, screening decisions, and extraction data retrieval.
- **Authentication:** API key; institutional access required.

### Semantic Scholar Academic Graph API
- **Description:** Free AI-powered academic search engine covering ~200 million papers across all disciplines, maintained by the Allen Institute for AI.
- **API Documentation:** https://api.semanticscholar.org/api-docs/
- **SDKs/Libraries:** Unofficial Python client (semanticscholar on PyPI); R client (semanticscholar on CRAN).
- **Developer Guide:** https://www.semanticscholar.org/product/api/tutorial
- **Standards:** REST/JSON; supports keyword and semantic search; bulk search endpoint for high-volume retrieval.
- **Authentication:** Public endpoints (rate-limited to 1,000 req/s shared); API key available for higher limits.

### OpenAlex
- **Description:** Fully open bibliographic database with 474+ million academic works (as of February 2026), including datasets, software, and grey literature; successor to Microsoft Academic.
- **API Documentation:** https://developers.openalex.org/
- **SDKs/Libraries:** openalexR (R; rOpenSci); pyalex (Python); unofficial JavaScript clients.
- **Developer Guide:** https://docs.openalex.org/api-entities/works/work-object
- **Standards:** REST/JSON; free up to 100,000 requests/day; supports filtering by institution, funder, topic, and OA status.
- **Authentication:** No authentication required for standard use; polite pool (email in user-agent) for higher rate limits.

### NCBI E-utilities (PubMed / PMC)
- **Description:** NIH's suite of eight server-side programs providing programmatic access to all Entrez databases including PubMed (35+ million citations) and PubMed Central (full-text open access articles).
- **API Documentation:** https://www.ncbi.nlm.nih.gov/home/develop/api/
- **SDKs/Libraries:** Biopython (Entrez module); rentrez (R); unofficial wrappers in Node.js and Go.
- **Developer Guide:** https://pmc.ncbi.nlm.nih.gov/tools/developers/
- **Standards:** XML/JSON; fixed URL syntax for esearch, efetch, elink, einfo, epost, esummary, egquery, espell.
- **Authentication:** API key optional (increases rate limit from 3 to 10 req/s); no registration required for basic use.

### Crossref REST API
- **Description:** Public REST API exposing bibliographic metadata for 150+ million scholarly works deposited by Crossref members; primary source for DOI resolution and citation graph data.
- **API Documentation:** https://www.crossref.org/documentation/retrieve-metadata/rest-api/
- **SDKs/Libraries:** rcrossref (R); habanero (Python); crossref-commons (Python).
- **Developer Guide:** https://github.com/CrossRef/rest-api-doc
- **Standards:** REST/JSON; Swagger documentation at https://api.crossref.org/; content negotiation for multiple metadata formats.
- **Authentication:** No authentication required; polite pool (mailto parameter) recommended for higher rate limits.

### Zotero Web API
- **Description:** RESTful API providing programmatic access to Zotero user and group libraries; widely used for reference management integration in systematic review workflows.
- **API Documentation:** https://www.zotero.org/support/dev/web_api/v3/basics
- **SDKs/Libraries:** pyzotero (Python, includes optional MCP server); Zotero.rb (Ruby); zotero-api-client (JavaScript/Node.js).
- **Developer Guide:** https://www.zotero.org/support/dev/web_api/v3/
- **Standards:** REST; JSON and Atom response formats; supports read/write of items, collections, tags, and attachments.
- **Authentication:** OAuth 2.0 for third-party apps; API key for direct access.

### ASReview (Python Library & REST API)
- **Description:** Open-source active learning platform for systematic review text screening; written in Python (Apache 2.0 licence); exposes a Python API and command-line interface.
- **API Documentation:** https://asreview.readthedocs.io/
- **SDKs/Libraries:** asreview package on PyPI; supports custom feature extractors, classifiers, query strategies, and balancers via a plugin architecture.
- **Developer Guide:** https://github.com/asreview/asreview
- **Standards:** Python API; JSON project files; plugin discovery via setuptools entry points.
- **Authentication:** Local deployment; no remote authentication required for open-source version.

### GRADEpro GDT
- **Description:** Guideline Development Tool for creating GRADE Summary of Findings tables and Evidence Profiles; the standard tool for GRADE-compliant evidence presentation in Cochrane and WHO guidelines.
- **API Documentation:** https://gdt.gradepro.org/app/help/user_guide/index.html
- **SDKs/Libraries:** No public API SDK; JSON-LD export format for inter-tool data exchange.
- **Developer Guide:** https://methods.cochrane.org/gradeing/gradepro-gdt
- **Standards:** JSON-LD export; RevMan 5 format import/export; MAGICapp PICO import from GDT files.
- **Authentication:** Account-based; no public API key system documented.

### arXiv API
- **Description:** Open access to preprints in physics, mathematics, computer science, and related fields; 2+ million papers; important for systematic reviews in technology-adjacent domains.
- **API Documentation:** https://info.arxiv.org/help/api/index.html
- **SDKs/Libraries:** arxiv.py (Python); aRxiv (R).
- **Standards:** XML (Atom feed); no API key required.
- **Authentication:** No authentication required.

---

## Notes

**No Unified Standard for Systematic Review Data Exchange:** As of 2026, there is no universally adopted open data exchange format for complete systematic review projects (search strategies, screening decisions, extraction data, GRADE assessments, and PRISMA flowcharts in a single schema). Each tool uses proprietary formats, though RIS and JSON-LD are common at the citation and evidence table levels respectively. This is a significant gap that an open-source tool could address.

**PROSPERO API:** PROSPERO does not currently offer a public programmatic API for protocol registration or data retrieval. Protocol export for pre-registration must be done manually via the web interface.

**Scopus and Web of Science:** These commercially licensed databases are essential for comprehensive systematic searches but their APIs require institutional subscriptions. Scopus is available via the Elsevier Developer Portal (https://dev.elsevier.com/) and Web of Science via the Clarivate API portal (https://developer.clarivate.com/apis/wos). Both return JSON and require API key authentication with institutional access validation.

**MCP Server Opportunity:** The pyzotero library already ships an optional MCP server exposing Zotero library contents to LLM agents. A systematic review tool built on MCP could expose search, screening, and extraction workflows as tools for AI agents, enabling agentic review automation without bespoke integrations.
