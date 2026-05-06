# Systematic Literature Review Tool

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> An AI-native, open-source platform for conducting rigorous systematic literature reviews with AI-powered paper screening, data extraction, and evidence synthesis.

The Systematic Literature Review Tool helps clinical researchers, health technology assessment teams, evidence synthesis researchers, and regulatory affairs professionals conduct PRISMA-compliant systematic reviews faster and more reliably. It addresses the core pain of evidence synthesis: screening thousands of records, extracting structured data from heterogeneous PDFs, and maintaining methodological rigour, all of which can otherwise take months of manual effort.

---

## Why Systematic Literature Review Tool?

- Gold-standard incumbent Covidence charges $55–$295/mo per review and offers limited AI automation compared to newer tools.
- Enterprise platforms such as LASER AI and AutoLit are custom-priced at $10,000–$50,000+/yr for institutional access, putting rigorous review tooling out of reach for many academic and policy teams.
- Free academic tools like Abstrackr, CADIMA, and Colandr have dated interfaces and limited extraction support.
- AI-native entrants (Elicit, Rayyan, Paperguide) demonstrate 30–80% time savings, but no single open-source platform combines best-in-class extraction, screening, and full PRISMA workflow support.
- Living reviews, semantic deduplication, and evidence gap mapping remain underserved across the incumbent landscape.

---

## Key Features

### Core Review Workflow

- Search strategy documentation with query history
- Screening workflow covering title/abstract and full-text phases
- Conflict management and resolution workflows
- User management and screening assignment
- PDF annotation and highlighting

### Data Extraction & Quality Assessment

- Customisable data extraction templates
- Cochrane Risk of Bias assessment tool
- Citation deduplication and matching
- Basic statistics and summary generation
- Export to PRISMA and other reporting formats

### AI-Augmented Review

- AI-powered title/abstract screening using active learning
- AI-assisted data extraction from PDF and text
- Risk of bias automation
- Study quality meta-analysis
- Living review update workflows

### Collaboration & Integrations

- Collaborative workflow with real-time updates
- Integration with publication databases (PubMed, Scopus)
- Meta-analysis calculation and visualization

### Advanced Synthesis (Backlog)

- Network meta-analysis visualization
- Semantic search understanding research intent
- Automated PRISMA compliance checking
- NLP-based question analysis and keyword suggestion
- Multi-language paper support and translation
- Bayesian and frequentist meta-analysis
- Subgroup analysis automation
- Publication bias detection

---

## AI-Native Advantage

An active learning screening engine continuously updates document relevance predictions as reviewers make inclusion/exclusion decisions, reaching confident recommendations with fewer human screens than static classifiers. Automated structured data extraction pulls values from PDFs, including tables, figures, and embedded supplementary files, mapped directly to user-defined extraction templates. Semantic deduplication identifies duplicate records across databases despite differing titles, author orderings, or publication formats. Evidence gap mapping visualises included studies across PICO dimensions, and living review automation monitors new publications and alerts teams when records meet inclusion criteria.

---

## Tech Stack & Deployment

The project targets browser-based collaborative workflows with API integrations to publication databases such as PubMed and Scopus. It aligns with established methodological standards including PRISMA 2020, the Cochrane Handbook, PROSPERO registration, GRADE, AMSTAR-2, the CEE Guidelines, and the broader EQUATOR Network reporting guidelines. Deployment modes and SDK details will be specified in the design phase.

---

## Market Context

The global systematic review and meta-analysis services and software market is estimated at USD 800 million to USD 1.5 billion, growing at approximately 12–18% CAGR (research.md). Incumbent pricing ranges from free academic tools to Covidence at $55–$295/mo per review and enterprise platforms at $10,000–$50,000+/yr. Primary buyers include clinical and health researchers, HTA teams at NICE, CADTH, and IQWiG, environmental and policy evidence synthesis researchers, regulatory affairs teams in pharma and medtech, and academics meeting journal review requirements.

---

## Project Status

> This project is in the **research and specification phase**.  
> Contributions, feedback, and domain expertise are welcome.

---

## Contributing

We welcome contributions from developers, domain experts, and potential users.
See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

**Important:** All contributions must be your own original work or clearly attributed
open-source material with a compatible licence. Copyright infringement and licence
violations will not be tolerated and will result in immediate removal of the offending
contribution. If you are unsure whether a piece of code, text, or other material is
safe to contribute, open an issue and ask before submitting.

---

## Licence

Licence to be determined. See [discussion](#) for context.
