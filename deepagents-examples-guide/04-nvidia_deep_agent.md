> 原始範例路徑：`docs/deepagents/examples/nvidia_deep_agent/`

# 把 GPU 塞進 Agent：NVIDIA Deep Agent 的多模型架構全解析 🚀

### 一個 frontier model 當指揮、Nemotron Super 做研究、GPU sandbox 跑運算——這就是真正的多模型 Deep Agent

---

## 引言：當 Agent 需要同時「思考」又「大量運算」

多數 LLM 應用只有一個模型、一個角色。但現實的研究與資料分析任務往往需要兩件同時很難做好的事：**深度推理**（設計分析計畫、彙整結論）和**大量重複勞動**（爬幾十個網頁、對百萬列資料集跑統計）。

NVIDIA Deep Agent 正面回答這個問題：**讓對的模型做對的事，再用 GPU sandbox 扛運算量。** 它結合了 LangGraph 的 Deep Agents 框架、NVIDIA Nemotron Super、Modal GPU sandbox 和 NVIDIA RAPIDS（cuDF/cuML），是目前官方範例中架構最複雜、元件最豐富的一個。

讀完這篇文章，你會知道：

- `create_deep_agent` 怎麼把多模型、subagent、skills、memory 和 backend 組成一個可部署的 agent
- `backend.py` 怎麼在執行階段建立 Modal GPU/CPU sandbox 並上傳 skills 與 memory
- `tools.py` 的 `tavily_search` 如何實作搜尋＋全文抓取
- `prompts.py` 的三套 prompt 各自扮演什麼角色
- `AGENTS.md` 的自我改進機制是怎麼運作的
- 四個 `SKILL.md` 如何以「漸進式揭露」的方式教會 agent 使用 RAPIDS

---

## 全貌：三層架構

在進入程式碼之前，先把整體架構放在腦海裡。這個 agent 有三個層次：

```
create_deep_agent  (orchestrator：frontier model)
    │
    ├── researcher-agent          ← Nemotron Super 驅動
    │       tavily_search（網路搜尋＋全文抓取）
    │
    ├── data-processor-agent      ← frontier model 驅動
    │       tavily_search（補充搜尋）
    │       skills: /skills/（cuDF / cuML / 視覺化 / 文件處理）
    │
    ├── memory/
    │       AGENTS.md（自我改進的持久化指令）
    │
    └── backend: Modal Sandbox
            GPU 模式：NVIDIA RAPIDS image + A10G GPU
            CPU 模式：debian_slim + pandas/scipy
            skills + memory 在沙箱建立時上傳
```

**第一層（orchestrator）**：frontier model 負責規劃、拆解任務、委派工作，最後彙整輸出。它不做粗重的工作。

**第二層（subagents）**：兩個 subagent 各司其職——`researcher-agent` 以 Nemotron Super 做網路研究；`data-processor-agent` 以 frontier model 在 GPU sandbox 上寫並執行 Python 腳本。

**第三層（支撐元件）**：skills（SKILL.md 格式的操作手冊）、memory（AGENTS.md 格式的自我改進記憶），以及可在執行階段切換 GPU/CPU 的 Modal sandbox backend。

---

## 環境與安裝

```toml
# pyproject.toml（節錄）
[project]
name = "nemotron-deep-agent"
requires-python = ">=3.11"
dependencies = [
    "deepagents>=0.6.8",
    "langchain-nvidia-ai-endpoints>=1.4.1",
    "modal>=1.4.3",
    "langchain-modal>=0.0.5",
    "tavily-python>=0.7.25",
    "markdownify>=1.2.2",
    ...
]
```

幾個關鍵依賴的作用：

| 套件 | 用途 |
|------|------|
| `deepagents` | `create_deep_agent` 框架本體 |
| `langchain-nvidia-ai-endpoints` | `ChatNVIDIA` 接入 NIM 端點 |
| `modal` / `langchain-modal` | GPU sandbox 建立與控制 |
| `tavily-python` | 網路搜尋 API |
| `markdownify` | 將抓取到的 HTML 轉成 markdown |

`langgraph.json` 只有三行，指定 entry point 為 `./src/agent.py:agent`，接著 `uv run langgraph dev --allow-blocking` 就能啟動。

---

## ★ 核心程式碼逐段導讀

### 1. `agent.py`：把一切組裝起來

這是整個 agent 的入口。它做三件事：定義模型、定義 subagents、呼叫 `create_deep_agent`。

**Context Schema：執行階段的沙箱選擇**

```python
class Context(TypedDict, total=False):
    sandbox_type: Literal["gpu", "cpu"]
```

