> docs/deepagents/examples/downloading_agents/

# Agent 其實就是一個資料夾：下載 zip、解壓即跑的可攜式 AI 助理

**副標：用 content-writer.zip 逐段拆解 Deep Agents 的「agent-as-folder」哲學——無程式碼、無依賴、秒速啟動。**

---

## 引言：一個顛覆「部署」直覺的觀念

當大多數人談到「分享一個 AI agent」，腦海裡浮現的通常是：Docker image、conda 環境、pip 依賴清單、設定檔、API key 注入……一大堆儀式。

Deep Agents 提出了一個截然不同的答案：

> **Agent 就是一個資料夾。**

資料夾裡只有兩種東西——一個叫做 `AGENTS.md` 的記憶文字檔，加上一個 `skills/` 目錄裝著各種技能（skill）。沒有可執行程式碼，沒有執行期依賴。這表示你可以把它壓成 zip、貼到任何地方、讓任何人下載回去，解壓縮後直接就能跑。

這篇文章會用官方範例 `downloading_agents` 裡的 `content-writer.zip` 為主線，逐段拆解它的內部結構，讓你真正理解「agent-as-folder」這個概念是怎麼運作的。

---

## 為什麼這樣行得通？🤔

傳統軟體之所以難以移植，是因為它有**執行期邏輯**：需要特定版本的 runtime、需要 compile、需要 link library。但一個 Deep Agents agent 根本沒有這些東西。

它的本質是**純文字的指令集**：

- `AGENTS.md` 是 agent 的**記憶（memory）與人格**——用 Markdown 寫成，告訴 agent「你是誰」、「你怎麼說話」、「你有哪些原則」。
- `skills/*.SKILL.md` 是一個個**工作流程（workflow）的文字描述**——用 Markdown 寫成，告訴 agent「這個任務要怎麼一步步完成」。

因為所有內容都是純文字，所以：

| 特性 | 傳統程式碼 agent | Deep Agents agent-as-folder |
|---|---|---|
| 分享方式 | Git repo / Docker image | zip 檔 / 任意存放 |
| 執行前設定 | 安裝依賴 / 設定環境 | 解壓縮即可 |
| 修改門檻 | 需懂程式語言 | 直接編輯 Markdown |
| 版本控制 | git diff on code | git diff on text |
| 可攜性（portable） | 中低 | 極高 |

這也是為什麼 Deep Agents 把這種設計稱為 **portable**——它的可攜性不是透過容器化來實現，而是因為根本不需要容器化。

---

## 快速開始：從零到執行只需四個步驟 🚀

先安裝命令列工具：

```bash
uv tool install deepagents-cli==0.0.13
```

然後：

```bash
# 步驟 1：建立專案資料夾並初始化 Git
mkdir my-project && cd my-project && git init

# 步驟 2：下載 agent（一個 zip 檔）
curl -L https://raw.githubusercontent.com/langchain-ai/deepagents/main/examples/downloading_agents/content-writer.zip -o agent.zip

# 步驟 3：解壓縮到 .deepagents 目錄
unzip agent.zip -d .deepagents

# 步驟 4：執行
deepagents
```

逐步說明：

1. **`git init`** — Deep Agents 建議在 Git repo 裡運作，因為 agent 產生的內容（部落格文章、社群貼文等）會直接存進資料夾，版本控制讓你隨時能回溯。
2. **`curl` 下載** — 注意這裡下載的就是一個普通的 zip 檔，沒有特殊格式，放在 GitHub raw URL 上就能取用。
3. **解壓縮到 `.deepagents`** — deepagents-cli 的慣例是讀取當前目錄下的 `.deepagents/` 資料夾作為 agent 的根目錄。把 zip 解壓到這裡，CLI 就知道從哪裡載入 memory 和 skills。
4. **`deepagents`** — 一個指令啟動，立刻進入對話介面。

或者你嫌麻煩，官方也提供了**一行搞定版**：

```bash
git init && curl -L https://raw.githubusercontent.com/langchain-ai/deepagents/main/examples/downloading_agents/content-writer.zip -o agent.zip && unzip agent.zip -d .deepagents && rm agent.zip && deepagents
```

---

## ★ 逐段導讀：content-writer.zip 的真實內容

把 `content-writer.zip` 解壓開來，結構如下：

```
.deepagents/
├── AGENTS.md
└── skills/
    ├── blog-post/
    │   └── SKILL.md
    └── social-media/
        └── SKILL.md
```

