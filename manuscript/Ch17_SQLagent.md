# Ch 17　SQL agent：讓 agent 安全查資料庫

> **本章目標**
>
> - 打造一個能用自然語言查 SQL 資料庫的 agent：列表 → 看 schema → 產生 SQL → 檢查 → 執行。
> - 把**安全**放在第一位：資料庫權限最小化、唯讀、執行前檢查——這是「讓模型產生並執行 SQL」這件危險事的必要防線。
> - 用 Ch5 的 Graph API 加上「專責節點」與條件邊，比一個籠統的 prompt 更可控。
> - 理解這套流程如何用 system prompt 與專責節點，把模型約束在安全的軌道上。
>
> **使用版本**：前半（LangGraph 版）用 `langgraph` 1.x、`langchain`、`langchain-google-genai`；後半補強（17.7–17.12，Deep Agents 版）用 `deepagents` 0.6.x、`langchain-community`、`langchain-anthropic`、`sqlalchemy`。配套 notebook 用內建 `sqlite3` 建真資料庫、用離線假 SQL 生成示範**流程**，免 API key；Deep Agents 版需 `ANTHROPIC_API_KEY`，配套 notebook 有可跑版本。

---

## 17.1 危險但實用：讓模型產生並執行 SQL

「用自然語言問資料庫」是極實用的應用，但它要求**執行模型產生的 SQL**——這有真實風險。模型可能產生 `DROP TABLE`、撈出整張敏感表、或寫出無效查詢。

所以本章第一原則就是安全，請刻在心裡：**把資料庫連線權限縮到 agent 需要的最小範圍——最好是唯讀、只開放必要的表。** 這能減輕（但無法完全消除）風險。把這當成不可妥協的前提，再來談功能。

## 17.2 流程設計：四個專責節點

LangChain 有現成的 SQL agent，但它靠 system prompt 約束行為（例如「一定先列表」「執行前先檢查」）。我們在 LangGraph 裡用**專責節點**把這些步驟「寫死進結構」，得到更高的控制度。一個典型的 ReAct 式 SQL agent 有這些步驟：

```
1. list_tables   列出資料庫有哪些表
2. get_schema    取出相關表的 schema（欄位、型別）
3. generate_query  依問題與 schema 產生 SQL
4. check_query   執行前檢查 SQL（語法、危險操作）
5. run_query     執行，把結果回給模型彙整成答案
```

把「列表」「看 schema」「檢查」做成**明確的節點**，而不是寄望模型「自己記得要做」，就是 Ch15 說的「能用 workflow 結構約束就別全交給 agent 自決」。

## 17.3 準備工具與資料庫

LangChain 提供讀 SQL 的工具集。先連資料庫、選一個支援 tool calling 的模型：

```python
import sqlite3
from langchain.chat_models import init_chat_model

model = init_chat_model("google_genai:gemini-2.5-flash", temperature=0)

# 開發用：內建 sqlite 建一個玩具資料庫
conn = sqlite3.connect(":memory:", check_same_thread=False)
conn.executescript("""
    CREATE TABLE products (id INTEGER PRIMARY KEY, name TEXT, price INTEGER, stock INTEGER);
    INSERT INTO products (name, price, stock) VALUES
        ('滑鼠', 590, 120), ('鍵盤', 1290, 45), ('螢幕', 6990, 8);
""")
```

讀 schema 與執行查詢，包成工具給模型用（唯讀！）：

```python
from langchain.tools import tool

@tool
def list_tables() -> str:
    """列出資料庫中所有表的名稱。"""
    rows = conn.execute("SELECT name FROM sqlite_master WHERE type='table'").fetchall()
    return ", ".join(r[0] for r in rows)

@tool
def get_schema(table: str) -> str:
    """取得指定表的 schema（欄位與型別）。"""
    rows = conn.execute(f"PRAGMA table_info({table})").fetchall()
    return "\n".join(f"{r[1]} {r[2]}" for r in rows)

@tool
def run_query(query: str) -> str:
    """執行一條唯讀 SQL 查詢並回傳結果。"""
    if not query.strip().lower().startswith("select"):   # 最起碼的防線：只准 SELECT
        return "錯誤：只允許 SELECT 查詢。"
    return str(conn.execute(query).fetchall())
```

