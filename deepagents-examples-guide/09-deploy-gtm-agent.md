> 原始範例路徑：`docs/deepagents/examples/deploy-gtm-agent/`

# Sync 等、Async 跑：用 Deep Agents 部署一個 Go-to-Market 策略 Agent

**副標：巢狀 subagent-as-folder、同步阻塞與非同步背景任務如何在一個 `deepagents deploy` 裡協同作戰**

---

## 引言：為什麼 GTM 需要協調多個子代理？

把一個新產品推向市場，從來都不是單一任務。你同時需要知道市場在哪裡（市場研究）、競爭格局長什麼樣（競爭者分析），以及用什麼語言說話（行銷文案）。這三件事的**時序關係**各不相同：

- 你必須先看完市場研究，才能寫出言之有物的策略。研究是「決策的前提」，策略依賴它，所以你只能等。
- 但部落格文章、到達頁（landing page）這些行銷素材，不需要等策略百分之百確定才能開始起草。它們可以在背景默默跑，等策略定案再整合進去。

這個直覺，正是 `deploy-gtm-agent` 範例要示範的核心模式：**用 sync subagent 處理阻塞型依賴，用 async subagent 處理可平行化的後台工作**。

而且，整個系統不需要一行 orchestration 膠水程式碼。你只需要幾個資料夾、幾份 Markdown，然後執行 `deepagents deploy`。

---

## 全貌：Sync vs Async Subagent 的差異與適用時機

### 為什麼同樣是 subagent，一個同步、一個非同步？

在 Deep Agents 的架構裡，subagent 的 sync/async 屬性描述的是**主 agent 要不要等這個 subagent 完成**：

| 模式 | 主 agent 行為 | 適用情境 |
|------|-------------|---------|
| **Sync（同步）** | 委派任務後阻塞（block），等回傳結果才繼續 | 後續步驟依賴這份結果，不能繞過 |
| **Async（非同步）** | 委派任務後立即繼續，不等結果 | 結果可以在流程末端整併，不影響主線 |

在這個範例裡，`market-researcher` 是 sync：主 agent 委派市場研究之後必須等研究結果，因為 GTM 策略的定位、定價、通路選擇全部建立在這份研究之上。

`content-writer` 是 async：內容產製任務（部落格、landing page、文案）耗時較長，但主 agent 不需要等它完成才能輸出 GTM 計畫骨架；最終只需要把產出整合進交付物即可。

這個搭配讓整體流程更有效率：研究是阻塞點，但內容產製可以與策略撰寫**平行**進行。

### 資料夾即 subagent：自動探索與串接

這個範例最有趣的部分之一，是 subagent 的**部署模型**。你不需要在主 agent 裡手動「import」或「register」子代理；只要把 subagent 的資料夾放在 `subagents/` 目錄下，`deepagents deploy` 就會在部署時**自動探索並串接**它們：

```
deploy-gtm-agent/
├── AGENTS.md                        # 主 agent 的協調指示
├── agent.json                       # 主 agent 的設定（名稱、模型）
├── skills/
│   └── competitor-analysis/
│       └── SKILL.md                 # 競爭者分析 skill
└── subagents/
    └── market-researcher/           # 這個資料夾 = 一個獨立的 subagent
        ├── AGENTS.md                # market-researcher 自己的角色指示
        ├── agent.json               # market-researcher 自己的模型設定
        └── skills/
            └── analyze-market/
                └── SKILL.md         # market-researcher 自己的 skill
```

注意 `subagents/market-researcher/` 這個子目錄的結構：它自己也有 `AGENTS.md`、`agent.json`、`skills/`。也就是說，**一個 subagent 本身就是一個完整的 agent**，只是被放進父 agent 的 `subagents/` 資料夾裡。這個「資料夾即 agent」的設計讓巢狀 orchestration 變得非常直觀。

> 📌 `content-writer` subagent 在這個範例的檔案結構裡並沒有對應的資料夾（README 中提到它是 async subagent，但 `subagents/` 下只有 `market-researcher/`）。這是 Deep Agents 的能力展示，實際整合細節可能在部署層處理。本文後續導讀以實際讀到的檔案為準。

---

## 環境與部署

### 環境變數

在執行 `deepagents deploy` 之前，需要設定兩個環境變數：

| 變數 | 說明 |
|------|------|
| `OPENAI_API_KEY` | 模型存取金鑰，主 agent 使用 `gpt-5.4-nano` |
| `LANGSMITH_API_KEY` | 部署到 LangSmith 平台的必填金鑰 |

把專案根目錄的 `.env` 範本複製一份填入自己的金鑰即可。

