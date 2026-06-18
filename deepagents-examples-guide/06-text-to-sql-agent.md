> 原始範例路徑：`docs/deepagents/examples/text-to-sql-agent/`

# 讓 AI 自己寫 SQL：Text-to-SQL Deep Agent 完整導讀 🗄️

**副標：從自然語言到資料庫查詢，一窺 planning、skill 按需載入與 FilesystemBackend 如何協同運作**

---

## 一、為什麼 text-to-SQL 要用 Deep Agent，而不是單次 prompt？

「幫我查出最暢銷的前五位藝人」——這句話聽起來簡單，背後卻藏著至少四個步驟：先知道資料庫裡有哪些表格，再看懂各表格的 schema，接著組裝跨表 JOIN，最後執行並整理成人讀得懂的格式。

如果把這四步全部塞進一個 prompt，有幾個現實問題：

1. **schema 龐大時超出 context window**：把所有表格的欄位定義一次餵給模型，很快就超量。
2. **中間結果無處存放**：「先看 Artist，再看 Album，再看 InvoiceLine……」這些探索步驟不能靠記憶，要有地方暫存。
3. **錯了沒辦法自我修正**：SQL 語法出錯時，單次 prompt 只能靜默失敗；agent 卻可以讀取錯誤訊息、重寫再試。
4. **安全邊界難以硬編碼**：INSERT、DROP 等危險語句應在 agent 層面徹底禁止，而不是交給 prompt 「希望模型能記住」。

Deep Agents 框架剛好為這四個痛點提供對應解答：planning（`write_todos`）、FilesystemBackend（暫存中間結果）、tool loop（自我修正）、AGENTS.md 安全規則（行為約束）。這篇文章就帶你逐段看清楚這個範例是怎麼組起來的。

---

## 二、全貌：四個階段，三個核心能力

```
使用者的自然語言問題
        ↓
 Deep Agent（具備 planning）
        ├─ 📋 write_todos          → 把複雜問題拆成可追蹤步驟
        ├─ 🔍 schema-exploration   → 列出表格、辨識外鍵關聯
        ├─ ✍️  query-writing        → 撰寫並驗證 SQL、執行查詢
        └─ 📂 FilesystemBackend    → 儲存 / 讀取中間結果（可選）
        ↓
  SQLite 資料庫（Chinook）
        ↓
  格式化的最終答案
```

**三個核心能力的分工：**

| 能力 | 對應機制 | 什麼時候啟動 |
|---|---|---|
| Planning | `write_todos` tool（Deep Agents 內建） | 面對複雜、多步驟問題 |
| Skill（按需載入） | `skills/` 目錄下的 SKILL.md | agent 自行判斷需要哪個 workflow |
| Filesystem | `FilesystemBackend` | 需要跨步驟持久化中間結果時 |

值得注意的是，這個範例的 `subagents=[]`——沒有啟用 subagent。所有工作都在同一個 agent 內透過 SQL tool + skill 完成。這讓架構保持簡潔，也示範了「不是每個 Deep Agent 都需要子代理」這個設計選擇。

---

## 三、環境與安裝

### 相依套件（`pyproject.toml` 精要）

```toml
[project]
name = "text2sql-deepagent"
version = "0.1.0"
requires-python = ">=3.11"
dependencies = [
    "deepagents>=0.6.8",
    "langchain>=1.3.4,<2.0.0",
    "langchain-anthropic>=1.4.4",
    "langchain-community>=0.4.2",
    "langgraph>=1.2.4",
    "sqlalchemy>=2.0.50",
    "python-dotenv>=1.2.2",
    "tavily-python>=0.7.25",
    "rich>=15.0.0",
]
```

幾個欄位說明：

- **`deepagents>=0.6.8`**：框架主體，提供 `create_deep_agent`、`FilesystemBackend`、`write_todos` 等核心元件。
- **`langchain-community`**：提供 `SQLDatabaseToolkit` 與 `SQLDatabase`，這是 SQL tool 的來源。
- **`sqlalchemy>=2.0.50`**：SQLite 連線底層驅動。
- **`rich>=15.0.0`**：CLI 輸出的彩色 Panel 格式。
- **`tavily-python`**：這個範例雖然沒有直接使用網路搜尋，但列為相依，保留擴充空間。

### 安裝步驟