注意 `run_query` 那道 `select` 檢查——這是把「唯讀」寫進程式的具體防線，不只靠 prompt 拜託模型乖乖的。

## 17.4 檢查節點：執行前再把關

除了工具內的基本防線，再加一個**獨立的檢查步驟**：在執行前，請模型（或用規則）檢查 SQL 是否合理、有沒有常見錯誤、會不會撈太多。這對應上面流程的 `check_query`。雙重保險的精神是：**危險操作前永遠多一道關卡。**

你甚至可以在這裡接上 Ch11 的 human-in-the-loop：對「可能改動資料」或「撈大量資料」的查詢，`interrupt` 暫停請人核准再執行。對 production 的資料庫 agent，這種人工關卡常常是值得的。

## 17.5 組成圖：用 system prompt + 結構雙重約束

把模型、工具與專責節點接成圖：模型節點 `bind_tools([list_tables, get_schema, run_query])`，用 system prompt 引導它「先列表、再看 schema、產生查詢前先檢查」，同時用圖結構（條件邊：有 tool call 就去 tools 節點、否則結束）確保流程。

```python
SYSTEM = (
    "你是查 SQL 資料庫的助理。務必：先用 list_tables 看有哪些表，"
    "再用 get_schema 看相關表結構，產生 SQL 後先檢查再用 run_query 執行。"
    "只能讀取（SELECT），絕不修改資料。回答用中文。"
)
```

這就是「prompt 引導 + 結構約束」的雙保險：prompt 告訴模型該怎麼做，結構（專責節點、唯讀工具、檢查關卡）確保即使模型不聽話，傷害也被框住。

## 17.6 為什麼不直接用 prebuilt？

> 提示：以下 17.7–17.12 是本章的「往上一層」補強。前面六節我們在 **LangGraph** 把 SQL agent 一根根接線、把每道防線焊進結構；接下來改用 **Deep Agents** 的高階組裝，看同一件事怎麼用 `create_deep_agent` 加上 `memory` 與 `skills` 寫成。這幾節用的是 `deepagents 0.6.x`，需要 `ANTHROPIC_API_KEY`（配套 notebook 有可跑版本）；只想看結構與簽名的話免金鑰。

LangChain 的 prebuilt SQL agent 上手快，但行為全靠 system prompt 約束。當你需要**更高的控制度**——強制某些步驟一定發生、在特定點插入人工核可、對不同查詢類型走不同路徑——就值得像本章這樣在 LangGraph 裡自訂。這再次呼應全書主軸：高階抽象快，但要精準控制就下沉到 LangGraph（Ch2）。

## 17.7 補強：用 Deep Agent 包一個 Text-to-SQL agent（Chinook 範例）

前面我們花了六節在 LangGraph 裡「焊」出一個 SQL agent：自己接專責節點、自己寫唯讀工具、自己掛檢查關卡。那是**低抽象、高控制**的做法。現在換個角度——同樣是「安全查資料庫」，如果你願意把控制度交給一個成熟的 harness，**Deep Agents** 能讓你少寫很多 scaffolding。

> 本節到 17.12 改寫自 deepagents 官方範例 `examples/text-to-sql-agent`。原範例用 Chinook（一個數位音樂商店的 SQLite 樣本庫），有 Artist／Album／Track／Customer／Invoice／Employee 等 11 張表，是練多表 JOIN 的經典教材。

核心想法只有一句：**用 `create_deep_agent` 把「SQL 工具 + 安全規則（memory）+ 領域工作流（skills）+ 規劃（write_todos）」一次組裝起來。** 完整的 factory 長這樣：

