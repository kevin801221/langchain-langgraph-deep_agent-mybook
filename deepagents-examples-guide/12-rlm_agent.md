> docs/deepagents/examples/rlm_agent/

# 一個回合，整批並行：用 RLM Agent 解鎖遞迴 REPL + PTC fan-out

**副標：深入剖析 `create_rlm_agent`——從 CodeInterpreterMiddleware 到 CompiledSubAgent 遞迴鏈，逐段讀懂唯一核心檔 `rlm_agent.py`**

---

## 引言：串行 vs. 並行的本質差距 🚀

傳統的 Deep Agent 也能透過 `task` 工具把工作分派給 subagent。但這種做法有個內建的節奏限制：**每一次 `task` 呼叫都佔用模型的一個獨立回合（turn）**，下一個 subagent 必須等前一個跑完才能啟動——本質上是串行的。

想像你要讓 agent 同時查詢十個資料來源：用傳統方式，模型得跑十個回合，依序等候。但如果你把工作流程寫成 JavaScript `Promise.all`，那十個查詢可以在**同一個 eval 呼叫裡一口氣扇出（fan-out）**，理論上只需一個回合。

這正是 **RLM（Recursive REPL Mode）** 模式要解決的問題。它在兩個維度同時突破：

- **PTC（Pass-Through Call）**：讓 `eval` 工具可以直接呼叫其他工具（如 `task`），實現單回合 fan-out。
- **遞迴 subagent 鏈**：每個被委派的 `general-purpose` subagent 自身也帶著 REPL + PTC，因此它「被呼叫的那一層」還能再次扇出，形成多層樹狀的並行結構。

---

## 核心概念：遞迴 REPL 與 PTC 是什麼？ 🧠

### CodeInterpreterMiddleware 與 PTC

`CodeInterpreterMiddleware` 來自 `langchain-quickjs` 套件，它在 agent 的 middleware 堆疊裡插入一個 JavaScript 直譯器。Agent 可以用 `eval` 工具呼叫執行任意 JS 程式碼。

關鍵在於 **`ptc`（Pass-Through Call）允許清單**：當你傳入 `ptc=["task", "add"]`，代表在 `eval` 的 JS 環境裡，模型可以直接呼叫這些工具名稱（透過全域的 `tools` 物件）。這讓以下程式碼成為可能：

```javascript
// 這段 JS 在一次 eval 呼叫裡並行執行四個 task
const results = await Promise.all([
  tools.task({ subagent_type: "general-purpose", description: "subtask 1" }),
  tools.task({ subagent_type: "general-purpose", description: "subtask 2" }),
  tools.task({ subagent_type: "general-purpose", description: "subtask 3" }),
  tools.task({ subagent_type: "general-purpose", description: "subtask 4" }),
]);
```

**一個 eval = 一個回合 = 四個 subagent 並行啟動。**

### 遞迴結構（depth 樹狀圖）

README 裡有一張清楚的層次圖，完整呈現了 `max_depth=2` 時的架構：

```
root (depth=2, has CodeInterpreterMiddleware)
└── general-purpose → compiled depth-1 graph (has CodeInterpreterMiddleware)
    └── general-purpose → compiled depth-0 graph (has CodeInterpreterMiddleware)
        └── general-purpose → built-in default (no REPL, no recursion)
```

重要觀念：每個節點都是一張**獨立編譯出來的圖（graph）**，不是同一張圖裡的循環（cycle）。Depth-0 是遞迴的 base case，它的 `general-purpose` 交還給 `create_deep_agent` 的內建版本，不再繼續往下遞迴。

---

## 環境與安裝 ⚙️

核心依賴來自 `pyproject.toml`：

```toml
[project]
name = "repl-swarm-example"
requires-python = ">=3.11"
dependencies = [
    "deepagents>=0.6.8",
    "langchain-quickjs",
    "langchain-anthropic>=1.4.4",
]
```

