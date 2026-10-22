# CurriculumIQ — Knowledge Graph-Based Curriculum Intelligence for Industry Skill Alignment

An MCA Minor Project (5 credits / 100 marks) that builds a real, working curriculum-intelligence
platform: it extracts skills from academic curriculum and industry job descriptions using a
hybrid NLP pipeline (NLTK lexical matching + spaCy named-entity recognition and word-vector
semantic matching), stores curriculum ↔ industry relationships as a Neo4j knowledge graph, runs
NetworkX graph analytics for industry demand and importance, and surfaces skill gaps and
explainable recommendations through a FastAPI + React application.

---

## 1. Problem Statement

University curricula are updated on a multi-year cycle, while industry skill requirements shift
continuously. There is no systematic, data-driven way to see *which* parts of a curriculum are
falling behind industry demand, *by how much*, and *what specifically* should be added. This
project builds that system for an MCA program.

## 2. Objectives

1. Develop a domain-specific knowledge graph linking academic course modules with evolving
   industry job requirements.
2. Implement a recommendation interface that highlights curriculum skill gaps for academic
   coordinators.
3. Evaluate mapping accuracy and gap-extraction efficiency against industry job-description
   datasets.
4. Apply graph-theoretic analytics — degree centrality **and PageRank** — over the job–skill
   network so industry skill importance reflects graph structure, not just mention counts.
5. Surface structural misalignment with a **Program Gap Report** (a program-level rollup with a
   course-wise breakdown underneath) and make the results shareable as downloadable **CSV/PDF**
   reports.

## 3. Features

- Hybrid NLP skill-extraction pipeline: NLTK (clean → tokenize → stopword removal → lemmatize →
  n-gram/regex skill matching) **plus** spaCy Named Entity Recognition and word-vector semantic
  matching, all normalized to canonical skills (`nlp_service.py`, `semantic_service.py`)
- Neo4j knowledge graph linking `Program → Course → Module → Topic → Skill` on the curriculum
  side and `Industry → JobRole → JobDescription → Skill` on the industry side, sharing canonical
  `Skill` nodes
- NetworkX-based analytics: job–skill bipartite graph, degree centrality as an industry-demand
  signal, PageRank as a complementary centrality signal, graph statistics
- Skill-gap engine: basic alignment (`covered / required × 100`) and a documented weighted
  alignment formula based on industry demand
- Rule-based, explainable recommendation engine with HIGH / MEDIUM / LOW priority
- Real evaluation: Precision, Recall, F1 (micro-averaged) against a hand-labeled ground-truth set,
  plus a Mapping Accuracy metric for the normalization step
- A full React dashboard: Dashboard, Curriculum, Industry Jobs, Skills, interactive Knowledge
  Graph explorer, Skill Gap Analysis, Program Gap Report (program-level + course-wise breakdown),
  Recommendations, Reports
- Report export: the program-level gap report downloads as **CSV** or **PDF** (`GET
  /api/reports/export/course-gaps`)

## 4. Technology Stack

| Layer | Technology |
|---|---|
| Backend | Python, FastAPI |
| NLP | NLTK + spaCy (NER, word vectors) |
| Knowledge Graph | Neo4j |
| Graph Analysis | NetworkX |
| Frontend | React.js (Vite), Recharts, d3-force |
| Communication | REST APIs (JSON over HTTP) |

## 5. Architecture

```
Curriculum JSON ─┐                              ┌─ Job Description JSON
                 v                              v
        ┌──────────────────────────────────────────────┐
        │     Hybrid NLP Pipeline (nlp_service.py)       │
        │  NLTK: clean → tokenize → stopwords →          │
        │        lemmatize → n-gram/regex skill match     │
        │  spaCy: NER entities + word-vector semantic     │
        │        match → canonical Skill (dedup, routes)  │
        └───────────────────────┬────────────────────────┘
                                 v
                   ┌─────────────────────────┐
                   │   Neo4j Knowledge Graph   │
                   │ Program..Skill..JobRole   │
                   └─────────────┬─────────────┘
                     ┌───────────┴────────────┐
                     v                        v
          ┌────────────────────┐   ┌────────────────────┐
          │   NetworkX Layer    │   │  Gap Analysis Engine │
          │ (centrality/demand) │──>│ (covered/missing/    │
          └──────────┬──────────┘   │  alignment)          │
                     └───────────┬───┴──────────┬──────────┘
                                 v               v
                       ┌────────────────────────────────┐
                       │     Recommendation Engine        │
                       └─────────────────┬────────────────┘
                                         v
                                    FastAPI (REST)
                                         v
                                     React.js
                                Curriculum Intelligence
                                     Dashboard
```