```python
import os

from deepagents import create_deep_agent
from deepagents.backends import FilesystemBackend
from langchain.chat_models import init_chat_model
from langchain_community.agent_toolkits import SQLDatabaseToolkit
from langchain_community.utilities import SQLDatabase


def create_sql_deep_agent():
    """組裝一個 text-to-SQL 的 Deep Agent。"""
    base_dir = os.path.dirname(os.path.abspath(__file__))

    # 連 Chinook 資料庫；sample_rows_in_table_info=3 會在 schema 裡附 3 列範例資料
    db_path = os.path.join(base_dir, "chinook.db")
    db = SQLDatabase.from_uri(f"sqlite:///{db_path}", sample_rows_in_table_info=3)

    # 同一個模型既給 toolkit（產 query / 檢查）用，也給 agent 推理用
    model = init_chat_model("anthropic:claude-sonnet-4-5", temperature=0)

    # 把 LangChain 的 SQL 工具集直接拿來當 deep agent 的 tools
    toolkit = SQLDatabaseToolkit(db=db, llm=model)
    sql_tools = toolkit.get_tools()

    agent = create_deep_agent(
        model=model,
        tools=sql_tools,                              # ← LangChain 的 SQL 工具，當「附加」工具
        memory=["./AGENTS.md"],                       # ← always-loaded：身份與唯讀安全規則
        skills=["./skills/"],                         # ← on-demand：query-writing / schema-exploration
        subagents=[],                                 # 本例不需要 subagent
        backend=FilesystemBackend(root_dir=base_dir),  # ← 用真實磁碟當後端，skills/AGENTS.md 從這讀
    )
    return agent
```

> 對照一下 17.5 的 LangGraph 版：那邊你要自己 `bind_tools`、自己畫條件邊、自己寫 `run_query` 的 `select` 防線。這裡 **`create_deep_agent` 一次到位**——`write_todos`、`ls`/`read_file`/`write_file`/`edit_file` 都是內建工具，你只負責「餵 SQL 工具 + 掛知識檔」。差別在抽象高度，不在能力。

跑跑看：

```python
agent = create_sql_deep_agent()
result = agent.invoke(
    {"messages": [{"role": "user", "content": "加拿大有幾位客戶？"}]}
)
print(result["messages"][-1].content)
```

跑下去你會看到 agent 先用 `sql_db_list_tables` 看有哪些表、用 `sql_db_schema` 看 `Customer` 結構、產一條 `SELECT COUNT(*) ... WHERE Country = 'Canada'`、執行、再把數字用中文回給你。整段流程沒有任何一行是你寫的節點——都長在 harness 裡。

> 💡 **冷知識** —— Chinook 這名字來自北美西岸的「Chinook 鮭魚」與同名的暖風（Chinook wind）。它是微軟 Northwind 範例庫的跨平台後繼者，幾乎每個 SQL 教學都拿它當試金石——你在 LangChain 文件裡看到的 SQL 範例八成也是它。

## 17.8 把 LangChain SQLDatabaseToolkit 當 tools 餵給 deep agent

上一節那行 `tools=sql_tools` 是整個範例的關鍵互通點：**Deep Agents 不自己造 SQL 工具，而是直接吃 LangChain 生態現成的 `SQLDatabaseToolkit`。** 這也是 deepagents 的設計哲學——任何 `BaseTool` 或 `@tool` 函數都能當 `tools=` 餵進去。

`SQLDatabaseToolkit(db, llm).get_tools()` 會回傳四個工具：

| 工具 `name` | 作用 | 對應 17.2 的步驟 |
|---|---|---|
| `sql_db_list_tables` | 列出資料庫所有表 | `list_tables` |
| `sql_db_schema` | 取指定表的 schema（含 `sample_rows_in_table_info` 設定的範例列） | `get_schema` |
| `sql_db_query_checker` | 用 LLM 檢查 SQL 有沒有常見錯誤 | `check_query` |
| `sql_db_query` | 實際執行 SQL | `run_query` |