`Context` 是一個 `TypedDict`，讓呼叫端在 invoke 時傳入 `context={"sandbox_type": "gpu"}` 或 `"cpu"`。`total=False` 表示這個欄位是選填的，預設走 GPU 路徑。

**兩個模型的定位**

```python
frontier_model = init_chat_model(
    os.environ.get("ORCHESTRATOR_MODEL", "anthropic:claude-sonnet-4-6")
)

nemotron_super = ChatNVIDIA(
    model="nvidia/nemotron-3-super-120b-a12b"
)
```

`init_chat_model` 使用 `"provider:model_name"` 格式，讓你不需要改程式碼就能換供應商（只要改環境變數 `ORCHESTRATOR_MODEL`）。`ChatNVIDIA` 則透過 NVIDIA NIM 端點，接入與 OpenAI API 相容的 Nemotron Super 120B 模型。

**Subagent 定義：字典格式**

```python
researcher_sub_agent = {
    "name": "researcher-agent",
    "description": (
        "Delegate research to this agent. Conducts web searches and gathers "
        "information on a topic. Give one focused research topic at a time."
    ),
    "system_prompt": RESEARCHER_INSTRUCTIONS.format(date=current_date),
    "tools": tools,
    "model": nemotron_super,
}

data_processor_sub_agent = {
    "name": "data-processor-agent",
    "description": "Delegate data analysis, ML, visualization, and document processing tasks...",
    "system_prompt": DATA_PROCESSOR_INSTRUCTIONS.format(date=current_date),
    "tools": tools,
    "model": frontier_model,
    "skills": ["/skills/"]
    # "interrupt_on": {"execute": True}  # human in the loop（選填）
}
```

注意兩個 subagent 的差異：
- `researcher-agent` 用 `nemotron_super`（速度快、成本低，適合量大的研究工作）
- `data-processor-agent` 用 `frontier_model`（推理能力強，適合寫程式碼）
- 只有 `data-processor-agent` 有 `"skills": ["/skills/"]`，告訴框架要把 skills 目錄掛載進去
- `interrupt_on` 被注解掉了，但開啟後可以在程式碼執行前要求人工確認

**最終組裝**

```python
agent = create_deep_agent(
    model=frontier_model,
    tools=tools,
    system_prompt=ORCHESTRATOR_INSTRUCTIONS.format(date=current_date),
    subagents=[researcher_sub_agent, data_processor_sub_agent],
    memory=["/memory/AGENTS.md"],
    backend=create_backend,
    context_schema=Context
)
```

`create_deep_agent` 接收六個關鍵參數：
- `model`：orchestrator 使用的模型
- `subagents`：list 格式的 subagent 定義
- `memory`：持久化記憶的路徑（sandbox 內的虛擬路徑）
- `backend`：一個 factory 函式，框架會在需要時呼叫它來建立 sandbox
- `context_schema`：告訴框架這個 agent 接受哪些執行階段參數

---

### 2. `backend.py`：Modal Sandbox 的建立與種子上傳

這是最具工程感的一個檔案。它做兩件事：定義沙箱的映像檔，以及在沙箱首次建立時把 skills 和 memory 上傳進去。

**兩種映像檔**

```python
rapids_image = (
    modal.Image.from_registry("nvcr.io/nvidia/rapidsai/base:25.02-cuda12.8-py3.12")
    .pip_install("numba-cuda>=0.28", "matplotlib", "seaborn")
)
cpu_image = modal.Image.debian_slim().pip_install(
    "pandas", "numpy", "scipy", "scikit-learn", "matplotlib", "seaborn"
)
```

`rapids_image` 從 NVIDIA Container Registry 拉取 RAPIDS 25.02 的基礎映像（已內含 cuDF、cuML），再補裝修正版 `numba-cuda>=0.28`（原版 0.2.0 有 IndexError bug）。`cpu_image` 則是輕量的 debian_slim，只裝 pandas 系列——沒有 GPU 也能跑，cuDF 會在 SKILL.md 的 boilerplate 中自動 fallback 回 pandas。

**`_seed_sandbox`：把本機檔案上傳到沙箱**

```python
def _seed_sandbox(backend: ModalSandbox) -> None:
    files: list[tuple[str, bytes]] = []
    for skill_dir in sorted(SKILLS_DIR.iterdir()):
        if not skill_dir.is_dir():
            continue
        skill_md = skill_dir / "SKILL.md"
        if not skill_md.exists():
            continue
        files.append(
            (f"/skills/{skill_dir.name}/SKILL.md", skill_md.read_bytes())
        )
    if MEMORY_FILE.exists():
        files.append(("/memory/AGENTS.md", MEMORY_FILE.read_bytes()))
    dirs = sorted({str(Path(p).parent) for p, _ in files})
    backend.execute(f"mkdir -p {' '.join(dirs)}")
    backend.upload_files(files)
```

