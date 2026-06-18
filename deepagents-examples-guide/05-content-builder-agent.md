> 原始範例路徑：`docs/deepagents/examples/content-builder-agent/`

# 用三個檔案定義一個 Agent：Deep Agents 的 Filesystem Primitive 完整導讀

**副標：memory、skills、subagents 如何分工，以及 content_writer.py 如何把它們接起來**

---

## 引言

如果你曾經構建過一個 LLM agent，你八成有過這樣的經歷：agent 的行為邏輯全部硬編碼在 Python 裡，每次改一個 prompt 都得動到程式碼、重新部署。這不只麻煩，更讓非工程師的協作者完全插不上手。

Deep Agents 的 `content-builder-agent` 範例提出了一個不同的答案：**把 agent 的「個性」、「技能」與「下屬」全部外部化成磁碟上的檔案**。程式碼只負責把這些檔案接起來，業務邏輯則活在可讀、可編輯的 Markdown 與 YAML 裡。

這篇文章會把這個範例的每一個核心檔案拆開來逐段解釋，讓你真正理解三種 filesystem primitive 的**差異**與**載入時機**。

---

## 🗂️ 全貌：三個 Primitive，一支組裝腳本

這個範例的目標是一個內容寫手 agent，能夠撰寫部落格文章、LinkedIn 貼文與推文，每一篇都附帶 AI 生成的封面圖。

整個專案只有五個核心設定檔，加上一支入口腳本：

```
content-builder-agent/
├── AGENTS.md                    # 🧠 Memory：品牌語調與風格指南
├── subagents.yaml               # 🤝 Subagents：研究員子代理定義
├── skills/
│   ├── blog-post/
│   │   └── SKILL.md             # 📖 Skill：部落格寫作工作流程
│   └── social-media/
│       └── SKILL.md             # 📖 Skill：社群媒體工作流程
└── content_writer.py            # 🔧 入口：組裝所有元件，定義 tool
```

三種 primitive 的本質差異，一張表說清楚：

| Primitive | 對應檔案 | 載入時機 | 作用 |
|-----------|---------|---------|------|
| **Memory** | `AGENTS.md` | 永遠（always）注入 system prompt | 長期上下文，例如品牌聲調 |
| **Skills** | `skills/*/SKILL.md` | 按需（on demand）載入 | 特定任務的工作流程手冊 |
| **Subagents** | `subagents.yaml`（程式碼讀取）| 啟動時定義，任務時委派 | 獨立跑的專責小幫手 |

> **一句話記法：** Memory 是「一直記得的東西」，Skills 是「需要時才翻出來的手冊」，Subagents 是「派出去的小幫手」。

---

## 🛠️ 環境與安裝

```toml
# pyproject.toml（節錄）
[project]
name = "content-builder-agent"
requires-python = ">=3.11"
dependencies = [
    "deepagents>=0.6.8",
    "google-genai>=2.8.0",
    "pillow>=12.2.0",
    "pyyaml>=6.0.3",
    "rich>=15.0.0",
    "tavily-python>=0.7.25",
]
```

專案的相依套件揭示了技術選型：
- `deepagents` 是核心框架，提供 `create_deep_agent`、middleware 與 `FilesystemBackend`
- `google-genai` 用來呼叫 Gemini 的圖片生成模型（`gemini-2.5-flash-image`）
- `pillow` 負責把 Gemini 回傳的 inline data 存成 PNG 檔
- `pyyaml` 用來解析 `subagents.yaml`
- `rich` 提供漂亮的終端機輸出（spinner、panel、markdown 渲染）
- `tavily-python` 是 researcher subagent 的網路搜尋後端（選用）

執行前需要準備三把 API key：

```bash
export ANTHROPIC_API_KEY="..."   # 主 agent 必備
export GOOGLE_API_KEY="..."      # 圖片生成必備
export TAVILY_API_KEY="..."      # 網路搜尋（選用）
```

使用 `uv run` 時，第一次執行會自動安裝相依套件，不需要手動建 virtualenv：

```bash
cd examples/content-builder-agent
uv run python content_writer.py "Write a blog post about prompt engineering"
```

---

## ★ 核心逐段導讀

### 1. `AGENTS.md`：Memory Primitive

`AGENTS.md` 是這個 agent 的「靈魂」。它不描述任務流程，它描述的是**這個 agent 是誰**。

