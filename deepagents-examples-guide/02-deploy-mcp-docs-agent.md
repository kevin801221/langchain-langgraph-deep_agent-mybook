> 原始路徑：docs/deepagents/examples/deploy-mcp-docs-agent/

# 先查文件，再開口：用 MCP 讓 AI 不再靠記憶回答技術問題

**副標：deploy-mcp-docs-agent 完整導讀——AGENTS.md 如何塑造「查了才說」的行為、agent.json 宣告式部署、MCP server 的 workspace 層級共用機制。**

---

## 引言：為什麼「先查文件再答」勝過純 LLM？

你有沒有遇過這種情況：問 AI「LangGraph 的 interrupt 怎麼設定」，它給了一段看起來很有道理的程式碼，結果跑起來卻報錯——因為 API 早在幾個版本前就改了？

這不是模型能力不足，而是一個根本性的問題：**LLM 的知識有截止日期，而技術文件每週都在更新。**

`deploy-mcp-docs-agent` 這個範例提供了一個直接的解法：讓 agent 在回答之前，先透過 MCP（Model Context Protocol）連到 LangChain 的線上官方文件，**搜尋 → 閱讀頁面 → 再作答**。這樣的「docs-first」架構能大幅降低資訊過時或模型捏造的風險，而且因為答案附有原始頁面的出處，使用者可以自行核對。

這個範例是 `deepagents deploy` 的宣告式部署示範。它沒有複雜的 Python 程式碼——整個專案只有兩個核心檔案：`AGENTS.md`（agent 的 prompt 與行為規範）和 `agent.json`（部署設定）。程式碼少，但每個設計決策都很值得細讀。

---

## 全貌：MCP → 搜尋 → 閱讀 → 作答的資料流

在深入每個檔案之前，先把整個系統的資料流看清楚：

```
使用者提問（LangSmith UI 或 SDK）
          │
          ▼
┌─────────────────────────────────────┐
│  deploy-mcp-docs-agent              │
│  模型：claude-sonnet-4-5            │
│  行為規範：AGENTS.md               │
│                                     │
│  1. 收到問題                        │
│  2. 呼叫 MCP 文件工具（搜尋）       │
│  3. 呼叫 MCP 文件工具（讀取頁面）   │
│  4. 整合文件內容，組織回答          │
│  5. 附上頁面標題或 URL 作為 citation │
└─────────────────────────────────────┘
          │
          ▼
     回答（附出處 URL）
```

MCP（Model Context Protocol）是這個架構的關鍵橋梁。LangChain 的官方文件網站提供了一個 MCP endpoint（`https://docs.langchain.com/mcp`），它對外暴露兩類 tool：**搜尋工具**（輸入關鍵字，找出相關頁面清單）和**頁面讀取工具**（輸入 URL，抓取完整頁面內容）。Agent 透過這兩支 tool，就能像人工查文件一樣，先找到相關章節，再仔細閱讀，最後作答。

**角色與元件一覽：**

| 元件 | 類型 | 職責 |
|------|------|------|
| `agent.json` | 設定檔 | 宣告 agent 名稱、使用的模型 |
| `AGENTS.md` | Prompt 規範 | 定義行為模式、回答格式、工具流程 |
| `docs-langchain` MCP server | Workspace 資源 | 提供搜尋與頁面讀取兩支 tool |
| `tools.json` | 工具引用（workspace 層級） | 把 MCP server 的 tool 接進 agent |
| LangSmith Deployments | 執行環境 | 部署後的 hosting 與 API 端點 |

---

## 環境與部署

### 必要的環境變數

README 列出了兩個必填的環境變數：

| Variable | 用途 |
|----------|------|
| `ANTHROPIC_API_KEY` | 存取 Claude 模型所需的金鑰 |
| `LANGSMITH_API_KEY` | 部署（deploy）時必填 |

這個範例沒有自定義的 Python 相依套件，因為所有的「工具」都來自外部的 MCP server，不需要在本地安裝額外的套件。這是宣告式部署的一大優點：agent 的能力擴充透過 MCP 掛接，而非程式碼修改。

### 部署流程：兩步驟

整個部署只需要兩個指令：

**第一步：把 MCP server 註冊到 workspace**

```bash
deepagents mcp-servers add --url https://docs.langchain.com/mcp --name docs-langchain
```

