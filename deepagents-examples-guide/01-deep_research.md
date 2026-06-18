> 原始路徑：docs/deepagents/examples/deep_research/

# 讓 AI 自己去查、自己去想：Deep Research Agent 完整程式碼導讀

**副標：一個 orchestrator、多個 subagent、兩支工具——看 deepagents 如何把「深度研究」變成可部署的 production agent。**

---

## 引言：當你問 AI 一個需要真正查資料的問題

「比較 OpenAI、Anthropic、DeepMind 在 AI 安全上的做法」——這類問題不是一次 prompt 就能解決的。它需要：瀏覽多個網頁、記住哪些資訊從哪裡來、交叉比對、最後整合成有條理的報告。

`deep_research` 這個範例，就是為了示範 **Deep Agents 如何以兩層架構解決這件事**：一個 orchestrator 負責拆解問題與整合結果，多個 research subagent 並行爬梳網路、蒐集原始資料。整個系統是可以直接用 `langgraph dev` 啟動、對外提供服務的 production-ready agent。

這篇文章會帶你逐一讀懂每一個核心檔案，理解每個設計決策背後的理由。

---

## 全貌：整體架構與資料流

先看清楚整個系統長什麼樣子：

```
使用者輸入問題
      │
      ▼
┌─────────────────────────────────┐
│  Orchestrator（create_deep_agent）│
│  - 讀 system_prompt（INSTRUCTIONS）│
│  - 使用 write_todos 規劃任務      │
│  - 呼叫 task() 委派給 subagent   │
│  - write_file 寫入 research_request.md │
│  - write_file 寫入 final_report.md    │
└──────────────┬──────────────────┘
               │  task() × N（最多 3 個並行）
               ▼
┌──────────────────────────────────┐
│  Research Subagent × N           │
│  name: "research-agent"          │
│  tools: tavily_search, think_tool│
│  - 搜尋網路、抓取完整頁面內容     │
│  - think_tool 反思進度            │
│  - 回傳帶引用的 findings          │
└──────────────────────────────────┘
               │  findings 回傳
               ▼
       Orchestrator 彙整引用、
       撰寫 final_report.md
```

**角色與職責一覽：**

| 角色 | 對應程式 | 職責 |
|------|----------|------|
| Orchestrator | `agent.py` 中的 `create_deep_agent(...)` | 規劃、委派、彙整 |
| Research Subagent | `research_sub_agent` dict | 聚焦搜尋單一主題 |
| `tavily_search` | `research_agent/tools.py` | 網路搜尋 + 全文抓取 |
| `think_tool` | `research_agent/tools.py` | 搜尋間反思與決策 |
| Prompt 集 | `research_agent/prompts.py` | 三層行為規範 |
| Virtual Filesystem | deepagents 內建 | 儲存 `.md` 中間檔案 |

---

## 環境與安裝

### 相依套件（`pyproject.toml`）

```toml
[project]
name = "deep-research-example"
version = "0.1.0"
requires-python = ">=3.11"
dependencies = [
    "deepagents>=0.6.8",
    "tavily-python>=0.7.25",
    "httpx>=0.28.1",
    "markdownify>=1.2.2",
    "langchain-anthropic>=1.4.4",
    "langchain-google-genai>=4.2.4",
    "rich>=15.0.0",
    "langgraph-cli[inmem]>=0.4.27",
    ...
]
```

幾個值得注意的依賴：

- **`deepagents>=0.6.8`**：整個框架的核心，提供 `create_deep_agent`、virtual filesystem、TODO 工具、task 委派機制。
- **`tavily-python`**：Tavily 官方 Python SDK，用來找到相關 URL。
- **`httpx` + `markdownify`**：直接對 URL 發出 HTTP 請求、把 HTML 轉 markdown——這是刻意繞開 Tavily 的摘要（snippet），保留完整頁面原文供 agent 分析。
- **`langgraph-cli[inmem]`**：讓你能用 `langgraph dev` 在本機起一個 LangGraph server。

### 安裝方式

```bash
cd examples/deep_research
uv sync
```

### API 金鑰設定

```bash
export ANTHROPIC_API_KEY=...  # 使用 Claude 時必填
export TAVILY_API_KEY=...     # 網路搜尋必填
export LANGSMITH_API_KEY=...  # 追蹤觀測（選填）
export GOOGLE_API_KEY=...     # 改用 Gemini 時才需要
```

---

## ★ 核心程式碼逐段導讀

### 1. `agent.py`：把所有零件組裝起來

這是整個 agent 的入口，也是 LangGraph server 的掛載點。全檔不到 60 行，卻是整個架構的樞紐。

