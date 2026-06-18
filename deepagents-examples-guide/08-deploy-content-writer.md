> 本文對應範例：`docs/deepagents/examples/deploy-content-writer/`

---

# 它記得你是誰：用 Deep Agents 打造具備 Per-User Memory 的內容寫作 Agent

**副標：一次 deploy，服務千人，每個人的偏好各自保存——auth + memory 的零程式碼整合實戰**

---

大多數 AI 寫作工具都有同一個盲點：它們沒有記憶。每次對話都從零開始，你得重新說「我喜歡輕鬆的語氣」、「我們公司是做 B2B SaaS 的」、「別用太多術語」。這些偏好設定，每次對話都要重複一遍。

`deploy-content-writer` 這個 Deep Agents 官方範例，正是為了解決這個問題而設計的。它不只是一個「會寫文章的 agent」，而是「會記得是誰在跟它聊」的 agent——每位使用者的語氣偏好、主題興趣、格式選擇，都被存在各自隔離的記憶檔案裡；下次回來，agent 能直接延續上次的脈絡，不需要你重新介紹自己。

更厲害的是：這一切不需要你寫任何 auth middleware。一行設定，Supabase JWT 驗證就自動接上。

---

## 全貌：per-user memory 如何隔離、auth 如何零程式碼接上

在深入逐個檔案之前，先建立一張全局地圖，理解這個範例的核心設計。

### Memory 隔離：`/memories/user/` 目錄

每位通過驗證的使用者，都在 `/memories/user/` 底下擁有**屬於自己**的記憶空間：

| 檔案 | 讀寫權限 | 用途 |
|------|----------|------|
| `preferences.md` | 可讀可寫 | 記錄使用者偏好的語氣、格式、主題 |
| `context.md` | 唯讀 | 使用者所屬公司與產品的靜態背景資料 |

「依使用者區隔」這件事，是由 auth layer 在底層自動完成的：當使用者 A 的 JWT 傳進來，Deep Agents 的 runtime 就把 `/memories/user/` 的讀寫範圍限定在 A 的資料；使用者 B 絕對看不到 A 的 `preferences.md`。同一個 deploy，服務千人，記憶不會串味。

### Auth 零程式碼：`deepagents.toml` 的 `[auth]` 區塊

README 說明：只要在 `deepagents.toml` 裡加上：

```toml
[auth]
provider = "supabase"
```

部署時，Deep Agents CLI 就會自動產生一個 Supabase token 驗證器（validator），並把它接進這次 deploy——你完全不需要自己寫 middleware。若想在不啟用驗證的情況下部署，把這個區塊移除即可。

> **注意**：`deepagents.toml` 本身不在範例資料夾內（它是 deploy 時由 CLI 管理的設定），但 README 與 `.env.example` 都對其 `[auth]` 區塊有明確說明，可視為此範例的核心設計文件之一。

---

## 環境與部署：`.env.example` 和 `agent.json`

### `.env.example`——四個變數，各司其職

```
OPENAI_API_KEY=        # 用來存取 GPT-4.1 模型（必填）
LANGSMITH_API_KEY=     # 部署時必填
SUPABASE_URL=          # 你的 Supabase 專案 URL（啟用 auth 時必填）
SUPABASE_ANON_KEY=     # 你的 Supabase anon/public key（啟用 auth 時必填）
```

檔案結構乾淨、直白。Supabase 相關的兩個變數，「只有在 `deepagents.toml` 保留 `[auth]` 區塊時才需要」——如果你想先不帶 auth 測試，只填前兩個就能 deploy。

### `agent.json`——最精簡的 runtime 設定

```json
{
  "name": "deepagents-deploy-content-writer",
  "runtime": {
    "model": {"model_id": "openai:gpt-4.1"}
  }
}
```

只有兩個欄位：

- **`name`**：這個 agent 部署後的識別名稱，會出現在 LangSmith 的 Deployments 頁面。
- **`runtime.model.model_id`**：指定使用 OpenAI 的 GPT-4.1 模型。格式為 `provider:model`，換模型只需改這一行。

整個 `agent.json` 只有六行，體現了 Deep Agents 的設計哲學：**能用慣例解決的，就不要讓開發者手寫**。

部署指令同樣一行搞定：

```bash
deepagents deploy
```

---

## ★ 逐段導讀：核心檔案解析

### 1. `AGENTS.md`——主 agent 的完整指示書