這個指令把 LangChain 的官方文件 MCP endpoint 以 `docs-langchain` 這個名稱註冊到你的 workspace。README 特別說明：**MCP 伺服器現在屬於 workspace 層級的資源（workspace-level resources）**，也就是說同一個 workspace 裡的多個 agent 都可以共用同一個 MCP server，不需要每個 agent 各自設定一遍。只要在 `tools.json` 裡引用 `docs-langchain` 這個名稱，就能讓任何 agent 存取這些 tool。

**第二步：部署 agent**

```bash
deepagents deploy
```

`deepagents deploy` 會讀取當前目錄的 `agent.json`（以及 workspace 層級的 `tools.json` 引用），把整個 agent 部署到 LangSmith 的 Deployments 上，生成一個可對外存取的 API 端點。

---

## ★ 逐段導讀

### 1. `AGENTS.md`：用 prompt 塑造「docs-first」行為

`AGENTS.md` 是這個 agent 最核心的設計文件。它不是一般的說明文件，而是會在每次 agent 啟動時被當成 system prompt 載入的指令集。全文共四個區塊：

#### 標題：明確宣告角色

```markdown
# LangChain Docs Research Agent

你是一個「以文件為優先（docs-first）」的技術研究代理，負責處理關於 LangChain、LangGraph 與 Deep Agents 的問題。

你的任務是：在依賴一般知識之前，先運用可用的 MCP 文件工具來回答開發者的問題。
```

第一段就清楚劃定了這個 agent 的邊界：它的服務對象是開發者問題（LangChain、LangGraph、Deep Agents），行為準則是「文件優先」。這個宣告不是廢話——它讓模型在決定「要不要先查文件」時，有一個明確的行為錨點。

#### 核心行為（Core behavior）

```markdown
## 核心行為（Core behavior）

- 對於涉及 API、功能、設定、部署、MCP、memory、tools、middleware、LangGraph 以及 Deep Agents 的事實性問題，優先使用文件 MCP 工具。
- 先搜尋，接著打開最相關的文件頁面，然後才作答。
- 盡可能依據文件中明確記載的行為來回答。
- 如果文件不完整或語意含糊，要明確說出這一點。
- 清楚區分「文件中記載的事實」與「你自己的推論」。
- 回答要簡潔、技術性、且具實務可行性。
```

這六條規則是整個行為模式的核心。幾個設計細節值得特別注意：

- **「先搜尋，接著打開…，然後才作答」**：這個順序明確規定了工具呼叫的流程，避免模型跳過搜尋直接憑記憶回答。
- **「如果文件不完整或語意含糊，要明確說出這一點」**：這是對抗「信心幻覺（hallucination of confidence）」的防線。模型被明確告知要承認不確定性，而不是填補空白。
- **「清楚區分文件記載的事實與你自己的推論」**：這讓使用者能知道哪些部分是有根據的，哪些是模型的推斷。

#### 回答格式（Answer format）

```markdown
## 回答格式（Answer format）

回答文件相關問題時：

1. 先給出直接的答案。
2. 附上一段以文件為依據的簡短說明。
3. 在有幫助時，引用相關的頁面標題或 URL。
4. 如果有多種可行的做法，簡短地比較它們。
5. 如果某個 API 或行為在文件中找不到，就回答 `I couldn't verify that in the docs.`
```

格式規範的第 5 點是這個 agent 的「誠實安全網」：如果文件裡真的沒有，就明確說沒找到，而不是硬給一個可能是錯的答案。這比「有底氣地說錯」要有價值得多。

第 3 點規定要附上 citation（頁面標題或 URL），這讓回答具備可驗證性——使用者拿到答案的同時，也拿到了「可以去核對的地方」。

#### 工具運用流程（Tooling workflow）

```markdown
## 工具運用流程（Tooling workflow）

對於任何關於 LangChain、LangGraph 或 Deep Agents 的問題：

1. 使用文件 MCP 的搜尋工具找出相關頁面。
2. 對最符合的結果，使用文件 MCP 的頁面閱讀工具。
3. 從文件內容中整合出答案。
4. 當文件無法支持某項主張時，避免用猜的。
```

這四步流程把「如何用 MCP tool」拆得很具體。它不是說「使用 MCP 工具」這樣抽象的指令，而是告訴模型要先用搜尋工具、再用頁面閱讀工具、然後整合答案——這個序列和 README 描述的「搜尋 → 閱讀 → 作答」資料流完全吻合。

第 4 點「避免用猜的」和核心行為的「不要捏造」相互呼應，在兩個地方都強調，是刻意的重複強化。

#### 邊界（Boundaries）

```markdown
## 邊界（Boundaries）