#### 1a. Import 與常數設定

```python
from deepagents import create_deep_agent
from research_agent.prompts import (
    RESEARCHER_INSTRUCTIONS,
    RESEARCH_WORKFLOW_INSTRUCTIONS,
    SUBAGENT_DELEGATION_INSTRUCTIONS,
)
from research_agent.tools import tavily_search, think_tool

max_concurrent_research_units = 3
max_researcher_iterations = 3
current_date = datetime.now().strftime("%Y-%m-%d")
```

**為什麼把上限抽成常數？** `max_concurrent_research_units` 和 `max_researcher_iterations` 會被注入到 `SUBAGENT_DELEGATION_INSTRUCTIONS` 的 format string 裡，這樣你只需要改一個地方就能調整整個系統的「探索深度」。`current_date` 則注入給 subagent，讓它知道今天是幾號——對於要查新聞或近況的查詢非常重要。

#### 1b. 組合 Orchestrator 的 Prompt

```python
INSTRUCTIONS = (
    RESEARCH_WORKFLOW_INSTRUCTIONS
    + "\n\n"
    + "=" * 80
    + "\n\n"
    + SUBAGENT_DELEGATION_INSTRUCTIONS.format(
        max_concurrent_research_units=max_concurrent_research_units,
        max_researcher_iterations=max_researcher_iterations,
    )
)
```

注意：**`RESEARCHER_INSTRUCTIONS` 沒有放進 `INSTRUCTIONS`**。它只屬於 subagent，不給 orchestrator 看。這是刻意的職責分離——orchestrator 只需要知道「如何規劃與委派」，subagent 才需要知道「如何搜尋」。

#### 1c. 定義 Research Subagent

```python
research_sub_agent = {
    "name": "research-agent",
    "description": "Delegate research to the sub-agent researcher. Only give this researcher one topic at a time.",
    "system_prompt": RESEARCHER_INSTRUCTIONS.format(date=current_date),
    "tools": [tavily_search, think_tool],
}
```

這是一個 **dict 格式的 subagent 定義**，是 deepagents 框架接受的規格。三個關鍵欄位：

- `name`：orchestrator 呼叫 `task()` 時識別目標 subagent 的名稱。
- `description`：告訴 orchestrator 這個 subagent 擅長什麼、怎麼用——這段文字直接影響 orchestrator 決定要不要呼叫它。`"Only give this researcher one topic at a time"` 是一個重要的使用說明，避免 orchestrator 一次塞太多事情。
- `system_prompt`：subagent 啟動時的 system prompt，注入了 `current_date`。
- `tools`：subagent 能用的工具，這裡只有 `tavily_search` 和 `think_tool`，不多也不少。

#### 1d. 建立 Agent

```python
model = init_chat_model(model="anthropic:claude-sonnet-4-5-20250929", temperature=0.0)

agent = create_deep_agent(
    model=model,
    tools=[tavily_search, think_tool],
    system_prompt=INSTRUCTIONS,
    subagents=[research_sub_agent],
)
```

`create_deep_agent` 是 deepagents 框架的核心工廠函式。它在底層：

1. 把 `model` 包裝進一個具備 virtual filesystem、TODO 管理、task 委派能力的 LangGraph agent。
2. 把 `tools` 掛到 orchestrator 上（orchestrator 本身也可以用 `tavily_search`，但 prompt 設計讓它通常透過 subagent 去做）。
3. 把 `subagents` 列表轉換成 orchestrator 可呼叫的 `task()` 工具。
4. 把 `system_prompt` 與 deepagents middleware 的內建指示合併。

`temperature=0.0` 讓 orchestrator 的規劃行為更穩定可預測。

---

### 2. `research_agent/prompts.py`：行為規範的三層結構

這個檔案定義了三個 prompt 字串，是整個 agent 「怎麼思考」的核心。

#### 2a. `RESEARCH_WORKFLOW_INSTRUCTIONS`：Orchestrator 的工作流程

```
1. Plan: Create a todo list with write_todos to break down the research into focused tasks
2. Save the request: Use write_file() to save the user's research question to /research_request.md
3. Research: Delegate research tasks to sub-agents using the task() tool
4. Synthesize: Review all sub-agent findings and consolidate citations
5. Write Report: Write a comprehensive final report to /final_report.md
6. Verify: Read /research_request.md and confirm you've addressed all aspects
```

這六個步驟定義了 orchestrator 的標準作業流程。值得注意的幾個設計：