三個純文字檔，構成一個完整、可執行的 content writer agent。下面逐一拆解。

---

### 📄 AGENTS.md：Agent 的記憶與人格

`AGENTS.md` 是整個 agent 的靈魂。它不是設定檔，是一份寫給 LLM 的「自我介紹 + 行為準則」。

```markdown
# Content Writer Agent

You are a content writer. Your job is to create engaging, informative content
that educates readers about technology topics.
```

**第一段**直接定義 agent 的身份（identity）：它是一個科技主題的內容寫作者。這句話會成為 LLM 系統提示的基礎。

```markdown
## Brand Voice

- **Professional but approachable**: Write like a knowledgeable colleague, not a textbook
- **Clear and direct**: Avoid jargon unless necessary; explain technical concepts simply
- **Engaging**: Use concrete examples, analogies, and stories to illustrate points
```

**Brand Voice 段落**定義「說話風格」。注意這裡完全沒有程式碼，只有三條 Markdown bullet。但這三條 bullet 會讓這個 agent 產出的文字風格，截然不同於沒有這段描述的 agent。這就是 **memory** 在發揮作用：它讓 agent 每次對話都「記得」自己應該怎麼說話。

```markdown
## Writing Standards

1. **Use active voice**: "The agent processes requests" not "Requests are processed by the agent"
2. **Lead with value**: Start with what matters to the reader, not background
3. **One idea per paragraph**: Keep paragraphs focused and scannable
4. **Concrete over abstract**: Use specific examples and case studies
5. **End with action**: Every piece should leave the reader knowing what to do next
```

**Writing Standards 段落**是更細緻的行為規範：主動語態、先說重點、每段一個觀念……這些規範在傳統軟體裡可能要用硬編碼的 prompt template 實作，但在 Deep Agents 裡只是幾行 Markdown，任何人都能直接改。

```markdown
## Formatting Guidelines

- Use headers (H2, H3) to break up long content
- Include code examples where relevant
- Add bullet points for lists of 3+ items
- Keep sentences under 25 words when possible
```

**Formatting Guidelines** 則控制輸出格式。整份 `AGENTS.md` 加起來不到 30 行，卻完整地定義了這個 agent 的「人格」、「說話方式」與「輸出標準」。

---

### 📄 skills/blog-post/SKILL.md：部落格寫作工作流

SKILL.md 的頂端是一段 YAML frontmatter：

```yaml
---
name: blog-post
description: Use this skill when writing long-form blog posts, tutorials, or educational articles
---
```

這兩個欄位讓 deepagents-cli 知道：
- `name`：這個 skill 的識別名稱，用於 CLI 的路由邏輯
- `description`：**觸發條件**——當使用者的輸入和這段描述語意接近時，agent 就會啟用這個 skill

frontmatter 之後是 Markdown 正文，定義具體的工作流程：

```markdown
## Output Structure

Save blog posts to:
```
blogs/<slug>/post.md
```
```

這段告訴 agent **輸出要存到哪裡**。注意這是用自然語言描述的檔案路徑慣例，不是程式碼。deepagents-cli 在執行時會把這個指令傳給 LLM，讓 LLM 在生成內容時同時決定要把檔案寫到 `blogs/ai-agents/post.md` 這樣的路徑。

接著是五段式文章結構：

```markdown
## Blog Post Structure

### 1. Hook (Opening)
- Start with a compelling question, statistic, or statement
- Make the reader want to continue

### 2. Context (The Problem)
- Explain why this topic matters
- Connect to the reader's experience

### 3. Main Content (The Solution)
- Break into 3-5 main sections with H2 headers
- Include code examples where helpful
...
```

這些不是模板字串，是給 LLM 的**流程指引**。LLM 在實際寫文章時，會把這五個步驟當成內部的 checklist，依序產出對應的內容段落。

最後是品質清單（Quality Checklist）：

```markdown
## Quality Checklist

Before finishing:
- [ ] Hook grabs attention in first 2 sentences
- [ ] Each section has a clear purpose
- [ ] Conclusion summarizes key points
- [ ] CTA tells reader what to do next
```

這個清單讓 agent 在「交付」之前先自我審查，本質上是在 skill 層面注入了一個輕量版的 self-reflection 機制。

---

### 📄 skills/social-media/SKILL.md：社群貼文工作流

同樣有 frontmatter：

```yaml
---
name: social-media
description: Use this skill when creating short-form social media content for LinkedIn or Twitter/X
---
```