這是整個 agent 的大腦。它定義了 agent 的**角色、語氣、工作流程，以及最關鍵的：如何讀寫 user memory**。

#### 品牌語氣（Brand Voice）

```
- 專業但平易近人：寫得像一位學識淵博的同事，而不是一本教科書
- 清楚且直接：除非必要，否則避免行話；用簡單的方式解釋技術概念
- 自信但不傲慢：分享專業見解，但不要居高臨下
- 有吸引力：用具體的例子、類比與故事來說明論點
```

這四條語氣準則，把「一個 AI 助手」塑造成「一個科技公司的專屬內容寫手」。它不是萬用 chatbot；它有特定的聲音。

#### 寫作準則（Writing Standards）

```
1. 使用主動語態
2. 開門見山先談價值——先講對讀者最重要的東西
3. 一段一個概念——讓每個段落聚焦、容易快速瀏覽
4. 具體勝於抽象——使用具體的例子、數字與案例研究
5. 以行動作結——每一篇內容都應該讓讀者知道接下來該做什麼
```

五條準則，覆蓋了從結構到收尾的寫作全流程，讓 agent 每次輸出都有一致的品質基準。

#### User Memory 區塊（最關鍵的部分）

```markdown
## User Memory（使用者記憶）

你可以存取位於 `/memories/user/` 的「依使用者區隔」記憶檔案。
用 `ls /memories/user/` 來查看有哪些可用的檔案。

- **preferences.md** — 可讀可寫。當你了解到使用者的內容偏好、語氣、
  感興趣的主題或格式選擇時，更新這個檔案。在每次對話開始時讀取它，
  以便個人化你的輸出。
- **context.md** — 唯讀。內含使用者所屬公司與產品的背景資料。
  建立內容時請參考它。

在開始工作前，務必先讀取你的使用者記憶檔案。當使用者分享偏好時，
使用 `edit_file` 更新 `/memories/user/preferences.md`。
```

這段指示有幾個設計細節值得注意：

1. **`ls /memories/user/` 先探測**：agent 不假設檔案一定存在，而是先列出可用檔案——這讓它能優雅地處理「新使用者尚無任何記憶檔案」的情況。
2. **讀寫分離**：`preferences.md` 可讀可寫，`context.md` 唯讀。設計上，背景資料由管理員或使用者事先填好，agent 不應覆蓋它；但偏好是動態的，agent 要主動更新。
3. **每次對話開始都要讀**：強制 agent 在任何動作之前先載入記憶，確保個人化從第一句回應就生效。
4. **用 `edit_file` 寫入**：具體指定工具名稱，避免 agent 用其他方式（如輸出文字叫使用者自己貼）更新記憶。

#### 工作流程（Workflow）

```
1. 先研究 — 動筆前，使用 `researcher` 子代理（subagent）對主題做深入研究
2. 擬大綱 — 用清楚的標題與合理的脈絡安排內容結構
3. 撰寫 — 依照品牌語氣與寫作準則草擬內容
4. 審閱 — 交付前，對照品質檢查清單逐項檢查
```

四步驟工作流程，其中第一步「委派 `researcher` subagent 做研究」是 Deep Agents 多代理架構的典型示範——主 agent 不自己爬文，而是把研究任務外包給專職的子代理。

---

### 2. `user/AGENTS.md`——每位使用者記憶的初始模板

```markdown
# Content Preferences

目前尚未設定任何偏好。當 agent 逐漸了解使用者偏好的主題、語氣調整與
格式選擇時，會自動更新這個檔案。
```

這個檔案是 `preferences.md` 的**初始狀態範本**。它非常短——因為它本來就應該是空的起點。

`user/` 目錄的存在意義：Deep Agents 的 per-user memory 機制，會把這個目錄的內容「複製」到每位新使用者的 `/memories/user/` 底下，作為他們記憶空間的初始值。第一次對話時，agent 看到的就是這段「目前尚未設定任何偏好」的提示；隨著使用者與 agent 互動，`preferences.md` 會被逐漸填充。

`context.md` 在 `user/` 目錄下並未出現——代表它需要由管理員或使用者自行建立，內容依使用者所屬公司而異，沒有通用的初始模板。

---

### 3. `skills/blog-post/SKILL.md`——部落格寫作 skill

```yaml
---
name: blog-post
description: 撰寫結構完整的長篇部落格文章，涵蓋研究調查、SEO 最佳化與封面圖片產生。
---
```

