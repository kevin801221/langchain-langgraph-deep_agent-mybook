> 原始路徑：docs/deepagents/examples/async-subagent-server/

# 把 AI 研究員「外包」成一個獨立服務：async-subagent-server 逐段深入導讀

**副標：用 FastAPI 自架 Agent Protocol HTTP 伺服器，讓 Deep Agents supervisor 以非同步方式遠端派任務、追進度、中途改指令——全部不阻塞主流程。**

---

## 引言：為什麼要把 subagent 拆成獨立服務？

在大多數 multi-agent 系統裡，supervisor 和 subagent 住在同一個 Python 行程、同一台機器上。這樣簡單，但擴展性有限：一旦 subagent 需要部署到 GPU 叢集、雲端託管服務，或者由另一個團隊維護，「同行程」模式就行不通了。

`async-subagent-server` 這個範例，示範的正是另一條路：

> **把 subagent 包成一個符合 Agent Protocol 的 HTTP 服務，supervisor 透過 LangGraph SDK 以非同步方式呼叫——送出任務後立刻拿到一個 task ID，不必等結果，需要時再回來查。**

這個設計讓通訊協定與代理邏輯徹底解耦。不論 subagent 內部跑的是研究員、程式碼生成器還是資料分析師，外層的 HTTP 合約完全一致。

---

## 全貌：supervisor ↔ Agent Protocol ↔ server

整個系統由兩個角色組成：

```
supervisor.py (REPL)
     │
     │  LangGraph SDK  (AsyncSubAgent)
     ▼
 HTTP / Agent Protocol
     │
     ▼
server.py  (FastAPI)
     │
     ▼
_agent  (create_deep_agent + web_search tool)
```

**非同步流程**分成五個核心操作，對應 REPL 的五條指令：

| 操作 | supervisor 送出 | server 回應 |
|------|----------------|------------|
| 啟動任務 | `POST /threads` → `POST /threads/{id}/runs` | 立刻回傳 `run_id`（status: pending） |
| 查詢狀態 | `GET /threads/{id}/runs/{run_id}` | status: pending / running / success / error |
| 取得結果 | `GET /threads/{id}` | `values.messages` 裡放最終輸出 |
| 更新指令 | `POST /threads/{id}/runs`（帶 `multitask_strategy: interrupt`） | 取消舊 run，啟動新 run |
| 取消任務 | `POST /threads/{id}/runs/{run_id}/cancel` | status: cancelled |

最關鍵的設計：**server 收到 create run 請求後，用 `asyncio.ensure_future` 把代理呼叫丟進背景，立刻回傳 HTTP 200**。supervisor 不必卡住等待，可以繼續處理其他事情，隨時再來輪詢。

---

## 環境與安裝

### 相依套件（pyproject.toml）

```toml
dependencies = [
    "deepagents>=0.6.8",
    "fastapi>=0.136.3",
    "uvicorn>=0.49.0",
    "langchain-anthropic>=1.4.4",
    "langchain_google_genai",
    "langgraph>=1.2.4",
    "langgraph-sdk>=0.4.2",
    "httpx>=0.28.1",
    "python-dotenv>=1.2.2",
]
```

核心依賴清單很精簡：`deepagents` 提供 `create_deep_agent` 與 `AsyncSubAgent`；`fastapi` + `uvicorn` 架起 HTTP 服務；`langgraph-sdk` 處理 supervisor 端的 Agent Protocol 通訊；`httpx` 用於 Tavily 搜尋的 async HTTP 呼叫。

### 環境變數（.env）

```
ANTHROPIC_API_KEY=       # 必填（server 與 supervisor 都需要）
TAVILY_API_KEY=...       # 選填；未設定時 server 改用 stub search
PORT=2024
RESEARCHER_URL=          # supervisor 用；預設 http://localhost:2024
```

`TAVILY_API_KEY` 選填是個貼心設計——讓你不需要立刻申請 API 金鑰也能把流程跑通。

### 啟動步驟（兩個終端機）

```bash
# 終端機 1：啟動 server
cd examples/async-subagent-server
uv sync
uv run uvicorn server:app --port 2024

# 終端機 2：啟動 supervisor REPL
cd examples/async-subagent-server
ANTHROPIC_API_KEY=... uv run python supervisor.py
```