- **`deepagents`**：提供 `create_deep_agent`、`CompiledSubAgent`、`SubAgent`、`GENERAL_PURPOSE_SUBAGENT` 等核心元件。
- **`langchain-quickjs`**：提供 `CodeInterpreterMiddleware`，這是 REPL + PTC 機制的底層實作。
- **`langchain-anthropic`**：Anthropic 模型的 LangChain 整合層。

安裝與執行（使用 `uv`）：

```bash
# 安裝依賴
uv sync

# 執行預設 demo（會跑並行加法示範）
uv run python rlm_agent.py

# 指定自訂任務
uv run python rlm_agent.py "Use eval to add 1+2 and 3+4 in parallel."

# 更深的遞迴層數
uv run python rlm_agent.py --max-depth 2
```

---

## ★ rlm_agent.py 逐段導讀 ★ 🔍

整支 `rlm_agent.py` 不到 201 行，卻緊湊地封裝了遞迴建構邏輯、守衛條件、以及一個可執行的 demo。以下逐段深入解析。

### 第一段：模組層級常數與 import（第 35–49 行）

```python
from __future__ import annotations

import argparse
from typing import Any

from deepagents import create_deep_agent
from deepagents.middleware.subagents import (
    GENERAL_PURPOSE_SUBAGENT,
    CompiledSubAgent,
    SubAgent,
)
from langchain_core.tools import BaseTool, tool
from langchain_quickjs import CodeInterpreterMiddleware

_MAX_DEPTH_LIMIT = 8  # 防呆：避免打錯數字而一口氣建立出成千上萬個 agent
```

這裡的 import 格局已經說明了整個設計意圖：

- `create_deep_agent`：所有遞迴層都由它建立，`create_rlm_agent` 只是它的 wrapper。
- `GENERAL_PURPOSE_SUBAGENT`：從 `deepagents` 匯入的常數字典，存著 `general-purpose` subagent 的名稱與描述——後續用於命名 `CompiledSubAgent` 以及守衛檢查。
- `CompiledSubAgent`：包裝已編譯圖（compiled graph）的特殊 subagent 類型，讓 `create_deep_agent` 知道「這個 subagent 不是抽象規格，而是一個真實可執行的圖」。
- `SubAgent`：使用者自定義 subagent 的型別（TypedDict 規格）。
- `CodeInterpreterMiddleware`：REPL 引擎，接受 `ptc` 參數設定允許清單。
- `_MAX_DEPTH_LIMIT = 8`：一個簡單但重要的安全防護——每多一層遞迴就多建一整張圖，`max_depth=8` 已是相當龐大的樹，超過幾乎沒有實際必要。

### 第二段：公開 API `create_rlm_agent`（第 52–107 行）

```python
def create_rlm_agent(
    *,
    model: str | None = None,
    tools: list[BaseTool] | None = None,
    subagents: list[SubAgent | CompiledSubAgent] | None = None,
    max_depth: int = 1,
    **kwargs: Any,
) -> Any:
```

注意簽名的幾個設計決策：

**`*`（keyword-only）**：所有參數都必須用關鍵字傳入，沒有位置參數，有效防止錯誤的參數順序。

**`model: str | None = None`**：模型名稱直接透傳給 `create_deep_agent`，預設由底層決定；範例 demo 用的是 `"anthropic:claude-haiku-4-5"`。

**`tools: list[BaseTool] | None = None`**：使用者提供的工具清單。這些工具會在**每一層遞迴**裡都可用，並且它們的名稱也會被加入 PTC 允許清單（稍後在 `_build` 裡看到）。

**`subagents: list[SubAgent | CompiledSubAgent] | None = None`**：額外的 subagent 規格，會原封不動地傳遞到每一層深度。有一個鐵律：**不能傳入名為 `general-purpose` 的規格**，因為這個名稱由 `create_rlm_agent` 自己管理。