這個函式把 `skills/` 底下所有子目錄的 `SKILL.md` 加上 `src/AGENTS.md`，打包成 `(沙箱路徑, bytes)` 的 list，先在沙箱內用 `mkdir -p` 建好目錄，再一次性 `upload_files`。這是 agent 能在沙箱內讀取 skills 和 memory 的關鍵步驟。

**`create_backend`：Factory 函式**

```python
def create_backend(runtime):
    ctx = runtime.context or {}
    sandbox_type = ctx.get("sandbox_type", "gpu")
    use_gpu = sandbox_type == "gpu"
    sandbox_name = f"{MODAL_SANDBOX_NAME}-{sandbox_type}"

    created = False
    try:
        sandbox = modal.Sandbox.from_name(MODAL_SANDBOX_NAME, sandbox_name)
    except modal.exception.NotFoundError:
        create_kwargs = dict(
            app=modal_app,
            workdir="/workspace",
            name=sandbox_name,
            timeout=3600,
            idle_timeout=1800,
        )
        if use_gpu:
            create_kwargs["image"] = rapids_image
            create_kwargs["gpu"] = "A10G"
        else:
            create_kwargs["image"] = cpu_image
        sandbox = modal.Sandbox.create(**create_kwargs)
        created = True

    backend = ModalSandbox(sandbox=sandbox)
    if created:
        _seed_sandbox(backend)
    return backend
```

這個函式是整個 backend 的核心邏輯。它先從 `runtime.context` 讀取 `sandbox_type`，再嘗試用 `modal.Sandbox.from_name` 找到已存在的沙箱（避免重複建立）。找不到時才呼叫 `modal.Sandbox.create`，根據模式選擇映像檔和 GPU 型號（預設 A10G）。沙箱的 `timeout=3600`（最長 1 小時）和 `idle_timeout=1800`（閒置 30 分鐘自動終止）可視需求調整。只有在 `created=True` 時才執行 `_seed_sandbox`，避免重複上傳。

---

### 3. `tools.py`：搜尋工具的實作

`tools.py` 只暴露一個工具：`tavily_search`。

```python
@tool(parse_docstring=True)
def tavily_search(
    query: str,
    max_results: Annotated[int, InjectedToolArg] = 1,
    topic: Annotated[
        Literal["general", "news", "finance"], InjectedToolArg
    ] = "general",
) -> str:
```

值得注意的設計是 `InjectedToolArg`：`max_results` 和 `topic` 被標記為「注入參數」，意思是這兩個參數由框架（或 runtime context）注入，而不是讓模型自己決定。模型只能控制 `query`——這避免了模型在不該時一次搜太多結果。

工具本身的邏輯是：先用 `TavilyClient` 取得搜尋結果的 URL 清單，再對每個 URL 呼叫 `fetch_webpage_content` 抓取完整內容，最後用 `markdownify` 轉成 markdown 格式。這讓 researcher-agent 拿到的不只是摘要，而是完整的網頁原文。

`fetch_webpage_content` 有 `try/except` 包裹，抓取失敗時回傳錯誤訊息而不中斷整個搜尋流程。

---

### 4. `prompts.py`：三套 prompt 各司其職

三套 prompt 分別對應三個角色：

**ORCHESTRATOR_INSTRUCTIONS（最簡短）**

```python
ORCHESTRATOR_INSTRUCTIONS = """You are a Deep Agent that handles research, data analysis, and optimization tasks. You produce thorough, well-structured outputs tailored to the user's request.

Current date: {date}
"""
```

orchestrator 的 system prompt 刻意極簡——只定義角色和日期。這是因為 orchestrator 的詳細行為指令已經寫在 `AGENTS.md`（memory）裡，並在 agent 啟動時自動載入。這樣設計的好處是 AGENTS.md 可以在執行時被修改（自我改進），而 system prompt 不行。

**RESEARCHER_INSTRUCTIONS（研究協議）**

Researcher 的 prompt 定義了一個五步驟的「Research Protocol」：先廣搜、每次搜索後反思、再針對缺口窄搜、有把握時停止。它明確限制了 tool call 的次數（簡單查詢 2-3 次，複雜查詢最多 8 次），並定義了何時應該立刻停止搜索（3 個以上相關來源、最後 2 次搜索回傳相似資訊）。最後還要求把研究結果用 `write_file` 寫到 `/shared/[query_topic].txt`。

