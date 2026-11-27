# FactFolio

**Facts on every company. Insight into every portfolio.**

FactFolio is a citation-grounded **Agentic RAG** assistant over SEC annual reports and mutual fund holdings. Ask about a company's figures, a multi-year trend, a comparison across companies, a sector, or a fund, and every number in the answer links to the filing page it came from. If the filings can't support an answer, FactFolio says so instead of guessing.

Capstone project, **Track B (API-side)**, Executive Post Graduate Programme in Generative AI and Agentic AI, IIT Kharagpur, November 2026. Team **SIMPLR**.

## At a glance

| | |
|---|---|
| Corpus | 509 SEC annual reports (Form ARS) from 143 companies in 13 sectors, FY2022–2026: 81,715 pages, 335,699 indexed chunks. Vanguard fund holdings from SEC Form N-PORT |
| Model | Hosted Qwen3-8B (8,192-token context, 1,024 output tokens), used through an OpenAI-compatible API with no access to weights |
| Retrieval | Chroma (bge-small-en-v1.5) + BM25, fused by reciprocal rank, filtered to one company and year, cross-encoder rerank |
| Agents | 7 LangGraph agents behind a router: Q&A, trend, sector trend, shareholder returns, dividend income, mutual fund, report hand-off. Capped at 10 steps |
| Safety | Input and evidence guards, policy refusals (advice, forecasts, personal data), a critic that withholds any figure not found on a cited page, human-review routing |
| Stack | FastAPI backend, Streamlit UI, LangSmith tracing, Docker Compose |

## Results

On 86 test questions frozen before the agents were built (hosted Qwen3-8B):

| Approach | Right answers |
|---|---|
| Prompt only (no retrieval) | 13% |
| Best single RAG prompt | 74% |
| **FactFolio agents** | **83%** |

Wrong or partial answers shown to the user fell from 11 to 2. Across all 113 questions, including the agent tasks, the agents answered 86% correctly against 62% for a single RAG prompt. These questions also guided our fixes, so this is not a blind test; details and limits are in the technical report.

## Submission deliverables