**`max_depth: int = 1`**：遞迴深度，預設值 1 表示建兩層圖（root + depth-0）。`0` 代表不遞迴，只有一個帶 REPL 的 agent。上限是 `_MAX_DEPTH_LIMIT`（8）。

**`**kwargs: Any`**：所有不認識的關鍵字參數都會被轉發給 `create_deep_agent`，讓底層的擴充彈性完整保留。

#### 守衛條件（Guard clauses，第 87–107 行）

```python
if max_depth < 0:
    msg = "max_depth must be >= 0"
    raise ValueError(msg)
if max_depth > _MAX_DEPTH_LIMIT:
    msg = f"max_depth {max_depth} exceeds safety cap {_MAX_DEPTH_LIMIT}"
    raise ValueError(msg)
for spec in subagents or []:
    if spec.get("name") == GENERAL_PURPOSE_SUBAGENT["name"]:
        msg = (
            "create_rlm_agent manages the `general-purpose` subagent "
            "itself; do not pass one via `subagents`."
        )
        raise ValueError(msg)

return _build(
    model=model,
    tools=tools,
    extra_subagents=list(subagents or []),
    max_depth=max_depth,
    **kwargs,
)
```

三道守衛依序檢查：
1. `max_depth < 0`：負數無意義，直接拒絕。
2. `max_depth > 8`：超過安全上限，防止意外建立爆炸性的圖。
3. 遍歷 `subagents`，如果有任何一個 `spec.get("name")` 等於 `GENERAL_PURPOSE_SUBAGENT["name"]`（字串 `"general-purpose"`），拋出錯誤——這是維護遞迴約定的關鍵守衛。

通過守衛後，把 `subagents` 轉為 `list`（確保可變），以 `extra_subagents` 之名傳入內部建構器 `_build`。

### 第三段：內部遞迴建構器 `_build`（第 110–156 行）

```python
def _build(
    *,
    model: str | None,
    tools: list[BaseTool] | None,
    extra_subagents: list[SubAgent | CompiledSubAgent],
    max_depth: int,
    **kwargs: Any,
) -> Any:
```

`_build` 是整個模式的心臟。它是純內部函式（沒有 docstring 裡的公開 API 宣告），只有 `create_rlm_agent` 知道它的存在。

#### Base case（depth 0，第 127–135 行）

```python
if max_depth == 0:
    ptc_tool_names = sorted({*(t.name for t in tools or []), "task"})
    return create_deep_agent(
        model=model,
        tools=tools,
        subagents=extra_subagents,
        middleware=[CodeInterpreterMiddleware(ptc=ptc_tool_names)],
        **kwargs,
    )
```

當 `max_depth == 0`，遞迴停止。此時：

- **`ptc_tool_names`**：用 set 去重、再 `sorted` 排序，確保 `"task"` 一定在清單裡，加上使用者自定義工具的所有名稱。這個清單傳給 `CodeInterpreterMiddleware(ptc=...)`，決定 JS eval 環境裡可以呼叫哪些工具。
- **`subagents=extra_subagents`**：不傳入自訂的 `general-purpose`，讓 `create_deep_agent` 的**內建自動注入機制**處理它——這就是遞迴的底板，也是讓整條鏈收尾的關鍵。
- **`middleware=[CodeInterpreterMiddleware(ptc=ptc_tool_names)]`**：即使是 depth-0，這個 agent 依然掛著 REPL；它只是沒有更深一層的 `general-purpose`。

#### Recursive case（depth N > 0，第 137–156 行）

```python
deeper = _build(
    model=model,
    tools=tools,
    extra_subagents=extra_subagents,
    max_depth=max_depth - 1,
    **kwargs,
)
compiled_gp = CompiledSubAgent(
    name=GENERAL_PURPOSE_SUBAGENT["name"],
    description=GENERAL_PURPOSE_SUBAGENT["description"],
    runnable=deeper,
)
ptc_tool_names = sorted({*(t.name for t in tools or []), "task"})
return create_deep_agent(
    model=model,
    tools=tools,
    subagents=[compiled_gp, *extra_subagents],
    middleware=[CodeInterpreterMiddleware(ptc=ptc_tool_names)],
    **kwargs,
)
```