看出來了嗎？**這四個工具幾乎一對一映射到我們 17.2 手工拆的四個專責節點。** 差別是：17.2 你把這四步寫成圖的節點與條件邊（結構約束）；這裡它們只是四個工具，**何時呼叫、呼叫順序交給模型自決**——靠 `AGENTS.md`（17.9）與 `skills/`（17.10）把正確流程「教」給它，而不是「焊」進結構。

`SQLDatabase.from_uri(..., sample_rows_in_table_info=3)` 這個參數很值得記：它讓 `sql_db_schema` 回傳時，每張表附上 3 列真實範例資料。模型「看過幾筆長相」才不會瞎猜欄位語意（例如 `Total` 是金額還是數量）——這是讓 text-to-SQL 準確度跳一階的小開關。代價是 schema 內容變長、吃更多 token，敏感資料庫要小心（見 17.12）。

> 🪞 **對照閱讀 —— 同一個 SQL agent 在三層怎麼組：**

| | LangGraph（17.1–17.6） | LangChain `create_agent` | Deep Agents（本節） |
|---|---|---|---|
| **怎麼做** | 自己畫 `StateGraph`，把 list/schema/check/run 接成節點與條件邊 | 一行 `create_agent(model, tools=sql_tools)`，行為靠 system prompt | `create_deep_agent(model, tools=sql_tools, memory=..., skills=...)` |
| **你親手做什麼** | 寫節點函數、`bind_tools`、條件邊、唯讀防線、檢查關卡——幾百行 | 寫一段 system prompt 約束流程——幾十行 | 寫 `AGENTS.md` 與 `SKILL.md`（知識檔）、餵工具——幾行程式碼＋幾份 markdown |
| **規劃／檔案** | 自己加（Ch5/Ch11） | 沒有，要自己接 | `write_todos`、`ls`/`read_file`/`write_file` 都內建 |

抽象高度由左到右遞增：程式碼從幾百行降到幾行，但你把「控制」換成了「對 harness 的信任」。要強制步驟一定發生、要在某點 `interrupt` 人工核可，仍然是最左邊的 LangGraph 最穩；要快速搭一個會規劃、會查、會存中間結果的 SQL 助理，最右邊最省事。

## 17.9 用 AGENTS.md memory 寫死唯讀安全規則（只允許 SELECT）

17.1 講過：**危險操作的防線不能只靠 prompt 拜託。** 在 LangGraph 版我們把 `select` 檢查焊進 `run_query`。在 Deep Agents 版，`memory=["./AGENTS.md"]` 是另一條防線——`AGENTS.md` 是 **always-loaded** 的記憶檔，每次對話都會被塞進 context，等於 agent 的「身份證 + 法規手冊」。把唯讀規則寫進去，模型每一步都看得到它。

`AGENTS.md` 的安全規則段落是這樣寫的：

```markdown
## Safety Rules

**NEVER execute these statements:**
- INSERT
- UPDATE
- DELETE
- DROP
- ALTER
- TRUNCATE
- CREATE

**You have READ-ONLY access. Only SELECT queries are allowed.**

## Query Guidelines

- Always limit results to 5 rows unless the user specifies otherwise
- Order results by relevant columns to show the most interesting data
- Only query relevant columns, not SELECT *
- Double-check your SQL syntax before executing
- If a query fails, analyze the error and rewrite
```

`memory` 載入的就是 `AGENTS.md`（這檔名是約定，對應 Ch36 的 Skills 與 Memory 機制）。它和 `system_prompt` 的差別是：`system_prompt` 是你硬塞的主提示；`memory` 是「一份放在檔案系統、可被讀寫的長期記憶」，由 `MemoryMiddleware` 在每輪載入。

