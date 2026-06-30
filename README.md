# Revenue Intelligence POC

This is an analysis proof of concept built around Excel-based revenue and inventory data. The project reads local Excel files, normalizes the data into analysis dimensions such as month, business group, and five major product lines, and provides querying, charting, and demo interfaces through a deterministic tool layer, multi-agent Q&A, a Python API, and a Next.js UI.

The current main workflow is the `real data` analysis path, which uses local files such as `data/inventory.xlsx` and `data/revenue.xlsx` to build the analysis context. The legacy offline reporting workflow still exists, but it requires an additional `data/mapping.xlsx` file.

> Note: The `data/` directory may be excluded from the public repository because it can contain internal or confidential source data.

## Project Capabilities

- Read real inventory and revenue Excel data
- Normalize month values into `YYYY-MM`
- Analyze entities by `business_group` and `product_line_5`
- Generate monthly revenue, inventory amount, inventory quantity, rankings, anomaly checks, and proxy ratio analysis
- Use `MultiAgentAssistant` to answer questions about sales, inventory, financial proxies, charts, and data quality
- Provide a Python HTTP API for frontend proxy calls
- Provide a Next.js dashboard and mobile demo route
- Optionally connect to Ollama as an LLM planner / rewriter, while keeping deterministic fallback enabled by default

## Project Structure

```text
.
├── data/                         # Local Excel source files, usually not committed
├── docs/                         # Design, API, demo, and LLM runbooks
├── eval/                         # Regression / evaluation outputs
├── frontend/                     # Next.js UI
├── scripts/                      # Backend/frontend startup and smoke scripts
├── tests/                        # unittest test suite
├── analysis_pipeline.py          # Builds PipelineContext
├── analysis_tools.py             # Deterministic analysis tools
├── demo_web.py                   # Python API server, port 8765
├── main.py                       # Legacy report CLI + agent CLI compatibility
├── multi_agent.py                # Multi-agent orchestration
├── real_data.py                  # Real inventory/revenue normalization
├── pyproject.toml                # Python dependency source of truth
└── requirements.txt              # pip fallback dependency list
```

## Data Files

The project expects the following local files:

- `data/inventory.xlsx`
  - Main sheet: `工作表1`
  - Columns include `Wn日期`, `年`, `月`, `HQBU`, `typename`, `金額`, `QTY`, `Productline_5`, `五大產品線`, `新事業群`

- `data/revenue.xlsx`
  - Main sheet: `工作表1`
  - Columns include `公司類別`, `年度`, `月份`, `合併事業群`, `產品類別名稱`, `實際營收`, `五大產品線`, `新事業群`

`real_data.py` converts the raw columns into standardized fields.

Inventory standardized fields:

```text
month_key
year
month
hqbu
inventory_type
inventory_amount
inventory_qty
productline_raw
product_line_5
business_group
```

Revenue standardized fields:

```text
month_key
year
month
company_type
merged_business_group
product_category_name
revenue_amount
product_line_5
business_group
```

The analysis join key is:

```text
month_key + business_group + product_line_5
```

Note: `data/mapping.xlsx` is not required for the real-data assistant path. `demo_web.py`, `agent_cli.py`, and `main.py --question` use the real-data assistant path and do not require the mapping file.

However, directly running `python main.py` will trigger the legacy offline reporting workflow, which still looks for `mapping.xlsx`.

## Environment Requirements

- Python 3.12
- Node.js LTS + npm, for `frontend/`
- Optional: `uv`
- Optional: Ollama, for live LLM planner / rewriter

`pyproject.toml` specifies:

```text
requires-python = ">=3.12,<3.13"
```

Main Python packages:

- `pandas`
- `openpyxl`
- `matplotlib`
- `requests`

## Installation

Using `uv` is recommended:

```powershell
uv venv --python 3.12
uv sync
```

If `uv` is not available, use a standard virtual environment:

```powershell
py -3.12 -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install --upgrade pip
pip install -r requirements.txt
```

Install frontend dependencies:

```powershell
cd frontend
npm install
```

## Common Startup Commands

### 1. Start the Python API

```powershell
python demo_web.py
```

The API will be available at:

```text
http://127.0.0.1:8765
```

### 2. Start the Next.js Dashboard

Open another PowerShell window:

```powershell
cd frontend
$env:PYTHON_API_BASE="http://127.0.0.1:8765"
npm run dev
```

Open:

```text
http://127.0.0.1:3000/dashboard
http://127.0.0.1:3000/mobile
```

### 3. Start with Scripts

```powershell
.\scripts\start_backend.ps1
.\scripts\start_frontend.ps1
```

Or start the backend, desktop UI, and mobile UI together:

```powershell
.\scripts\start_all.ps1
```

`start_all.ps1` expects the services to run at:

- API: `http://127.0.0.1:8765`
- Web UI: `http://127.0.0.1:3000`
- Mobile UI: `http://127.0.0.1:3001`

## CLI Usage

Generate a multi-agent project summary:

```powershell
python main.py --project-summary
```