### 一鍵部署

```bash
deepagents deploy
```

Deploy 完成後，到 LangSmith 的 **Deployments** 分頁找到部署網址，就可以開始使用。你也可以用 LangGraph SDK 從程式碼呼叫：

```python
from langgraph_sdk import get_client

client = get_client(url="https://<your-deployment-url>")
thread = await client.threads.create()

async for chunk in client.runs.stream(
    thread["thread_id"], "agent",
    input={"messages": [{"role": "user", "content": "Build a GTM plan for our new Python SDK for AI agents"}]},
    stream_mode="messages",
):
    print(chunk.data, end="", flush=True)
```

---

## ★ 逐段導讀

### 1. 主 AGENTS.md：協調策略的核心

**路徑：`deploy-gtm-agent/AGENTS.md`**

這份檔案定義了主 agent（GTM Strategy Agent）的角色與行為。全文約 24 行，但每一個段落都有明確的語義分工：

**§ 角色宣告**

```markdown
# GTM Strategy Agent

你是一個進入市場策略（go-to-market）代理人，負責協助團隊規劃並執行產品上市。
```

簡短的角色定義，讓 LLM 在整個對話中保持一致的自我認知。

**§ 能力（Capabilities）**

```markdown
## 能力

- **market-researcher**（同步）：負責委派市場研究任務 — 競爭者分析、TAM/SAM/SOM 市場規模估算，以及受眾分群。
- **content-writer**（非同步）：負責啟動耗時較長的內容產製任務 — 部落格文章、到達頁（landing page），以及行銷文案。
```

這裡直接在 Markdown 中宣告了兩個 subagent 的**名稱**與**同步/非同步屬性**。`（同步）`、`（非同步）` 這兩個標記讓 LLM 知道該如何排程委派工作——哪個要等、哪個可以先繼續。

**§ 工作流程（Workflow）**

```markdown
## 工作流程

1. 拿到要上市的產品或功能後，先把市場研究委派給 market-researcher 子代理人。
2. 根據研究結果，發展出涵蓋定位（positioning）、定價（pricing）與通路選擇（channel selection）的 GTM 策略。
3. 針對所需的行銷素材，透過 content-writer 這個非同步子代理人啟動內容產製任務。
4. 持續追蹤非同步任務，並把產出的交付物整併進最終的 GTM 計畫。
```

四個步驟的工作流程明確定義了**執行順序**：先研究（同步等待）→ 再策略（主 agent 自己做）→ 再啟動內容任務（非同步觸發）→ 最後整合。這是整個協調機制的劇本。

**§ 準則（Guidelines）**

```markdown
## 準則

- 所有建議都必須以 market-researcher 提供的研究資料為依據。
- 提出策略時，要附上清楚的理由與佐證。
- 在為 content-writer 撰寫內容簡報（content brief）時，務必包含目標受眾、關鍵訊息，以及語氣（tone）準則。
```

準則段落是 guardrail：確保主 agent 不會憑空捏造策略，並且在移交給 content-writer 時傳遞足夠的上下文（target audience、key messages、tone guidelines）。

---

### 2. 主 agent.json：名稱、描述與模型

**路徑：`deploy-gtm-agent/agent.json`**

```json
{
  "name": "deepagents-deploy-gtm-agent",
  "description": "Go-to-market strategy agent that coordinates research and content creation",
  "runtime": {
    "model": {"model_id": "openai:gpt-5.4-nano"}
  }
}
```

這份設定檔只有三個欄位：

| 欄位 | 說明 |
|------|------|
| `name` | Agent 在 LangSmith 平台上的識別名稱 |
| `description` | 描述這個 agent 的用途，供平台和其他工具識別 |
| `runtime.model.model_id` | 指定使用的 LLM，格式為 `provider:model-name`，這裡是 OpenAI 的 `gpt-5.4-nano` |

主 agent 使用 `gpt-5.4-nano`，這是一個輕量模型——因為主 agent 的主要工作是協調（orchestration）而非深度推理，所以不需要最強的模型。實際推理工作交給子代理自行處理。

---

### 3. competitor-analysis SKILL.md：主 agent 的 Skill

**路徑：`deploy-gtm-agent/skills/competitor-analysis/SKILL.md`**

```yaml
---
name: competitor-analysis
description: >-
  分析特定市場區隔中的競爭者。
  觸發時機：競爭格局、競爭者分析、
  市場比較、競爭定位。
---
```

SKILL.md 的 frontmatter 定義了兩個關鍵欄位：