這個 skill 定義了主 agent 在「撰寫部落格文章」任務時應遵循的詳細流程。

**研究優先（Research First）**——這是強制步驟：

```markdown
在撰寫任何部落格文章之前，先把研究工作委派出去：
1. 使用 `task` 工具，並指定 `subagent_type: "researcher"`
2. 同時指明「研究主題」以及「研究結果要存到哪裡」
```

**文章結構五段式**：

| 段落 | 說明 |
|------|------|
| Hook（開場鉤子） | 引人入勝的問題、統計數字或陳述句，2-3 句以內 |
| Context（問題脈絡） | 說明主題為何重要，連結讀者自身經驗 |
| Main Content（3-5 段） | 每段用 H2 標題涵蓋一個關鍵重點，可放程式碼範例 |
| Practical Application（實際應用） | 逐步操作說明或程式碼片段 |
| Conclusion & CTA（結論與行動呼籲） | 最多 3 個條列重點 + 明確 CTA |

**輸出位置**：`blogs/<slug>/post.md`

**SEO 注意事項**：
- 在標題與第一段放入主要關鍵字
- 標題長度控制在 60 字元以內
- 撰寫 150-160 字元的 meta description

這個 skill 的設計體現了一個重要原則：**把「做什麼」和「怎麼做」都寫進 SKILL.md**，讓 agent 不需要依賴隱性知識，照著指示就能產出一致品質的長篇文章。

---

### 4. `skills/social-media/SKILL.md`——社群媒體 skill

```yaml
---
name: social-media
description: 製作社群媒體內容，包含 Twitter/X 討論串（thread）、LinkedIn 貼文與短篇更新。
---
```

這個 skill 涵蓋三種格式，每種都有具體的長度與結構規範：

**Twitter/X Thread**：
- 開頭推文：引人入勝的問題或大膽陳述，< 280 字元
- 3-7 則延伸推文展開主題
- 最後一則放 CTA 或關鍵重點

**LinkedIn Post**：
- 開頭鉤子（前 2 行顯示在「see more」展開之前）——這個細節直接對應平台的 UI 行為
- 3-5 個簡短段落帶出關鍵洞見
- 以問題作結，藉此帶動留言互動
- 目標長度約 1,300 字元

**Short-form Update**：
- 單一段落，控制在 280 字元以內，方便跨平台共用

**共通準則**：
- LinkedIn 用第一人稱，公司帳號用第三人稱
- 放入 2-3 個 hashtag（不要更多）
- 視平台調整語氣：LinkedIn 較專業，Twitter 較口語
- 用具體的數字與成果，取代含糊的說法

**輸出位置**：`social/<platform>/<slug>.md`（例如 `social/twitter/ai-agents-thread.md`）

把不同平台的規格寫進同一個 skill，讓主 agent 不需要自己記憶各平台的格式差異——只要呼叫 `social-media` skill，格式自動對齊。

---

### 5. `test_user_memory.py`——四個測試，驗證記憶隔離的完整性

這個測試腳本是整個範例最具說服力的部分。它用四個 thread 場景，系統性地驗證 per-user memory 的每一個關鍵行為：

```python
DEPLOY_URL = "https://deepagents-deploy-content-w-6909480a63d7575eb597d5a1b3c6e61e.us.langgraph.app"
USER_ID = "test-user-sydney"
```

**Thread 1：設定偏好**

```python
"I prefer concise, bullet-point style content. Please remember this preference."
```

使用 `user_id=USER_ID` 呼叫 agent，請它把「偏好簡潔條列式內容」這個偏好記下來。此時 agent 應該讀取 `/memories/user/preferences.md`，並用 `edit_file` 把這個偏好寫入。

**Thread 2：跨 thread 驗證記憶持續性（同一使用者）**

```python
"What are my content preferences? Read your memory files and tell me."
```

開一個**全新的 thread**，但 `user_id` 相同（`test-user-sydney`）。若 per-user memory 正常運作，agent 應該能從 `preferences.md` 讀出 Thread 1 設定的偏好，並回報出來。這一步驗證的是：**記憶跨越 thread 存活，而不是只在一次對話中有效**。

**Thread 3：不同使用者——不應看到別人的偏好**

```python
user_id="other-user-xyz"
```

同樣的問題，但換成完全不同的 `user_id`。這個使用者的 `/memories/user/preferences.md` 是全新的（初始狀態），不應該看到 `test-user-sydney` 設定的任何偏好。這一步驗證的是：**記憶隔離真的有效，不同帳號之間不會串味**。

