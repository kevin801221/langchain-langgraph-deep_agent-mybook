> 原始路徑：`docs/deepagents/examples/deploy-coding-agent/`

# 用 `deepagents deploy` 打造自主 Coding Agent：從設定檔到 sandbox 的完整導讀

**副標：Plan → Implement → Review → Deliver——四個 phase 全部由 AGENTS.md 驅動，零行 Python 也能跑起一個會寫 code 的 agent。**

---

## 引言

大多數「AI 寫程式」的展示，都是把 LLM 當成一個比較聰明的自動完成工具：你貼一段需求，它吐一段 code，你再手動測試、手動 commit。這個流程裡，人仍然是最外層的 orchestrator。

`deploy-coding-agent` 這個範例要示範的，是另一個層次的自主性：**你只要給它一句自然語言任務，它會自己探索程式碼庫、擬計畫、改檔案、跑測試、修 bug、最後 commit 並回報——全程在 LangSmith sandbox 裡執行，不碰你的本機環境。**

更值得注意的是，整個 agent 的「大腦」沒有一行 LangGraph Python 或自訂 orchestration 程式碼。它是一個**宣告式（declarative）部署**：行為完全由 `AGENTS.md`（系統提示）、`agent.json`（部署設定）、以及三個 skill 的 `SKILL.md` 所定義。理解這些設定檔的結構與互動方式，就是理解 Deep Agents 宣告式架構的核心。

---

## 全貌：Plan → Implement → Review → Deliver

在讀個別檔案之前，先建立一個全局印象會讓後面的導讀更容易定位。

```
deploy-coding-agent/
├── AGENTS.md                  # agent 的系統提示：四個 phase 的工作流程
├── agent.json                 # 部署設定：名稱、model
├── .env.example               # 環境變數範本
└── skills/
    ├── code-review/
    │   ├── SKILL.md           # code review checklist + 使用說明
    │   └── lint_check.py      # 可執行的 lint 輔助腳本
    ├── coding-prefs/
    │   └── SKILL.md           # 讀寫使用者偏好的指引
    └── planning/
        └── SKILL.md           # 任務拆解與風險評估的指引
```

這個 agent 的工作流程分成四個 phase，全部定義在 `AGENTS.md`：

| Phase | 名稱 | 核心動作 |
|-------|------|---------|
| 1 | **Plan** | 探索 repo 結構、用 `grep`/`glob` 找相關檔案、寫 todo list |
| 2 | **Implement** | 依計畫改 code、每步跑測試、debug 修好再繼續 |
| 3 | **Review** | 跑完整測試套件、執行 linter、逐一讀過所有修改的檔案 |
| 4 | **Deliver** | 寫 commit message commit 進去、回報決策摘要 |

整個過程在 **LangSmith sandbox** 裡執行，agent 擁有完整的 shell 權限，可以直接操作 git、pytest、ruff 等工具。

---

## 環境與部署

### 兩個必要的環境變數

`.env.example` 非常簡單，只有兩行：

```
ANTHROPIC_API_KEY=
LANGSMITH_API_KEY=
```

- `ANTHROPIC_API_KEY`：呼叫 Claude 模型所需。
- `LANGSMITH_API_KEY`：兩個用途都需要——一是執行 `deepagents deploy` 把 agent 上傳到 LangSmith，二是讓 agent 在 LangSmith sandbox 裡執行時取得沙箱資源。

使用方式是把 `.env.example` 複製成 `.env`，填入兩組金鑰後執行：

```bash
deepagents deploy
```

### agent.json：部署的靜態設定

```json
{
  "name": "deepagents-deploy-coding-agent",
  "runtime": {
    "model": {"model_id": "anthropic:claude-sonnet-4-5"}
  }
}
```

這個設定檔定義了兩件事：

1. **`name`**：agent 在 LangSmith 上的識別名稱，也是部署後 endpoint 的一部分。
2. **`runtime.model.model_id`**：使用 `anthropic:claude-sonnet-4-5`，格式是 `<provider>:<model-name>`。

注意這裡沒有任何 workflow 邏輯、tool 定義或 skill 綁定——這些全部由 AGENTS.md 和 skills 目錄決定。`agent.json` 只負責「用哪個模型、叫什麼名字」這件事，夠精簡。

