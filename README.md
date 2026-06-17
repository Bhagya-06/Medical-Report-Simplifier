# Medical Report Simplifier AI

> **Understand your medical reports in plain language**

A full-stack web application that uses **LangGraph** to orchestrate a 7-node AI workflow, transforming complex medical jargon into clear, patient-friendly language. Built as a portfolio-ready project for students learning AI-powered web development.

---

## Table of Contents

- [What It Does](#what-it-does)
- [Technology Stack](#technology-stack)
- [Architecture Overview](#architecture-overview)
- [LangGraph Workflow Explained](#langgraph-workflow-explained)
- [Node Descriptions](#node-descriptions)
- [Project Structure](#project-structure)
- [Setup Instructions](#setup-instructions)
- [Running the Project](#running-the-project)
- [API Reference](#api-reference)
- [Sample Reports](#sample-reports)
- [Important Notes](#important-notes)
- [Future Enhancements](#future-enhancements)

---

## What It Does

Patients often receive medical reports filled with clinical abbreviations, reference ranges, and terminology that is difficult to understand without a medical background. This application:

- Accepts pasted text or uploaded PDF medical reports
- Runs the report through a **7-node LangGraph workflow**, each node powered by OpenAI GPT-4o-mini
- Returns:
  - **Key Findings** — important clinical observations extracted from the report
  - **Medical Terms Explained** — plain-language definitions of jargon
  - **Abnormal Values** — flagged lab results with color-coded severity and interpretation
  - **Simplified Report** — the full report rewritten in patient-friendly language
  - **Patient Summary** — the most actionable section: what to know, what to ask your doctor
  - **LangGraph Workflow Viewer** — visual progress through all 7 nodes

> **Disclaimer:** This application is for educational purposes only. It does not provide medical diagnoses or treatment recommendations.

---

## Technology Stack

| Layer | Technology |
|-------|-----------|
| Frontend | React 18, Vite, TypeScript, Tailwind CSS v4 |
| UI Components | shadcn/ui, Radix UI, Framer Motion |
| API Client | TanStack React Query (auto-generated via Orval) |
| Backend | Python 3.11, FastAPI, Uvicorn |
| AI Workflow | LangGraph, OpenAI GPT-4o-mini |
| PDF Processing | pypdf |
| Monorepo | pnpm workspaces |

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────┐
│                    Browser                          │
│                                                     │
│  ┌──────────────────────────────────────────────┐  │
│  │         React + Vite Frontend (/)            │  │
│  │                                              │  │
│  │  Sidebar              Main Area              │  │
│  │  ─────────────        ──────────────────     │  │
│  │  • API Key Input      • Report Textarea      │  │
│  │  • Sample Reports     • Analyze Button       │  │
│  │  • PDF Upload         • Workflow Visualizer  │  │
│  │                       • Results Sections     │  │
│  └──────────────────────────────────────────────┘  │
│                        │                            │
│              POST /api/analyze                      │
│              POST /api/upload                       │
└────────────────────────┼────────────────────────────┘
                         │
┌────────────────────────▼────────────────────────────┐
│           Python FastAPI Backend (/api)             │
│                                                     │
│  POST /analyze ──► LangGraph Workflow               │
│  POST /upload  ──► pypdf text extraction            │
│  GET  /healthz ──► health check                     │
└─────────────────────────────────────────────────────┘
                         │
┌────────────────────────▼────────────────────────────┐
│              LangGraph Workflow (7 nodes)           │
│                                                     │
│  1. Extract Findings                                │
│  2. Medical Terms     ──► Each node calls           │
│  3. Abnormal Values       OpenAI GPT-4o-mini        │
│  4. Simplification        with a specialized        │
│  5. Patient Summary       prompt                    │
│  6. Self Review                                     │
│  7. Final Output                                    │
└─────────────────────────────────────────────────────┘
```

The monorepo uses **pnpm workspaces** with three layers:
- `artifacts/` — runnable applications (frontend, backend)
- `lib/` — shared libraries (OpenAPI spec, generated client, Zod schemas, DB)
- `scripts/` — utility scripts

The OpenAPI spec at `lib/api-spec/openapi.yaml` is the **single source of truth** for the API contract. Running codegen generates both the React Query hooks used by the frontend and the Zod validation schemas used by the backend.

---

## LangGraph Workflow Explained

**LangGraph** is a library for building stateful multi-step AI workflows as graphs. Key concepts used in this project:

### State
A shared `TypedDict` object (`MedicalReportState`) that flows through every node. Each node receives the full state, does its work, and returns a dictionary of fields to update.

```python
class MedicalReportState(TypedDict):
    original_report: str          # Input: the raw medical report
    api_key: str                  # Input: user's OpenAI key
    extracted_findings: list      # Populated by Node 1
    medical_terms: list           # Populated by Node 2
    abnormal_values: list         # Populated by Node 3
    simplified_explanations: str  # Populated by Node 4
    patient_summary: str          # Populated by Node 5
    review_notes: str             # Populated by Node 6
    final_output: dict            # Populated by Node 7
    workflow_steps: list          # Tracking which nodes completed
```

### Nodes
Pure functions: `State → partial State`. Each node reads what it needs and returns only the fields it modifies.

### Edges
Sequential connections between nodes. After Node N completes, Node N+1 starts. LangGraph also supports conditional edges for branching workflows.

### Compilation
`graph.compile()` validates the graph structure and returns an executable object. `graph.invoke(initial_state)` runs all nodes in order and returns the final state.

```python
# How the graph is built (simplified)
graph = StateGraph(MedicalReportState)
graph.add_node("extract_findings", extract_findings_node)
graph.add_node("medical_terms", medical_terms_node)
# ... 5 more nodes ...
graph.set_entry_point("extract_findings")
graph.add_edge("extract_findings", "medical_terms")
# ... 5 more edges ...
graph.add_edge("final_output", END)
compiled = graph.compile()

# How the graph is invoked per request
final_state = compiled.invoke(initial_state)
```

---

## Node Descriptions

| # | Node | Responsibility | OpenAI Task |
|---|------|---------------|-------------|
| 1 | **Extract Findings** | Identify tests, diagnoses, measurements, and key clinical observations | Extract structured findings from the raw text |
| 2 | **Medical Terms** | Detect complex jargon and abbreviations; create plain-language explanations | Map medical terms to everyday language |
| 3 | **Abnormal Values** | Identify lab values outside reference ranges; classify as HIGH / LOW / BORDERLINE | Interpret values educationally (no diagnosis) |
| 4 | **Simplification** | Rewrite the entire report in patient-friendly language using "you/your" | Transform clinical prose into accessible text |
| 5 | **Patient Summary** | Generate main findings, things to discuss with a doctor, and important observations | Synthesize actionable patient-facing summary |
| 6 | **Self Review** | Review all generated content for clarity, simplicity, and consistency | Quality-gate the output before delivery |
| 7 | **Final Output** | Combine all sections into one structured response | Assembly node — no LLM call needed |

---

## Project Structure

```
medical-report-simplifier/
│
├── artifacts/
│   ├── api-server/                  # Python FastAPI backend
│   │   ├── main.py                  # FastAPI app, routes
│   │   ├── graph.py                 # LangGraph workflow definition
│   │   ├── nodes.py                 # 7 node functions
│   │   ├── state.py                 # TypedDict state schema
│   │   ├── pdf_processor.py         # PDF text extraction
│   │   ├── requirements.txt         # Python dependencies
│   │   └── src/                     # (Legacy Node.js files, unused)
│   │
│   └── medical-simplifier/          # React + Vite frontend
│       ├── src/
│       │   ├── App.tsx              # Root app with routing
│       │   ├── index.css            # Tailwind + theme tokens
│       │   ├── pages/
│       │   │   └── home.tsx         # Main page (layout + logic)
│       │   └── components/
│       │       ├── workflow-visualizer.tsx   # Animated node progress
│       │       └── results-display.tsx       # 6-section results layout
│       ├── index.html               # HTML entry point (font links)
│       ├── vite.config.ts           # Vite configuration
│       └── package.json             # Frontend dependencies
│
├── lib/
│   ├── api-spec/
│   │   └── openapi.yaml             # OpenAPI contract (source of truth)
│   ├── api-client-react/
│   │   └── src/generated/           # Auto-generated React Query hooks
│   └── api-zod/
│       └── src/generated/           # Auto-generated Zod schemas
│
├── pnpm-workspace.yaml              # Workspace configuration
├── package.json                     # Root dev dependencies
└── README.md                        # This file
```

---

## Setup Instructions

### Prerequisites

- **Node.js 18+** and **pnpm 8+**
- **Python 3.11+** and **uv** (fast Python package manager)
- An **OpenAI API key** (entered in the UI at runtime — never hardcoded)

### 1. Install Node.js dependencies

```bash
pnpm install
```

### 2. Set up the Python virtual environment

```bash
cd artifacts/api-server
uv venv .venv
uv pip install --python .venv/bin/python -r requirements.txt
cd ../..
```

### 3. (Optional) Regenerate API client from OpenAPI spec

```bash
pnpm --filter @workspace/api-spec run codegen
```

This regenerates the React Query hooks in `lib/api-client-react/src/generated/` and Zod schemas in `lib/api-zod/src/generated/`. Only needed if you modify `lib/api-spec/openapi.yaml`.

---

## Running the Project

You need to run **two processes** simultaneously: the Python backend and the React frontend.

### Terminal 1 — Python FastAPI Backend

```bash
cd artifacts/api-server
PORT=8080 .venv/bin/python main.py
```

The API server starts at `http://localhost:8080`. Verify with:

```bash
curl http://localhost:8080/api/healthz
# → {"status":"ok"}
```

### Terminal 2 — React Frontend

```bash
PORT=3000 BASE_PATH=/ pnpm --filter @workspace/medical-simplifier run dev
```

Open `http://localhost:3000` in your browser.

> **Note:** The frontend sends API requests to `/api/...` which need to be proxied to port 8080. For local development, you can add a Vite proxy in `vite.config.ts`:
>
> ```typescript
> server: {
>   proxy: {
>     '/api': 'http://localhost:8080'
>   }
> }
> ```

---

## API Reference

### `GET /api/healthz`

Health check. Returns `{"status": "ok"}`.

---

### `POST /api/analyze`

Run the full LangGraph workflow on a medical report.

**Request body:**
```json
{
  "api_key": "sk-...",
  "report_text": "Hemoglobin: 10.2 g/dL..."
}
```

**Response:**
```json
{
  "findings": [
    { "title": "Low Hemoglobin", "description": "Hemoglobin is 10.2 g/dL, below the reference range of 13.5–17.5 g/dL." }
  ],
  "medical_terms": [
    { "term": "Hemoglobin", "explanation": "A protein in red blood cells that carries oxygen throughout your body." }
  ],
  "abnormal_values": [
    {
      "test": "Hemoglobin",
      "value": "10.2 g/dL",
      "reference_range": "13.5-17.5 g/dL",
      "status": "low",
      "interpretation": "Low hemoglobin generally indicates that your blood may not be carrying enough oxygen, which can cause fatigue and shortness of breath."
    }
  ],
  "simplified_report": "Your blood test results show...",
  "patient_summary": "**Main Findings**\n- Your hemoglobin is lower than normal...",
  "review_notes": "The explanations are clear and use accessible language...",
  "workflow_steps": [
    { "node": "Extract Findings", "status": "completed", "description": "Found 6 key findings" },
    { "node": "Medical Terms", "status": "completed", "description": "Explained 8 medical terms" }
  ]
}
```

---

### `POST /api/upload`

Upload a PDF and extract its text.

**Request:** `multipart/form-data` with field `file` (PDF file)

**Response:**
```json
{
  "text": "Extracted text from the PDF..."
}
```

---

## Sample Reports

### Complete Blood Count + Lipid Panel

```
Complete Blood Count Report
Patient: John Doe | Date: 2024-01-15

Hemoglobin: 10.2 g/dL (Reference: 13.5-17.5 g/dL) - LOW
Hematocrit: 31% (Reference: 41-53%) - LOW
WBC: 11.8 K/uL (Reference: 4.5-11.0 K/uL) - HIGH
Platelets: 145 K/uL (Reference: 150-400 K/uL) - BORDERLINE LOW

Lipid Panel:
Total Cholesterol: 245 mg/dL (Reference: Below 200 mg/dL) - HIGH
LDL Cholesterol: 165 mg/dL (Reference: Below 100 mg/dL) - HIGH
HDL Cholesterol: 38 mg/dL (Reference: Above 40 mg/dL) - LOW

Fasting Blood Sugar: 128 mg/dL (Reference: Below 100 mg/dL) - HIGH (Pre-diabetic range)

Assessment: The patient exhibits mild normocytic anemia, hypercholesterolemia, leukocytosis,
and impaired fasting glucose consistent with pre-diabetes.
```

### Thyroid Function Panel

```
Thyroid Function Panel
Patient: Jane Smith | Date: 2024-02-10

TSH (Thyroid Stimulating Hormone): 6.8 mIU/L (Reference: 0.4-4.0 mIU/L) - HIGH
Free T4: 0.7 ng/dL (Reference: 0.8-1.8 ng/dL) - LOW NORMAL
Free T3: 2.1 pg/mL (Reference: 2.3-4.2 pg/mL) - BORDERLINE LOW

HbA1c: 7.2% (Reference: Below 5.7%) - HIGH (Diabetic range)
Fasting Glucose: 142 mg/dL (Reference: 70-99 mg/dL) - HIGH

Assessment: Findings consistent with subclinical hypothyroidism and poorly controlled
type 2 diabetes mellitus. Recommend endocrinology referral.
```

---

## Important Notes

### OpenAI API Key Security
- The API key is entered by the user in the browser UI
- It is stored only in React component state (`useState`) — **never** in `localStorage`, `sessionStorage`, cookies, or environment variables
- It is transmitted to the backend only as part of each analysis request
- The backend does not store or log the API key

### Educational Use Only
This application is designed for educational understanding. It:
- Does **not** provide medical diagnoses
- Does **not** recommend treatments or medications
- Does **not** replace consultation with a qualified healthcare professional

---

## Future Enhancements

| Feature | Description |
|---------|-------------|
| **Report History** | Save past analyses in the browser using IndexedDB |
| **Multi-language Support** | Generate patient summaries in the user's language |
| **Conditional Graph Branching** | Use LangGraph's `conditional_edges` to skip nodes when the report lacks certain content |
| **Streaming Output** | Stream each node's result as it completes using FastAPI's `StreamingResponse` and Server-Sent Events |
| **Comparison View** | Side-by-side view of original vs. simplified report |
| **Export to PDF** | Generate a patient-friendly PDF of the analysis results |
| **Image OCR** | Process scanned PDF reports using Tesseract or a vision model |
| **Follow-up Q&A** | Ask follow-up questions about the report using the full analysis as context |
| **Doctor Letter Generator** | Generate a prepared summary letter the patient can share with their doctor |

---

## Learning Resources

- [LangGraph Documentation](https://langchain-ai.github.io/langgraph/)
- [FastAPI Documentation](https://fastapi.tiangolo.com/)
- [OpenAI API Reference](https://platform.openai.com/docs/api-reference)
- [React Query (TanStack)](https://tanstack.com/query/latest)
- [Tailwind CSS v4](https://tailwindcss.com/docs)