這段是整個模式最精妙的地方，分三步走：

**步驟一：先遞迴建好「深度減一」的 agent**。`_build(..., max_depth=max_depth - 1, ...)` 先跑，回傳一張已編譯好的圖，存為 `deeper`。這是 Python 的一般遞迴：先建底層再建上層。

**步驟二：把 `deeper` 包進 `CompiledSubAgent`**。`CompiledSubAgent` 的三個欄位：
  - `name=GENERAL_PURPOSE_SUBAGENT["name"]`：讓這個 subagent 的識別名稱等同於 `"general-purpose"`。
  - `description=GENERAL_PURPOSE_SUBAGENT["description"]`：沿用官方描述，讓模型在推理時不感知到差異。
  - `runnable=deeper`：把下一層的已編譯圖塞進去——模型呼叫 `task({subagent_type: "general-purpose"})` 時，實際上執行的就是這個 `deeper` 圖。

**步驟三：建立這一層的 agent，把 `compiled_gp` 排在 `extra_subagents` 前面**。`subagents=[compiled_gp, *extra_subagents]` 確保 `general-purpose` 的位置是顯式指定的，`create_deep_agent` 看到清單裡已有 `general-purpose` 就不會再自動注入內建版本——這個行為是整條遞迴鏈正確運作的隱性合約。

### 第四段：demo driver（第 159–200 行）

```python
@tool
def add(a: int, b: int) -> int:
    """Add two integers and return their sum."""
    return a + b
```

一個最小化的工具示範，讓 agent 可以在 JS REPL 裡呼叫 `tools.add(1, 2)`。

```python
def _main() -> None:
    args = _parse_args()
    agent = create_rlm_agent(
        model=args.model,
        tools=[add],
        max_depth=args.max_depth,
    )
    result = agent.invoke(
        {"messages": [{"role": "user", "content": args.task}]},
    )
    for message in result["messages"]:
        print(f"--- {type(message).__name__} ---")
        print(message.content)
```

`_parse_args()` 設定三個 CLI 參數：`task`（預設任務字串）、`--max-depth`（預設 1）、`--model`（預設 `"anthropic:claude-haiku-4-5"`）。`_main` 建立 agent、呼叫 `invoke`、然後逐條印出 messages 的類型與內容，方便肉眼觀察整個推理過程。

---

## RLM Agent 展示的三項 Deep Agents 能力 💡

### 1. Middleware 架構

`CodeInterpreterMiddleware` 作為 middleware 掛入 agent，而不是作為普通工具——這代表它在 agent 執行的 middleware 堆疊裡注入行為，可以攔截、轉換或擴充工具呼叫。RLM 模式示範了如何用一個 middleware 為整個 agent 增加 JavaScript eval 能力，而不改動 agent 的核心邏輯。

### 2. CompiledSubAgent：把圖當 subagent

`CompiledSubAgent` 讓一張已編譯的 LangGraph 圖成為另一張圖的 subagent。這是 Deep Agents 的重要能力：**你可以把任何符合介面的 runnable 掛成 subagent**，不限定必須使用框架內建的 subagent 類型。

### 3. 單回合並行 fan-out

透過 PTC 機制，模型能在一個 eval 呼叫裡觸發多個並行 task，這遠比多回合串行委派有效率。對於「任務可以拆解成多個彼此獨立子任務，但事先不知道怎麼拆」的場景，這個模式讓 agent 自行決定如何拆分，並在運行時動態並行化。

---

## 怎麼用（Usage 範例）🛠️

### 最小化用法

