# Revenue Intelligence POC

這是一個以 Excel 營收與庫存資料為核心的分析與問答 POC。專案會讀取 `data/` 內的三份工作簿，清理欄位、解析匿名化 mapping、建立跨資料集分析表，輸出 Excel、Markdown 報告與圖表，並提供 multi-agent 問答 API 與 Next.js 前端介面。

目前專案重點是「可追溯的商業分析」：回答會保留資料範圍、工具證據、限制說明與前端可呈現的 display blocks，避免把背景資訊或不充分證據包裝成確定結論。

## 功能總覽

- 讀取營收、庫存、mapping 三份 Excel 檔案
- 將 `YYYYMM`、日期欄位等資料格式標準化
- 解析匿名化 business group、HQBU、platform 對應關係
- 建立月度趨勢、平台比較、庫存金額、庫存數量、營收/庫存 proxy 等分析資料
- 產出清理後 Excel、彙總 Excel、Markdown 報告與 PNG 圖表
- 提供 deterministic multi-agent 問答流程
- 可選擇接入 Ollama，啟用 LLM planner / rewriter
- 提供 Python HTTP API 給前端代理呼叫
- 提供 Next.js 桌面 dashboard 與 mobile executive demo 路由
- 內建 regression tests、demo smoke、planner eval 與 live LLM smoke 腳本

## 專案結構

```text
.
├─ data/                 # 輸入資料：inventory.xlsx、revenue.xlsx、mapping.xlsx
├─ output/               # 分析輸出與 logs；執行流程後產生或更新
│  └─ charts/            # 圖表 PNG
├─ frontend/             # 主要 Next.js app
│  ├─ app/dashboard/     # 桌面分析工作台
│  ├─ app/mobile/        # 行動版 executive demo
│  ├─ app/api/           # Next.js proxy routes
│  ├─ components/        # shared charts、chat、KPI components
│  └─ lib/python-api.js  # Python API client
├─ scripts/              # 啟動、smoke test、eval 腳本
├─ tests/                # pytest 測試
├─ eval/                 # 評測資料與輸出報告
├─ docs/                 # 架構、API、前端整併與 response logic 文件
├─ main.py               # 離線分析與 CLI 入口
├─ demo_web.py           # Python HTTP API 入口
├─ multi_agent.py        # multi-agent 問答主流程
├─ analysis_tools.py     # 分析工具箱與圖表/觀測資料查詢
├─ answer_contract.py    # 回答契約與前端 display blocks
├─ evidence_projector.py # evidence 到 executive display 的投影邏輯
└─ pyproject.toml        # Python 依賴定義
```

## 環境需求

- Python `3.12`
- `uv`，建議使用
- Node.js LTS 與 npm，用於 `frontend/`
- 可選：Ollama，用於 live LLM planner / rewriter

`pyproject.toml` 是 Python 依賴的主要來源；`requirements.txt` 保留作為相容安裝方式。

## 快速開始

### 1. 安裝 Python 依賴

建議使用 `uv`：

```powershell
uv venv --python 3.12
uv sync
```

或使用一般 venv：

```powershell
python -m venv .venv
.\.venv\Scripts\activate
pip install -r requirements.txt
```

### 2. 執行離線分析流程

```powershell
uv run python main.py
```

此流程會讀取：

- `data/inventory.xlsx`
- `data/revenue.xlsx`
- `data/mapping.xlsx`

並更新 `output/` 內的分析產物。

如需重建測試資料：

```powershell
uv run python main.py --generate-test-data
```

### 3. 啟動 Python API

```powershell
uv run python demo_web.py
```

預設 API 位址：

```text
http://127.0.0.1:8765
```

### 4. 啟動前端

另開一個 PowerShell：

```powershell
cd frontend
npm install
$env:PYTHON_API_BASE="http://127.0.0.1:8765"
npm run dev -- --hostname 127.0.0.1 --port 3000
```

前端入口：

- Dashboard: `http://127.0.0.1:3000/dashboard`
- Mobile demo: `http://127.0.0.1:3000/mobile`
- Root `/` 會自動導向 `/dashboard`

也可以使用腳本啟動後端或前端：

```powershell
.\scripts\start_backend.ps1
.\scripts\start_frontend.ps1
```

## CLI 使用方式

執行完整分析：

```powershell
uv run python main.py
```

分析後進入終端問答模式：

```powershell
uv run python main.py --chat
```

輸出 multi-agent 專案摘要：

```powershell
uv run python main.py --project-summary
```

詢問單一問題：

```powershell
uv run python main.py --question "summarize current project capability"
```

輸出完整 JSON 回答：

```powershell
uv run python main.py --question "which platform has the weakest revenue inventory proxy?" --agent-json
```

也可以使用獨立 assistant CLI：

```powershell
uv run python agent_cli.py --project-summary
uv run python agent_cli.py --question "summarize current project capability" --json
```

## API 端點

`demo_web.py` 使用 Python 標準庫 `ThreadingHTTPServer` 啟動 API。主要端點如下：