**DATA_PROCESSOR_INSTRUCTIONS（七步驟工作流程）**

Data processor 的 prompt 是三套中最長的，定義了七個明確步驟：

1. 理解任務
2. 用 `read_file` 讀取對應的 SKILL.md（**強制要求，不可省略**）
3. 用 `write_file` 把 Python 腳本寫到 `/workspace/[name].py`
4. 用 `execute` 跑腳本
5. 對每張圖呼叫 `read_file("/workspace/<chart>.png")` 內嵌顯示
6. 檢查輸出，最多重試 2 次
7. 把結論寫到 `/shared/[task_topic].txt`

其中最有意思的是「更新 Skills（Self-Improvement）」的段落：當 agent 解決了一個 bug 或發現 API 行為與預期不同時，要**立即**用 `edit_file` 更新對應的 SKILL.md，而不是之後再做。

---

### 5. `AGENTS.md`：Orchestrator 的持久化大腦

`AGENTS.md`（位於 `src/` 本機，在 sandbox 內以 `/memory/AGENTS.md` 存取）是 orchestrator 的「操作手冊」，也是這個系統最有創意的設計之一。

它定義了 orchestrator 的完整工作流程（七步驟），包含：

**委派策略**：多數查詢從 1 個 subagent 開始，只有任務中包含明顯獨立面向時才平行化。最多同時 3 個並行呼叫，總共最多 5 輪委派。

**data-processor-agent 的觸發條件**：CSV 資料分析、大型資料統計運算、ML 模型訓練、圖表製作、大型 PDF 處理——這些全部委派，orchestrator 自己不寫分析程式碼。

**程式碼執行的界線**：orchestrator 可以用 `execute` 做輕量操作（下載檔案、列出目錄），但不能自己寫資料分析程式碼——這屬於 data-processor-agent 的職責。

**自我改進（Self-Improvement）機制**：AGENTS.md 本身定義了何時、如何、在哪裡更新知識。原則是：
- **任務專屬資訊** → 不儲存
- **整個 agent 通用的工作流程知識** → 更新 `/memory/AGENTS.md`
- **某個技能的 API 細節** → 更新對應的 `/skills/<skill>/SKILL.md`
- 更新要**立即執行**，不要累積到之後

AGENTS.md 甚至記錄了已學到的生產知識，例如下載大型資料集時要用串流 + 提早中止，而不是整份緩衝進記憶體。

---

### 6. Skills：漸進式揭露的操作手冊

四個 skill 都遵循相同的格式：YAML frontmatter 定義觸發條件，然後是初始化 boilerplate、快速參考 API、常見陷阱。

**cudf-analytics / cuml-machine-learning**

這兩個 skill 的 boilerplate 設計得很精妙——先做 smoke test，確認 GPU 計算**和** host transfer 都正常，再設定 `GPU` 旗標：

```python
try:
    import cudf
    _test = cudf.Series([1, 2, 3])
    assert _test.sum() == 6
    assert _test.to_pandas().tolist() == [1, 2, 3]
    GPU = True
except Exception as e:
    print(f"[GPU] cudf unavailable, falling back to pandas: {e}")
    GPU = False
```

後續所有讀 CSV、to_pandas() 的操作都通過 `read_csv()` 和 `to_pd()` wrapper，自動根據 `GPU` 旗標選路徑。cuML 也有同樣的模式，差別在於 smoke test 執行了一次 KMeans 來確認 GPU ML 端到端正常。

**data-visualization**

關鍵是一行：在 import pyplot 之前先呼叫 `matplotlib.use('Agg')`，才能在無頭（headless）沙箱環境中正常運作。Skill 還定義了發布品質的預設 rcParams 和色盲友善的 Okabe-Ito 調色盤。

**gpu-document-processing**

這個 skill 採用「sandbox 即工具（sandbox-as-tool）」模式：agent 在 CPU 側推理，把文件解析、切塊（chunking）、嵌入向量計算等重工丟到 GPU sandbox 跑。好處是 API 金鑰不進沙箱、各處理工作彼此獨立，且 GPU 只在實際處理時使用。

---

## 它示範的 Deep Agents 核心能力