**Thread 4：沒有 `user_id`——應優雅地略過**

```python
"Hello, just say hi back briefly."
```

不傳 `user_id`，看 agent 會不會崩潰。用 `try/except` 包住，預期 agent 能正常回應（只是沒有個人化的記憶可用）。這一步驗證的是：**auth 不是強制的——就算沒有 user_id，agent 也能運作**，不會因為找不到記憶檔案而報錯。

`run_thread()` 的實作細節也值得一看：它用 `stream_mode="values"` 串流，並在 AI 訊息的 content 中處理兩種格式——純字串，以及帶有 `type: "text"` 的 block list（對應工具呼叫的回應結構）。這讓測試腳本能穩健地從不同格式的回應中擷取最終文字。

---

## 這個範例示範的 Deep Agents 核心能力

| 能力 | 在此範例的體現 |
|------|--------------|
| **Per-user memory** | `/memories/user/` 依使用者 ID 自動隔離，不同帳號讀寫各自的記憶空間 |
| **Auth middleware** | `deepagents.toml` 的 `[auth] provider = "supabase"` 一行啟用 JWT 驗證，零自訂程式碼 |
| **Skills** | `blog-post` 和 `social-media` 兩個 skill，把平台特定的格式規範封裝進獨立模組 |
| **Deploy** | `deepagents deploy` 一行部署，自動整合 auth validator |
| **Subagent** | 主 agent 委派 `researcher` subagent 先做研究，再動筆 |

---

## 怎麼用：SDK 帶 JWT 查詢

部署完成後，用 `langgraph_sdk` 連上 agent，在 `Authorization` header 帶上 Supabase JWT：

```python
from langgraph_sdk import get_client

client = get_client(
    url="https://<your-deployment-url>",
    headers={"Authorization": "Bearer <your-supabase-jwt>"},
)
thread = await client.threads.create()

async for chunk in client.runs.stream(
    thread["thread_id"], "agent",
    input={"messages": [{"role": "user", "content": "Write a tweet about AI agents"}]},
    stream_mode="messages",
):
    print(chunk.data, end="", flush=True)
```

deploy URL 在 LangSmith 的 **Deployments** 頁面可以找到。部署端收到請求後，會自動驗證 JWT、解析出使用者身分，並把後續所有的 memory 讀寫都導向對應使用者的空間——你不需要在 client 端做任何額外處理。

---

## 延伸方向

這個範例是一個扎實的起點，可以往幾個方向延伸：

**豐富 `context.md` 的內容**：為不同的使用者預先填入他們的公司介紹、產品定位、目標受眾，讓 agent 一開始就有足夠的背景知識，產出更貼近使用者業務的內容。

**增加更多 skill**：目前有 `blog-post` 和 `social-media` 兩個 skill。可以繼續新增，例如 `press-release`（新聞稿）、`email-newsletter`（電子報）等，每個格式都有各自的結構規範與輸出路徑。

**多語言支援**：在 `preferences.md` 記錄使用者偏好的語言，讓 agent 自動切換輸出語言，同一個部署服務不同語系的使用者。

**串接其他 auth provider**：`[auth]` 區塊目前示範的是 Supabase，但同樣的設計模式可以套用到其他支援 JWT 的 auth provider。

---

## 小結

`deploy-content-writer` 這個範例，用最少的程式碼示範了一個「真正能商業化部署」的 AI 寫作服務需要什麼：

- **每個使用者都有自己的記憶**（`/memories/user/` + per-user auth 隔離）
- **auth 不需要自己寫**（`deepagents.toml` 的 `[auth]` 區塊）
- **寫作規範可以封裝成獨立模組**（`blog-post` 和 `social-media` skill）
- **品質可以被量化與驗證**（`test_user_memory.py` 的四個場景）

從「一個會寫文章的 chatbot」到「會記得每個使用者是誰的內容寫作服務」，差距不在於模型能力，而在於**記憶架構與部署設計**。這個範例把那個差距填上了，而且做得非常乾淨。

---

*📁 範例資料夾：`docs/deepagents/examples/deploy-content-writer/`*
*🔗 相關文件：[deepagents deploy](https://docs.langchain.com/deepagents/deploy) · [custom auth](https://docs.langchain.com/deepagents/auth) · [per-user memory](https://docs.langchain.com/deepagents/memory)*