```markdown
# Content Writer Agent

你是一家科技公司的內容寫手。你的工作是產出引人入勝且具資訊量的內容，
幫助讀者了解 AI、軟體開發與新興技術。

## Brand Voice

- **專業但平易近人**：像一位學識淵博的同事在說話，而不是一本教科書
- **清楚且直接**：除非必要，否則避免術語；用淺顯的方式解釋技術概念
...

## Writing Standards

1. **使用主動語態**：...
2. **價值先行**：...
...

## Research Requirements

在針對任何主題動筆前：
1. 使用 `researcher` 子代理（subagent）進行深入的主題研究
2. 蒐集至少 3 個可信的來源
```

**Memory 的關鍵特性**：無論 agent 接到什麼任務，這份文件的內容**永遠**都在 system prompt 裡。這意味著：

- **品牌聲調（brand voice）**：專業但平易近人、主動語態、價值先行——這些原則不需要每次都在 prompt 裡重複。
- **寫作標準（writing standards）**：句子長度控制、每段一個重點、以行動呼籲收尾——這些成為 agent 的預設行為。
- **研究要求（research requirements）**：在任何主題動筆前必須先用 `researcher` 子代理做研究——這個硬性規定被「燒進」了 agent 的記憶。
- **Content pillars**：AI agents、開發者工具、軟體架構——圈定了內容的邊界。

這裡有一個設計巧思：`AGENTS.md` 在 research requirements 一節明確提到了要使用 `researcher` subagent，讓 memory 和 subagents 之間形成呼應——memory 告訴 agent「要做研究」，subagents 則提供「怎麼做研究」的能力。

**想改語調？** 只要編輯 `AGENTS.md` 即可，程式碼完全不用動。

---

### 2. `subagents.yaml`：Subagents Primitive

```yaml
researcher:
  description: >
    ALWAYS use this first to research any topic before writing content.
    Searches the web for current information, statistics, and sources.
    When delegating, tell it the topic AND the file path to save results
    (e.g., 'Research renewable energy and save to research/renewable-energy.md').
  model: anthropic:claude-haiku-4-5-20251001
  system_prompt: |
    You are a research assistant. You have access to web_search and write_file tools.

    ## Your Tools
    - web_search(query, max_results=5, topic="general") - Search the web
    - write_file(file_path, content) - Save your findings

    ## Your Process
    1. Use web_search to find information on the topic
    2. Make 2-3 targeted searches with specific queries
    3. Gather key statistics, quotes, and examples
    4. Save findings to the file path specified in your task
    ...
  tools:
    - web_search
```

每個欄位的意義：

| 欄位 | 說明 |
|------|------|
| 頂層鍵（`researcher`） | subagent 的名稱，主 agent 用這個名稱透過 `task` tool 委派任務 |
| `description` | 主 agent 決定「要不要委派」時看的說明，也說明了委派時需要提供什麼資訊 |
| `model` | 這個 subagent 使用的模型，此處用 `claude-haiku-4-5` 這個較輕量的選項，節省成本 |
| `system_prompt` | subagent 自身的 system prompt，完整定義它有哪些工具、怎麼做事 |
| `tools` | 這個 subagent 能用的工具清單（字串名稱，由 `load_subagents()` 解析成物件） |

值得注意的設計決策：

1. **model 降規**：主 agent 用 Claude 高階模型，researcher subagent 用 `claude-haiku-4-5`。研究員負責的是資料蒐集，不需要高階推理，省成本合理。
2. **description 裡的指令**：`ALWAYS use this first` 和 `tell it the topic AND the file path` 這種明確指令放在 description 裡，是要引導主 agent 正確使用這個 subagent。
3. **寫檔到 `research/`**：subagent 把研究結果存到磁碟，主 agent 之後再 `read_file` 讀取——這是 agent 之間以「檔案系統」溝通的模式，清楚留下可審查的中間產物。

**Subagents 和 Memory/Skills 的根本差異**：subagents 必須在程式碼中定義（傳給 `create_deep_agent` 的 `subagents` 參數）。這個範例透過 `load_subagents()` 工具函式把設定外部化到 YAML，但這是範例自製的輔助機制，不是 deepagents 框架原生支援的。

---

### 3. `skills/blog-post/SKILL.md`：Skill Primitive（部落格）

#### Frontmatter：按需載入的觸發器