- 不要捏造未經記載的旗標（flag）、API 或設定。
- 在文件沒有明確顯示時，不要宣稱自己很確定。
- 如果使用者要求程式碼，請提供一個與你所查到文件一致的最小範例。
- 如果使用者問的不是文件相關的問題，你仍然可以幫忙，但要註明你已超出文件範圍。
```

邊界（Boundaries）區塊的最後一條值得特別關注：**如果使用者問的不是文件相關問題，agent 仍然可以回答，但要說明已超出文件範圍。** 這是個聰明的設計——它沒有讓 agent 完全拒絕文件範圍外的問題，而是允許回答但加上透明度標籤。這讓 agent 更實用，同時維持誠信。

---

### 2. `agent.json`：最小化的部署設定

```json
{
  "name": "deploy-mcp-docs-agent",
  "runtime": {
    "model": {"model_id": "anthropic:claude-sonnet-4-5"}
  }
}
```

這是整個範例裡最短的設定檔——三個欄位，六行 JSON。這種極簡設計是宣告式部署的體現：你只需要告訴 `deepagents deploy` 這個 agent 叫什麼名字、要跑哪個模型，其餘的（工具連接、prompt 載入、API 端點設定）都由框架處理。

逐欄說明：

- **`name`**：`"deploy-mcp-docs-agent"` 是這個 agent 在 LangSmith Deployments 上的識別名稱，也是透過 LangGraph SDK 呼叫時會用到的 graph 名稱（在 `client.runs.stream(thread_id, "agent", ...)` 中）。
- **`runtime.model.model_id`**：`"anthropic:claude-sonnet-4-5"` 指定了使用 Anthropic 的 Claude Sonnet 4.5 模型。`anthropic:` 前綴是 deepagents 框架識別 provider 的格式。

沒有出現在 `agent.json` 裡的東西同樣重要：這裡沒有工具列表，沒有 prompt 路徑。工具是透過 workspace 層級的 `tools.json` 引用（`docs-langchain` MCP server），prompt 是 `AGENTS.md`——這些都由 deepagents 框架在部署時自動整合。

---

### 3. MCP Server 的註冊與工具引用流程

這是這個範例最需要理解的機制：MCP server 如何從一個外部 URL 變成 agent 可以呼叫的 tool。

#### 步驟一：`deepagents mcp-servers add`

```bash
deepagents mcp-servers add --url https://docs.langchain.com/mcp --name docs-langchain
```

這個指令在 workspace 層級新增一筆 MCP server 記錄：名稱 `docs-langchain` 對應到 URL `https://docs.langchain.com/mcp`。Workspace 層級的意思是：這個設定和你的帳號/workspace 綁定，不屬於某個特定 agent，所以同一個 workspace 裡的所有 agent 都可以引用 `docs-langchain`。

#### 步驟二：`tools.json` 引用

雖然範例目錄裡沒有 `tools.json` 檔案（它是 workspace 層級的設定，不存在於個別範例目錄），但 README 說明了機制：在 `tools.json` 裡引用 `docs-langchain` 這個名稱，就能把這個 MCP server 的 tool 接進 agent。

#### MCP server 提供的 tool

`https://docs.langchain.com/mcp` 這個 endpoint 暴露的是兩類操作：
- **搜尋工具**：輸入關鍵字，返回相關文件頁面的清單。
- **頁面閱讀工具**：輸入頁面 URL，返回完整的頁面內容。

這兩支 tool 加上 `AGENTS.md` 裡定義的工具使用流程，就構成了「搜尋 → 閱讀 → 作答」的完整機制。

---

## 這個範例示範了 Deep Agents 的哪些能力

| 能力 | 如何體現 |
|------|---------|
| **宣告式部署（`deepagents deploy`）** | 只有 `AGENTS.md` + `agent.json` 兩個檔案，無需撰寫 Python agent 程式碼 |
| **MCP 工具整合** | 透過 `deepagents mcp-servers add` 把外部 MCP endpoint 接進 agent |
| **Workspace 層級資源共用** | MCP server 註冊一次，同 workspace 的所有 agent 都能引用 |
| **AGENTS.md 作為 prompt** | `AGENTS.md` 在部署時自動被當成 system prompt 載入，定義 docs-first 行為 |
| **Citation 機制** | Prompt 規定回答必須附上頁面標題或 URL，讓使用者可自行核對 |