- **`name`**：skill 的唯一識別名稱
- **`description`**：描述這個 skill 的用途與觸發時機。Deep Agents 用這段文字決定「何時啟用這個 skill」——當對話內容匹配到「競爭格局」、「市場比較」等語義時，這個 skill 就會被載入。

Skill 的正文是具體的工作流程指示：

```markdown
# Competitor Analysis

當被要求分析競爭者時：

1. 找出目標區隔中前 3 到 5 名的競爭者
2. 針對每一個競爭者，評估：
   - 產品定位與關鍵差異化要素
   - 定價模式與方案層級
   - 目標受眾與市佔率估計
   - 優勢與劣勢
3. 製作一份比較矩陣（comparison matrix）
4. 找出市場缺口，以及可供差異化的機會
```

四個步驟定義了一套標準化的競爭者分析框架：識別競爭者 → 多維度評估 → 製作比較矩陣 → 找出差異化機會。這個 skill 掛在**主 agent** 層級，意思是當主 agent 判斷需要做競爭者分析時，它可以直接使用這個 skill，不必委派給 market-researcher。

---

### 4. market-researcher 的 AGENTS.md：子代理的角色定義

**路徑：`deploy-gtm-agent/subagents/market-researcher/AGENTS.md`**

market-researcher 有自己的 AGENTS.md，結構和主 agent 一樣，但內容聚焦在市場研究的專業職能：

**§ 重點領域（Focus Areas）**

```markdown
## 重點領域

- **市場規模估算**：TAM、SAM、SOM 估計，並附上推算方法
- **競爭者分析**：產品定位、定價、市佔率
- **受眾分群**：人口統計、心理統計（psychographics）、購買行為
- **趨勢分析**：產業趨勢、新興技術、法規變化
```

四個領域覆蓋了 GTM 研究的主要面向，並且每一條都附有具體的子項目——這些子項目直接成為 LLM 輸出時的結構指引。

**§ 產出（Output）**

```markdown
## 產出

撰寫一份完整的 markdown 報告 — 內容須包含推算方法、各主題的詳細發現，以及對 GTM 策略具體可行的建議。使用 write_file 工具，把報告存到 `/memories/subagents/market-researcher/market-research-report.md`。其餘的結構化欄位應該是從這份報告中萃取出的精簡內容。並在 `full_report_path` 中回傳這個檔案路徑。
```

這個段落值得特別注意：它不只告訴 agent「輸出什麼」，還告訴它**怎麼輸出**：

1. 用 `write_file` 工具把完整報告存到指定路徑（`/memories/subagents/market-researcher/market-research-report.md`）
2. 在回傳值中附上 `full_report_path`，讓主 agent 知道報告存在哪裡

這個設計讓市場研究報告**持久化到 memory 層**，後續的 GTM 策略撰寫可以隨時引用，不會隨對話消失。

**§ 準則（Guidelines）**

```markdown
## 準則

- 盡可能標註資料來源
- 區分「硬資料（hard data）」與「估計值」
- 標示出不確定的部分，或還需要進一步研究的地方
- 分析聚焦在「對進入市場規劃實際有用」的內容上
```

準則段落強調資料品質與誠實標註——明確區分「有來源的事實」與「估算值」，這在市場研究裡非常重要，避免 agent 過度自信地提供沒有依據的數字。

---

### 5. market-researcher 的 agent.json：子代理的模型設定

**路徑：`deploy-gtm-agent/subagents/market-researcher/agent.json`**

```json
{
  "description": "Researches market trends, competitors, and target audiences to inform GTM strategy",
  "model_id": "openai:gpt-5.4-mini"
}
```

和主 agent 的 `agent.json` 相比，這份設定有兩個值得注意的差異：

1. **沒有 `name` 欄位**：子代理的名稱由它所在的**資料夾名稱**（`market-researcher`）決定，不需要在 JSON 裡重複定義。
2. **使用 `model_id`（而非 `runtime.model.model_id`）**：欄位路徑的差異說明子代理的設定格式比主 agent 更精簡。
3. **模型是 `gpt-5.4-mini`**：比主 agent 的 `gpt-5.4-nano` 稍強——市場研究需要更深的推理能力（計算 TAM/SAM/SOM、評估競爭格局），所以使用更大的模型。

---

### 6. market-researcher 的 analyze-market SKILL.md：子代理的 Skill

**路徑：`deploy-gtm-agent/subagents/market-researcher/skills/analyze-market/SKILL.md`**

```yaml
---
name: analyze-market
description: >-
  針對某個產品類別或市場區隔執行市場分析。
  觸發時機：市場分析、市場規模、TAM SAM SOM、
  市場機會、產業分析。
---
```