---

## ★ 核心檔案逐段導讀 ★

### AGENTS.md：agent 的「大腦基底」

`AGENTS.md` 在 Deep Agents 框架裡扮演 **system prompt** 的角色，在每一輪對話開始時被載入，是 agent 行為的最根本定義。

#### 開場人設

```markdown
# Coding Agent

你是一位資深軟體工程師，能夠自主（autonomously）解決各種程式設計任務。
你在一個沙箱（sandboxed）環境中工作，並擁有完整的 shell 權限。
```

這幾行做了三件事：定義角色（資深工程師）、聲明環境（sandbox）、聲明能力邊界（完整 shell 權限）。告訴 agent「你可以自由使用 shell」，是讓它敢放手執行 `execute("python -m pytest")` 的前提。

#### Phase 1：Plan

```markdown
- 使用 grep 與 glob 找出相關的檔案
- 使用 write_todos 寫出一份逐步的實作計畫
- 如果任務描述含糊不清，先請求澄清再動手
```

`write_todos` 是 Deep Agents 框架內建的 tool，讓 agent 把計畫以結構化清單存起來，後續 phase 可以逐項更新。這個設計讓 agent 的「思考過程」可被觀察、可被中斷後繼續。最後一條「先請求澄清」是一個重要的防呆機制，避免 agent 在需求不清時盲目動工。

#### Phase 2：Implement

```markdown
- 每完成一個重要變更後就跑一次測試
- 如果測試失敗，先除錯並修好，再繼續下一步
- 隨著步驟完成，持續更新你的 todo 清單
```

「每步跑測試」而不是「最後才跑測試」——這個指令讓 agent 在小範圍內維持測試通過，避免累積太多問題到最後爆炸。`todo 清單` 的即時更新讓外部可以追蹤進度。

#### Phase 3：Review

```markdown
- 執行完整的測試套件：execute("python -m pytest")
- 如果專案有設定 linter，就執行它：execute("ruff check .")
- 審查你自己的變更：把每一個被修改過的檔案從頭到尾讀過一遍
- 如果有任何不對的地方，回到 Phase 2
```

這段最值得注意的是「**回到 Phase 2**」——這是一個顯式的 loop 結構，用自然語言描述，而不是程式碼裡的 `while` 迴圈。Agent 被授權自我修正，直到所有檢查都通過為止。

#### Phase 4：Deliver

```markdown
- 用一句清楚、具描述性的 commit message 把變更 commit 進去
- 摘要說明你做了什麼，以及過程中做了哪些決策
```

Commit message 的品質要求被明確寫進指令，確保 agent 留下可讀的 git 歷史。

#### Coding Standards 與 Common Patterns

```markdown
## Common Patterns
- 尋找檔案：在閱讀檔案前，先用 glob("**/*.py") 或 grep("pattern") 找出目標
- 測試變更：每次編輯後都務必跑測試，不要假設程式一定正確
- Shell 指令：用 execute() 來執行 git、pytest、linters、builds 等指令
```

`glob()`、`grep()`、`execute()` 這三個都是 Deep Agents sandbox 提供的內建 tool。把這些 patterns 列在 AGENTS.md 裡，等於是給 agent 一份「你有哪些工具、何時用哪個」的速查手冊，減少它在 tool selection 上的猶豫。

#### Subagents

```markdown
## Subagents
遇到複雜任務時，把工作委派（delegate）給子代理（subagents）：
- 用 task(subagent_type="researcher") 來研究 API、文件或既有模式
- 用 task(subagent_type="general-purpose") 來處理彼此獨立的子任務
```

`task()` 是 Deep Agents 的 subagent 委派 tool。當任務夠複雜時，主 agent 可以把子任務交給專門型別的子 agent 並行處理。這段指令讓 agent 知道「什麼時候該自己做、什麼時候該分工」。

---

### skills/planning/SKILL.md：任務拆解的 SOP

```yaml
---
name: planning
description: 把一項程式設計任務拆解成結構化的實作計畫，包含清楚的步驟、相關檔案的辨識，以及風險評估。
---
```