---

## ★ 核心逐段導讀

### server.py — 把研究員包成 Agent Protocol HTTP 服務

#### 持久化：記憶體內 SQLite

```python
_conn = sqlite3.connect(":memory:", check_same_thread=False)
_conn.row_factory = sqlite3.Row
```

server 用的是行程內的 in-memory SQLite，不需要任何外部資料庫設定。`check_same_thread=False` 允許多個 async handler 共用同一個連線（FastAPI 的 async route 在同一執行緒的事件迴圈上跑）。

`_init_db()` 在啟動時建立兩張資料表：

- **threads**：一個對話脈絡一列，存放 `messages`（JSON 陣列）和 `values_`（JSON 物件，用來放最終狀態）。
- **runs**：一次執行嘗試一列，`status` 欄位可以是 `pending | running | success | error | cancelled`。

這個設計故意簡單：只要把 `":memory:"` 換成檔案路徑，或整個換成 PostgreSQL，就能做到跨行程的持久化。

#### web_search tool：真實呼叫 vs. stub fallback

```python
@tool
async def web_search(query: str) -> str:
    if os.environ.get("TAVILY_API_KEY"):
        async with httpx.AsyncClient() as client:
            res = await client.post(
                "https://api.tavily.com/search",
                json={"api_key": os.environ["TAVILY_API_KEY"], "query": query, "max_results": 5},
                timeout=30,
            )
        data = res.json()
        results = data.get("results") or []
        ...
        return "\n\n".join(
            f"{i + 1}. **{r['title']}**\n   {r['content']}\n   Source: {r['url']}"
            for i, r in enumerate(results)
        )

    # stub fallback
    return "\n".join([
        f'[stub] Search results for "{query}":',
        f"1. Key finding: Recent developments show significant progress in {query}",
        ...
    ])
```

這個 `@tool` 函式有個清晰的雙軌設計：有 `TAVILY_API_KEY` 就打真實 API，沒有就回傳格式相同的假資料。**stub 的意義在於讓 HTTP 合約測試與代理邏輯測試可以在沒有外部服務的情況下進行**。

#### create_deep_agent：代理定義

```python
_agent = create_deep_agent(
    model=ChatGoogleGenerativeAI(model="gemini-3.5-flash"),
    system_prompt=(
        "You are a thorough research agent. Investigate topics using web search and produce "
        "a well-structured research summary (300–500 words). Cite sources where possible.\n\n"
        "If you receive new instructions mid-conversation, follow them immediately without "
        "asking for clarification — discard prior work and start fresh on the new task."
    ),
    tools=[web_search],
)
```

`create_deep_agent` 是 `deepagents` 函式庫的核心工廠函式，回傳一個相容 LangGraph 的 agent graph。這裡只傳入 `model`、`system_prompt` 和 `tools=[web_search]`，不傳 `checkpointer`——因為持久化由 server 自行管理的 SQLite 負責，而不是 LangGraph 的 checkpoint 機制。

system prompt 裡「收到新指令時立刻放棄先前工作、重新開始」這一句，正是配合 `update_async_task`（帶 `multitask_strategy: interrupt`）的設計——讓代理的行為語意和 HTTP 層的中斷語意一致。

#### _execute_run：射後不理的非同步執行器

```python
async def _execute_run(run_id: str, thread_id: str, user_message: str) -> None:
    _conn.execute("UPDATE runs SET status = 'running' WHERE run_id = ?", (run_id,))
    _conn.commit()
    try:
        result = await _agent.ainvoke({"messages": [HumanMessage(user_message)]})
        last = result["messages"][-1]
        output = last.content if isinstance(last.content, str) else json.dumps(last.content)
        assistant_msg = {"role": "assistant", "content": output}
        # ... 把回應存回 threads 資料表 ...
        _conn.execute("UPDATE runs SET status = 'success' WHERE run_id = ?", (run_id,))
    except Exception as exc:
        _conn.execute(
            "UPDATE runs SET status = 'error', error = ? WHERE run_id = ?",
            (str(exc), run_id),
        )
    _conn.commit()
```

這是整個 server 最核心的函式。它做的事情：