## 6. Knowledge Graph Design

**Nodes:** `Program`, `Course`, `Module`, `Topic`, `Skill`, `Industry`, `JobRole`, `JobDescription`

**Relationships:** `HAS_COURSE`, `HAS_MODULE`, `HAS_TOPIC`, `COVERS`, `HAS_ROLE`,
`HAS_DESCRIPTION`, `REQUIRES`

Curriculum and industry both link into the **same canonical `Skill` node** (e.g. `React.js`),
which is what makes cross-domain analysis possible: a single Cypher traversal can walk from a
`Course` to its covered skills and from there to every `JobRole` that requires them.

Constraints (in `graph_service.py::CONSTRAINTS`) enforce uniqueness on `Skill.canonical_name` and
on every node's `id`, so ingestion is idempotent (`MERGE`, not `CREATE`).

## 7. Hybrid NLP Pipeline

See `backend/app/services/nlp_service.py` (orchestration) and
`backend/app/services/semantic_service.py` (spaCy layer). Skill extraction is a **hybrid of three
matching routes**, all normalized through the same controlled skill dictionary
(`datasets/skill_dictionary/skills.json`) into one deduplicated set of canonical `Skill` names:

**Route 1 — NLTK lexical (baseline, always on).** Text is cleaned, tokenized, stripped of
stopwords, and lemmatized with `WordNetLemmatizer`. Matching happens two ways: n-grams of the
lemmatized tokens (for plain word/phrase skills like "Machine Learning"), and a punctuation-safe
regex pass over the cleaned text (for skills like `React.js`, `C++`, `CI/CD` that generic
tokenizers would mangle).

**Route 2 — spaCy Named Entity Recognition (context-aware).** spaCy's statistical model tags
spans such as `PRODUCT`, `LANGUAGE`, `ORG`, `WORK_OF_ART`, `NORP`. Each entity span is treated as a
*candidate* skill mention and normalized through the skill dictionary — so a span tagged ORG that
happens to be "React.js" still lands on the canonical Skill node. This is statistical, context-
aware detection rather than string matching; noisy ORG/NORP spans are harmless because the
dictionary normalization rejects anything that is not an actual skill alias.

**Route 3 — Semantic (fuzzy + word-vector) matching.** Candidate n-grams the dictionary does not
know are compared to every canonical skill with two complementary signals:

- **Edit distance** (`difflib.SequenceMatcher`): near-miss spellings like `"Kubernetees"` →
  `Kubernetes` are rescued, but only for out-of-vocabulary words / multi-word phrases so ordinary
  English ("network", "data") can never be remapped to a skill by accident.
- **Word-vector confirmation** (`en_core_web_md`, 300-dim embeddings): the candidate's embedding
  is compared to the matched skill's and recorded as an extra confidence signal; for multi-word
  candidates a vector similarity ≥ 0.95 can alone accept a match.

Every result keeps **provenance** — which route found each skill (`skills_by_route`, plus
`ner_entities` and `semantic_matches` for the viva demo) — so the pipeline is auditable, and the
NLTK route remains a clean baseline. Honest calibration note (also documented in the source):
`en_core_web_md` ships only 20k vector rows, so OOV tech jargon often hashes to a zero vector;
the accept decision is therefore dominated by edit-distance + NER, with vectors used as
confirmation — all signals are recorded so the tradeoff can be discussed.

**Graceful degradation.** spaCy imports are lazy. If the library or model is missing, every
entry point reports the layer as unavailable and the NLTK-only pipeline keeps working (same
philosophy as the PageRank fallback in `networkx_service.py`).

**Configuration** (see `backend/app/config.py`, overridable via `.env`):