```yaml
---
name: blog-post
description: 撰寫並編排長篇部落格文章、建立教學大綱，並針對 SEO 最佳化內容，
  同時產生封面圖片。當使用者要求撰寫部落格文章、文章、how-to 指南、
  教學、技術說明、思想領導（thought leadership）類文章，或任何長篇內容時，
  使用此技能。
---
```

這個 YAML frontmatter 是 skills 機制的核心。`description` 字段扮演的角色，就是讓 SkillsMiddleware 判斷「這個任務要不要載入這個 skill」的依據——有點像 tool calling 裡的 tool description。

當主 agent 收到「Write a blog post about AI agents」這個請求，它不會一開始就把所有 skill 都載入。middleware 會先看各個 SKILL.md 的 description，判斷哪個最相關，才把那個 skill 的完整內容加進 context。這就是「按需載入」的機制，**避免了把所有工作流程一次塞滿 context window**。

#### 文件本體：Progressive Disclosure

SKILL.md 的正文是給 agent 看的工作流程手冊，採用了 **progressive disclosure** 的結構：

1. **Research First（強制）**：
   ```
   task(
       subagent_type="researcher",
       description="Research [TOPIC]. Save findings to research/[slug].md"
   )
   ```
   手冊直接給出 `task` tool 的呼叫格式，讓 agent 不需要自己摸索。

2. **Output Structure（強制）**：
   ```
   blogs/
   └── <slug>/
       ├── post.md        # 部落格文章內容
       └── hero.png       # 必要：產生的封面圖片
   ```
   明確規定輸出的目錄結構，讓結果可預期。

3. **Blog Post Structure**：Hook → Context → Main Content → Practical Application → Conclusion & CTA 的五段式結構。

4. **Cover Image Generation**：`generate_cover` tool 的呼叫方式，以及如何撰寫有效的圖片 prompt（subject、style、composition、color palette、lighting、technical details 六個維度）。

5. **SEO Considerations**：關鍵字密度、標題長度、meta description 等具體數字規範。

6. **Quality Checklist**：最後的 checklist 用 `[ ]` 格式列出，讓 agent 能逐項自我審查。

手冊裡有一句點睛之語：**「沒有封面圖片的部落格文章不算完成」**（"No image generated" would be incomplete）。這種強硬的完成條件讓 agent 不會在生成文字後就「認為任務結束」。

---

### 4. `skills/social-media/SKILL.md`：Skill Primitive（社群媒體）

社群媒體 skill 的 frontmatter：

```yaml
---
name: social-media
description: 撰寫吸睛的社群媒體貼文、寫出開場鉤子（hook）、建議 hashtag、
  設計討論串（thread）結構，並產生搭配的圖片。當使用者要求撰寫 LinkedIn 貼文、
  推文、Twitter/X 討論串、社群媒體圖說、社群貼文，或把內容改寫為適合社群平台
  的版本時，使用此技能。
---
```

這個 skill 和 blog-post skill 的**架構相同**（Research First → Output Structure → Platform Guidelines → Image Generation → Quality Checklist），但**內容完全不同**。這種「結構一致、內容專門」的設計方便新增 skill——你可以依樣畫葫蘆新增 `skills/newsletter/SKILL.md`。

**平台差異化** 是這個 skill 的核心價值所在：

| 平台 | 輸出路徑 | 字數限制 | 特殊規定 |
|------|---------|---------|---------|
| LinkedIn | `linkedin/<slug>/post.md` | 1,300 字元 | 第一行當 hook、結尾 3-5 個 hashtag |
| Twitter/X | `tweets/<slug>/thread.md` | 每則 280 字元 | 討論串格式 `1/🧵`，每則最多 2 個 hashtag |

圖片 prompt 的撰寫指南也針對社群媒體的特性調整：強調「單一焦點、高對比、不放文字」——因為社群圖片在動態消息裡以小尺寸呈現，複雜構圖完全無效。

---

### 5. `content_writer.py`：入口腳本逐段解析

#### 段落一：Tool 定義

腳本最前面定義了三個 tool，都用 `@tool` 裝飾器包裝：

```python
@tool
def web_search(query: str, max_results: int = 5,
               topic: Literal["general", "news"] = "general") -> dict:
    """在網路上搜尋最新資訊。"""
    try:
        from tavily import TavilyClient
        api_key = os.environ.get("TAVILY_API_KEY")
        if not api_key:
            return {"error": "TAVILY_API_KEY not set"}
        client = TavilyClient(api_key=api_key)
        return client.search(query, max_results=max_results, topic=topic)
    except Exception as e:
        return {"error": f"Search failed: {e}"}
```