1. 把 run 狀態從 `pending` 改成 `running`
2. 呼叫 `_agent.ainvoke()`——這可能跑幾秒到幾分鐘
3. 成功時，把最後一條 AI 訊息附加到 thread 的 `messages` 和 `values_` 欄位，狀態改成 `success`
4. 失敗時，把錯誤訊息存入 `error` 欄位，狀態改成 `error`

這個函式本身是 `async`，但它**從不被 `await`**——呼叫端用 `asyncio.ensure_future(_execute_run(...))` 把它丟進背景。

#### Routes：Agent Protocol 的 HTTP 端點

**`POST /threads`** — 建立 thread，回傳 `thread_id`：

```python
@app.post("/threads")
async def create_thread() -> dict[str, Any]:
    thread_id = str(uuid.uuid4())
    now = datetime.now(UTC).isoformat()
    _conn.execute(
        "INSERT INTO threads (thread_id, created_at) VALUES (?, ?)",
        (thread_id, now),
    )
    _conn.commit()
    return {"thread_id": thread_id, "created_at": now, "messages": [], "values": {}}
```

**`POST /threads/{thread_id}/runs`** — 在 thread 上啟動 run，這是整個 server 最複雜的端點：

```python
@app.post("/threads/{thread_id}/runs")
async def create_run(thread_id: str, request: Request) -> dict[str, Any]:
    body = await request.json()
    multitask_strategy = body.get("multitask_strategy")

    if multitask_strategy == "interrupt":
        _conn.execute(
            "UPDATE runs SET status = 'cancelled' WHERE thread_id = ? AND status = 'running'",
            (thread_id,),
        )
        _conn.execute(
            "UPDATE threads SET values_ = '{}' WHERE thread_id = ?",
            (thread_id,),
        )
        _conn.commit()
    # ... 取出 user_message，建立 run 記錄 ...
    asyncio.ensure_future(_execute_run(run_id, thread_id, user_message))
    return {"run_id": run_id, ..., "status": "pending", ...}
```

`multitask_strategy == "interrupt"` 時，server 會：先取消同一個 thread 上所有 `running` 狀態的 run，清空 thread 的 `values_`，再建立新的 run。這讓 `update_async_task` 的語意成立：「丟掉舊工作，從新指令重新開始」。

**`GET /threads/{thread_id}/runs/{run_id}`** — 輪詢 run 狀態，supervisor 會反覆呼叫這個端點直到 status 不再是 `running`。

**`GET /threads/{thread_id}`** — 取得 thread，LangGraph SDK 讀取 `values["messages"]` 拿結果。

**`POST /threads/{thread_id}/runs/{run_id}/cancel`** — 在 DB 裡把 run 標記成 `cancelled`。注意：代理的 `ainvoke` 呼叫不會被真正中斷（如果想要真正的 mid-execution cancel，需要額外接上 `asyncio.Task` 的取消機制）。

**`GET /ok`** — 健康檢查，回傳 `{"ok": True}`。

---

### supervisor.py — 主管 REPL：用 LangGraph SDK 派任務

#### AsyncSubAgent 設定

```python
async_subagents: list[AsyncSubAgent] = [
    {
        "name": "researcher",
        "description": (
            "A research agent that investigates any topic using web search. "
            "Runs in the background and returns a detailed summary."
        ),
        "graph_id": "researcher",
        "url": RESEARCHER_URL,
        "headers": {"x-auth-scheme": "custom"},
    },
]
```

`AsyncSubAgent` 是一個 TypedDict，描述一個遠端的非同步子代理。`url` 指向 server.py 的地址（預設 `http://localhost:2024`），`graph_id` 對應 server 端的 `assistant_id`，`headers` 讓你可以傳自訂認證標頭。`deepagents` 的 middleware 會根據這份設定，自動把 `start_async_task`、`check_async_task`、`update_async_task`、`cancel_async_task`、`list_async_tasks` 等工具注入給 supervisor agent。

#### Supervisor agent 定義

```python
checkpointer = MemorySaver()
thread_id = str(uuid.uuid4())

supervisor = create_deep_agent(
    model=ChatGoogleGenerativeAI(model="gemini-3.5-flash"),
    checkpointer=checkpointer,
    system_prompt=(...),
    subagents=async_subagents,
)
```