這個 skill 同時涵蓋 LinkedIn 和 Twitter/X 兩個平台，並在正文裡分別定義了不同的格式規範：

```markdown
## Platform Guidelines

### LinkedIn
- 1,300 character limit
- First line is crucial - make it hook
- 3-5 hashtags at the end

### Twitter/X
- 280 character limit per tweet
- Threads for longer content (use 1/🧵 format)
- No more than 2 hashtags per tweet
```

這裡有個值得注意的設計：**兩個平台的格式差異，完全靠文字描述來區分**。當使用者說「幫我寫 LinkedIn 貼文」，agent 就會根據這份文字規範，自動套用 LinkedIn 的字數限制和 hashtag 慣例。

輸出路徑同樣有明確指定：

```markdown
**LinkedIn posts:** linkedin/<slug>/post.md
**Twitter/X threads:** tweets/<slug>/thread.md
```

---

## 這個範例展示了哪些 Deep Agents 核心能力？ 🎯

| 能力 | 在 content-writer.zip 裡的體現 |
|---|---|
| **Agent-as-folder** | 整個 agent 就是三個 Markdown 檔 |
| **Memory（記憶）** | `AGENTS.md` 讓 agent 跨對話保持一致的品牌聲音與寫作標準 |
| **Skills（技能）** | 兩個 SKILL.md 分別處理長文與社群貼文兩種截然不同的任務 |
| **零程式碼可攜（portable）** | zip 下載、解壓即跑，不需要安裝任何語言 runtime 或 library |
| **自然語言路由** | SKILL.md 的 `description` 欄位讓 CLI 根據語意自動選擇正確的 skill |

---

## 怎麼跑：一次對話示範

啟動後，你可以直接用自然語言對 agent 說：

```
寫一篇關於 RAG（Retrieval-Augmented Generation）的部落格文章
```

deepagents-cli 會：
1. 根據語意比對，觸發 `blog-post` skill（因為 description 說「long-form blog posts, tutorials」）
2. 依照 `AGENTS.md` 的品牌聲音規範生成文字
3. 依照 skill 的五段式結構組織文章
4. 把最終結果存到 `blogs/rag/post.md`

或者你說：

```
幫我把這篇文章改寫成 LinkedIn 貼文
```

這次就會觸發 `social-media` skill，自動套用 LinkedIn 的格式規範，存到 `linkedin/rag/post.md`。

---

## 延伸：怎麼把自己的 agent 打包成 zip 分享？ 📦

反過來，如果你已經在某個 git repo 裡調整出一個很好用的 agent，想分享給同事或社群，過程非常直觀：

**一、確認 `.deepagents/` 的結構正確**

```
.deepagents/
├── AGENTS.md          ← 把你的 memory/指令寫在這裡
└── skills/
    └── your-skill/
        └── SKILL.md   ← frontmatter + 工作流程
```

**二、打包成 zip**

```bash
cd .deepagents
zip -r ../my-agent.zip .
```

注意：zip 的根目錄直接是 `AGENTS.md` 和 `skills/`，不要多包一層資料夾，這樣別人 `unzip agent.zip -d .deepagents` 才能正確展開。

**三、分享**

把 zip 丟到 GitHub、Google Drive、Slack，或任何你習慣的地方。收到的人只需要：

```bash
curl -L <你的zip網址> -o agent.zip && unzip agent.zip -d .deepagents && deepagents
```

就能立刻跑起你設計的 agent，完全不需要你陪同說明環境設定。

---

## 小結：文字就是最好的介面

`downloading_agents` 範例用極簡的方式示範了一件事：**當 agent 的本質是「文字指令的集合」，分享 agent 這件事就會變得和分享一個 PDF 一樣簡單**。

- 你不需要打包執行環境
- 你不需要寫部署腳本
- 你不需要對方懂任何程式語言

三個 Markdown 檔、一個 zip、一行 `deepagents`——這就是 Deep Agents 對「agent 應該長什麼樣子」這個問題的答案。

而 `content-writer.zip` 裡 `AGENTS.md` 的品牌聲音定義、`blog-post/SKILL.md` 的五段式結構、`social-media/SKILL.md` 的雙平台格式規範，每一行都在示範：**你能用多自然的語言去定義一個 agent 的行為，它就能用多一致的方式把那個行為帶到任何地方。**

這就是 portable 的真正意義。

---

*本文基於 Deep Agents 官方範例 `downloading_agents`，所有程式碼與 Markdown 片段均取自 `content-writer.zip` 的實際內容。*