```python
from rlm_agent import create_rlm_agent

agent = create_rlm_agent(
    model="claude-sonnet-4-6",
    max_depth=1,
)

result = agent.invoke({
    "messages": [{"role": "user", "content": "Do three independent research tasks in parallel."}]
})
```

### 帶自訂工具

```python
from langchain_core.tools import tool
from rlm_agent import create_rlm_agent

@tool
def lookup(key: str) -> str:
    """Fetch a value by key."""
    return database.get(key)

agent = create_rlm_agent(
    model="claude-sonnet-4-6",
    tools=[lookup],
    max_depth=2,
)
```

`lookup` 工具會自動加入每一層的 PTC 清單，所以 agent 在 `eval` 裡可以直接寫 `tools.lookup("key")` 並行呼叫。

### 帶額外 subagent

```python
agent = create_rlm_agent(
    tools=[lookup],
    subagents=[
        {
            "name": "writer",
            "description": "Writes polished prose from structured notes.",
            "system_prompt": "You are a professional writer...",
        },
    ],
    max_depth=1,
)
```

**注意**：`subagents` 裡**絕對不能**放名為 `"general-purpose"` 的規格，否則會收到 `ValueError`。

---

## 什麼情境適合 RLM Agent？ 🎯

**適合的場景**：

- **任務結構動態未知**：無法預先知道需要幾個子任務、每個子任務的性質是什麼，需要讓 agent 在運行時自行決定拆分方式。
- **高度可並行化的工作負載**：例如同時爬取多個資料來源、並行執行多個獨立分析、批量處理不相依的工作項目。
- **多層委派需求**：每個 subagent 自己也需要能再扇出——例如「先拆成三個大任務，每個大任務再拆成五個小任務」的多層樹狀結構。

**不適合的場景**：

- 任務有明確的串行依賴（A 完成後 B 才能開始）。
- 子任務之間需要共享狀態——每一層都是獨立圖，狀態不跨層共享，資料只能透過 `task` 的 `description` 帶入或透過工具回傳值帶出。
- 需要極度節省資源——每多一層就多建一整張 Deep Agent 圖，`max_depth=2` 對大多數場景已經足夠。

**深度設定建議**：
- `max_depth=0`：單層帶 REPL，適合簡單的並行工具呼叫。
- `max_depth=1`（預設）：兩層，root 可把任務扇出給 depth-0 agent，depth-0 agent 用內建 `general-purpose`。
- `max_depth=2`：三層，適合需要多層動態拆解的複雜任務，是大多數情境的實際上限。

---

## 小結 📝

`rlm_agent.py` 只有 201 行，卻示範了 Deep Agents 幾個重要的組合模式：

1. **Wrapper 哲學**：`create_rlm_agent` 不重造輪子，而是在 `create_deep_agent` 外層疊加遞迴邏輯，保持底層的完整彈性。

2. **遞迴建構，非運行時遞迴**：整條遞迴鏈在**建立 agent 時**就展開完畢，生成多張靜態的已編譯圖；運行時沒有動態遞迴，每一層都是獨立執行的圖。

3. **隱性合約的顯式維護**：`CompiledSubAgent` 用 `GENERAL_PURPOSE_SUBAGENT` 的名稱與描述「假扮」內建 subagent，讓模型從不感知到「這個 general-purpose 其實是更深一層的圖」。守衛條件（guard clauses）則確保使用者不會不小心破壞這個約定。

4. **PTC 的關鍵地位**：如果只有遞迴 subagent 但沒有 PTC，模型只能一次委派一個任務（串行）。PTC 讓 `eval` 裡的 `Promise.all` 真正把多個 task 壓縮進同一回合，這才是整個模式的效能核心。

想要讓 agent 在不確定性中找到最佳並行結構，RLM 模式提供了一個優雅的答案：不要硬編碼拆解策略，讓每一層的 agent 自己決定怎麼扇出，然後讓遞迴鏈確保它們都有能力這樣做。