- **步驟 1 先 `write_todos`**：利用 deepagents 內建的 TODO 工具，讓 orchestrator 先把任務拆解「寫下來」，再開始執行。這避免了「邊想邊做」導致遺漏某些研究面向的問題。
- **步驟 2 寫入 `research_request.md`**：把使用者的原始問題存到 virtual filesystem。這是 Deep Agents 的 virtual filesystem 能力——在整個 agent 執行期間，所有寫入的 `.md` 檔案都存在 LangGraph 的 state 裡，可以重複讀取。
- **步驟 6 驗證**：回頭讀 `research_request.md`，確認報告有沒有遺漏任何面向。這是一個自我驗證迴圈，降低「忘記回答某個子問題」的機率。

引用格式的設計也很細：

```
- Cite sources inline using [1], [2], [3] format
- Assign each unique URL a single citation number across ALL sub-agent findings
- End report with ### Sources section listing each numbered source
```

由於多個 subagent 會各自回傳帶有引用的 findings，orchestrator 需要「重新統一編號」——每個唯一 URL 在最終報告中只出現一個序號，不能重複。這個規則是在 prompt 層面硬性規定的。

#### 2b. `RESEARCHER_INSTRUCTIONS`：Subagent 的搜尋守則

```python
RESEARCHER_INSTRUCTIONS = """You are a research assistant conducting research on the user's input topic.
For context, today's date is {date}.
...
<Hard Limits>
**Tool Call Budgets** (Prevent excessive searching):
- **Simple queries**: Use 2-3 search tool calls maximum
- **Complex queries**: Use up to 5 search tool calls maximum
- **Always stop**: After 5 search tool calls if you cannot find the right sources
...
"""
```

這段 prompt 有幾個關鍵設計：

- **`{date}` 動態注入**：在 `agent.py` 的 `research_sub_agent` 定義時 `.format(date=current_date)` 填入，讓 subagent 知道時間背景。
- **硬性搜尋上限（2-5 次）**：避免 subagent 陷入無限搜尋迴圈，控制 API token 消耗。
- **`<Hard Limits>` / `<Instructions>` 等 XML 標籤格式**：Claude 模型對 XML 標籤區塊的結構理解特別好，用標籤把不同類型的指示隔開，有助於模型更精確地遵守各區塊的規則。
- **「Stop Immediately When」的明確停止條件**：告訴 subagent 什麼情況下「夠了就好」，避免過度搜尋。

#### 2c. `SUBAGENT_DELEGATION_INSTRUCTIONS`：委派策略

```python
SUBAGENT_DELEGATION_INSTRUCTIONS = """...
## Delegation Strategy

**DEFAULT: Start with 1 sub-agent** for most queries:
- "What is quantum computing?" → 1 sub-agent
...

**ONLY parallelize when the query EXPLICITLY requires comparison:**
- "Compare OpenAI vs Anthropic vs DeepMind" → 3 parallel sub-agents
...

## Parallel Execution Limits
- Use at most {max_concurrent_research_units} parallel sub-agents per iteration
- Make multiple task() calls in a single response to enable parallel execution
"""
```

這段 prompt 的核心洞見是：**並行不是越多越好**。預設用 1 個 subagent，只有在明確需要比較或地理分區時才並行。這是 token 效率的考量——拆太細反而產生更多 overhead 和 context 浪費。

`{max_concurrent_research_units}` 和 `{max_researcher_iterations}` 由 `agent.py` 在 `.format()` 時填入（分別是 3 和 3），讓上限設定集中在 `agent.py` 管理。

---

### 3. `research_agent/tools.py`：兩支關鍵工具

#### 3a. `tavily_search`：URL 探索引擎 + 全文抓取器

```python
@tool(parse_docstring=True)
def tavily_search(
    query: str,
    max_results: Annotated[int, InjectedToolArg] = 1,
    topic: Annotated[Literal["general", "news", "finance"], InjectedToolArg] = "general",
) -> str:
    # 使用 Tavily 探索 URL
    search_results = tavily_client.search(query, max_results=max_results, topic=topic)

    # 逐一抓取每個 URL 的完整內容
    for result in search_results.get("results", []):
        url = result["url"]
        content = fetch_webpage_content(url)
        ...
```

這個工具有兩個值得深究的設計決策：

**`InjectedToolArg` 的用途**：`max_results` 和 `topic` 標記為 `InjectedToolArg`，代表這兩個參數**不會暴露給 LLM 選擇**，而是由 deepagents 框架在呼叫時自動注入。LLM 只需要提供 `query`，框架控制其餘參數，避免 LLM 做出不當的參數選擇。