> 🩸 **血淚教訓** —— 我曾經把唯讀規則「只」寫在 `AGENTS.md` 裡，覺得 always-loaded 應該夠了。結果模型在一次多步推理的中段，被一個「幫我把這筆訂單金額更新成 0 來測試」的誘導 prompt 帶偏，差點產出 `UPDATE`。**always-loaded memory 是『教育』，不是『圍欄』。** 真正擋住它的，是資料庫帳號本身就是唯讀（`GRANT SELECT`）、以及工具層那道 `startswith("select")` 檢查。教訓很簡單：**memory 寫安全規則是好習慣，但永遠要有一道『模型再不聽話也越不過去』的程式／權限防線。** 三層防護——唯讀 DB 帳號、工具層檢查、`AGENTS.md` 規則——缺一不可。

> ⏱ **三秒判斷** —— 安全規則該寫哪？
> - 「模型違反就會釀災」（DML/DDL）→ **資料庫權限 + 工具層程式檢查**（硬防線）
> - 「希望模型養成的習慣」（先看 schema、預設 LIMIT 5、不要 `SELECT *`）→ **`AGENTS.md` memory**（軟引導）
> - 「特定任務才需要的細節流程」→ **`skills/`**（見 17.10，按需載入）

## 17.10 用 skills/ 漸進式揭露掛 query-writing 與 schema-exploration 工作流

如果把所有 SQL 撰寫技巧、JOIN 範例、錯誤復原指南全塞進 `AGENTS.md`，context 會爆，而且大部分查詢根本用不到那些細節。Deep Agents 的解法是 **skills 的 progressive disclosure（漸進式揭露）**：

> **平時 agent 只看到每個 `SKILL.md` 的 `description`（一句「我是誰、何時該用我」），真正需要時才把整份 `SKILL.md` 的完整工作流程載進 context。**

這個範例有兩個 skill：

```
skills/
├── query-writing/
│   └── SKILL.md      # 簡單 / 複雜查詢的撰寫流程、JOIN 與聚合範例、錯誤復原
└── schema-exploration/
    └── SKILL.md      # 列表、描述欄位、辨識外鍵、映射實體關係的流程
```

每個 `SKILL.md` 用 YAML frontmatter 的 `name` 與 `description` 控制觸發時機。**`description` 要寫得像「何時該叫我」的觸發條件**，這是 skill 設計的精髓：

```yaml
---
name: query-writing
description: Writes and executes SQL queries from simple SELECTs to complex
  multi-table JOINs, aggregations, and subqueries. Use when the user asks to
  query a database, write SQL, run a SELECT statement, retrieve data, filter
  records, or generate reports from database tables.
---
```

```yaml
---
name: schema-exploration
description: Lists tables, describes columns and data types, identifies foreign
  key relationships, and maps entity relationships in a database. Use when the
  user asks about database schema, table structure, column types, what tables
  exist, ERD, foreign keys, or how entities relate.
---
```

注意 `description` 裡那串 **「Use when the user asks to ...」**——它列出一堆具體觸發語境（query a database、write SQL、retrieve data...）。寫得越具體，模型越能在對的時機把這份 skill 拉進來。frontmatter 之後才是真正的工作流程內容（簡單查詢五步、複雜查詢先 `write_todos` 規劃、JOIN 範例、錯誤復原指南），那些**只有被觸發時才進 context**。

`skills=["./skills/"]` 載入時，`SkillsMiddleware` 會掃這個目錄、把每個 `SKILL.md` 的 `description` 注入 agent 的可用技能清單。這就是用幾乎零 token 成本掛載深度領域知識的辦法——更完整的機制在 Ch36 細講。

> 💡 **冷知識** —— skills 用 `FilesystemBackend(root_dir=base_dir)` 從**真實磁碟**讀。換句話說，你不改一行 Python，光是新增一個 `skills/report-formatting/SKILL.md` 檔，就替 agent 加了一項新本事——這是「用檔案而非程式碼擴充 agent」的設計，也是 deepagents 跟 `deepagents-code`（`dcode` CLI）共用的同一套 skills 格式。