和 server 端的 `_agent` 不同，supervisor 傳入了 `checkpointer=MemorySaver()`。這讓 supervisor 本身具備 checkpoint 能力，可以在多輪對話中記住之前派出去的 task ID，使用者才能在後續輪次說「check status of \<task-id\>」。

#### System prompt 的五段操作語意

supervisor 的 system prompt 用非常清楚的結構告訴模型什麼時候呼叫哪個工具：

```
START: 當使用者說要 research 某個東西時：
  1. Call start_async_task with subagent_type "researcher" and the topic.
  2. Report the task_id and stop. Do NOT immediately check status.

CHECK: 當使用者詢問進度或結果時：
  1. Call check_async_task with the exact task_id.
  2. Report what the tool returns.

UPDATE: 當使用者要改變研究員正在做的事時：
  1. Call update_async_task with the task_id and new instructions.

CANCEL: 當使用者要取消任務時：
  1. Call cancel_async_task with the exact task_id.

LIST: 當使用者要列出所有任務時：
  1. Call list_async_tasks.
```

特別值得注意的 rules：**"Never report a stale status from memory. Always call a tool."** 和 **"Never poll in a loop. One tool call per user request."** 這兩條規則，確保 supervisor 不會猜測狀態，也不會在一次回應裡發出多次 HTTP 請求造成意外的輪詢風暴。

#### REPL 主迴圈

```python
async def chat(user_input: str) -> None:
    result = await supervisor.ainvoke(
        {"messages": [HumanMessage(user_input)]},
        config={"configurable": {"thread_id": thread_id}},
    )
    last = result["messages"][-1]
    content = last.content
    print(
        "\n"
        + (content if isinstance(content, str) else __import__("json").dumps(content, indent=2))
        + "\n"
    )

async def main() -> None:
    print(f"Supervisor connected to researcher at {RESEARCHER_URL}")
    while True:
        user_input = input("> ").strip()
        if not user_input:
            continue
        await chat(user_input)
```

REPL 每次把使用者輸入包成 `HumanMessage`，帶著同一個 `thread_id` 呼叫 `supervisor.ainvoke`，印出最後一條訊息。`thread_id` 在整個 REPL session 保持不變，讓 supervisor 的 `MemorySaver` 能夠在多輪對話中維持上下文。

---

### test_server.py — 驗證 Agent Protocol HTTP 合約

test_server.py 用 `pytest` + FastAPI 的 `TestClient` 做端對端測試，**不呼叫真實 LLM**——`_agent.ainvoke` 全部被 `unittest.mock.patch` 替換成預先準備好的固定回應。

#### _fresh_db fixture

```python
@pytest.fixture(autouse=True)
def _fresh_db():
    server._conn.executescript("DROP TABLE IF EXISTS runs; DROP TABLE IF EXISTS threads;")
    server._init_db()
```

每個測試前都重新初始化資料庫，確保測試之間的隔離。用 `DROP TABLE IF EXISTS` 再重建，比清空資料更徹底。

#### 涵蓋的測試案例

| 測試函式 | 驗證的行為 |
|---------|-----------|
| `test_health` | `GET /ok` 回傳 `{"ok": True}` |
| `test_create_thread` | `POST /threads` 回傳含 `thread_id` 的物件，`messages` 為空陣列 |
| `test_create_run_starts_agent` | `POST /threads/{id}/runs` 回傳 `status: pending`，`run_id` 存在 |
| `test_full_lifecycle` | 建立 thread → 建立 run → 等待背景任務完成（`asyncio.sleep(0.5)`）→ run status 變 `success` → thread `values.messages` 含助理回應 |
| `test_cancel_run` | 用 `slow_ainvoke`（`asyncio.sleep(10)`）模擬長時間執行，cancel 後 status 變 `cancelled` |
| `test_interrupt_strategy` | 第一個 run 正在執行時，帶 `multitask_strategy: interrupt` 建立第二個 run，第一個 run 變 `cancelled` |
| `test_404_for_missing_thread` | 不存在的 thread 回傳 404 |
| `test_404_for_missing_run` | 不存在的 run 回傳 404 |