| Setting | Default | Meaning |
|---|---|---|
| `NLP_USE_SPACY` | `true` | enable the spaCy NER + semantic routes |
| `NLP_SEMANTIC_ENABLED` | `true` | enable the word-vector semantic matcher |
| `NLP_SEMANTIC_THRESHOLD` | `0.90` | minimum accept score for semantic matches |

## 8. NetworkX Usage

Neo4j answers relationship queries. NetworkX (`networkx_service.py`) builds an **in-memory
bipartite graph** of `JobDescription <-> Skill` edges and computes:
- degree centrality per skill (used as the industry-demand weight)
- PageRank per skill (a complementary centrality signal that captures how "central" a skill is
  in the job–skill network, not just how many jobs mention it)
- job-skill graph statistics (density, average skills per job)

These numbers feed directly into the weighted alignment formula and recommendation priority — a
real graph-theoretic computation, not just a stored count. If `numpy`/`scipy` are not installed,
PageRank falls back to a pure-Python power-iteration implementation (`_pagerank` in
`networkx_service.py`), so no extra dependencies are required.

## 9. Skill-Gap Algorithm

```
covered  = curriculum_skills ∩ industry_skills
missing  = industry_skills − curriculum_skills
critical = missing skills with demand% >= 70

basic_alignment    = covered_count / required_count × 100
weighted_alignment = Σ demand%(covered) / Σ demand%(required) × 100
```

## 10. Recommendation Algorithm

For every missing skill: `demand% >= 70 → HIGH`, `40–69% → MEDIUM`, `< 40% → LOW`. Recommendation
text is templated per skill **category** (Frontend, Backend, DevOps & Cloud, etc.) — see
`recommendation_service.CATEGORY_TEMPLATES` — so the logic stays a transparent lookup table, not
free-form generated text.

## 11. Evaluation

Run against `datasets/ground_truth/labeled_jobs.json` (40 hand-labeled samples):

- **Precision / Recall / F1** (micro-averaged) measure extraction quality
- **Mapping Accuracy** measures normalization correctness: every alias in the skill dictionary is
  checked against its declared canonical skill