## 17.11 用 write_todos 拆解多表 JOIN 的複雜分析問題

簡單問題（「加拿大有幾位客戶？」）模型一條 `SELECT COUNT(*)` 就解掉。但「**哪位員工在哪些國家賺最多錢？**」這種要跨 `Employee`→`Customer`→`Invoice` 三張表 JOIN 再雙重聚合的問題，直接硬幹很容易漏 JOIN 條件或 `GROUP BY` 不完整。

Deep Agents 內建 `write_todos` 規劃工具（對應 Ch30 的 Planning），`AGENTS.md` 與 `query-writing` skill 都明確指示模型：複雜問題先寫 todo 拆解再執行。跑「哪位員工在哪些國家賺最多錢」你會看到 agent 先吐出一張 todo list：

```
write_todos:
- [ ] 列出資料庫有哪些表
- [ ] 看 Employee、Customer、Invoice 的 schema 與外鍵
- [ ] 規劃 Employee → Customer → Invoice 的 JOIN 結構
- [ ] 執行查詢，依員工與國家聚合營收
- [ ] 整理並格式化結果
```

接著它才照這份計畫一步步走：`sql_db_list_tables` → 多次 `sql_db_schema` → 產 JOIN query → （需要時）`sql_db_query_checker` → `sql_db_query` → 彙整。`write_todos` 的價值不是「多一個工具」，而是**把長任務的計畫攤在 context 裡當外部記憶**，讓模型每一步都對齊原始目標、不會做到一半忘了要 `GROUP BY country`。

中間若想存檔（例如把第一版查詢結果落地再交叉比對），`FilesystemBackend` 給的 `write_file`/`read_file` 隨手可用——這正是 Deep Agents 比 prebuilt SQL agent「深」的地方：它有規劃、有檔案系統當工作區，扛得住多步分析。

## 17.12 常見坑：sample_rows_in_table_info、權限防護與 query_checker

把 Deep Agent 版 SQL agent 推上線前，這幾個地雷要先排掉：

- **以為 `AGENTS.md` 唯讀規則就夠**：它是 always-loaded 的「教育」，不是「圍欄」。production 一定要疊上**唯讀 DB 帳號**（`GRANT SELECT`）與工具層程式檢查——和 17.1 的第一原則完全一致，換成 Deep Agents 也不打折。
- **`sample_rows_in_table_info` 在敏感庫直接漏資料**：它讓 `sql_db_schema` 附真實範例列，準確度大增，但等於把幾筆真資料送進 LLM context（與 trace、log）。敏感資料庫請調小或關掉，或先做去識別化。
- **以為 `sql_db_query_checker` 是安全閘**：它是「**語法／常見錯誤**」檢查（靠 LLM），不是權限檢查，擋不住惡意的 `DROP`。安全靠帳號權限與工具層，別把 `query_checker` 當防火牆。
- **沒鎖 LIMIT，一條查詢撈爆**：`AGENTS.md` 寫了「預設 LIMIT 5」，但模型可能忽略。對 production 在 DB 端設 `statement_timeout` 與結果列數上限，必要時接 Ch11 的 `interrupt` 對大查詢人工核可——和 17.4 的精神相同。
- **版本與相依沒鎖死**：官方範例 `uv.lock` 鎖在 `deepagents 0.4.4`，書中以 `deepagents 0.6.x` 為準；不同版的內建工具與參數可能有出入（例如 `model=None` 已 deprecated，**範例一律明確傳模型**）。安裝用 uv：`uv add deepagents langchain-community sqlalchemy`，並記得 `chinook.db` 是 gitignored、要另外 `curl` 下載。

---

## 常見坑（Pitfalls）