| Method | Path | 說明 |
|---|---|---|
| `GET` | `/` | API 服務資訊與端點清單 |
| `GET` | `/api/health` | API、pipeline、資料狀態摘要 |
| `GET` | `/api/data-version` | 資料版本、月份範圍、row counts |
| `GET` | `/api/pipeline-status` | pipeline infos / warnings / errors |
| `GET` | `/api/data-quality` | 資料品質、mapping 狀態、限制說明 |
| `GET` | `/api/summary` | 專案與資料摘要 |
| `GET` | `/api/chart-catalog` | 可用圖表清單 |
| `GET` | `/api/observe-options` | 可查詢維度與篩選選項 |
| `POST` | `/api/ask` | multi-agent 問答 |
| `POST` | `/api/chart` | 取得圖表 payload 與可選圖像 |
| `POST` | `/api/observe` | 取得觀測資料表 |

`/api/ask` 範例：

```powershell
$body = @{
  question = "which platform has the weakest revenue inventory proxy?"
  use_llm_planner = $false
  use_llm_rewriter = $false
} | ConvertTo-Json

Invoke-RestMethod `
  -Method Post `
  -Uri "http://127.0.0.1:8765/api/ask" `
  -ContentType "application/json; charset=utf-8" `
  -Body $body
```

## 前端

`frontend/` 是目前主要前端 app。Next.js routes：

- `/dashboard`：桌面分析工作台
- `/mobile`：行動版 executive demo
- `/api/*`：代理 Python API 的 shared routes

前端 proxy 預設連到 `http://127.0.0.1:8765`，可用 `PYTHON_API_BASE` 覆寫。

常用命令：

```powershell
cd frontend
npm install
npm run dev
npm run build
npm run start
```

## 輸出檔案

執行 `main.py` 或建立 pipeline context 後，主要輸出會放在 `output/`：

- `output/cleaned_inventory.xlsx`
- `output/cleaned_revenue.xlsx`
- `output/parsed_mapping.xlsx`
- `output/merged_analysis.xlsx`
- `output/summary_metrics.xlsx`
- `output/analysis_report.md`
- `output/llm_explanation.md`
- `output/qa_transcript.md`，使用 `--chat` 時產生
- `output/charts/*.png`
- `output/logs/`，API、CLI、eval 等流程的 logs

## Ollama 與 LLM 選配

專案可以在沒有 LLM 的情況下用 deterministic 規則回答。若要測試 live LLM planner / rewriter，可先啟動 Ollama 並拉取模型。

目前程式設定位於 `config.py`：

- Base URL: `http://localhost:11434`
- Model: `gemma4:e4b`
- Timeout: `90` 秒

啟動 Ollama：

```powershell
ollama serve
ollama pull gemma4:e4b
```

啟動 Python API 時可用環境變數控制：

```powershell
$env:USE_LLM_PLANNER="true"
$env:USE_LLM_REWRITER="true"
uv run python demo_web.py
```

若 Ollama 不可用，建議保留 `use_llm_planner=false`、`use_llm_rewriter=false`，讓 demo 使用 deterministic path。

## 測試與驗證

執行 pytest：

```powershell
uv run pytest
```

執行 deterministic eval：

```powershell
uv run python eval/run_eval.py
```

執行 demo smoke，直接呼叫 assistant：

```powershell
uv run python scripts/demo_smoke.py
```

執行 demo smoke，透過 API：

```powershell
uv run python scripts/demo_smoke.py --api-url http://127.0.0.1:8765
```

測試 LLM planner：

```powershell
uv run python scripts/eval_llm_planner.py --mock
```

Live LLM smoke：

```powershell
uv run python scripts/smoke_llm_live.py --live
```

前端 production build：

```powershell
cd frontend
npm run build
```

## 重要設計概念

- `analysis_pipeline.py` 建立共享 pipeline context，供 CLI、API、tests 使用。
- `analysis_tools.py` 封裝可被 assistant 使用的 deterministic 分析工具。
- `business_question_classifier.py`、`task_profile.py`、`answer_plan.py` 負責問題分類與回答規劃。
- `analysis_rubrics.py` 將 evidence 分成 primary、supporting、background、debug。
- `evidence_projector.py` 只把合適的 primary / supporting evidence 投影到前端 display blocks。
- `answer_contract.py` 維持前後端穩定契約，保留 answer、evidence、limitations、tools_used、display_blocks 等欄位。
- `frontend/components/chat/` 會讀取 API 回答中的 chart evidence 與 display blocks，更新對話與圖表狀態。

## 開發注意事項

- 新增 Python 依賴時，優先更新 `pyproject.toml`。
- 新前端功能應放在 `frontend/`；`/dashboard` 與 `/mobile` 是目前主要入口。
- 回答邏輯應優先使用 deterministic evidence，不應讓 LLM 直接決定商業結論。
- 若修改 `/api/ask` 回傳格式，請同步檢查 frontend display、tests 與 eval。
- `output/` 是產物目錄，內容可重新產生。

## 常見問題

### API 啟動後前端仍無法取得資料

確認 Python API 正在 `http://127.0.0.1:8765`，且前端環境變數有正確設定：

```powershell
$env:PYTHON_API_BASE="http://127.0.0.1:8765"
```

### 問答結果沒有使用 LLM

這是預期行為。預設路徑是 deterministic，只有在 API payload 或環境變數啟用 LLM planner / rewriter 時才會嘗試呼叫 Ollama。

### 中文輸出看起來亂碼

請確認終端機與編輯器使用 UTF-8。PowerShell 可先執行：

```powershell
chcp 65001
```

### 圖表或報告沒有更新

重新執行：

```powershell
uv run python main.py
```

如果輸入 Excel schema 有變動，請先檢查 `data/` 內的欄位是否仍符合 `config.py` 的欄位契約。