這個範例**沒有**用到：subagent、virtual filesystem、HITL（interrupt）、write_todos、或 `langgraph dev` 本機模式。它純粹展示 `deepagents deploy` 的宣告式部署路徑，以及 MCP 工具整合的最小可行模式。

---

## 怎麼試用

### 在 LangSmith 介面問問題

部署完成後，在 LangSmith 的 Deployments 分頁找到這個 agent 並打開對話界面，試試這些問題：

- `"How do I configure memory in Deep Agents?"`
- `"What's the difference between sync and async subagents?"`
- `"Show me how to add an MCP server to deepagents.toml"`
- `"What models are supported for deploy?"`

README 說明：這個 agent 一律會**先搜尋文件**，並在回答時**附上找到答案的頁面出處**，方便自行核對。你可以觀察它是否確實先呼叫了搜尋工具，再呼叫頁面讀取工具，然後才整合出回答。

### 透過 SDK 以程式方式查詢

```python
from langgraph_sdk import get_client

client = get_client(url="https://<your-deployment-url>")
thread = await client.threads.create()

async for chunk in client.runs.stream(
    thread["thread_id"], "agent",
    input={"messages": [{"role": "user", "content": "How do I add an MCP server to deepagents.toml?"}]},
    stream_mode="messages",
):
    print(chunk.data, end="", flush=True)
```

這段程式碼做三件事：建立 client（`url` 填入部署 URL）、開一個新 thread、以串流方式送出問題並逐塊印出回應。部署 URL 可以在 LangSmith 的 Deployments 分頁找到。

---

## 延伸方向

讀完這個範例後，有幾個清楚的延伸方向：

**1. 接入更多 MCP 文件來源**
目前只接了 LangChain 的官方文件。可以用同樣的 `deepagents mcp-servers add` 指令，把其他套件（如 FastAPI、Pydantic、Anthropic）的文件 MCP endpoint 也加進來，讓 agent 能回答更廣泛的技術問題。

**2. 調整 AGENTS.md 的覆蓋範圍**
`AGENTS.md` 的「核心行為」區塊列出了觸發文件查詢的關鍵字（API、功能、設定、部署……）。可以根據你的使用場景擴充或縮減這個清單，讓 agent 的行為更精準地符合你的需求。

**3. 加入 subagent 架構**
目前是單一 agent 負責搜尋與作答。如果問題複雜到需要查多個主題，可以考慮在 `agent.json` 加入 subagent 設定，讓 orchestrator 把不同主題委派給不同的 research subagent 並行查詢（可參考 `deep_research` 範例的架構）。

**4. 增加 citation 驗證步驟**
可以在 prompt 裡加一個額外步驟：整合完答案後，回頭確認每個 citation URL 確實來自剛才讀過的頁面，避免模型在「填補」不確定之處時引用了不存在的 URL。

---

## 小結

`deploy-mcp-docs-agent` 是 Deep Agents 生態裡最「宣告式」的範例之一：沒有 Python 邏輯、沒有工具定義程式碼，只有兩個檔案——`AGENTS.md` 定義行為，`agent.json` 宣告部署設定，加上一條 `deepagents mcp-servers add` 指令接入外部 MCP server。

這個設計傳達了一個重要的訊息：**agent 的「個性」可以完全由 prompt 定義，agent 的「能力」可以透過 MCP 外掛。** 你不需要寫任何程式碼，也能部署一個有明確行為邊界、能查文件、會附 citation 的技術助理。

幾個值得帶走的設計原則：

- **Docs-first 是一個 prompt 策略**：把「先查再答」的順序明確寫進工具使用流程，比單純說「你可以使用文件工具」更有效。
- **Citation 是信任的基礎**：強制附上出處 URL，讓使用者能核對，比「你相信我就好」更有說服力。
- **誠實承認局限**：明確規定「找不到就說找不到」，是比「硬給一個可能錯的答案」更有價值的行為設計。
- **MCP 的 workspace 共用**：把常用的文件 MCP server 在 workspace 層級只註冊一次，所有 agent 都能引用——這是可維護性的關鍵。

如果你想在自己的組織裡部署一個「不亂說、有根據、附出處」的技術文件助理，這個範例是最直接的起點。