- **給 agent 的資料庫帳號權限過大**：頭號風險。一定要最小權限、最好唯讀、只開必要的表。
- **只靠 system prompt 防護**：prompt 是「拜託模型乖」，不是保證。把唯讀、檢查寫進程式（工具內檢查、檢查節點），結構性地框住。
- **不看 schema 就產 SQL**：模型會猜欄位名，產出無效或錯誤查詢。強制先 `list_tables` + `get_schema`。
- **沒有執行前檢查 / 人工關卡**：對會改資料或撈大量資料的查詢，加 `check_query` 與（必要時）Ch11 的 interrupt。
- **把 DB 連線跨執行緒亂用**：sqlite 範例記得 `check_same_thread=False`；production 用連線池與正確的並發處理。

## 本章小結

SQL agent 是極實用但有風險的應用——它要執行模型產生的 SQL。第一原則是**安全**：資料庫權限最小化、唯讀、把防線寫進程式而非只靠 prompt。我們用 Ch5 的 Graph API 把流程拆成專責節點（list_tables → get_schema → generate → check → run_query），用唯讀工具與檢查節點構築結構性防線，並可在執行前接 Ch11 的人工關卡。相較於 prebuilt SQL agent 全靠 prompt 約束，在 LangGraph 自訂讓你獲得「強制步驟、插入核可、分流查詢」的控制度——這正是 Ch2 說的「要精準控制就下沉」。本章後半（17.7–17.12）再往上一層：用 **`create_deep_agent`** 把同一件事用高階組裝寫出來——`tools=` 直接吃 LangChain 的 `SQLDatabaseToolkit`、`memory=["./AGENTS.md"]` 掛 always-loaded 的唯讀安全規則、`skills=["./skills/"]` 用 **progressive disclosure** 按需載入 query-writing 與 schema-exploration 工作流、內建 `write_todos` 拆解多表 JOIN。**抽象越高、程式碼越少，但「模型再不聽話也越不過去」的硬防線（唯讀帳號、工具層檢查）在任何一層都不能省**——這是 SQL agent 唯一不可妥協的原則。至此 Part I 只剩最後一塊拼圖：替你辛苦打造的 agent 寫測試，確保它改了不會壞。

## 延伸閱讀 / 練習

1. **離線流程版**：用內建 `sqlite3` 建一個玩具資料庫，把「SQL 生成」用一個寫死的 `SELECT` 取代（不呼叫 LLM），跑通 list_tables → run_query 的流程（配套 notebook 有可跑版本，免 API key）。
2. **加防線**：在 `run_query` 裡擋掉 `;`（多語句）與非 `SELECT`，丟幾個惡意查詢測試它擋得住。
3. **加人工關卡**：對「撈超過 N 列」的查詢用 Ch11 的 `interrupt` 暫停請人核准。
4. **Deep Agent 版實作**：`curl` 下載 `chinook.db`，照 17.7 用 `create_deep_agent(model, tools=SQLDatabaseToolkit(...).get_tools(), memory=["./AGENTS.md"], skills=["./skills/"], backend=FilesystemBackend(...))` 跑起來，分別問一個簡單問題與「哪位員工在哪些國家賺最多錢？」，比較有沒有觸發 `write_todos`（需 `ANTHROPIC_API_KEY`，配套 notebook 有可跑版本）。
5. **設計一個新 skill**：在 `skills/` 下新增一個 `report-formatting/SKILL.md`，frontmatter 的 `description` 寫好「何時該用我」的觸發條件，內容寫成「把查詢結果整理成 markdown 表格」的工作流；不改任何 Python，驗證 agent 會在對的時機把它載進來（progressive disclosure，對應 Ch36）。
6. **思考題**：為什麼「把唯讀寫進工具程式 / 資料庫帳號」比「在 `AGENTS.md` memory 寫『請只讀取』」更可靠？always-loaded memory 既然每輪都看得到，為何仍不算「圍欄」？如果三層（DB 權限、工具層檢查、`AGENTS.md`）都做，是多餘還是必要？