SKILL.md 最前面是 **YAML frontmatter**，`name` 和 `description` 是框架用來識別這個 skill 的 metadata。`description` 同時也是 agent 在決定「要不要載入這個 skill」時看到的說明——寫得夠清楚，agent 才能在對的時機主動引用。

Skill 的正文定義了五個步驟：理解任務、探索程式碼庫、辨識相關檔案、撰寫計畫、評估風險。其中最關鍵的是第四步：

```markdown
### 4. Write the Plan（撰寫計畫）
使用 write_todos 建立一份結構化的計畫：

write_todos([
    "1. <specific change in specific file>",
    "2. <next specific change>",
    "3. Write tests for <feature>",
    "4. Run test suite and fix failures",
    "5. Review all changes"
])
```

這裡用具體的 `write_todos` 呼叫範例，示範 todo item 的粒度。規則是「計畫應包含 3 到 10 個步驟，每個步驟明確到不需要再額外規劃就能直接執行」。

Skills 在 Deep Agents 裡是**按需載入**的：agent 不會在啟動時一次把所有 skill 塞進 context，而是在需要時讀取對應的 SKILL.md。這和 AGENTS.md 的「永遠存在」形成對比——AGENTS.md 是 memory（常駐記憶），skills 是按需查閱的工具書。

---

### skills/coding-prefs/SKILL.md：跨對話的使用者偏好持久化

```yaml
---
name: coding-prefs
description: 在做出非瑣碎的風格決策前，先從 /memory/coding-prefs.md 讀取使用者的程式撰寫偏好；
             當使用者給出長期適用的回饋時，再把新的偏好附加進去。
---
```

這個 skill 示範了 Deep Agents 的 **memory 機制**。`/memory/coding-prefs.md` 是一個 **user-scoped** 的持久化檔案：

- 每個 user 有自己獨立的一份，不同 user 之間不會互相影響。
- Agent 在做風格決策前（例如要不要加 docstring、用 pytest 還是 unittest）應該先讀這份檔案。
- 當 user 給出長期適用的回饋時，應該把新偏好 **附加**（append）進去，而不是整個覆蓋。

```markdown
## When to Write
每當使用者給出「應該套用到日後工作」的回饋時，就附加一筆新紀錄：
- 「除非我要求，不然不要加 docstring」→ 存起來
- 「比起 unittest，我偏好 pytest」→ 存起來

## How to Write
先讀取這個檔案（它可能還不存在），再附加內容。不要整個覆蓋——偏好是隨時間累積的。
```

這個設計讓 agent 具備跨對話的學習能力：它不是每次都從零開始猜你的偏好，而是隨著合作時間增長，越來越了解你的工作風格。

---

### skills/code-review/SKILL.md：結構化審查清單

SKILL.md 的正文定義了四個審查維度：

- **Correctness**：是否真的解決了 issue、有沒有副作用、邊界情況。
- **Code Quality**：風格一致性、命名、沒有死碼或未處理的 TODO。
- **Tests**：新功能有測試、既有測試仍然通過、測試夠健壯。
- **Safety**：沒有 hardcoded 密鑰、輸入驗證、SQL/XSS/command injection 風險。

其中 Process 段落定義了七步執行順序：

```markdown
## Process
1. 把每一個被修改過的檔案從頭到尾讀過（不要只看 diff）
2. 執行測試套件：execute("python -m pytest -v")
3. 若有 linter 就執行：execute("ruff check .")
4. 執行內附的 lint 檢查：execute("python /skills/code-review/lint_check.py .")
5. 逐項對照審查清單（review checklist）檢查
6. 若發現任何問題，修正後重新審查
7. 全部通過後，審查即完成
```

第 4 步特別引用了 skill 目錄裡的輔助腳本 `lint_check.py`，透過 `execute()` 呼叫它。這示範了 skill 不只是純文字指引，**也可以附帶可執行的 helper script**。

---

### skills/code-review/lint_check.py：逐段解釋

這是整個範例裡唯一一支 Python 腳本，也是「skill 可以附帶工具」的具體實作。

#### 模組文件字串與 import

```python
#!/usr/bin/env python3
"""提供給 code-review 技能使用的快速 lint 檢查輔助工具。

掃描 Python 檔案，找出完整 linter 可能漏掉、或值得在程式碼審查時提出來的
常見問題：
- 缺少模組層級 docstring 的檔案
- 超過 50 行的函式
- 裸露的 except: 子句
"""

import ast
import sys
from pathlib import Path
```

