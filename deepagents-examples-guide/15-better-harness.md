> 原始碼位置：docs/deepagents/examples/better-harness/

# 讓 Agent 優化 Agent：better-harness 核心導讀 🔁

**副標：用 Deep Agent 跑 eval 驅動的 harness engineering——從 baseline 到 train/holdout 決策，一份自動化的提案→評測→保留迴圈**

---

## 一、引言：「harness engineering」是什麼？

過去幾年，AI 工程師花了大量時間在調整模型本身——微調、RLHF、強化學習……但實踐上有另一條路成本低很多：**不動模型，只動模型外圍那層可調整的東西**。這層東西就叫做 harness。

harness 包含什麼？

- **prompt**：系統提示、指令風格、邊界條件的描述
- **tools**：模型可以呼叫的工具函式
- **skills**：提示式的「任務手冊」（Markdown 格式）
- **middleware**：包在工具呼叫外層的邏輯，例如日誌、重試、輸入轉換

LangChain 研究團隊把這套概念稱為 Harness Engineering，靈感來源包括 [karpathy/autoresearch](https://github.com/karpathy/autoresearch) 與 [Meta-Harness 論文](https://arxiv.org/abs/2603.28052)。`better-harness` 把這個想法具體化成一套可執行的系統：

> **讓一個 Deep Agent（外層 agent）透過 evals，自動去改善另一個 agent（內層 agent）的 harness。**

這篇文章會帶你逐段讀懂 `better-harness` 的核心程式碼，理解它的設計決策，以及背後那個「eval 驅動的自我優化」迴圈是如何運作的。

---

## 二、全貌：六步優化流程 + 可編輯 harness surface

### 六步流程

README 把這個系統的優化流程說得很清楚。整個迴圈做的事如下：

| 步驟 | 描述 |
|------|------|
| 1 | 跑 **baseline**（基準線），記錄當前 train / holdout 通過題數 |
| 2 | 為外層 agent 建立一個隔離的 **proposer workspace**（提案用工作區） |
| 3 | 外層 agent 閱讀失敗案例，**編輯**它被允許動的那些 surface 檔案 |
| 4 | 用編輯過的 harness 跑 **train + holdout**，計算候選版本的分數 |
| 5 | **只有在合計通過題數變多**時，才保留這次修改；否則丟棄 |
| 6 | （可選）在 baseline 與最終版本上各跑一次 **scorecard** |

整個迴圈最多跑 `max_iterations` 輪，每一輪都有完整的決策紀錄存到磁碟。

### 什麼是「可編輯 harness surface」？

**surface** 是這個系統的核心概念：它是一個「外層 agent 被允許編輯的單一介面」。每個 surface 對應到內層 agent 在 eval 過程中實際載入的某個東西——一段 prompt 文字、一個工具定義檔、一個 skills Markdown、一段 middleware 程式碼。

系統支援兩種載入模式（`kind`）：

- **`module_attr`**：直接 patch 一個 Python 屬性，例如 `deepagents.graph:BASE_AGENT_PROMPT`
- **`workspace_file`**：在 eval 執行期間，暫時取代目標工作區裡的某個檔案

這個設計讓「不同介面採用不同注入策略」成為可能——prompt 可以直接改記憶體裡的 Python 屬性，工具/技能/middleware 則改磁碟上的檔案。

---

## 三、環境與安裝

`better-harness` 的依賴極輕——`pyproject.toml` 的 `dependencies = []`，也就是沒有強制依賴。開發依賴只有 `pytest` 和 `ruff`。

```bash
# 安裝開發依賴
uv sync --extra dev

# 複製範例設定（唯一公開完整範例）
cp examples/deepagents_example.toml my_experiment.toml
```

執行前需要有：

- **Python 3.12+**（`pyproject.toml` 要求）
- **`uv`**（套件管理與執行）
- **Deep Agents**：已安裝套件，或透過環境變數 `DEEPAGENTS_ROOT` 指向本機 clone

```bash
# 驗證設定是否合法
uv run better-harness validate my_experiment.toml

# 跑優化迴圈，最多 3 輪
uv run better-harness run my_experiment.toml \
  --output-dir runs/my-harness \
  --max-iterations 3
```

---

## 四、★ 逐段導讀

### 4.1 `examples/deepagents_example.toml`——如何曝露 surface 介面給外層 agent 編輯

這是 repo 裡**唯一一份公開、完整可跑的範例**，README 明確指出「從這裡開始看」。

**`[experiment]` 區塊**定義基本資訊：

```toml
[experiment]
name = "my-deepagents-harness"
runner = "pytest"
workspace_root = "${DEEPAGENTS_ROOT}"
model = "claude-sonnet-4-6"
max_iterations = 3
```

`workspace_root = "${DEEPAGENTS_ROOT}"` 示範了環境變數展開語法。`core.py` 裡的 `expand_env()` 函式（用 `re.compile(r"\$\{([^}]+)\}")`）會在載入設定時把它展開成真實路徑。

**`[better_agent]` 區塊**定義外層 Deep Agent 的行為：

```toml
[better_agent]
model = "claude-sonnet-4-6"
max_turns = 11000
```

`max_turns = 11000` 是一個非常寬鬆的上限，意思是「讓外層 agent 跑到它自己覺得做完為止」。

---

**五個 surface 的定義**是這份設定最值得細讀的部分。

**① prompt（`module_attr` 類型）**：

```toml
[surfaces.prompt]
kind = "module_attr"
target = "deepagents.graph:BASE_AGENT_PROMPT"
filename = "prompt.txt"
base_value = """
You are a helpful agent.

If a request is underspecified, ask the minimum followup needed to make the next useful move.
Do not ask for details the user already supplied.
Use reasonable defaults when the request clearly implies them.
"""
```

`kind = "module_attr"` 搭配 `target = "deepagents.graph:BASE_AGENT_PROMPT"` 的意思是：eval 執行時，`patching.py` 的 `patch_module_attrs()` 函式會用 `importlib.import_module("deepagents.graph")` 匯入模組，然後 `setattr(module, "BASE_AGENT_PROMPT", <新文字>)`。這讓你不需要修改任何檔案就能換掉 prompt。

`base_value` 直接內嵌在設定檔裡，讓整份設定「自我完整（self-contained）」——外層 agent 一打開設定就能看到起始 prompt。

---

**② tools（`workspace_file` 類型）**：

```toml
[surfaces.tools]
kind = "workspace_file"
target = "libs/deepagents/deepagents/custom_tools.py"
filename = "custom_tools.py"
base_value = """
from langchain.tools import tool

@tool
def send_report(recipient: str, subject: str, body: str) -> str:
    \"\"\"Send a report to a user.\"\"\"
    return f"sent report to {recipient} with subject {subject}"
"""
```

`kind = "workspace_file"` 的意思是：eval 執行期間，`patching.py` 的 `workspace_override_context()` 函式會把 `workspace_root/libs/deepagents/deepagents/custom_tools.py` 暫時換成這個新內容，eval 結束後還原。外層 agent 若想新增或修改工具，就直接改 `current/custom_tools.py`。

---

**③ skills（`workspace_file` 類型）**：

```toml
[surfaces.skills]
kind = "workspace_file"
target = "libs/deepagents/deepagents/skills/reporting.md"
filename = "reporting.md"
base_value = """
# Reporting skill

- Ask domain-defining questions before implementation questions.
- For recurring summaries, clarify format or detail level before broad setup details.
- Prefer acting when the destination and core intent are already clear.
"""
```

skills 是 Deep Agents 架構裡的「任務手冊」——Markdown 格式，讓 agent 在面對特定任務時知道該遵守什麼原則。把它曝露成 surface，意味著外層 agent 可以根據失敗案例改寫這份指引。

---

**④⑤ middleware 的兩個 surface——最重要的設計雷區**

README 特別標注了 ⚠️ 警告：**middleware 通常需要同時曝露「實作」與「接線（wiring）」兩個 surface**。若只曝露了實作程式碼，卻沒有曝露 `create_deep_agent(middleware=[...])` 那個地方，外層 agent 就算寫了中介程式碼，也無法讓它真的生效。

```toml
[surfaces.middleware_impl]
kind = "workspace_file"
target = "libs/deepagents/deepagents/custom_middleware.py"
filename = "custom_middleware.py"
base_value = """
from langchain.agents.middleware import wrap_tool_call

@wrap_tool_call
def log_tool_calls(request, handler):
    \"\"\"Simple example middleware that logs tool calls.\"\"\"
    result = handler(request)
    return result
"""

[surfaces.middleware_registration]
kind = "workspace_file"
target = "libs/deepagents/deepagents/agent_setup.py"
filename = "agent_setup.py"
base_value = """
from deepagents import create_deep_agent
from deepagents.custom_middleware import log_tool_calls
from deepagents.custom_tools import send_report

def build_agent(model: str):
    return create_deep_agent(
        model=model,
        tools=[send_report],
        middleware=[log_tool_calls],
        system_prompt="You are a helpful agent.",
    )
"""
```

`middleware_impl` 是實作，`middleware_registration` 是把 `log_tool_calls` 接進 `create_deep_agent(middleware=[...])` 的那個地方。兩個都曝露，外層 agent 才能完整地「新增一個 middleware 並讓它生效」。

---

**eval cases 的三種 split**：

```toml
[[cases]]
case_id = "tests/evals/test_tool_selection.py::test_indirect_email_report[{model}]"
split = "train"
stratum = "tool_use"

[[cases]]
case_id = "tests/evals/test_tool_selection.py::test_direct_request_slack_dm[{model}]"
split = "holdout"
stratum = "tool_use"

[[cases]]
case_id = "tests/evals/test_tool_selection.py::test_direct_request_github_pr[{model}]"
split = "scorecard"
stratum = "tool_use"
```

`{model}` 佔位符讓同一份設定可以套用不同模型。`stratum` 用來確保 train 與 holdout 涵蓋相同的任務類別（`validate_experiment()` 會檢查 `train` 和 `holdout` 的 strata 集合必須相等）。

---

### 4.2 `better_harness/core.py`——優化迴圈的骨架

`core.py` 是整個系統的核心，承擔三件事：資料模型定義、設定載入/驗證、優化迴圈執行。

**資料模型層級**

```
Surface          一個可編輯介面的定義（kind/target/base_value/filename）
EvalCase         一個 eval 案例（case_id/split/stratum）
Experiment       完整的實驗設定（surfaces + cases + runner config）
Variant          一組「已填好內容的介面值」快照
CaseOutcome      單一案例的執行結果
SplitResult      單一 split 的聚合結果
Proposal         外層 agent 產出的一份提案
CandidateEvaluation  對候選版本的完整評估（train + holdout + accepted）
RunReport        整次執行的最終報告
```

這些 dataclass 全部用 `frozen=True`，意味著它們是不可變的值物件，狀態變更只透過建立新物件來完成。

**`load_experiment()` 的設定解析邏輯**

`load_experiment()` 用 `tomllib.loads()` 讀取 TOML，解析過程有幾個值得注意的地方：

1. `expand_env()` 展開 `${VAR}` 語法，讓 `workspace_root = "${DEEPAGENTS_ROOT}"` 這種寫法可行
2. surface 必須定義 `base_file` 或 `base_value` 其中之一（兩者都有或都沒有都會 `raise ValueError`）
3. `validate_experiment()` 強制 train 和 holdout 的 strata 集合相等，防止你評測的方向偏掉
4. `normalize_split()` 支援 `"acceptance"` 和 `"final_eval"` 這兩個別名，都對應到 `"scorecard"`

**`run_experiment()` 的優化迴圈**

這是 `core.py` 裡最核心的函式，值得完整看清楚它的結構：

```python
def run_experiment(experiment, *, output_dir, max_iterations=None, reuse_existing=False):
    runner = build_runner(experiment)
    layout = RunLayout(output_dir.resolve())
    layout.write_manifest(experiment)

    baseline = build_baseline_variant(experiment)
    current = baseline

    # 先跑 baseline 的 train + holdout
    baseline_train = runner.run_split(experiment, variant=baseline, split="train", ...)
    baseline_holdout = runner.run_split(experiment, variant=baseline, split="holdout", ...)
    current_train = baseline_train
    current_holdout = baseline_holdout

    iterations = []
    for index in range(1, iteration_limit + 1):
        # 如果已全過，提前結束
        if current_train.passed == current_train.total and current_holdout.passed == current_holdout.total:
            break

        # 呼叫外層 agent，產生提案與候選 variant
        proposal, candidate_variant = propose_variant(
            experiment=experiment, current=current,
            train_result=current_train, layout=layout, iteration=index,
        )

        if not proposal.changed_surfaces:
            # agent 說它沒有需要改的，提前結束
            iterations.append(IterationRecord(iteration=index, ..., candidate=None))
            break

        # 用候選版本跑 train + holdout
        train = runner.run_split(experiment, variant=candidate_variant, split="train", ...)
        holdout = runner.run_split(experiment, variant=candidate_variant, split="holdout", ...)

        # 決策：只看合計通過題數
        current_combined = current_train.passed + current_holdout.passed
        candidate_combined = train.passed + holdout.passed
        accepted = candidate_combined > current_combined

        # 只有接受時才更新 current
        if accepted:
            current = candidate_variant
            current_train = train
            current_holdout = holdout

    # 可選的 scorecard（只跑 baseline 和最終版）
    baseline_scorecard = _run_optional_scorecard(...)
    final_scorecard = _run_optional_scorecard(experiment=experiment, variant=current, ...)

    report = RunReport(baseline=baseline, final=current, ...)
    layout.write_report(report)
    return report
```

幾個設計決定值得特別標注：

- **決策準則只有一個**：`candidate_combined > current_combined`，合計通過題數必須嚴格變多才接受。這個「嚴格大於」防止了 train 進步但 holdout 退步的假改善。
- **scorecard 只跑兩次**：baseline 和最終版，避免 scorecard 案例「洩漏」進優化迴圈而讓外層 agent 對它 overfit。
- **`proposal.changed_surfaces` 為空就停下**：外層 agent 可以藉此表達「我沒有什麼要改的了」，系統尊重這個信號。

**`RunLayout` 的目錄結構設計**

```
output_dir/
  manifest.json              # 實驗 metadata
  split.json                 # split 清單
  variants/
    baseline.json            # baseline variant 的完整值
    iter-001.json            # 第一輪候選 variant
  history/
    visible/
      train/                 # train 結果（外層 agent 可見）
      iterations/
        001/
          decision.json      # 決策：accepted/rejected
          proposer_workspace/  # 外層 agent 的工作區
    private/
      holdout/               # holdout 結果（外層 agent 不可見）
  report.json                # 最終報告
  report.md                  # 最終報告（Markdown）
```

這個分層設計刻意把 holdout 結果放在 `private/` 路徑下，讓外層 agent 「看不到」 holdout 的失敗細節，防止它針對 holdout 案例做 overfit。

---

### 4.3 `better_harness/agent.py`——外層 Deep Agent 的建構與執行

`agent.py` 負責建立外層 Deep Agent、為它搭建 proposer workspace、執行它、並把它的輸出轉換成候選 variant。

**`DEFAULT_SYSTEM_PROMPT`：外層 agent 的指令**

```python
DEFAULT_SYSTEM_PROMPT = """You are Better Agent, an outer-loop Deep Agent that improves another agent harness.

Rules:
- Edit only files under /current.
- Do not edit train_cases, history, or bookkeeping files except /proposal.md.
- Prefer general harness fixes over case-specific hacks.
- Do not overfit to the visible examples. Infer the broader policy or behavior they expose...
- If a surface is a code file such as a tool or middleware file, write the real code...
- If you change tool or middleware behavior, update both the implementation and any registration or wiring surfaces you were given.
- Use surface_manifest.json and task.md to understand how each editable file maps back to the target harness.
- Stop as soon as /current and /proposal.md are updated.
..."""
```

這段 system prompt 有幾個重要約束：「只改 `/current` 下面的檔案」、「不要 overfit 到可見案例」、「改了 middleware 就要同時改接線」——這些都是把 README 裡的 ⚠️ 警告轉譯成 agent 行為規則。

**`build_proposer_workspace()` 的工作區建構**

每一輪迭代都會在 `layout.proposer_workspace_dir(iteration)` 建一個全新的工作區：

```python
def build_proposer_workspace(*, experiment, current, train_result, layout, iteration):
    root = layout.proposer_workspace_dir(iteration)
    current_dir = root / "current"
    current_dir.mkdir(parents=True, exist_ok=True)

    # 把各 surface 的目前值寫成檔案，放進 /current
    for name, surface in experiment.surfaces.items():
        path = current_dir / surface.filename
        path.write_text(current.values[name])

    # 寫出 surface_manifest.json（讓 agent 知道每個檔案對應哪個 target）
    (root / "surface_manifest.json").write_text(json.dumps(manifest, ...))

    # 把 train 失敗案例與摘要寫進去
    _write_train_artifacts(experiment, train_result, root)

    # 把先前輪的可見歷史（decision.json/md）複製進來
    _write_visible_history(layout, root)
    _copy_prior_visible_artifacts(layout, root, iteration)

    # 寫出 task.md（清楚說明有哪些 surface、目前失敗什麼）
    _write_task_file(experiment, current, train_result, root)

    # 建立空的 proposal.md 讓 agent 填寫
    proposal_file = root / "proposal.md"
    proposal_file.write_text("# Proposal\n\n- Summary:\n- Why this should help:\n- Surfaces changed:\n")
    ...
```

外層 agent 看到的工作區長這樣：

```
proposer_workspace/
  current/
    prompt.txt          # 目前 prompt（可改）
    custom_tools.py     # 目前 tools（可改）
    reporting.md        # 目前 skills（可改）
    custom_middleware.py
    agent_setup.py
  surface_manifest.json # surface 名稱 → 檔案 → target 的對應表
  train_failures.json   # 這一輪的 train 失敗案例
  train_summary.json    # train 整體結果
  train_cases/          # 失敗案例的原始測試程式碼
  history/
    visible_history.md  # 先前各輪的決策摘要
    prior_visible/      # 先前輪的詳細紀錄
  task.md               # 任務說明（可見 surface 清單 + 失敗清單）
  proposal.md           # agent 完成後要填的摘要
```

**`invoke_deepagents_proposer()` 的兩條路徑**

這個函式根據是否有 `deepagents_root` 而走兩條路：

1. **有 `deepagents_root`（常見情境）**：用 `subprocess.run()` 以 `uv run --project <deepagents_root>` 啟動一個子行程，執行 `better_harness.agent` 的 `main()` 函式。這確保外層 agent 在 Deep Agents 的正確 Python 環境裡執行，不受當前環境污染。
2. **沒有 `deepagents_root`**：在當前 process 裡用 `importlib.import_module` 動態匯入 `deepagents.backends`、`deepagents.graph` 等模組。

兩條路徑最終都呼叫：

```python
agent = create_deep_agent(
    model=experiment.better_agent_model,
    system_prompt=_compose_system_prompt(experiment),
    backend=filesystem_backend_cls(root_dir=str(workspace.root), virtual_mode=True),
)
result = agent.invoke(
    {"messages": [HumanMessage(content="Read /task.md first. Then inspect...")]},
    config={"recursion_limit": experiment.better_agent_max_turns},
)
```

`FilesystemBackend(virtual_mode=True)` 是 Deep Agents 的 filesystem backend，讓 agent 把 `proposer_workspace/` 的路徑當作它的「虛擬根目錄」來讀寫。

兩條路徑都包含**最多 3 次的 retry 邏輯**，只在偵測到 `"overloaded"`、`"rate limit"`、`"timeout"` 等暫態錯誤時才重試。

**`propose_variant()` 的整合**

```python
def propose_variant(*, experiment, current, train_result, layout, iteration):
    workspace = build_proposer_workspace(...)
    final_message = invoke_deepagents_proposer(...)
    values = load_candidate_values(current=current, workspace=workspace)
    changed_surfaces = tuple(
        sorted(name for name in experiment.surfaces if values[name] != current.values[name])
    )
    proposal = Proposal(changed_surfaces=changed_surfaces, ...)
    candidate = build_variant(experiment=experiment, label=f"iter-{iteration:03d}", values=values)
    return proposal, candidate
```

這個函式是 `core.py` 的 `run_experiment()` 和 `agent.py` 的外層 agent 之間的橋樑：執行外層 agent、讀回被修改的檔案、計算哪些 surface 真的改了、建立候選 variant。

---

### 4.4 `runners.py` / `patching.py` / `better_harness_plugin.py`——各司其職

**`runners.py`**：實作兩個 runner 類別——`PytestRunner` 和 `HarborRunner`——它們都支援 `run_split()` 和 `collect_inventory()` 介面。`PytestRunner.run_split()` 對每個案例個別跑一次 pytest 子行程，解析 JUnit XML 輸出；`HarborRunner` 則呼叫 `harbor run` 命令並解析 `result.json` 或 `reward.txt`。兩個 runner 都用 `workspace_override_context()` 在 eval 執行期間暫時置換 workspace 檔案，並在結束後還原。

**`patching.py`**：負責兩種 patch 機制。`workspace_override_context()` 是一個 context manager，進入時寫入覆寫內容，離開時還原備份。`patch_from_env()` 讀取 `BETTER_HARNESS_VARIANT_FILE` 環境變數指向的 variant JSON，並對 `module_attr` 類型的 surface 執行 `setattr` patch。`ensure_sitecustomize()` 則在 `.runtime/` 目錄寫出一個 `sitecustomize.py`，讓 Python 在 subprocess 啟動時自動呼叫 `patch_from_env()`。

**`better_harness_plugin.py`**：整個檔案只有兩行——匯入 `patch_from_env` 並呼叫它。這是 pytest plugin 的進入點；`runners.py` 的 `_base_command()` 在組裝 pytest 命令時永遠加上 `-p better_harness_plugin`，確保每次 pytest 執行都會在啟動時套用 `module_attr` 的 patch。

---

## 五、示範的 Deep Agents 能力

`better-harness` 展示了幾個 Deep Agents 的核心能力：

**1. Eval 驅動的自我優化**

外層 agent 不是亂猜——它讀 `train_failures.json`、讀 `train_cases/` 裡的原始測試程式碼、讀先前輪的決策歷史，然後做出有根據的編輯決定。這個閉環讓系統在沒有人工介入的情況下迭代改善。

**2. 雙 agent 架構的職責分離**

外層 agent（提案者）和內層 agent（被優化對象）完全隔離：外層 agent 只能看到 proposer workspace 的內容，無法直接存取目標 repo；內層 agent 在真實的 eval 環境裡執行，不知道自己正在被優化。

**3. Surface 作為可調介面**

把 prompt / tools / skills / middleware 全部建模成「可編輯 surface」，讓 harness engineering 有了明確的操作對象。這個抽象層讓同一套優化迴圈可以套用在不同的 agent 技術棧上。

**4. FilesystemBackend 的虛擬工作區**

使用 `FilesystemBackend(virtual_mode=True)` 讓外層 agent 在一個沙箱化的虛擬檔案系統裡工作，不會直接寫到系統上的任何其他位置。

---

## 六、怎麼用

最快的起步方式是複製 `examples/deepagents_example.toml`，然後：

1. 把 `workspace_root` 改成你的 agent repo 路徑
2. 把各 `surfaces.*` 的 `target` 改成你 repo 裡真實存在的模組屬性或檔案路徑
3. 把 `[[cases]]` 改成你自己 eval suite 裡的測試 node id
4. 確認 train / holdout 兩個 split 各至少有一個 case，且 strata 集合相等

若你還沒有 evals，README 建議直接用你慣用的 agent 給它以下指令：

```
set up this repo to optimize for X task. I don't have evals so go and bootstrap them
in this repo and run the optimization loop
```

---

## 七、延伸閱讀

- [Deep Agents repo](https://github.com/langchain-ai/deepagents)：外層 agent 實際使用的基礎
- [Improving Deep Agents with Harness Engineering](https://blog.langchain.com/improving-deep-agents-with-harness-engineering/)：LangChain 的 harness engineering 研究博文
- [Custom middleware in LangChain](https://docs.langchain.com/oss/python/langchain/middleware/custom)：middleware 如何自訂
- [Meta-Harness 論文](https://arxiv.org/abs/2603.28052)：理論背景

---

## 八、小結

`better-harness` 用約 1500 行程式碼實現了一個完整的 eval 驅動 harness 優化系統。它的核心設計非常簡潔：

- **Surface 概念**把「可以調整的東西」從「不能動的模型」分離出來
- **train/holdout 分割** + **嚴格大於**的接受準則防止 overfit 和假改善
- **proposer workspace** 讓外層 agent 在一個有上下文、有限制的沙箱裡工作
- **middleware 雙 surface 設計**（實作 + 接線）是一個非常實際的工程細節，避免了「寫了但不生效」的常見坑

整個系統最吸引人的地方，是它把「讓 agent 優化 agent」這個抽象概念，落地成了一套可以真的跑起來、有紀錄、可重現的工程基礎設施。作為 research artifact，它保持了刻意的簡單——容易讀、容易改、容易套用到你自己的 agent 技術棧。

如果你正在考慮如何系統性地改善一個 agent 的 harness，`better-harness` 是一個值得參考的起點。