`web_search` 是給 **researcher subagent** 用的，主 agent 不能直接呼叫它。它把金鑰缺失和例外都包成 `{"error": ...}` dict 回傳，而不是拋例外——這讓 subagent 能繼續運行，並在 context 裡看到錯誤訊息。

```python
@tool
def generate_cover(prompt: str, slug: str) -> str:
    """為部落格文章產生封面圖片。"""
    from google import genai
    client = genai.Client()
    response = client.models.generate_content(
        model="gemini-2.5-flash-image",
        contents=[prompt],
    )
    for part in response.parts:
        if part.inline_data is not None:
            image = part.as_image()
            output_path = EXAMPLE_DIR / "blogs" / slug / "hero.png"
            output_path.parent.mkdir(parents=True, exist_ok=True)
            image.save(str(output_path))
            return f"Image saved to {output_path}"
    return "No image generated"
```

`generate_cover` 是給**主 agent** 用的。值得關注的細節：
- 使用 `gemini-2.5-flash-image` 模型（Gemini Imagen）
- 圖片以 `inline_data` 的形式回傳，透過 `part.as_image()` 轉成 Pillow Image 物件
- 儲存路徑固定為 `blogs/<slug>/hero.png`，路徑由 `slug` 參數決定
- `EXAMPLE_DIR / "blogs" / slug / "hero.png"` 用 `pathlib.Path` 構建，`mkdir(parents=True, exist_ok=True)` 確保目錄存在

`generate_social_image` 的邏輯與 `generate_cover` 幾乎相同，差別在於接受 `platform` 參數（`"linkedin"` 或 `"tweets"`），並把圖片存到對應的目錄。

#### 段落二：`load_subagents()`

```python
def load_subagents(config_path: Path) -> list:
    """從 YAML 載入子代理（subagent）定義並接上對應的工具。"""
    available_tools = {
        "web_search": web_search,
    }
    with open(config_path) as f:
        config = yaml.safe_load(f)
    subagents = []
    for name, spec in config.items():
        subagent = {
            "name": name,
            "description": spec["description"],
            "system_prompt": spec["system_prompt"],
        }
        if "model" in spec:
            subagent["model"] = spec["model"]
        if "tools" in spec:
            subagent["tools"] = [available_tools[t] for t in spec["tools"]]
        subagents.append(subagent)
    return subagents
```

這個輔助函式做了一件重要的事：把 YAML 裡的工具**字串名稱**（`"web_search"`）對應到 Python 函式物件。`available_tools` 字典就是這個映射表。

注意 `model` 和 `tools` 都用 `if ... in spec` 做選擇性讀取——這讓 YAML 裡的欄位可以省略，不會因為缺欄位而崩潰，設計上更容錯。

#### 段落三：`create_content_writer()`

```python
def create_content_writer():
    """建立一個由檔案系統上的檔案來設定的內容寫手代理。"""
    return create_deep_agent(
        memory=["./AGENTS.md"],
        skills=["./skills/"],
        tools=[generate_cover, generate_social_image],
        subagents=load_subagents(EXAMPLE_DIR / "subagents.yaml"),
        backend=FilesystemBackend(root_dir=EXAMPLE_DIR),
    )
```

五個參數，分工明確：

| 參數 | 傳入值 | 誰處理 | 作用 |
|------|-------|-------|------|
| `memory` | `["./AGENTS.md"]` | MemoryMiddleware（框架原生） | 注入 system prompt |
| `skills` | `["./skills/"]` | SkillsMiddleware（框架原生） | 按需載入 SKILL.md |
| `tools` | `[generate_cover, generate_social_image]` | 直接傳給 agent | 圖片生成能力 |
| `subagents` | `load_subagents(...)` 回傳的 list | 框架轉成 `task` tool | 委派研究任務 |
| `backend` | `FilesystemBackend(root_dir=EXAMPLE_DIR)` | 框架 | 提供 read/write file 等 tool |

`FilesystemBackend` 提供了 agent 存取本機檔案系統的能力（`read_file`、`write_file` 等），`root_dir` 限制了可以存取的根目錄。

#### 段落四：`main()` 與串流輸出