只用標準函式庫（`ast`、`sys`、`pathlib`），**零外部依賴**，確保在任何 Python 環境都能直接執行。選用 `ast` 模組做靜態分析，而不是用 `re` 搜尋文字，意味著分析是語法層面的，不會被格式排版干擾。

#### `check_file(path)` — 單一檔案的分析核心

```python
def check_file(path: Path) -> list[str]:
    """回傳單一 Python 檔案的警告清單。"""
    warnings: list[str] = []
    try:
        source = path.read_text(encoding="utf-8")
    except Exception as exc:
        return [f"{path}: could not read ({exc})"]

    try:
        tree = ast.parse(source, filename=str(path))
    except SyntaxError as exc:
        return [f"{path}:{exc.lineno}: syntax error: {exc.msg}"]
```

第一段處理兩種「連分析都做不了」的失敗：讀取失敗（權限問題、binary 檔案）和語法錯誤。都以「警告字串」形式回傳，不丟 exception——設計目標是「掃完所有檔案、印出所有問題」，而不是「遇到第一個問題就停下來」。

```python
    # 檢查是否缺少模組層級 docstring
    if not ast.get_docstring(tree):
        warnings.append(f"{path}:1: missing module docstring")
```

`ast.get_docstring(tree)` 直接讀 AST 樹的第一個表達式，判斷是否為字串字面量（即 docstring）。比正則表達式更可靠，不會被縮排或引號類型混淆。

```python
    for node in ast.walk(tree):
        # 過長的函式
        if isinstance(node, (ast.FunctionDef, ast.AsyncFunctionDef)):
            length = (node.end_lineno or node.lineno) - node.lineno + 1
            if length > 50:
                warnings.append(
                    f"{path}:{node.lineno}: function '{node.name}' is {length} lines long (>50)"
                )

        # 裸露的 except
        if isinstance(node, ast.ExceptHandler) and node.type is None:
            warnings.append(f"{path}:{node.lineno}: bare 'except:' clause")
```

`ast.walk(tree)` 遞迴走訪整個 AST，同時偵測兩種問題：

- **函式長度**：同時處理 `FunctionDef`（同步）和 `AsyncFunctionDef`（async），閾值 50 行。`node.end_lineno` 是 Python 3.8+ 才有的屬性，加上 `or node.lineno` 作為 fallback。
- **裸露 except**：`ast.ExceptHandler` 的 `type` 為 `None` 代表 `except:` 而非 `except SomeError:`。裸露 except 會吞掉所有例外包括 `KeyboardInterrupt`，是常見的 anti-pattern。

#### `main(paths)` — 入口與批次處理

```python
def main(paths: list[str]) -> int:
    targets = [Path(p) for p in paths] if paths else [Path(".")]
    all_warnings: list[str] = []

    for target in targets:
        if target.is_file() and target.suffix == ".py":
            all_warnings.extend(check_file(target))
        elif target.is_dir():
            for py_file in sorted(target.rglob("*.py")):
                all_warnings.extend(check_file(py_file))

    for w in all_warnings:
        print(w)

    if all_warnings:
        print(f"\n{len(all_warnings)} warning(s) found.")
        return 1

    print("No warnings found.")
    return 0
```

幾個設計細節值得注意：

- **有無路徑參數都能用**：`paths` 為空時預設掃目前目錄，agent 呼叫 `execute("python /skills/code-review/lint_check.py .")` 即可。
- **同時支援檔案與目錄**：對目錄用 `rglob("*.py")` 遞迴搜尋，對單一 `.py` 檔直接分析。
- **回傳值作為 exit code**：有警告時回傳 `1`（非零），讓 `execute()` 的呼叫方可以根據 exit code 判斷是否需要修正。這和 `ruff`、`pytest` 的慣例一致。

---

## 這個範例示範了哪些 Deep Agents 能力

整理一下這個範例在架構層面揭示的 Deep Agents 設計：

**1. AGENTS.md 作為永久記憶（persistent memory）**
AGENTS.md 在每次對話都被載入，是 agent 行為的「基因」。四個 phase 的工作流程、coding standards、tool 使用模式全部寫在這裡，不需要任何程式碼。

