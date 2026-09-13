# 🩺 MedDocAI — Healthcare Document Intelligence Pipeline

**A production-style, multi-agent LLM pipeline that converts unstructured clinical
documents (discharge summaries, referral letters, lab reports) into structured,
ICD-10-coded, validated JSON — with an automatic human-review safety net.**

Built with **LangGraph**, **AWS Bedrock** (Claude models), and **FastAPI**, with a
**Streamlit** demo site you can click through in the browser.

> ⚠️ Built entirely on synthetic/sample data for demonstration purposes. Not a
> medical device, not HIPAA-certified as-is, not for clinical use. See
> [Limitations](#limitations--what-id-do-for-a-real-deployment).

---

## Why this project

Hospitals and clinics generate huge volumes of unstructured text — discharge
summaries, referral letters, radiology reports — that need to become structured
data for billing, care coordination, and analytics. Doing this by hand doesn't
scale, and a single LLM call with no checks is too risky for clinical data. This
project shows the pattern that actually works in production: **specialized
agents, each doing one job well, orchestrated as a graph with validation and
human-review gates built in** — not just a single prompt.

## What it does

Feed it raw clinical text → get back structured JSON like this:

```json
{
  "source_document_id": "doc_0001",
  "document_type": "discharge_summary",
  "demographics": {"full_name": "Jane Doe", "age": 64, "sex": "F", "mrn": "MRN12345"},
  "diagnoses": [
    {"description": "Type 2 diabetes mellitus", "icd10_code": "E11.9", "icd10_confidence": 0.94, "is_primary": true}
  ],
  "medications": [
    {"name": "Metformin", "dosage": "500mg", "frequency": "BID", "route": "oral"}
  ],
  "labs": [
    {"test_name": "HbA1c", "value": "7.8", "unit": "%", "flag": "HIGH"}
  ],
  "clinical_summary": "64F with T2DM discharged on Metformin; HbA1c elevated at 7.8%.",
  "extraction_confidence": 0.91,
  "validation_flags": [],
  "requires_human_review": false
}
```

## Architecture

4 specialized agents orchestrated as a **LangGraph state machine** (not just a
linear chain — it can loop back and retry):

```
raw document → [Extractor] → [Validator] → [Coder] → [QA] → structured JSON
                                  │  ▲
                                  └──┘ retry on critical validation failure
```

| Agent | Job |
|---|---|
| **Extractor** | Pulls demographics, diagnoses, meds, vitals, labs, follow-up into structured JSON |
| **Validator** | Deterministic rule checks + LLM check for hallucinated/missed facts vs. source text |
| **Coder** | Maps diagnoses → ICD-10-CM and meds → RxNorm codes, with confidence scores |
| **QA** | Writes a plain-language summary, computes a final explainable confidence score, decides if human review is needed |

Full diagrams and production AWS deployment design: **[architecture/architecture.md](architecture/architecture.md)**

## Tech stack

- **Orchestration**: LangGraph (stateful multi-agent graph, conditional routing, bounded retries)
- **Model**: AWS Bedrock (Claude) for production; direct Anthropic API for local dev — one env var swaps between them
- **API**: FastAPI (`/process`, `/process/batch`, `/health`, `/schema`)
- **Demo site**: Streamlit
- **Validation**: Pydantic schemas (FHIR-inspired: Patient/Condition/MedicationStatement shapes)
- **Infra**: Docker, docker-compose, GitHub Actions CI

## Quickstart (no AWS account needed)

```bash
git clone <your-repo-url>
cd healthcare-doc-intelligence
python -m venv venv && source venv/bin/activate
pip install -r requirements.txt

cp .env.example .env
# edit .env: set ANTHROPIC_API_KEY=sk-ant-...   (LLM_PROVIDER=anthropic is the default)

# Run the demo website
streamlit run frontend/app.py

# Or run the API
uvicorn src.api.main:app --reload
# then: curl -X POST localhost:8000/api/v1/process -H "Content-Type: application/json" \
#   -d '{"document_text": "..."}'
```

### Switching to AWS Bedrock (production path)

```bash
# in .env:
LLM_PROVIDER=bedrock
AWS_REGION=us-east-1
BEDROCK_MODEL_ID=anthropic.claude-sonnet-4-5-20250929-v1:0
# configure AWS credentials via aws configure / IAM role — no code changes needed
```

### Docker

```bash
docker-compose up --build
# API:  http://localhost:8000/docs
# Demo: http://localhost:8501
```

## Project structure

```
healthcare-doc-intelligence/
├── src/
│   ├── agents/            # extractor, validator, coder, qa, orchestrator (LangGraph)
│   ├── llm/                # Bedrock / Anthropic client abstraction
│   ├── models/             # Pydantic output schemas
│   ├── api/                 # FastAPI service
│   └── config.py
├── frontend/app.py          # Streamlit demo website
├── data/sample_documents/   # synthetic sample clinical documents
├── tests/                    # pytest suite (unit tests run with no LLM/API key)
├── architecture/             # architecture diagrams + design rationale
├── Dockerfile / docker-compose.yml
└── .github/workflows/ci.yml
```

## Running tests

```bash
pytest tests/ -v
```

Deterministic-logic tests (validation rules, confidence scoring) run with **no
API key required** — the end-to-end LLM test is auto-skipped unless credentials
are configured, so CI stays green without secrets.

## Roadmap / what I'd add next

- [ ] Human-review UI (approve/edit flagged records → feeds back as few-shot examples)
- [ ] Real terminology service integration (UMLS / AWS Comprehend Medical) instead of LLM-only ICD-10 mapping
- [ ] PDF/OCR ingestion (Textract) for scanned documents
- [ ] Multi-document patient timeline aggregation
- [ ] Prompt/response logging + eval harness for regression testing agent quality over time

## Limitations & what I'd do for a real deployment

- Trained/tested on **synthetic data only** — real-world clinical text is messier (abbreviations, handwriting artifacts from OCR, non-English text).
- ICD-10/RxNorm coding is LLM-assisted, not a certified coding engine — a real deployment needs a licensed terminology service and a certified coder in the loop for billing use cases.
- No PHI de-identification layer included here — a real deployment needs one before any data leaves a controlled environment, plus a signed BAA with the model provider.
- This is a portfolio/demo project, not audited or certified for clinical or billing use.

## License

MIT — see [LICENSE](LICENSE). Use freely for learning, portfolio, and interview prep.

---

*Built as a portfolio project demonstrating production-pattern multi-agent LLM
pipelines for AI/ML Engineer roles. Feedback and PRs welcome.*