Ask a single question:

```powershell
python main.py --question "What months are covered by the current dataset?"
```

Output full JSON:

```powershell
python main.py --question "How is the current data quality?" --agent-json
```

You can also use the agent CLI directly:

```powershell
python agent_cli.py --project-summary
python agent_cli.py --question "Compare the latest monthly revenue performance by business group" --json
```

Legacy reporting workflow:

```powershell
python main.py
```

This path reads `data/inventory.xlsx`, `data/revenue.xlsx`, and `data/mapping.xlsx`, then writes outputs to `output/`.

If you want to run the legacy report, provide the mapping file first or use the test data generator.

Generate test data:

```powershell
python main.py --generate-test-data
```

Warning: this will overwrite `data/inventory.xlsx` and `data/revenue.xlsx`, and it will also create `data/mapping.xlsx`. If you want to keep the current real data, back up the `data/` directory first.

## API

`demo_web.py` provides the following endpoints:

- `GET /api/health`
- `GET /api/data-version`
- `GET /api/pipeline-status`
- `GET /api/data-quality`
- `GET /api/summary`
- `GET /api/chart-catalog`
- `GET /api/observe-options`
- `POST /api/ask`
- `POST /api/chart`
- `POST /api/observe`

Example:

```powershell
Invoke-RestMethod http://127.0.0.1:8765/api/health
```

```powershell
Invoke-RestMethod `
  -Method Post `
  -Uri http://127.0.0.1:8765/api/ask `
  -ContentType "application/json; charset=utf-8" `
  -Body '{"question":"What months are covered by the current dataset?"}'
```

The Next.js frontend proxies requests to the Python API through `frontend/app/api/*`.

The default Python API base is:

```text
http://127.0.0.1:8765
```

It can be overridden with an environment variable:

```powershell
$env:PYTHON_API_BASE="http://127.0.0.1:8765"
```

## LLM / Ollama

The LLM planner and rewriter are disabled by default:

```powershell
$env:USE_LLM_PLANNER="false"
$env:USE_LLM_REWRITER="false"
```

Even if Ollama is not available, the system will continue to use deterministic fallback.

To test live LLM mode:

```powershell
ollama serve
ollama pull gemma4:e4b
```

You can configure:

```powershell
$env:OLLAMA_BASE_URL="http://localhost:11434"
$env:OLLAMA_MODEL="gemma4:e4b"
$env:OLLAMA_TIMEOUT_SECONDS="90"
```

Planner-only rehearsal:

```powershell
$env:USE_LLM_PLANNER="true"
$env:USE_LLM_REWRITER="false"
```

For details, see:

```text
docs/llm_enablement_runbook.md
```

## Testing and Validation

Run the full Python unittest suite:

```powershell
python -m unittest discover -s tests -v
```

Run deterministic demo smoke:

```powershell
python scripts\demo_smoke.py
```

Run smoke tests against a running API:

```powershell
python scripts\demo_smoke.py --api-url http://127.0.0.1:8765
```

Run LLM availability smoke:

```powershell
python scripts\smoke_llm_live.py
```

Run a production-like frontend check:

```powershell
cd frontend
$env:PYTHON_API_BASE="http://127.0.0.1:8765"
npm run build
npm run start
```

## Outputs

The Python pipeline and smoke scripts generate local outputs such as:

- `output/`
- `output/logs/`
- `output/charts/`
- `eval/*.csv`
- `eval/*.md`

These are mostly local execution artifacts and should not be treated as the source of truth.

## FAQ

### `uv` is not found

You can use `py -3.12 -m venv .venv` and `pip install -r requirements.txt` instead.

`uv` is recommended, but it is not the only installation option.

### `python main.py` says mapping is missing

This is the legacy offline reporting workflow.

If you only want to use the current real-data assistant, API, or UI, run:

```powershell
python demo_web.py
python main.py --project-summary
python main.py --question "How is the current data quality?"
```

If you specifically want to run the legacy report, provide `data/mapping.xlsx` or run:

```powershell
python main.py --generate-test-data
```

### Frontend `/api/*` requests fail

Make sure the Python API is running and that the frontend environment variable points to the correct location:

```powershell
$env:PYTHON_API_BASE="http://127.0.0.1:8765"
```

### Chinese text becomes garbled in the terminal or logs

Use a terminal that supports UTF-8, or enter Chinese questions through the frontend UI.

Some older documents and legacy display strings may contain historical encoding issues. This README is based on the currently verifiable program flow and data contract.

## Important Documents

- `docs/real_data_contract.md`
- `docs/pipeline_architecture.md`
- `docs/api_status_endpoints.md`
- `docs/demo_checklist.md`
- `docs/llm_enablement_runbook.md`
- `frontend/README.md`

## Data Security Note

This project may use internal revenue and inventory data during local development.

Do not commit confidential Excel files, raw exports, or generated reports to a public repository. Recommended `.gitignore` rules:

```gitignore
data/
*.xlsx
*.csv
output/
eval/
```