```bash
# 1. Clone 並進入範例目錄
git clone https://github.com/langchain-ai/deepagents.git
cd deepagents/examples/text-to-sql-agent

# 2. 下載 Chinook SQLite 資料庫
curl -L -o chinook.db \
  https://github.com/lerocha/chinook-database/raw/master/ChinookDatabase/DataSources/Chinook_Sqlite.sqlite

# 3. 建立虛擬環境並安裝套件（推薦用 uv）
uv venv --python 3.11
source .venv/bin/activate
uv pip install -e .

# 4. 設定環境變數
cp .env.example .env
# 編輯 .env，填入 ANTHROPIC_API_KEY
```

`.env` 必填：

```
ANTHROPIC_API_KEY=your_anthropic_api_key_here
```

選填（啟用 LangSmith 追蹤）：

```
LANGCHAIN_TRACING_V2=true
LANGSMITH_ENDPOINT=https://api.smith.langchain.com
LANGCHAIN_API_KEY=your_langsmith_api_key_here
LANGCHAIN_PROJECT=text2sql-deepagent
```

---

## 四、★ 逐段導讀

### 4-1 `agent.py`：組裝整個 agent 的核心

全檔只有約 110 行，卻把所有關鍵決策都集中在這裡。

#### 第一段：import 與資料庫連線

```python
from deepagents import create_deep_agent
from deepagents.backends import FilesystemBackend
from langchain_anthropic import ChatAnthropic
from langchain_community.agent_toolkits import SQLDatabaseToolkit
from langchain_community.utilities import SQLDatabase
```

- `create_deep_agent`：Deep Agents 框架的主要工廠函式，接收 model、memory、skills、tools、subagents、backend 六個參數，一次組裝完成。
- `FilesystemBackend`：把 agent 的暫存空間掛在本地目錄，讓 agent 可以在步驟之間讀寫檔案。
- `SQLDatabaseToolkit`：來自 `langchain_community`，會根據傳入的 `SQLDatabase` 物件自動生成一組 SQL tool。

#### 第二段：`create_sql_deep_agent()` 函式

```python
def create_sql_deep_agent():
    base_dir = os.path.dirname(os.path.abspath(__file__))

    db_path = os.path.join(base_dir, "chinook.db")
    db = SQLDatabase.from_uri(f"sqlite:///{db_path}", sample_rows_in_table_info=3)

    model = ChatAnthropic(model="claude-sonnet-4-5-20250929", temperature=0)

    toolkit = SQLDatabaseToolkit(db=db, llm=model)
    sql_tools = toolkit.get_tools()

    agent = create_deep_agent(
        model=model,
        memory=["./AGENTS.md"],
        skills=["./skills/"],
        tools=sql_tools,
        subagents=[],
        backend=FilesystemBackend(root_dir=base_dir),
    )

    return agent
```

幾個細節值得注意：

**`sample_rows_in_table_info=3`**：`SQLDatabase.from_uri` 的這個參數讓 schema tool 回傳時附帶 3 筆範例資料，幫助模型理解欄位內容，而不是只看欄位名稱。

**`temperature=0`**：SQL 生成要求精確、可重現，設成 0 是標準做法。

**`toolkit.get_tools()`**：這行呼叫會回傳四個 tool，分別是：
- `sql_db_list_tables`：列出資料庫裡的所有表格
- `sql_db_schema`：取得指定表格的欄位定義與範例資料
- `sql_db_query_checker`：在執行前先做語法檢查
- `sql_db_query`：實際執行 SQL 並回傳結果

**`memory=["./AGENTS.md"]`**：永遠載入，每次 agent 啟動時都會讀入，作為身份與安全規則的基礎。

**`skills=["./skills/"]`**：指向整個 skills 目錄。agent 在 context 中只會看到每個 SKILL.md frontmatter 裡的 `description`，只有在判斷當前任務需要某個 skill 時，才把完整的 SKILL.md 內容載入進來——這就是「漸進式揭露（progressive disclosure）」。

**`subagents=[]`**：明確設為空，這個範例不啟用子代理。

**`FilesystemBackend(root_dir=base_dir)`**：把 agent 的工作目錄設在 `agent.py` 所在的資料夾，讓 agent 可以用 `ls`、`read_file`、`write_file`、`edit_file` 等 filesystem tool 在本地讀寫暫存檔。

#### 第三段：`main()` 與 CLI

```python
result = agent.invoke(
    {"messages": [{"role": "user", "content": args.question}]}
)
final_message = result["messages"][-1]
```

`agent.invoke` 接收 LangChain 標準的 messages dict。執行完成後取 `result["messages"][-1]` 的 content 即為最終答案。CLI 用 `rich` 的 `Panel` 把問題和答案分別用藍色（cyan）和綠色（green）框起來，錯誤則用紅色。