| Deliverable | Where |
|---|---|
| Git repository with reproducible code | This repository: [Quick start](#quick-start) below, `uv.lock` for exact package versions, an offline test suite |
| Trace dashboard (Track B) | [TRACES.md](TRACES.md): LangSmith trace links and screenshots. How tracing works: [docs/observability_langsmith.md](docs/observability_langsmith.md) |
| Model card and Responsible AI | [MODEL_CARD.md](MODEL_CARD.md): intended use, failure modes, refusals, adversarial tests |
| Technical report | [docs/final_tech_document/FactFolio_Financial_Report_Technical_Documentation.docx](docs/final_tech_document/FactFolio_Financial_Report_Technical_Documentation.docx) |
| Working deployed system | Live on Google Cloud (Mumbai, e2-standard-4, Docker Compose, HTTPS): https://8-234-95-222.sslip.io. Login shared with evaluators. Deployment files: `docker-compose.gcp.yml`, `deploy/Dockerfile.local`; guide: [docs/gcp.md](docs/gcp.md) |
| Demo video (10 min) | [link to be added] |

## Quick start

Python 3.12 and [uv](https://docs.astral.sh/uv/) are recommended. Commands are shown for macOS/Linux; on Windows use `.venv\Scripts\python.exe` and backslashes in paths.

### 1. Install and run the tests (no keys, models or data needed)

```bash
git clone https://github.com/sp-261/FactFolio.git
cd FactFolio
uv sync
.venv/bin/python -m pytest tests -q
```

About 720 tests run offline in a few minutes. They cover the agents, guardrails, retrieval, API and UI.

### 2. Offline demo (mock LLM, no keys)

```bash
.venv/bin/python -m uvicorn scripts.api.api_app:app --host 127.0.0.1 --port 8000
.venv/bin/streamlit run scripts/ui/api_streamlit_app.py
```

Open http://localhost:8501. Without a hosted model the app runs in **Test mode** (an orange notice on the page): the full pipeline, guards and UI work, but answers are placeholder text.

### 3. Full system (real index and hosted model)

1. Download the prebuilt index (`chroma_db_final.tar`) and `mutual_fund_data.zip` from the team's shared drive and unpack them into `indexes/rag_home/indexes/`.
2. Put the course API credentials in place. These are **never committed**: `secrets/` and `indexes/` are ignored by Git.
   - Chat key: `secrets/llm_api_key.txt`
   - Chat server certificate: `indexes/rag_home/secrets/llm_ca.crt`
3. Start the API with the hosted model:

```bash
export LLM_API_KEY="$(tr -d '[:space:]' < secrets/llm_api_key.txt)"
export LLM_API_BASE=https://103.42.50.41:1032/v1 LLM_MODEL=Qwen/Qwen3-8B LLM_MAX_TOKENS=1024
export DEFAULT_LLM_PROVIDER=hosted HF_HUB_OFFLINE=1
export RAG_REQUEST_TIMEOUT_SECONDS=180 RAG_HTTP_TIMEOUT_SECONDS=90
.venv/bin/python -m uvicorn scripts.api.api_app:app --host 127.0.0.1 --port 8000
```

4. In a second terminal: `.venv/bin/streamlit run scripts/ui/api_streamlit_app.py`

The step-by-step guide, with checks after each step, is [docs/local_setup_guide.html](docs/local_setup_guide.html). To build the index from scratch (Colab, about 509 filings), see [Data download](#data-download) and [docs/deployment.md](docs/deployment.md).

### Try these questions

| Agent | Question |
|---|---|
| Q&A | Compare Costco and Walmart revenue in 2025 |
| Trend | How did JPMorgan Chase's net income trend from 2022 to 2025? |
| Sector trend | How did banking sector net income trend from 2022 to 2025? |
| Mutual fund | What are the top 10 holdings of the Vanguard 500 Index Fund? |
| Refusal | Should I buy Apple stock? |

More questions with expected answers: [docs/agent_test_questions.html](docs/agent_test_questions.html). How to read an answer and its checks: [docs/agent_user_guide.html](docs/agent_user_guide.html).

## Documentation map

| Topic | Document |
|---|---|
| How each agent works, step by step | [docs/agent_flows.md](docs/agent_flows.md) |
| One question per agent, end to end, with traces | [docs/end_to_end_walkthrough.md](docs/end_to_end_walkthrough.md) |
| API reference | [docs/fastapi_api.md](docs/fastapi_api.md) |
| Week 1 baseline and failure scenarios | [docs/week1_failure_scenarios.md](docs/week1_failure_scenarios.md) |
| Mutual fund data and agent | [docs/mutual_fund_rag_design.md](docs/mutual_fund_rag_design.md) |
| Deterministic financial report generation | [docs/financial_report_generation.md](docs/financial_report_generation.md) |
| Deployment options and costs | [docs/deployment.md](docs/deployment.md) |

## Team SIMPLR and contributions

**Contributors:** Laxmi Mohta, Sandeep Patil, Rama Krishna Dasaraju and Praket Sanganeria.

Contributions follow the [CRediT](https://credit.niso.org/) contributor roles. All contributors took part in every role below.

| CRediT role | What it covered in FactFolio |
|---|---|
| Conceptualization | Problem framing, scope and the agentic RAG approach |
| Methodology | Agent design, hybrid retrieval, page routing, guardrails and the evaluation protocol |
| Investigation | Research on RAG architectures, page readers and the SEC EDGAR and N-PORT data |
| Data curation | The 509-filing corpus, the N-PORT fund holdings and the frozen evaluation set |
| Software | Ingestion, retrieval, agents, guardrails, the FastAPI backend and the Streamlit UI |
| Validation | Unit tests, the frozen evaluation, agent testing and adversarial testing |
| Formal analysis | Baselines, results analysis and failure analysis |
| Visualization | The FactFolio interface, architecture diagrams and result figures |
| Writing – original draft | Technical report, design notes, guides and the model card |
| Writing – review & editing | Review of the report, documentation and demo script |
| Project administration | Planning, milestones, repository management and submission |

**Also on the team roster:** Manideep Gubbala and Jai Kishan.

## Project history

Development started on 23 October 2026 in the team's working repository (`rama159065/Capstone`) and moved here on 26 November 2026 with its full history preserved: every commit, branch, author and date is unchanged. A dated progress log is in [docs/project_timeline.md](docs/project_timeline.md).

## Acknowledgements

SEC EDGAR for the annual reports and N-PORT filings; the course team for the hosted Qwen3-8B and embedding endpoints. We used Claude (Anthropic) as a coding and writing assistant; all design decisions, experiments and evaluation results are the team's own.

---

# Technical reference

The sections below are the full command and design reference.

## Why This Is Agentic RAG

This is **Agentic RAG** because retrieval is only one step inside a stateful agent workflow. A plain RAG chain usually does `query -> retrieve -> answer`; this project adds LangGraph nodes that validate the request, plan retrieval, call hybrid search tools, inspect retrieved evidence, draft an answer, filter unsafe output, validate citations, and route risky cases to human review.

The implemented agent loop has these decision-making stages:

```text
security -> plain_text -> planner (tools) -> retrieve <-> evidence_guard (retry) -> answer -> calculator -> critic (repair) -> governance
```

That makes the system agentic in the capstone sense: it uses tools, maintains structured state, applies bounded decisions, checks its own evidence/citations, records observability metadata, and can stop or escalate instead of always answering.

## Scope

The product is **financial report question answering** only (`financial_report_qa`): SEC annual reports (Form ARS) from 281 companies in 13 sectors, served as `full` (1,004 filings) or `min500` (509).

The five other Track B domains (procurement, travel and expense, competitive research, incident postmortem, HR policy) were removed on 10 November 2026; they remain in git history, as does the shared domain agent (`rag_agent.py`) removed on 11 November 2026.

## Serving backend: Chroma first

The capstone answers from **pipeline 1, the Chroma index** (`RAG_BACKEND=financial_vlm`, folder `<RAG_HOME>/indexes/chroma_db`): 509 filings (`min500`), 335,699 chunks, with chart and table pages read by OCR, Qwen3-VL or Gemini. Each search runs bge-small dense vectors and a BM25 sidecar and fuses them by reciprocal rank. This is the default in every environment (local PC, Colab, a server or Google Cloud): `scripts/common/runtime_profile.py` picks it whenever its `vlm_ingest.sqlite3` exists, and `GET /health` reports it as `financial_backend`.

Two other backends are kept for testing. Neither is chosen automatically while the Chroma index exists; set `RAG_BACKEND` before starting the API:

| `RAG_BACKEND` | Index | Status |
|---|---|---|
| `financial_vlm` (default) | pipeline 1: Chroma `annual_reports_vlm` + BM25 | Main backend for the capstone |
| `financial_qdrant` | pipeline 2: Qdrant `annual_reports_text` + ColPali `annual_reports_visual` | Test option; served automatically only when it is the only index |
| `financial_hybrid` | both: Chroma's readings plus ColPali page finding | Proof of concept, tested on one filing (Chevron 2022: expected value reached the evidence for 9/9 questions with Chroma, 4/9 with Qdrant, 9/9 with the hybrid). See [docs/hybrid_retrieval.html](docs/hybrid_retrieval.html) |

```powershell
# default: nothing to set
uvicorn scripts.api.api_app:app --port 8000
# Qdrant only, or the hybrid (needs both indexes; RAG_FIN_VISUAL=auto, ColPali in bfloat16 on CPU)
$env:RAG_BACKEND="financial_qdrant"; uvicorn scripts.api.api_app:app --port 8000
$env:RAG_BACKEND="financial_hybrid"; $env:RAG_FIN_COLPALI_DTYPE="bfloat16"; uvicorn scripts.api.api_app:app --port 8000
```

On Colab, `cp.serve(s)` serves the notebook's pipeline, and `cp.serve(s, backend="financial_qdrant" | "financial_hybrid")` picks a test backend. Docker and Google Cloud take `RAG_BACKEND` from `.env`. If the chosen backend's index is missing, the API logs a warning and `/health` lists it under `missing_indexes`.

## Implementation Status

| Area | Status |
|---|---|
| Financial PDF RAG | Implemented: pipeline 1 (Chroma + BM25, RRF, routed OCR/VLM page reading, cross-encoder rerank) is the served index; pipeline 2 (Qdrant + ColPali) and the hybrid are test backends. |
| Agent orchestration | Implemented with LangGraph nodes for security, plain-text validation, planning, hybrid retrieval, evidence guard, answer generation, output filtering, citation criticism, and governance/HITL. |
| Guardrails and HITL | Implemented for input filtering, evidence prompt-injection filtering, output content filtering, citation validation, and human-review routing. |
| API backend and UI | Implemented with FastAPI backend endpoints plus Streamlit frontends, including an API-backed Streamlit app for the final architecture. |
| Observability | Optional LangSmith tracing is implemented for agents and financial retrieval evaluation. |
| Evaluation | Financial retrieval recall@k evaluation is implemented separately from answer generation. |

## Repository Layout

```text
docs/                         Design notes and capstone documentation
scripts/
  api/            FastAPI backend (api_app.py): financial chat, conversation, retrieval, companies
  ui/             Streamlit front ends: API-backed FactFolio app (api_streamlit_app.py), standalone financial app
  agents/         Financial agent graph and tools, clarifier, safe answer (multimodal)
  guardrails/     Shared input / evidence / output safety checks and their regex pattern sets
  prompts/        Prompt templates for the financial and domain agents
  rag/            Ingestion, indexing and retrieval: Chroma/BM25 indexer, Qdrant ingest (financial_ingest_qdrant_db.py),
                  multimodal RAG, ColPali, selective OCR, retrieval metrics
  observability/  JSONL traces, LangSmith helpers, report-trace monitor
  downloads/      Corpus and model downloaders (EDGAR ARS/10-K, SEC company facts, models)
  evaluation/     Eval set builder, prompt baselines, retrieval/OCR/ColPali evaluations, validation-data generators
  reports/        Deterministic financial report generation
  common/         Repo paths, runtime profile (local/colab/server), resilience, local models, company directory
notebooks/                    Colab ingestion and serving notebook
tests/                        Unit tests
```

Generated data is intentionally ignored by git:

```text
data/raw/
data/processed/
indexes/
output/
eval/
```

## Setup

Python 3.12 is recommended.

Using `uv`:

```powershell
uv sync
```

Or with plain `pip`:

```powershell
python -m venv .venv
.\.venv\Scripts\Activate.ps1
pip install -e .
```

Optional OCR support for scanned pages, tables, or chart images requires Tesseract installed on the operating system. The current implementation can run without OCR for text-based PDFs.

## Developer smoke test (hashing embeddings, mock LLM)

Build a tiny index from 5 PDFs and run one agent offline:

```powershell
python scripts\downloads\download_financial_report_pdfs.py --target-pdfs 5
python scripts\rag\index_financial_report_pdfs.py --embedding-provider hashing --limit-pdfs 5 --max-pages-per-pdf 10 --reset
python scripts\agents\financial_prompting_agent.py --llm-provider mock --technique agentic_graph --query "What does Apple say can affect quarterly net sales?" --ticker AAPL
```

Start the final API-side demo stack:

```powershell
uvicorn scripts.api.api_app:app --host 127.0.0.1 --port 8000
streamlit run scripts\ui\api_streamlit_app.py
```

## Data Download

Download financial annual report PDFs:

```powershell
python scripts\downloads\download_financial_report_pdfs.py --target-pdfs 100
```

Optional raw SEC XBRL company-facts records (JSON) for corpus analysis:

```powershell
python scripts\downloads\download_rag_corpora.py --domains all --target-docs 500 --overwrite --sec-user-agent "capstone-rag/0.1 your_email@example.com"
```

PDF manifests are written to:

```text
data/raw/<domain>/pdf_manifest.jsonl
```

### Sector annual-report corpus (1,004 EDGAR ARS PDFs, 13 sectors)

The corpus planned in `docs/annual_report_corpus_plan.html`. The plan ships as `data/raw/financial_report_qa/annual_reports/corpus_plan.jsonl`. The downloader writes one folder and one manifest per sector, plus a combined manifest; every row carries `sector` / `sector_slug` / `sic`:

```text
data/raw/financial_report_qa/annual_reports/
  pdf_manifest.jsonl               all sectors (local_path = banking/EDGAR_ARS_JPM_2025.pdf)
  banking/pdf_manifest.jsonl       banking only (local_path = EDGAR_ARS_JPM_2025.pdf)
  banking/EDGAR_ARS_JPM_2025.pdf
  oil_and_gas/...
```

```powershell
python -m scripts.downloads.download_annual_report_corpus --dry-run                       # per-sector counts and GB, no network
python -m scripts.downloads.download_annual_report_corpus --user-agent "capstone-research you@example.com" --sectors banking,oil_and_gas --max-per-sector 20
# Index by sector: pass the corpus folder as the manifest; every chunk gets sector metadata.
python -m scripts.rag.financial_ingest_qdrant_db ingest --manifest data\raw\financial_report_qa\annual_reports --sectors banking
python -m scripts.rag.financial_ingest_qdrant_db query "net interest income" --sector banking
python scripts\rag\index_financial_report_pdfs.py --manifest data\raw\financial_report_qa\annual_reports --sectors banking --filter-sector banking --query "net interest income"
```

- **Reruns:** downloads are resumable. PDFs already in `data/raw/financial_report_qa/edgar/pdfs` are hard-linked or copied instead of fetched.
- **One collection for all sectors:** sectors are a payload-indexed field, not separate collections.
- **Older indexes:** `financial_ingest_qdrant_db tag-sectors --manifest <corpus>` adds the field to points ingested before it existed.
- **On Colab:** see the "sector corpus" cells in `notebooks/colab_financial_ingest.ipynb`. Use `--output-dir` to put the corpus on Drive.

### Page audit: which parser each page needs

After downloading the corpus (`full` or `min500`), `scripts.rag.page_audit` analyses every page and writes one CSV per PDF, in folders that mirror the sector folders: `page_audit/<sector_slug>/<pdf name>.csv` with `pageNumber,parserUsage`. `parserUsage` is `pdfParser`, `pdfParser+colpali`, `ocr`, `qwen3VL` or `paidVL` (rules in the module docstring); `_summary.csv` / `_summary.json` give the counts per filing and sector. Reruns skip finished filings.

```powershell
python -m scripts.rag.page_audit --manifest data\raw\financial_report_qa\annual_reports --corpus-preset min500 --out output\page_audit
awinancial_report_qannual_reports --corpus-preset min500 --out output\page_audit
python -m scripts.rag.colab_pipeline audit --corpus-preset min500      # Colab: writes <drive_root>/page_audit
```

### Two ingestion pipelines: 1 = Chroma text index, 2 = Qdrant ColPali index

Both pipelines read the same manifests (a file or the sector corpus folder, with `--sectors`), and both put `sector`, `ticker`, `fiscal_year` and a citable `chunk_id` on every point.

| | Pipeline 1: OCR / VLM, `scripts.rag.financial_ingest_chroma_db` (**Chroma only**) | Pipeline 2: ColPali, `scripts.rag.financial_ingest_qdrant_db` (**Qdrant only**) |
|---|---|---|
| Visual pages become | Text written by the page reader (tables as Markdown tables, charts as labelled values), then chunked and embedded | ColPali multivectors: mean-pooled rows/columns in RAM (first stage), full 1030×128 on disk (MaxSim rerank) |
| Pages | `--vlm-pages visual`, `tables_charts` or `all`; other pages keep their text layer | `--visual-pages rule` (tables, charts, pictures, unreadable pages), `tables_charts` or `all` |
| Store | Chroma `annual_reports_vlm` (dense) + a BM25 sidecar (`chroma/bm25.sqlite3`, SQLite FTS5 on number-aware terms), fused by RRF | Qdrant `annual_reports_visual` (+ `annual_reports_text` for text, dense + sparse BM25) |
| Page reader | `--vlm routed` (default: a reader per page from the page audit rules, with the `paidVL` fallback OCR → Qwen3-VL → paid model, plus a Qwen3-VL search description chunk for each chart and image page), `--vlm ocr` (RapidOCR + native tables, CPU), `--vlm transformers` (local Qwen3-VL-4B (2B / 8B), 4-bit on GPU), `--vlm openai` (OpenAI-compatible: self-hosted Qwen-VL on vLLM / llama.cpp / Ollama, or a paid API such as OpenAI or Gemini via `RAG_VISION_API_BASE/MODEL/KEY`) or `mock` | `vidore/colpali-v1.3-hf` (retrieval only; OCR for unreadable text pages) |
| Numbers | figures the text layer doesn't print are flagged as `unverified_numbers` | stay as exact strings in the payload, never in vectors |
| Chunk schema | the same fields in both indexes (`scripts/rag/chunk_schema.py`): filing, page, chunk, and provenance (`pipeline`, `text_source`, `reader`, `reader_model`, `page_route`, `escalation`, `number_check`, `unverified_numbers`); Chroma stores lists as JSON strings | same fields |

```powershell
python -m scripts.rag.financial_ingest_qdrant_db ingest --manifest data\raw\financial_report_qa\annual_reports --sectors banking --stages visual --visual-pages tables_charts
python -m scripts.rag.financial_ingest_chroma_db ingest --manifest data\raw\financial_report_qa\annual_reports --sectors banking --vlm ocr
python -m scripts.rag.financial_ingest_chroma_db ingest --manifest data\raw\financial_report_qa\annual_reports --sectors banking --vlm transformers
# paid API (bills per page; keep the page selection narrow)
$env:RAG_VISION_API_BASE="https://api.openai.com/v1"; $env:RAG_VISION_MODEL="gpt-4o-mini"; $env:RAG_VISION_API_KEY="sk-..."
python -m scripts.rag.financial_ingest_chroma_db ingest --root indexes\financial_vlm_paid --sectors banking --vlm openai --vlm-pages tables_charts
# a reader per page (page audit rules); paidVL pages: Qwen3-VL checked against OCR, paid model only when >10% of its figures are unconfirmed
$env:RAG_PAID_VISION_API_BASE="https://api.openai.com/v1"; $env:RAG_PAID_VISION_MODEL="gpt-4o"; $env:RAG_PAID_VISION_API_KEY="sk-..."
python -m scripts.rag.financial_ingest_chroma_db ingest --sectors banking --vlm routed --paid-threshold 0.1
python -m scripts.rag.financial_ingest_chroma_db query "net interest income" --sector banking
```

**On Colab:** first run `python -m scripts.common.colab_bundle` and upload `output/colab/capstone_code.zip` to `MyDrive/capstone_rag/`; every notebook that imports `scripts` unpacks it (see `notebooks/README.md`). Then open `notebooks/colab_rag_pipelines.ipynb`, choose `PIPELINE` (`colpali_qdrant` or `vlm_chroma`), the sectors and `SMOKE_TEST`, then Run all. Its defaults are the full plan: all 13 sectors (`SECTORS = ""`), every planned PDF (`MAX_PER_SECTOR = 0`), and ColPali on every page. It calls `scripts/rag/colab_pipeline.py`, which you can also run from a terminal:
- **Live index:** on the VM's local disk. Each sector's PDFs are copied there from Drive before ingesting (`stage_pdfs`).
- **Drive:** the PDFs by sector, the VLM transcriptions, and a copy of the index (a Qdrant snapshot or the Chroma folder after each sector; `snapshot_every=N` snapshots every N batches of 10 filings instead).
- **Next session:** the index is restored from Drive, and finished filings are skipped.

```powershell
python -m scripts.rag.colab_pipeline run --pipeline vlm_chroma --sectors banking --vlm openai --qdrant-url ""   # local
python -m scripts.rag.colab_pipeline status --pipeline colpali_qdrant --qdrant-url ""
```

## Financial RAG Index (Week 1 index)

This is the original single-folder index (`index_financial_report_pdfs.py`), used for offline smoke runs and the prompting baselines. The capstone serves the pipeline 1 index above; this one is used only when neither sector-corpus index exists (`RAG_BACKEND` unset, shown as `chroma`).

Fast local smoke-test mode with deterministic hashing embeddings:

```powershell
python scripts\rag\index_financial_report_pdfs.py --embedding-provider hashing --reset
```

Hosted embedding mode for final capstone runs:

```powershell
$env:EMBEDDING_API_BASE="https://your-hosted-endpoint/v1"
$env:EMBEDDING_API_KEY="your-key"
$env:EMBEDDING_MODEL="your-embedding-model"
python scripts\rag\index_financial_report_pdfs.py --embedding-provider hosted --reset
```

The financial indexer writes:

- chunks: `data/processed/financial_report_qa/pdf_chunks.jsonl`
- chart images: `data/processed/financial_report_qa/chart_images/`
- Chroma index: `indexes/chroma/financial_report_qa_pdfs`

## Retrieval Strategy (Week 1 index)

The Week 1 RAG pipeline uses:

- PDF text extraction with page-level metadata
- Recursive character chunking with overlap: financial defaults to `chunk_size=2000`, `chunk_overlap=250` (the Qdrant sector-corpus ingest uses 1,200 / 200)
- ChromaDB vector index using cosine HNSW
- BM25 lexical retrieval over `pdf_chunks.jsonl`
- Reciprocal rank fusion of dense and BM25 results
- Lightweight lexical reranking
- Citation-grounded answers using chunk IDs

For 500-2,000 PDFs, HNSW in ChromaDB is the recommended default. IVF/PQ or IVF-PQ is mainly useful when the vector count becomes much larger or memory pressure becomes the main bottleneck.

## Run Agents

Financial agent, offline mock mode:

```powershell
python scripts\agents\financial_prompting_agent.py --llm-provider mock --technique all --query "What does Apple say can affect quarterly net sales?" --ticker AAPL
```

Financial agent with a hosted OpenAI-compatible <=8B chat API:

```powershell
$env:LLM_API_BASE="https://your-hosted-endpoint/v1"
$env:LLM_API_KEY="your-key"
$env:LLM_MODEL="your-chat-model"
python scripts\agents\financial_prompting_agent.py --llm-provider hosted --technique agentic_graph --query "What are revenue risks?" --ticker AAPL
```

## Streamlit Chat UIs

API-backed Streamlit UI (FactFolio) for the final architecture:

```powershell
uvicorn scripts.api.api_app:app --host 127.0.0.1 --port 8000
streamlit run scripts\ui\api_streamlit_app.py
```

Financial UI:

```powershell
streamlit run scripts\ui\financial_streamlit_app.py
```

## FastAPI Backend

Run the backend:

```powershell
uvicorn scripts.api.api_app:app --host 127.0.0.1 --port 8000
```

Interactive docs are available at `http://127.0.0.1:8000/docs` (Swagger UI) and `http://127.0.0.1:8000/redoc` (ReDoc). The raw OpenAPI schema is at `http://127.0.0.1:8000/openapi.json`. See [docs/fastapi_api.md](docs/fastapi_api.md) for request fields, response shapes, error codes, examples, rate limits, and tracing setup. Streaming is not implemented; see the [proposed FastAPI streaming design](docs/fastapi_streaming_design.md).

Useful endpoints:

```text
GET  /health
GET  /domains
GET  /companies
POST /conversation/financial_report_qa
POST /chat/financial_report_qa
POST /retrieve/financial_report_qa
```

Example request:

```powershell
$body = @{
  query = "What was Costco's net income in fiscal 2025?"
  llm_provider = "mock"
  embedding_provider = "hashing"
  enable_guardrails = $true
  enable_human_review = $true
} | ConvertTo-Json

Invoke-RestMethod -Method Post -Uri "http://127.0.0.1:8000/chat/financial_report_qa" -ContentType "application/json" -Body $body
```

The API allows 60 requests per minute per client IP across all routes. Excess requests return `429 Too Many Requests` with a `Retry-After` header. The limiter is in-memory and per application process; use a shared rate-limit store such as Redis when running multiple workers or replicas. Client IPs come from the direct connection, so configure trusted proxy handling at the deployment boundary if the API is behind a reverse proxy.

Enable LangSmith for API calls by setting `LANGSMITH_API_KEY` and either passing `enable_langsmith=true` in the request body or setting `LANGSMITH_TRACING=true` in the environment. Tracing is per request: one request asking for it never turns it on for others. Every run carries the request ID, conversation turns share a `thread_id`, and LLM calls show their prompts and tokens. `/health` reports the setup without the key. See [docs/observability_langsmith.md](docs/observability_langsmith.md) ([HTML](docs/observability_langsmith.html)).

API access logs include request IDs, route, status, and duration without recording request bodies or prompts. Each response returns `X-Request-ID`; when LangSmith tracing is active, the ID is attached as trace metadata. See [docs/fastapi_api.md](docs/fastapi_api.md#request-logging-and-trace-correlation).

Chat endpoints return generated answers only when citations validate against retrieved evidence; otherwise the answer is withheld and the response explains the grounding failure. The agent's critic also checks support: every figure must appear in a cited chunk and an LLM judge must support every claim; with a hosted LLM an unverified answer is replaced by cited excerpts (see [docs/agent_flows.md](docs/agent_flows.md)). Financial chat also returns `not_available` when the filings can't answer the question; see [docs/fastapi_api.md](docs/fastapi_api.md#when-the-filings-cant-answer).

## Deployment

The API, UI and Qdrant ship as one Docker Compose stack: CPU only, using the hosted LLM. It runs the same on a laptop or on any cloud VM, and optionally adds HTTPS with a login.

```bash
cp .env.example .env                                   # LLM endpoint, API_AUTH_TOKEN, ...
docker compose up -d qdrant
docker compose run --rm --user root api sh deploy/restore_index.sh   # Colab snapshots in ./snapshots
docker compose up -d --build                           # UI http://127.0.0.1:8501, API http://127.0.0.1:8000/docs
docker compose --profile https up -d                   # on a VM with DOMAIN set: https://$DOMAIN
```

[docs/deployment.md](docs/deployment.md) ([HTML](docs/deployment.html)) covers:
- the deployment options (laptop, Colab, a VM, Qdrant Cloud);
- every setting;
- costs: building the index, closed-model VLM transcription, serving, and the LLM per 1,000 questions.

Capstone build plan: [docs/capstone_implementation_plan.md](docs/capstone_implementation_plan.md). Feature specifications: [docs/features_to_implement.md](docs/features_to_implement.md) ([HTML](docs/features_to_implement.html)).

## End-to-end commands

Use hashing embeddings and mock LLM mode for local smoke tests. For final capstone runs, replace `--embedding-provider hashing` with `--embedding-provider hosted` after setting the embedding endpoint variables, and use `--llm-provider hosted` after setting the chat endpoint variables.

### Financial Report QA

```powershell
python scripts\downloads\download_financial_report_pdfs.py --target-pdfs 100
python scripts\rag\index_financial_report_pdfs.py --embedding-provider hashing --reset
python scripts\agents\financial_prompting_agent.py --llm-provider mock --technique agentic_graph --query "What does Apple say can affect quarterly net sales?" --ticker AAPL
streamlit run scripts\ui\financial_streamlit_app.py
```

Optional top-k retrieval check:

```powershell
python scripts\rag\index_financial_report_pdfs.py --embedding-provider hashing --query-only --retrieval-mode hybrid --query "What does Apple report about net sales and revenue?" --top-k 5 --filter-ticker AAPL
```

## Guardrails, Governance, and HITL

The agent flow includes:

- LangGraph orchestration for security, validation, planning, retrieval, evidence checks, answer generation, output filtering, citation criticism, and governance
- Input guard for prompt injection, code/tool payloads, and secrets
- LLM-backed plain-text/domain-intent classifier
- Retrieval evidence guard against injected instructions inside retrieved chunks
- Output content filter for generated prompt-injection echoes, executable payloads, secrets, sensitive data, and domain-sensitive recommendations
- Citation validation against retrieved chunk IDs
- Financial agent only: a plain "not available" reply, with no figure, for a company or year without filings or a measure a bank doesn't report; answers citing a company the question didn't ask about are withheld
- Human review routing for high-impact decisions, unsafe evidence, low confidence, or citation failure
- HTML/JSON audit artifacts for review

See [docs/agent_flows.md](docs/agent_flows.md) and `docs/financial_report_agent_design.md` for flow diagrams and governance details.

Content filtering is layered:

```text
user input -> input filter -> plain-text classifier -> retrieval -> evidence filter -> answer -> output filter -> citation/HITL
```

The filters return auditable JSON fields such as `security`, `evidence_guard`, `output_filter`, and `human_review`. Output filter decisions are `allow`, `review`, or `block`; review-level content adds `output_content_filter_review` to HITL triggers.

## LangSmith Logging and Monitoring

LangSmith tracing is optional and can be enabled for agent runs and retrieval evaluation.

Set your LangSmith credentials:

```powershell
$env:LANGSMITH_API_KEY="your-langsmith-key"
$env:LANGSMITH_PROJECT="capstone-financial-report-qa"
```

Run the financial agent with LangSmith tracing:

```powershell
python scripts\agents\financial_prompting_agent.py --enable-langsmith --langsmith-project capstone-financial-report-qa --llm-provider mock --technique agentic_graph --query "What does Apple say can affect quarterly net sales?" --ticker AAPL
```

Run retrieval evaluation with LangSmith tracing:

```powershell
python scripts\evaluation\evaluate_financial_rag.py --enable-langsmith --langsmith-project capstone-financial-rag-eval --embedding-provider hashing
```

Each trace includes tags and metadata for domain, retrieval mode, LLM provider, embedding provider, top-k settings, guardrail status, and HITL status. If `--enable-langsmith` is used without `LANGSMITH_API_KEY`, the run still completes locally and reports LangSmith as disabled in the JSON output.

## Retrieval Evaluation

Generate a silver retrieval set and measure recall separately from answer quality:

```powershell
python scripts\evaluation\evaluate_financial_rag.py --embedding-provider hashing --generate-silver
```

Run evaluation:

```powershell
python scripts\evaluation\evaluate_financial_rag.py --embedding-provider hashing
```

The evaluation reports recall@k and related retrieval metrics without calling the LLM.

## Final Capstone Run Checklist

Before a final demo or report run:

- Download the corpus (`min500` or `full`) and ingest it (see [docs/deployment.md](docs/deployment.md)).
- Rebuild indexes with hosted embeddings instead of hashing embeddings.
- Use hosted chat mode for the final Agentic RAG answer path through FastAPI.
- Run `uvicorn scripts.api.api_app:app --host 127.0.0.1 --port 8000` and `streamlit run scripts\ui\api_streamlit_app.py` for the final API-backed demo.
- Run retrieval recall@k evaluation separately from answer-quality evaluation.
- Capture LangSmith traces for at least representative successful, refused, and HITL-routed runs.
- Keep generated data, indexes, HTML reports, and eval artifacts out of git.

## Tests

Run the full unit test suite:

```powershell
python -m pytest tests
```

Current coverage includes:

- financial PDF downloader helpers
- financial indexing helpers
- hybrid retrieval and evaluation helpers
- financial prompting agent guardrails, content filters, LangGraph workflows, and HITL routing
- agent routing, tools, retrieval retry, calculator, support-checking critic and step budget
- LangSmith optional tracing helpers
- the Streamlit feature guide and the financial-only API surface

## Notes

- Hashing embeddings are for local smoke tests and deterministic CI-style checks.
- Hosted embeddings and hosted chat APIs should be used for final capstone inference runs.
- ChromaDB (pipeline 1, `financial_vlm`) is the served vector database; Qdrant (pipeline 2, `financial_qdrant`) and the hybrid (`financial_hybrid`) are opt-in test backends. The older local Qdrant multimodal path (`RAG_BACKEND=qdrant`) is described below.
- Downloaded PDFs, processed chunks, Chroma indexes, outputs, and eval artifacts are generated files and are excluded from git.

## Local Qdrant multimodal RAG

See [the local setup guide](docs/local_multimodal_rag.md) for PDF ingestion, hybrid retrieval, image-aware local answers, and six included sample PDFs covering all domains. Set `RAG_BACKEND=qdrant` to use Qdrant from the existing app.

The complete CPU profile uses learned embeddings, a cross-encoder reranker, and a local Qwen vision model. Run `.venv/Scripts/python.exe -m scripts.rag.run_local_multimodal evaluate` to validate text, tables, and an image-only chart; run the same module with `serve` to start the prepared local API and model service.

For ColPali page multivectors and Qdrant MaxSim retrieval, see [ColPali implementation and Qdrant storage](docs/colpali_qdrant.md). Backfill the existing index with `python -m scripts.rag.colpali_index backfill`, then start the API with `python -m scripts.rag.run_local_multimodal serve --colpali`.

Selective OCR now adds scanned-page and embedded-image text to the same text-vector collection. Use `python -m scripts.rag.ocr_index` to backfill existing PDFs; see [OCR implementation and validation](docs/selective_ocr.md).