```python
async def main():
    agent = create_content_writer()
    display = AgentDisplay()
    async for chunk in agent.astream(
        {"messages": [("user", task)]},
        config={"configurable": {"thread_id": "content-writer-demo"}},
        stream_mode="values",
    ):
        if "messages" in chunk:
            messages = chunk["messages"]
            if len(messages) > display.printed_count:
                live.stop()
                for msg in messages[display.printed_count:]:
                    display.print_message(msg)
                display.printed_count = len(messages)
                live.start()
```

Agent 以 `astream` 方式執行，`stream_mode="values"` 代表每次新訊息加入 state 時都會推送完整的 messages list。`AgentDisplay` 類別負責把不同類型的訊息（HumanMessage、AIMessage、ToolMessage）渲染成不同樣式——tool call 顯示進行中的動作，`rich.live` 的 spinner 在等待期間旋轉。

---

## 🔍 Deep Agents 能力對照

這個範例示範了 Deep Agents 的以下核心能力：

| 能力 | 範例中的體現 |
|------|------------|
| Memory primitive | `AGENTS.md` 永遠注入 system prompt |
| Skill primitive | 兩個 SKILL.md 按需載入，frontmatter 作為匹配觸發器 |
| Subagent delegation | `researcher` 接受委派，結果落檔再由主 agent 讀取 |
| Filesystem backend | `read_file`/`write_file` 讓 agent 讀寫本機目錄 |
| Custom tools | `generate_cover`、`generate_social_image` 以 `@tool` 定義 |
| 非同步串流 | `astream` + `rich.live` 實現即時終端機 UI |

---

## ▶️ 怎麼跑

```bash
# 部落格文章
uv run python content_writer.py "Write a blog post about prompt engineering"

# LinkedIn 貼文
uv run python content_writer.py "Create a LinkedIn post about AI agents"

# Twitter 討論串
uv run python content_writer.py "Write a Twitter thread about the future of coding"
```

執行成功後，輸出會依內容類型分目錄存放：

```
blogs/prompt-engineering/
├── post.md      # 文章本文
└── hero.png     # AI 生成封面圖

linkedin/ai-agents/
├── post.md      # LinkedIn 貼文
└── image.png    # 搭配圖片

research/
└── prompt-engineering.md   # 研究筆記（可審查的中間產物）
```

---

## 🚀 延伸：如何客製化

### 新增內容類型

在 `skills/` 目錄下建立新的 skill，frontmatter 是關鍵：

```yaml
# skills/newsletter/SKILL.md
---
name: newsletter
description: 撰寫電子報，當使用者要求寫電子報、週報或訂閱信件時使用此技能。
---
# Newsletter Skill
...
```

Skill 只要放對目錄、有正確的 frontmatter，就會自動被 SkillsMiddleware 掃到。

### 新增 subagent

在 `subagents.yaml` 新增一段：

```yaml
editor:
  description: Review and improve drafted content
  model: anthropic:claude-haiku-4-5-20251001
  system_prompt: |
    You are an editor. Review the content for clarity and tone...
  tools: []
```

同時在 `AGENTS.md` 的 Research Requirements 段落加上「寫完後交給 `editor` 審稿」的指示，讓主 agent 知道何時委派。

### 新增 tool

在 `content_writer.py` 定義，再加進 `create_deep_agent` 的 `tools` 清單：

```python
@tool
def publish_to_cms(slug: str, platform: str) -> str:
    """把寫好的文章發布到 CMS。"""
    ...

return create_deep_agent(
    ...
    tools=[generate_cover, generate_social_image, publish_to_cms],
)
```

---

## 小結

`content-builder-agent` 最有價值的設計示範是「**把 agent 的行為邏輯從程式碼裡解放出來**」：

- **Memory**（`AGENTS.md`）定義 agent 的性格與長期原則，一直在場、不需要重複。
- **Skills**（`skills/*/SKILL.md`）是任務手冊，frontmatter 讓 middleware 按需載入，避免 context 膨脹。
- **Subagents**（`subagents.yaml` + `load_subagents()`）是可委派的專責下屬，每個都有自己的 model、system prompt 與 tool 集合。
- **`content_writer.py`** 只做一件事：把上述三者接起來，再加上圖片生成的 custom tool。

理解了這三個 primitive 的差異與載入時機，你就掌握了 Deep Agents 設計哲學的核心：**agent 的能力由磁碟上的檔案定義，程式碼只是膠水**。

---

*本文對應的範例版本：deepagents >= 0.6.8*