---

### 4-2 `AGENTS.md`：行為規範，永遠在線

這個檔案對應 `create_deep_agent` 的 `memory` 參數，每次 agent 啟動都會載入。它做了五件事：

**① 定義角色**：「你是一個專門用來與 SQL 資料庫互動的 Deep Agent」，並列出五個步驟的工作流程（探索表格 → 檢視 schema → 產生 SQL → 執行 → 整理答案）。

**② 資料庫背景知識**：告知 agent 這是 SQLite 的 Chinook 資料庫，有藝人、專輯、曲目、客戶、發票、員工等資料，讓 agent 在解讀問題時有領域背景。

**③ 查詢指引**：預設 LIMIT 5、避免 SELECT *、執行前再次確認語法、查詢失敗要分析錯誤並重寫——這些規則是讓 agent 輸出品質穩定的關鍵。

**④ 安全規則**（最重要的部分）：

```
絕對不要執行以下這些語句：
INSERT / UPDATE / DELETE / DROP / ALTER / TRUNCATE / CREATE
你只有唯讀（READ-ONLY）權限，只允許 SELECT 查詢。
```

這不只是 prompt 裡的建議，而是直接寫進 agent 的「身份記憶」，每次都會讀到。結合 `SQLDatabase.from_uri` 預設的唯讀連線，形成雙層防護。

**⑤ 針對複雜問題的規劃指引**：明確告訴 agent 面對複雜問題時應使用 `write_todos` 拆解步驟，並可用 filesystem tool 儲存中間結果。

---

### 4-3 `skills/schema-exploration/SKILL.md`：資料庫結構探索工作流

```yaml
---
name: schema-exploration
description: 列出表格、描述欄位與資料型別、辨識外鍵（foreign key）關聯，
             並繪製資料庫中各實體（entity）之間的關係。當使用者詢問資料庫
             結構（schema）、表格結構、欄位型別、有哪些表格、ERD、外鍵，
             或各實體之間如何關聯時使用此技能。
---
```

frontmatter 的 `description` 就是 agent 在 context 中看到的全部資訊，它決定了 agent 「什麼時候該載入這個 skill」。

skill 正文定義了四步工作流：

1. **`sql_db_list_tables`**：取得完整表格清單。
2. **`sql_db_schema`**：對指定表格取欄位名稱、資料型別、範例資料、主鍵、外鍵。
3. **繪製關聯**：找出以 `Id` 結尾的欄位，追蹤父子關係，建立心智地圖。
4. **回答問題**：說明表格用途、欄位內容、關聯路徑，附上範例資料。

SKILL.md 還提供了三個範例（列出所有表格、描述 Customer 表格、如何查詢藝人營收），並給出品質指引，例如「把相關表格分組」、「標註主鍵與外鍵」、「說明整條關聯鏈」。

這個 skill 的定位是**探索而不執行**——它的最後通常會說「需要 query-writing skill 來執行」，明確交棒。

---

### 4-4 `skills/query-writing/SKILL.md`：SQL 撰寫與執行工作流

```yaml
---
name: query-writing
description: 撰寫並執行 SQL 查詢，範圍從簡單的 SELECT 到複雜的多表 JOIN、
             彙總（aggregation）與子查詢（subquery）。當使用者要求查詢資料
             庫、撰寫 SQL、執行 SELECT 語句、取得資料、篩選紀錄，或從資料
             庫表格產生報表時使用此技能。
---
```

這個 skill 依問題複雜度拆成兩條路徑：

**簡單查詢（單一表格）：**
識別表格 → `sql_db_schema` 看欄位 → 撰寫 SELECT（含 WHERE/LIMIT/ORDER BY）→ `sql_db_query` 執行 → 整理答案。

**複雜查詢（多表 JOIN）：**
先用 `write_todos` 拆解步驟 → 對每個表格分別呼叫 `sql_db_schema` → 依 FK = PK 組裝 JOIN → 確認 GROUP BY 完整性 → 執行並驗證。

SKILL.md 內嵌了一個具體的 SQL 範例：

```sql
SELECT
    c.Country,
    ROUND(SUM(i.Total), 2) as TotalRevenue
FROM Invoice i
INNER JOIN Customer c ON i.CustomerId = c.CustomerId
GROUP BY c.Country
ORDER BY TotalRevenue DESC
LIMIT 5;
```

這個範例示範了表格別名（`i`、`c`）、ROUND 函式、GROUP BY + ORDER BY 的標準寫法，直接成為 agent 的學習樣本。