**為什麼不用 Tavily 的摘要，改自己抓全文？** Tavily 回傳的 `snippet` 是截斷過的摘要，可能漏掉關鍵細節。這個工具直接對 `url` 做 HTTP GET，再用 `markdownify` 把 HTML 轉成 markdown，讓 agent 看到完整的頁面內容。

`fetch_webpage_content` 的細節：

```python
def fetch_webpage_content(url: str, timeout: float = 10.0) -> str:
    headers = {
        "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) ..."
    }
    try:
        response = httpx.get(url, headers=headers, timeout=timeout)
        response.raise_for_status()
        return markdownify(response.text)
    except Exception as e:
        return f"Error fetching content from {url}: {str(e)}"
```

**為什麼用假的 User-Agent？** 直接用 Python httpx 的預設 User-Agent 請求許多網站會收到 403 Forbidden，模擬瀏覽器的 User-Agent 能大幅降低被擋的機率。10 秒的 `timeout` 設定則避免因某個慢速網站而卡住整個搜尋流程。錯誤會被 catch 並回傳錯誤訊息，讓 agent 知道這個 URL 抓不到、可以繼續下一個。

#### 3b. `think_tool`：強制反思的暫停機制

```python
@tool(parse_docstring=True)
def think_tool(reflection: str) -> str:
    """用於對研究進度與決策進行策略性反思的工具。
    ...
    """
    return f"Reflection recorded: {reflection}"
```

這是這個範例裡最「輕量」但設計最精妙的工具——它的實作就是把輸入字串包成一句話回傳，**什麼都不做**。

那它有什麼用？它的價值在 prompt 層面：`RESEARCHER_INSTRUCTIONS` 強制規定「每次搜尋後都要用 `think_tool`」。當 LLM 呼叫 `think_tool(reflection="...")` 時，它被迫把當下的評估「寫下來」——這個動作本身就是鏈式思考（chain-of-thought）的一種實現，幫助模型在下一步工具呼叫前整理思緒、評估進度、決定是否繼續搜尋。

**`@tool(parse_docstring=True)` 的作用**：LangChain 的 `@tool` 裝飾器搭配 `parse_docstring=True` 時，會自動從 docstring 提取工具描述與參數說明，生成給 LLM 看的 tool schema。這讓工具文件和程式碼保持在同一個地方，不需要手動維護分開的 schema。

---

### 4. `research_agent/__init__.py`：公開介面聲明

```python
from research_agent.prompts import (
    RESEARCHER_INSTRUCTIONS,
    RESEARCH_WORKFLOW_INSTRUCTIONS,
    SUBAGENT_DELEGATION_INSTRUCTIONS,
)
from research_agent.tools import tavily_search, think_tool

__all__ = [
    "tavily_search",
    "think_tool",
    "RESEARCHER_INSTRUCTIONS",
    "RESEARCH_WORKFLOW_INSTRUCTIONS",
    "SUBAGENT_DELEGATION_INSTRUCTIONS",
]
```

`__init__.py` 的內容很精簡，主要功能是：把 `research_agent` 子套件的公開介面統一從一個地方 import。`agent.py` 用的就是 `from research_agent.prompts import ...` 和 `from research_agent.tools import ...`，這個 `__init__.py` 讓外部使用者也可以直接 `from research_agent import tavily_search`。

---

### 5. `utils.py`：Notebook 開發輔助工具

```python
def format_messages(messages):
    """以 Rich 格式化並顯示一串訊息。"""
    for m in messages:
        msg_type = m.__class__.__name__.replace("Message", "")
        content = format_message_content(m)

        if msg_type == "Human":
            console.print(Panel(content, title="🧑 Human", border_style="blue"))
        elif msg_type == "Ai":
            console.print(Panel(content, title="🤖 Assistant", border_style="green"))
        elif msg_type == "Tool":
            console.print(Panel(content, title="🔧 Tool Output", border_style="yellow"))
```

`utils.py` 是給 Jupyter Notebook 用的視覺化輔助模組，生產環境的 `agent.py` 不依賴它。它使用 `rich` 套件把 LangChain 的 message 物件渲染成有顏色邊框的 Panel，方便在 notebook 裡逐步觀察 agent 的對話流程。

`format_message_content` 的實作處理了兩種格式：Anthropic 的 content list 格式（`{"type": "tool_use", ...}`）和 OpenAI 的 `tool_calls` 屬性格式，讓同一個函式能適用於不同 model provider 的輸出。

---

### 6. `langgraph.json`：LangGraph Server 部署設定