這個 skill 掛在 **market-researcher 子代理**層級，不是主 agent。它的觸發時機是「TAM SAM SOM」、「市場機會」等語義——當主 agent 把市場研究任務委派給 market-researcher 時，market-researcher 會用這個 skill 來結構化它的分析過程。

正文工作流程：

```markdown
# Market Analysis

當被要求分析某個市場時：

1. 定義市場邊界（地理範圍、區隔、時間範圍）
2. 估算市場規模（TAM/SAM/SOM），並附上推算方法
3. 找出關鍵趨勢與成長動能
4. 描繪競爭格局
5. 評估進入障礙（barriers to entry）
6. 彙整市場機會，並提出明確的建議
```

六個步驟從「定義邊界」到「提出建議」，形成完整的市場分析框架。特別是**第 1 步「定義市場邊界」**——這是很多分析師容易跳過的步驟，但沒有邊界，TAM/SAM/SOM 的數字就沒有意義。把它明確寫進 SKILL.md，等於強制 agent 在計算市場規模之前先釐清範疇。

---

## 這個範例展示的 Deep Agents 能力

透過 `deploy-gtm-agent`，這個範例完整展示了四個 Deep Agents 的核心能力：

| 能力 | 在本範例中的體現 |
|------|----------------|
| **Sync subagent** | `market-researcher` — 主 agent 委派後阻塞等待，確保策略建立在研究資料之上 |
| **Async subagent** | `content-writer` — 主 agent 委派後繼續執行，內容在背景平行產製 |
| **巢狀 agent-as-folder** | `subagents/market-researcher/` 本身也是一個完整 agent，有自己的 AGENTS.md、agent.json、skills/ |
| **Skills（按需載入）** | 主 agent 有 `competitor-analysis` skill；market-researcher 有 `analyze-market` skill，各自在對應的語義觸發時載入 |

---

## 怎麼使用這個 Agent

部署完成後，你可以丟給它以下類型的提示：

```
"We're launching a new Python SDK for AI agents next month — build me a GTM plan"
"Help us position our vector database product against Pinecone and Weaviate"
"We're targeting mid-market engineering teams — what channels should we prioritize?"
```

Agent 接到提示後的執行流程：

1. 🔍 觸發 `market-researcher`（sync）→ 等待完整市場研究報告
2. 🧠 主 agent 根據研究結果撰寫 GTM 策略（定位、定價、通路）
3. ✍️ 觸發 `content-writer`（async）→ 背景產製行銷素材
4. 📋 整合所有交付物，輸出最終 GTM 計畫

---

## 延伸思考

這個範例的架構有幾個值得延伸的方向：

**增加更多子代理**：`subagents/` 下可以再加 `seo-analyst/`、`pricing-strategist/` 等資料夾，`deepagents deploy` 會自動發現並串接它們。每個子代理都可以有自己的模型選擇，依推理需求分級配置。

**調整 sync/async 策略**：如果你的 content-writer 產出對策略有回饋影響（例如 A/B 測試文案決定通路選擇），可以把它改成 sync，讓主 agent 等待結果再繼續。

**在 SKILL.md 裡加入 MCP 工具呼叫**：若有對接真實資料來源的需求（如 Crunchbase API、SEMrush API），可以透過 `mcp.json` 設定 MCP server，讓 agent 在 skill 執行過程中呼叫外部工具取得真實市場數據。

**Memory 持久化**：market-researcher 已經示範了把報告存到 `/memories/subagents/market-researcher/` 的模式——這個路徑的報告在後續任務中可以被重複引用，讓 agent 隨著任務累積的知識愈來愈豐富。

---

## 小結

`deploy-gtm-agent` 是一個設計精緻的 orchestration 範例，它用最少的檔案清晰地示範了兩件事：

**第一，sync/async 不是技術細節，是業務邏輯。** 「需要拿來當決策依據」的工作走 sync、阻塞等待；「可以平行進行、最後整合」的工作走 async、背景執行。這個判斷應該發生在系統設計層，而不是程式碼層。

**第二，「資料夾即 agent」讓巢狀 orchestration 零摩擦。** 你不需要寫任何 registration 程式碼——把 subagent 的資料夾放進 `subagents/`，`deepagents deploy` 自動處理探索與串接。每個 subagent 本身就是一個完整的 agent，有自己的角色指示、模型設定，甚至自己的 skills。

如果你正在構建需要多步驟、多專業領域協作的 agent 系統，這個範例提供了一個清晰的設計模板：**用資料夾結構表達 orchestration 拓撲，用 AGENTS.md 定義協調邏輯，讓框架處理其餘的一切。**