**錯誤復原**部分定義了三種失敗情境的處理方式：空結果（確認欄位名稱與 NULL 值）、語法錯誤（重檢 JOIN 與 alias）、逾時（加 WHERE 篩選縮小結果集）。這讓 agent 在遇到 SQL 執行失敗時有清楚的修正路徑，而不是直接放棄。

品質指引最後強調兩點：`write_todos` 用於複雜查詢，以及禁止 DML 語句（與 AGENTS.md 的安全規則呼應，形成雙重約束）。

---

## 五、這個範例示範的 Deep Agents 能力

| 能力 | 實作方式 | 在哪裡體現 |
|---|---|---|
| 漸進式揭露（Progressive Disclosure） | skill frontmatter + 按需載入 | `skills/*/SKILL.md` 的 `description` |
| Planning | `write_todos` 內建 tool | AGENTS.md 與 query-writing SKILL.md 都提及 |
| Filesystem 持久化 | `FilesystemBackend` | `create_deep_agent` 的 `backend` 參數 |
| 工具組整合 | `SQLDatabaseToolkit` 自動生成 SQL tool | `agent.py` 的 `toolkit.get_tools()` |
| 安全邊界 | AGENTS.md 安全規則 + 唯讀連線 | 雙層防護 |
| 自我修正 | 錯誤訊息讀取 + 重寫 SQL | query-writing SKILL.md 的 Error Recovery |

**`subagents=[]` 的設計選擇**：這個範例刻意不使用子代理，展示了 Deep Agents 的靈活性——即使只用單一 agent + skill，也能處理多步驟的複雜查詢。這對於不需要平行處理或高度專門化分工的場景，是更輕量的選擇。

---

## 六、怎麼跑：可以試的自然語言問題

```bash
# 簡單計數
python agent.py "How many customers are from Canada?"

# 排行榜（需要 JOIN Artist + Album + Track + InvoiceLine）
python agent.py "What are the top 5 best-selling artists?"

# 多維度分析（需要 planning + 多表 JOIN）
python agent.py "Which employee generated the most revenue by country?"
```

**觀察 agent 的思考過程**：啟用 LangSmith tracing 後，你可以在 `https://smith.langchain.com/` 看到完整的執行軌跡，包括：
- `write_todos` 拆出了哪些步驟
- 哪個 skill 被載入
- `sql_db_schema` 被呼叫了幾次
- 最終執行的 SQL 語句是什麼
- token 用量與費用

---

## 七、延伸方向

1. **接更大的資料庫**：把 `SQLDatabase.from_uri` 的連線字串換成 PostgreSQL 或 MySQL，SQL tool 的行為幾乎不需要改動。
2. **啟用 subagent**：把 schema exploration 和 query writing 各自拆成獨立的 subagent，讓主 agent 只負責 planning 與最終整理，適合資料庫超大、schema 探索耗時的場景。
3. **新增 `data-analysis` skill**：在查詢完成後，新增一個負責統計分析（平均值、趨勢、異常值）的 skill，讓 agent 不只是「查到就回傳」，而是主動給出洞察。
4. **寫入保護的進階做法**：除了 AGENTS.md 的規則，可以在 `SQLDatabase.from_uri` 層面設定唯讀使用者，或在資料庫層建立 view-only 連線，讓安全防護更徹底。

---

## 八、小結

Text-to-SQL Deep Agent 是一個精巧的範例，它用不到 120 行的 `agent.py` 展示了 Deep Agents 框架的三個核心設計思想：

- **Memory（AGENTS.md）** 定義「誰」：角色、安全規則、查詢原則——永遠在線，形成穩固的行為基礎。
- **Skill（SKILL.md）** 定義「怎麼做」：schema-exploration 和 query-writing 各司其職，只在需要時才載入，保持 context 精簡。
- **Tool（SQLDatabaseToolkit）** 定義「能做什麼」：`sql_db_list_tables`、`sql_db_schema`、`sql_db_query_checker`、`sql_db_query` 四個 tool 覆蓋了完整的 SQL 工作流。

如果你想在自己的專案裡接入任何 SQL 資料庫，這個範例是一個幾乎可以直接改用的起點——把連線字串換掉，調整 AGENTS.md 的資料庫背景說明，再視需要新增 skill，就能快速得到一個生產等級的 text-to-SQL agent。

---

*本文基於 `deepagents>=0.6.8`、`langchain>=1.3.4`、Claude Sonnet 4.5（`claude-sonnet-4-5-20250929`）。*