```json
{
  "dependencies": ["."],
  "graphs": {
    "research": "./agent.py:agent"
  },
  "env": ".env"
}
```

這個三行設定檔決定了 `langgraph dev` 如何啟動服務：

- **`dependencies: ["."]`**：把當前目錄（`deep_research/`）安裝為 Python 套件，確保 `research_agent` 子套件可以被 import。
- **`graphs: { "research": "./agent.py:agent" }`**：把 `agent.py` 裡的 `agent` 物件掛到 `/research` 這個 graph endpoint 上。LangGraph Studio 和 API 都會透過這個名稱來呼叫你的 agent。
- **`env: ".env"`**：告訴 LangGraph server 從 `.env` 檔案載入環境變數（API 金鑰等）。

---

## 這個範例示範了 Deep Agents 的哪些能力

這個範例實際用到了以下 Deep Agents 核心能力：

| 能力 | 如何體現 |
|------|---------|
| **write_todos 規劃** | Orchestrator prompt 第一步強制 `write_todos`，把研究分解成具體任務清單 |
| **subagents + task()** | `create_deep_agent(subagents=[...])` 定義 subagent，orchestrator 透過 `task()` 委派並行執行 |
| **Virtual Filesystem** | `write_file("/research_request.md")`、`write_file("/final_report.md")` 把中間產物存在 agent state |
| **Middleware（內建指示）** | `create_deep_agent` 的 `system_prompt` 與框架 middleware 的內建指示合併，`INSTRUCTIONS` 是「補強」而非替換 |

這個範例**沒有**用到 HITL（interrupt）、skill、memory（AGENTS.md）、或 sandbox。

---

## 怎麼跑起來

### 方式一：LangGraph Server（推薦）

```bash
cd examples/deep_research
langgraph dev
```

瀏覽器會自動開啟 LangGraph Studio。在 Studio 裡選擇 `research` graph，輸入你的研究問題，就能即時觀察 orchestrator 規劃、subagent 搜尋、報告生成的完整流程。

### 方式二：Jupyter Notebook

```bash
uv run jupyter notebook research_agent.ipynb
```

適合想逐步觀察每個步驟輸出的學習場景。

---

## 可以怎麼延伸

讀完程式碼之後，這個範例有幾個清楚的延伸方向：

**1. 換 tool：加入 PDF 閱讀或資料庫查詢**
目前 subagent 只有 `tavily_search` 和 `think_tool`。可以加入 PDF 讀取工具（讓它能讀論文 PDF）、或是內部知識庫的查詢工具，讓研究範圍不侷限於公開網路。

**2. 換模型：Gemini 或 OpenAI**
`agent.py` 裡已有 Gemini 3 的範例程式碼（被注解掉），只要取消注解並設定 `GOOGLE_API_KEY` 即可切換。`create_deep_agent` 接受任何 LangChain chat model。

**3. 加入 HITL（interrupt）**
可以在 orchestrator 完成規劃（`write_todos` 之後）加入一個人工確認步驟，讓使用者審核研究計畫再開始執行，適合需要人工監督的企業場景。

**4. 調整上限參數**
`max_concurrent_research_units` 和 `max_researcher_iterations` 是最直接的調鈕：前者控制並行度（token 消耗速度），後者控制最大研究輪次（探索深度）。

**5. 加入 AGENTS.md memory**
讓 orchestrator 在每次研究後把「什麼問題適合幾個 subagent」的經驗寫入 `AGENTS.md`，逐漸累積個人化的研究策略。

---

## 小結

`deep_research` 範例的優雅之處，在於它用極少的程式碼（主要就是 `agent.py` 加上 `prompts.py` 和 `tools.py`）示範了 Deep Agents 最核心的兩層架構：**orchestrator 規劃 + subagent 執行**。

幾個值得帶走的設計原則：

- **Prompt 分層**：orchestrator prompt 和 subagent prompt 嚴格分開，各管各的職責。
- **工具設計的克制**：subagent 只有兩支工具，`think_tool` 看起來什麼都不做，卻在 prompt 配合下強制了高品質的鏈式思考。
- **Virtual Filesystem 的作用**：`.md` 檔案不只是輸出，也是 orchestrator 的中間狀態——寫入 `research_request.md`、最後回頭讀它來驗證，是一個簡單有效的自我稽核機制。
- **上限設計的重要性**：搜尋次數上限、並行數上限、迭代輪次上限，這些「硬性限制」寫在 prompt 裡，是讓 agent 在可控成本內運作的關鍵。

如果你正在思考「怎麼把 LLM 搜尋能力做成真正可用的工具」，這個範例是一個值得細讀的起點。