`test_full_lifecycle` 是最完整的一個：它把整條生命週期串起來，確認 `values["messages"]` 裡真的包含 `"Here are the research results."`（也就是 `FAKE_RESPONSE` 裡的內容）。

`test_interrupt_strategy` 直接驗證了 `multitask_strategy: interrupt` 的核心語意：新任務進來時，舊的 running run 必須被取消。

---

## Deep Agents 能力：這個範例展示了什麼

1. **async subagent**：透過 `AsyncSubAgent` TypedDict + `create_deep_agent(subagents=...)` 把遠端服務接入 supervisor，無需任何 RPC 框架。
2. **研究員工具**：`web_search` 工具結合 Tavily API，展示 Deep Agent 如何與外部 HTTP API 互動。
3. **stub fallback**：沒有 `TAVILY_API_KEY` 時自動降級為假資料，讓開發體驗不依賴外部服務。
4. **interrupt 語意**：`multitask_strategy: interrupt` 讓主管可以在任務進行中改變指令，server 端配合清空狀態重新執行。
5. **記憶體內持久化**：零設定的 in-memory SQLite 讓範例開箱即用，同時展示了持久化的責任邊界。

---

## 怎麼跑：REPL 指令

啟動 REPL 後，可以試試這五種操作：

```
# 啟動一個研究任務（supervisor 會立刻回傳 task ID）
> research the latest developments in quantum computing

# 查詢進度（把 <task-id> 替換成實際的 UUID）
> check status of <task-id>

# 中途更新指令（帶 interrupt 語意：丟掉舊工作，重新開始）
> update <task-id> to focus on commercial applications only

# 取消任務
> cancel <task-id>

# 列出所有任務
> list all tasks
```

🎯 第一條指令的重點是：supervisor **不會等結果**，它會立刻印出 task ID 然後回到 `>` 提示符。這就是 async 的精髓——你可以同時派出多個研究任務，隨時用 `check status` 查看哪個先完成。

---

## 延伸：部署到任何基礎設施

這個範例的最大優點，在 README 最後一段說得很清楚：

> 只要把 `server.py` 裡的 `create_deep_agent` 換成你自己的代理即可。不論代理內部做什麼，外層的 Agent Protocol 都維持不變。

```python
_agent = create_deep_agent(
    model=ChatAnthropic(model="claude-sonnet-4-5"),
    system_prompt="You are a ...",
    tools=[your_tool],
)
```

這意味著：

- **換模型**：`ChatGoogleGenerativeAI` → `ChatAnthropic` 或任何 LangChain 相容的 LLM
- **換工具**：把 `web_search` 替換成你的業務工具
- **換持久化**：把 in-memory SQLite 換成 PostgreSQL + pgvector
- **部署到雲端**：把整個 FastAPI 應用部署到任何支援 HTTP 的基礎設施（Cloud Run、ECS、Kubernetes），supervisor 只需要改一個 `RESEARCHER_URL` 環境變數

Agent Protocol 在這裡扮演的是「標準化 HTTP 介面」的角色——就像 REST API 讓不同語言的服務能互相溝通，Agent Protocol 讓不同基礎設施上的 agent 能互相協作。

---

## 小結

`async-subagent-server` 用不到 400 行 Python，示範了一個完整的非同步子代理部署模式：

- **server.py**：FastAPI + in-memory SQLite + `asyncio.ensure_future` = 零設定的 Agent Protocol HTTP 服務
- **supervisor.py**：`AsyncSubAgent` + `create_deep_agent` + `MemorySaver` = 能跨輪對話管理多個背景任務的主管 REPL
- **test_server.py**：mock patch + `TestClient` = 不依賴 LLM、不依賴網路的 HTTP 合約測試

最值得帶走的設計思想是：**非同步不只是技術細節，而是架構決策**。把 subagent 拆成獨立服務、用 task ID 追蹤進度、以 interrupt 語意處理中途改指令——這三個設計加在一起，讓 Deep Agents 系統能真正支撐長時間、多任務、可中途介入的複雜研究工作流程。

---

*涵蓋檔案：`server.py`、`supervisor.py`、`test_server.py`、`pyproject.toml`、`.env`*