**2. Skills 的按需載入（on-demand loading）**
三個 skill（planning、coding-prefs、code-review）不是一次全部載入，而是 agent 在需要時主動讀取對應的 SKILL.md。這讓 context window 保持精簡，也讓 skill 可以獨立演進。

**3. SKILL.md 的 progressive disclosure**
Frontmatter 的 `description` 是第一層（讓 agent 決定要不要讀這個 skill），SKILL.md 正文是第二層（讓 agent 知道怎麼用這個 skill），helper script（如 `lint_check.py`）是第三層（可執行的工具）。三層遞進，信息密度逐漸增加。

**4. User-scoped memory 與 coding-prefs**
`/memory/coding-prefs.md` 讓 agent 跨對話記住個別 user 的偏好，實現個人化的長期協作關係。

**5. Sandbox + shell 作為執行環境**
Agent 用 `execute()` 直接呼叫 `git`、`pytest`、`ruff`，sandbox 提供安全隔離。不需要 function calling schema，也不需要自己實作工具，shell 就是 interface。

**6. 宣告式部署（declarative deployment）**
整個 agent 沒有一行 LangGraph 或 Python orchestration 程式碼。行為由文字定義，部署由 `agent.json` 定義，執行由 `deepagents deploy` 完成。

---

## 怎麼用：可以試試看的任務

部署完成後，在 LangSmith 介面開啟這個 agent，丟以下任務給它：

```
Add a function that reverses a string and write a test for it
```
→ 觀察它如何 Plan（探索 repo、寫 todo）→ Implement（新增函式和測試）→ Review（跑 pytest + lint_check.py）→ Deliver（commit + 摘要）

```
Find all TODO comments in the repo and create a summary
```
→ 這個任務不需要改 code，觀察 agent 如何用 `grep` 搜集資訊再整理成報告。

```
Refactor the main module to use dataclasses
```
→ 這是一個風格型重構任務，觀察 agent 是否會先讀 `coding-prefs` skill 確認你的偏好。

也可以用 LangGraph SDK 程式化呼叫：

```python
from langgraph_sdk import get_client

client = get_client(url="https://<your-deployment-url>")
thread = await client.threads.create()

async for chunk in client.runs.stream(
    thread["thread_id"], "agent",
    input={"messages": [{"role": "user", "content": "Add a hello_world function and test it"}]},
    stream_mode="messages",
):
    print(chunk.data, end="", flush=True)
```

Deployment URL 可以在 LangSmith 的 **Deployments** 頁面找到。

---

## 延伸：可以如何擴展

這個範例提供了一個最小可行的 coding agent，有幾個自然的擴展方向：

- **加 MCP server**：用 `deepagents mcp-servers add --url <url>` 在 workspace 層級註冊外部工具（例如 LangChain 文件的 MCP server），再在 `tools.json` 引用，讓 agent 能查閱外部文件。
- **擴充 lint_check.py**：目前只檢查三種問題（缺 docstring、長函式、裸露 except），可以加入更多專案特定的規則。
- **豐富 coding-prefs**：隨著和 agent 合作，把偏好逐漸寫進 `/memory/coding-prefs.md`，讓它越來越了解你的風格。
- **換模型**：修改 `agent.json` 的 `model_id`，試試其他支援的模型。

---

## 小結

`deploy-coding-agent` 是一個展示 Deep Agents **宣告式架構**上限的範例。它的精髓在於：

- **AGENTS.md** 定義了 agent 的完整工作流程，四個 phase 用自然語言寫成，比程式碼更容易讀懂、更容易修改。
- **agent.json** 把部署設定縮到最小（名稱 + 模型），部署一個指令搞定。
- **Skills** 示範了三種不同的 skill 用途：planning（任務拆解 SOP）、coding-prefs（跨對話個人化）、code-review（帶可執行工具的審查清單）。
- **lint_check.py** 是一個實用的小工具，展示 skill 可以不只是文字，還可以附帶實際跑的腳本。

如果你想讓某個自動化程式設計流程跑在雲端、不碰本機環境、又想完全控制 agent 的行為準則，這個範例的結構值得直接拿來改。