| 能力 | 在此範例中的體現 |
|------|-----------------|
| **多模型架構** | orchestrator 用 Claude、researcher 用 Nemotron Super、data-processor 用 Claude |
| **Subagents** | 兩個 subagent 各自有獨立的 system_prompt、tools 和 model |
| **Sandbox backend** | `create_backend` 工廠函式，在執行階段決定 GPU 或 CPU 沙箱 |
| **Skills** | 四個 SKILL.md，只在需要時載入，不占用 system prompt 篇幅 |
| **Self-improving memory** | AGENTS.md + SKILL.md 在執行時被 agent 直接編輯更新 |
| **Human in the loop** | `interrupt_on={"execute": True}` 選項，可在程式碼執行前要求人工確認 |
| **Context schema** | `Context(TypedDict)` 讓每次 invoke 都能傳入不同的執行參數 |

---

## 怎麼跑

**前置條件**：安裝 `uv`，設定以下環境變數：

```bash
export ANTHROPIC_API_KEY=...    # Claude（frontier model）
export NVIDIA_API_KEY=...       # Nemotron Super via NIM
export TAVILY_API_KEY=...       # 網路搜尋
export LANGSMITH_API_KEY=...    # 追蹤（選填）
```

**認證 Modal**（選擇一種）：

```bash
uv run modal setup
# 或在 .env 加入 MODAL_TOKEN_ID 和 MODAL_TOKEN_SECRET
```

**啟動**：

```bash
cd docs/deepagents/examples/nvidia_deep_agent
uv sync
uv run langgraph dev --allow-blocking
```

打開 LangSmith Studio，試試這個 prompt：

```
Generate a 1000-row random dataset about credit card transactions with columns
(id, value, category, score), use your cudf skill, then do some cool analysis
and give me some insights on that data!
```

如果觸發了 human in the loop interrupt，在 Studio 貼上以下內容繼續：

```json
{"decisions": [{"type": "approve"}]}
```

想切換 CPU 模式，在呼叫時傳入 `context={"sandbox_type": "cpu"}`，或在 Studio 的「manage assistants」面板調整。

---

## 延伸：改造成自己的領域 Agent

README 提供了五個調整方向，這裡補充一些具體思路：

**換掉研究模型**：`nemotron_super` 可以換成任何 OpenAI-compatible 端點。如果你有 Groq、Together AI 或自架的 vLLM，只需要調整 `ChatNVIDIA` 的初始化參數，或者改用 `ChatOpenAI(base_url=...)` 來對接。

**新增 domain-specific skill**：在 `skills/` 底下建一個資料夾，放入 `SKILL.md`，描述你的 API、提供 boilerplate 程式碼，列出已知限制。agent 會在沙箱建立時自動上傳它，subagent 在遇到對應任務時就會去讀取。

**換掉 sandbox 供應商**：backend 只依賴 `langchain-modal` 的 `ModalSandbox` 抽象層。如果想換成 Daytona 或 E2B，替換 `create_backend` 函式中的 sandbox 建立邏輯即可，`_seed_sandbox` 的 `upload_files` 模式是通用的。

**啟用 human in the loop**：把 `agent.py` 中 `interrupt_on={"execute": True}` 的注解去掉，每次 data-processor-agent 準備執行程式碼之前，就會暫停等待人工審核。

---

## 小結

NVIDIA Deep Agent 是 Deep Agents 框架一個全面展示的範例，它把多模型分工、GPU sandbox、skills、memory 和自我改進全部塞進一個可運行的專案裡。

從程式碼的角度看，它最值得學的設計有三個：

1. **`create_deep_agent` 的組裝方式**：subagent 用字典定義，backend 用 factory 函式，skills 用路徑列表——每個元件都可以獨立替換。

2. **`backend.py` 的 seed 模式**：sandbox 首次建立時上傳 skills 和 memory，之後 agent 直接在 sandbox 內讀寫——這讓 skills 變成可以被 agent 自行更新的「活文件」。

3. **SKILL.md 的漸進式揭露**：skills 預設不占用 system prompt，只在需要時載入。加上 self-improvement 機制，skills 會在 agent 踩過坑之後自動變得更聰明。

如果你正在考慮把一個需要研究 + 資料分析的任務做成 agent，這個範例是一個非常好的起點。

---

**相關資源**

- [Deep Agents 文件](https://docs.langchain.com/oss/python/deepagents/overview)
- [Agent Skills 規格](https://agentskills.io/specification)
- [NVIDIA NIM](https://build.nvidia.com/)
- [Modal](https://modal.com)
- [NVIDIA AIQ Blueprint](https://github.com/langchain-ai/aiq-blueprint)
- [The Two Patterns for Agent Sandboxes](https://blog.langchain.com/the-two-patterns-by-which-agents-connect-sandboxes/)