On the shipped datasets this currently computes to **Precision ≈ 99.26%, Recall ≈ 99.51%, F1 ≈
99.38% (403 true positives / 3 false positives / 2 false negatives), Mapping Accuracy = 100%** —
all computed live by `GET /api/evaluation`, never hard-coded. (The tiny number of false
positives/negatives comes from generic filler phrases in the templated job descriptions — e.g. a
ground-truth labeler not listing CSS in a Full-Stack JD whose text mentions "Tailwind CSS and
CSS" — a good real-world example of the precision/recall tradeoff to discuss in a viva.) The same
numbers hold with the spaCy layer on (the hybrid evaluation above) and with `NLP_USE_SPACY=false`
(NLTK-only baseline), and the semantic matcher is what rescues near-miss spellings such as
"Kubernetees" → Kubernetes in `tests/test_semantic_nlp.py`.

Two additional report endpoints turn the same live analysis into downloadable files:

- `GET /api/analysis/program-gap?role_id=...` — program-level gap report with a course-wise
  breakdown under each program
- `GET /api/analysis/course-gaps?role_id=...` — every course ranked against one target role
- `GET /api/reports/export/course-gaps?role_id=...&format=csv|pdf` — export of the above as CSV
  or PDF (see `report_service.py`)

## 12. Project Structure

```
curriculum-intelligence/
├── backend/                  # FastAPI app (see app/api, app/services, app/database, app/utils)
├── datasets/                 # curriculum, job_descriptions, skill_dictionary, ground_truth
├── frontend/                 # React (Vite) app
├── docs/                     # architecture notes, diagrams, screenshots, evaluation output
├── docker-compose.yml        # one-command local run: Neo4j + backend + frontend
└── README.md
```

## 13. Installation & Setup

### Prerequisites
- Python 3.10+
- Node.js 18+
- A running Neo4j instance (Neo4j Desktop, or `docker run` if you have Docker available) —
  Community Edition is sufficient

### Option A: Docker Compose (recommended, zero manual setup)
With Docker installed you can run Neo4j, the backend and the frontend all at once:

```bash
docker compose up --build
```

Then load the datasets into the graph and open the dashboard:

```bash
# wait for the backend healthcheck to pass, then ingest once
docker compose exec backend python -c "from app.services.skill_service import ingest_all_data; ingest_all_data()"
```

- Dashboard: http://localhost:5173
- API docs: http://localhost:8000/docs
- Neo4j browser: http://localhost:7474 (`neo4j` / `curriculumneo4j`)

### Option B: Manual (Neo4j Desktop / local run)

### Neo4j Setup (the one manual step)
1. Install [Neo4j Desktop](https://neo4j.com/download/) (or run
   `docker run -p 7474:7474 -p 7687:7687 -e NEO4J_AUTH=neo4j/your_password neo4j:5`).
2. Create/start a database and note its Bolt URI (typically `bolt://localhost:7687`), username
   (`neo4j`) and the password you set.
3. Copy `backend/.env.example` to `backend/.env` and fill in `NEO4J_URI`, `NEO4J_USERNAME`,
   `NEO4J_PASSWORD`.

### Backend Setup
```bash
cd backend
python3 -m venv .venv
source .venv/bin/activate        # Windows: .venv\Scripts\activate
pip install -r requirements.txt
cp .env.example .env             # then edit .env with your Neo4j credentials
uvicorn app.main:app --reload --port 8000
```
On first run (or whenever you want to rebuild the graph from the JSON datasets), call:
```bash
curl -X POST http://localhost:8000/api/admin/reingest
```
This wipes and repopulates the graph from `datasets/`. Re-run it any time you edit a dataset file.

Interactive API docs: http://localhost:8000/docs

### Frontend Setup
```bash
cd frontend
npm install
npm run dev
```
Open http://localhost:5173

### Running Tests
```bash
cd backend
python -m pytest tests/ -v
```
40 tests cover skill normalization, extraction, the spaCy NER + semantic-matching routes (with
graceful degradation when spaCy is unavailable), alignment/gap calculation (including the
program-level gap report rollup), recommendation priority, NetworkX demand/statistics and the
PageRank demand signal, evaluation metrics, and CSV/PDF report export — none require a live
Neo4j connection.

## 14. Dataset Information

- `datasets/curriculum/curriculum.json` — MCA program, 8 courses (Web Development, Machine
  Learning, Database Management Systems, Cloud Computing, Mobile Application Development,
  **Computer Networks**, **Computer Architecture**, **Operating Systems**), with modules/topics
  and real descriptions for the NLP pipeline to process
- `datasets/job_descriptions/jobs.json` — 138 generated job postings across 11 roles (Full Stack
  Developer, Frontend Developer, Backend Developer, Data Analyst, Data Scientist, ML Engineer,
  Cloud Engineer, DevOps Engineer, Mobile App Developer, Data Engineer, QA Engineer), each with a
  varied skill subset so demand percentages are meaningful rather than trivial; networking,
  security, operating-systems and GPU skills are seeded across the Cloud, DevOps, Backend and ML
  roles so the new core courses have real industry demand to align against
- `datasets/skill_dictionary/skills.json` — 119 canonical skills (with aliases and categories)
  across **Frontend Development**, **Backend Development**, **DevOps & Cloud**, **Database**,
  **Data Science & Analytics**, **Machine Learning**, **Mobile Development**, **Programming
  Languages**, **Tools & Version Control**, **Networking & Security**, **Computer Architecture**
  and **Operating Systems**
- `datasets/ground_truth/labeled_jobs.json` — 40 hand-labeled samples for evaluation

`backend/data/generate_jobs.py` is the (re-runnable) generator script used to build the job and
ground-truth datasets.

## 15. Future Scope

- Move the semantic layer to a larger embedding model (`en_core_web_lg`) or a transformer-based
  sentence encoder so vector similarity becomes the primary accept signal rather than a
  confirmation signal
- Support curriculum file upload (PDF/DOCX syllabus parsing) instead of hand-authored JSON
- Add authentication for academic-coordinator accounts if the project scope is extended beyond
  this Minor Project
- Track alignment trends over multiple curriculum revisions over time

## 16. Screenshots

_Add screenshots of the running application here (`docs/screenshots/`) once you have Neo4j
running locally — Dashboard, Knowledge Graph Explorer, and Skill Gap Analysis are the strongest
ones to include for the report._
