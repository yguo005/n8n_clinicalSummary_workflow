# Vithea

Health and wellness tooling: **doctor's note processing** (OCR, SOAP, patient summaries) and **n8n-based mental health questionnaire pipelines** (preprocessing, trend analysis, validation, and parenting assessment reports).

---

## Overview

This repository contains two main areas:

1. **Doctor's note API** — A Deno HTTP server that accepts doctor's note images or text, runs OCR (via Groq), and returns structured SOAP format plus a patient-friendly summary.
2. **n8n questionnaire workflows** — Python and JavaScript used inside [n8n](https://n8n.io) to preprocess mental health questionnaires (PHQ-9, GAD-7, WHO-5, PROMIS, PedsQL, SDQ, etc.), validate data, analyze trends, and generate parenting coach reports.

---

## 1. Doctor's Note API (`index.ts`)

Deno-based edge/server that:

- **POST (multipart/form-data)** — Upload an image of a doctor's note → OCR (Groq) → SOAP conversion + patient summary.
- **POST (application/json)** — Send `{ "noteText": "..." }` → SOAP conversion + patient summary (no OCR).
- **GET /doctors-note** — Returns the contents of `doctors_note.json`.

### Requirements

- [Deno](https://deno.land/)
- **GROQ_API_KEY** in environment or `.env`

### Run locally

```bash
# From repo root (ensure .env has GROQ_API_KEY)
deno run --allow-net --allow-env --allow-read index.ts
```

Default serve runs on the port Deno assigns (e.g. 8000); check console output.

### Example requests

**Image upload:**

```bash
curl -X POST -F "file=@/path/to/doctors_note.jpg" http://localhost:8000
```

**JSON (text only):**

```bash
curl -X POST -H "Content-Type: application/json" -d '{"noteText":"Patient reports headache. BP 120/80. Plan: rest, follow up in 2 weeks."}' http://localhost:8000
```

**Response** includes `extracted_text` (if image), `soap`, and `patient_summary`.

---

## 2. n8n Questionnaire Pipelines

Scripts and logic are intended to run inside n8n (Code nodes, HTTP nodes, etc.). The workflow export `My workflow4 (3).json` can be imported into n8n.

### Main components

| File | Purpose |
|------|--------|
| **n8n_questionnaire_preprocessor.py** | Converts raw questionnaire rows into structured results: scores, severities, clinical cutoffs, and derived fields for PHQ-9, GAD-7, WHO-5, PROMIS (Depression, Anxiety, Life Satisfaction), PedsQL, CES-DC, SCARED, RSES, SDQ, PSC-17. |
| **n8n_trend_analyzer.py** | Analyzes score changes over time using administration frequency and clinical significance guidelines for the same instruments. |
| **n8n_data_validator.py** | Validates preprocessed data (required fields, dates, derived fields) before trend analysis or LLM steps. Use as a checkpoint between Preprocessor and Trend Analyzer. |
| **evaluate_summary.js** | Evaluates AI-generated parent summaries (length, safety cues, domains, insight/actionability, relevance). |
| **facts_contract.js** | Structures “facts” for summary generation. |
| **format_parent_email.js** / **format_admin_email.js** | Format content for parent and admin emails. |

### Supported questionnaires (preprocessor)

- **PHQ-9** — Depression (0–27, severity bands, MDD flag)
- **GAD-7** — Anxiety (0–21)
- **WHO-5** — Well-being (raw → 0–100 index)
- **PROMIS** — Depression, Anxiety, Life Satisfaction (pediatric/parent, T-scores)
- **PedsQL** — HRQoL (dimensions, psychosocial/total ratio)
- **CES-DC** — Depression risk (youth)
- **SCARED** — Anxiety (total + subscales)
- **RSES** — Self-esteem
- **SDQ** — Strengths and Difficulties (parent/self, subscales)
- **PSC-17** — Pediatric Symptom Checklist (internalizing, attention, externalizing)

### Parenting assessment pipeline (`parenting_n8n/`)

- **All_in_One_FIXED.py** — Single Code node that normalizes input (handles JsProxy), finds assessment data, and runs the full parenting pipeline.
- **prepare_analysis.py** — Prepares data for analysis (e.g. trend dimensions, significant changes).
- **prepare_for_vetting_FIXED.py** / **assemble_vetting_insight_FIXED.py** / **verify_vetting_FINAL.py** — Vetting and assembly of insights for reports.
- **verify_all_insights.py** — Verification of insights.
- **report_FIXED.md** / **insight_score_FIXED.md** / **analysis_FIXED.md** — Prompt and structure docs for report generation.

Inputs are typically assessment session JSON; outputs feed into “parenting coach” reports (strengths, areas for growth, recommendations).

### Running the Python scripts outside n8n

The preprocessor, trend analyzer, and validator expect input in the same shape as n8n items (e.g. a list of objects with a `json` payload). For example:

```python
# Example: run preprocessor on a list of rows
from n8n_questionnaire_preprocessor import preprocess_questionnaire_data

items = [{"json": {"questionnaire": "PHQ-9", "timepoint": 1, "date": "2025-01-15", "question": "Q1", "answer": 2, ...}}, ...]
results = preprocess_questionnaire_data(items)
```

Use Python 3 with standard library; no extra pip dependencies are required for the core preprocessor/validator/trend logic.

---

## Project layout (summary)

```
Vithea/
├── index.ts                    # Doctor's note API (Deno)
├── doctors_note.json           # Static doctor's note (optional; used by GET /doctors-note)
├── .env                        # GROQ_API_KEY (not committed)
├── n8n_questionnaire_preprocessor.py
├── n8n_trend_analyzer.py
├── n8n_data_validator.py
├── data_validatot.py           # Alternate/legacy validator
├── n8n_questionnaire_gemini_preprocessor.js
├── evaluate_summary.js
├── facts_contract.js
├── final_decision.js
├── format_parent_email.js
├── format_admin_email.js
├── retry_counter.js
├── My workflow4 (3).json       # n8n workflow export
├── parenting_n8n/              # Parenting assessment pipeline
│   ├── All_in_One_FIXED.py
│   ├── prepare_analysis.py
│   ├── prepare_for_vetting_FIXED.py
│   ├── assemble_vetting_insight_FIXED.py
│   ├── verify_vetting_FINAL.py
│   ├── verify_all_insights.py
│   ├── report_FIXED.md
│   ├── insight_score_FIXED.md
│   ├── analysis_FIXED.md
│   └── response_data_*.json    # Sample assessment data
├── questionnaire_document.csv  # Questionnaire reference
└── INTERVIEW_PREP.md           # Interview/design notes
```

---

## Environment

- **Doctor's note API:** set `GROQ_API_KEY` in `.env` or the environment.
- **n8n:** configure any API keys (e.g. OpenAI, Groq) and endpoints in n8n credentials and node settings.

---

## License

See repository license file (if present).
