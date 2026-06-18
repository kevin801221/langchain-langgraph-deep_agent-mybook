> 原始範例路徑：`docs/deepagents/examples/llm-wiki/`

# 🧠 用 Deep Agents 打造永不遺忘的 LLM Wiki：llm-wiki 範例逐段導讀

**副標：script-first 架構 × LangSmith Sandbox × Context Hub 三位一體，讓知識庫隨每次執行持續演進**

---

## 引言：什麼是「script-first」？什麼是「持久累積的維基」？

大多數 LLM 應用的知識只活在單次對話的 context window 裡——問完就忘、重跑又從零開始。**llm-wiki** 提出了一個截然不同的設計哲學：

1. **script-first**：沒有複雜的圖（Graph）拓撲，也不需要拖拉介面；整個 agent 工作流程由 Python 腳本直接驅動，`runner.py` 就是入口，跑一行指令就能完成一個知識操作。
2. **持久累積的維基**：每次執行（ingest、query、lint）的成果都會被寫進本地 `wiki/` 目錄，再透過 `langsmith hub push` 同步到 LangSmith 的 Context Hub，讓知識以版本（revision）形式累積，而不是每次重來。

這個模型特別有趣的地方在於：它示範了 **create_deep_agent** 如何在 **LangSmith Sandbox** 這個隔離執行環境中，以 **filesystem** 作為知識的持久化介質——agent 讀 `/raw/`、寫 `/wiki/`，runner 管理 `log.md` 的時間線，三者分工明確，協作構成一個可以長期維護的 topic knowledge base。

---

## 全貌：各檔職責、三模式資料流與 wiki/ 目錄的演進

### 各檔職責一覽表

| 檔案 | 職責 |
|---|---|
| `runner.py` | 輕量 CLI 入口：解析→執行→輸出 |
| `helpers.py` | 共用工具 + CLI 解析 + 模式調度邏輯 |
| `models.py` | `RunnerConfig` / `CliDeps` / `RunResult` dataclass |
| `ingest.py` | ingest mode：來源展開、review/apply 兩階段 |
| `query.py` | query mode：唯讀分析 + 選擇性存檔 |
| `lint.py` | lint mode：單階段健檢與調和 |
| `init.py` | init mode：建立本機 scaffold + 驗證 internal source |
| `index.py` | 重建 `wiki/index.md` 目錄頁 |
| `log.py` | append-only 時間線條目格式化與寫入 |

### 三模式資料流

```
ingest  →  [stage raw/]  →  (review →) apply  →  refresh index  →  append log  →  hub push
query   →  [read wiki/]  →  grounded 答案  →  (file → apply)   →  append log  →  hub push
lint    →  [read log.md] →  apply 健檢     →  refresh index    →  append log  →  hub push
```

### `wiki/` 目錄如何逐步累積

```
<topic-dir>/
├── AGENTS.md          ← 維基 schema/規則（init 建立，後續手動維護）
├── raw/               ← 不可變動的來源素材（ingest 寫入）
├── wiki/
│   ├── index.md       ← 內容目錄頁（每次模式結束後重建）
│   ├── <concept>.md   ← 正規概念/實體頁（ingest apply 寫入）
│   └── query/
│       └── <slug>.md  ← 有保存價值的 query 答案（query apply 寫入）
└── log.md             ← 只新增的時間線（runner 管理，agent 不可直接編輯）
```

每次執行都讓 `wiki/` 長大一點；Context Hub 上保存每次 push 的版本，團隊成員可以看到知識庫的演進歷史。

---

## 環境與安裝

```bash
# 從 deepagents repo 根目錄執行
uv sync --project examples/llm-wiki

# 驗證 hub 指令是否可用
uv run --project examples/llm-wiki langsmith hub --help

# 確認 API 金鑰已設定（ingest/query/lint 三模式都需要）
echo "${LANGSMITH_API_KEY:+set}"
```

**依賴重點**：
- Python 3.11+
- `langsmith[sandbox]`（含 `hub` 系列指令）
- `LANGSMITH_API_KEY`（執行任何 agent 操作前必須設定）

---

## ★ 逐段導讀

### 1. `runner.py`：極簡的 CLI 入口

```python
def main(argv: Sequence[str] | None = None) -> int:
    try:
        config = parse_config(argv)
        run_result = run(config)
    except WikiError as exc:
        print(f"error: {exc}")
        return 1

    if run_result.answer:
        print(run_result.answer)
    if run_result.hub_url:
        print(f"Context Hub: {run_result.hub_url}")
    return 0
```

`runner.py` 只有 28 行，設計極其克制。它做的事情僅有三步：

1. 呼叫 `parse_config(argv)` 把 CLI 參數轉成 `RunnerConfig`
2. 呼叫 `run(config)` 執行選定的模式
3. 列印答案與 Context Hub URL，或在遇到 `WikiError` 時回傳 exit code 1

所有複雜邏輯全部下沉到 `helpers.py`，`runner.py` 只負責 I/O 介面。這正是 script-first 哲學的體現：**腳本是人類可讀的工作流程敘述，而不是業務邏輯的容身之處**。

---

### 2. `helpers.py`：共用工具 + 模式調度的核心

`helpers.py` 是整個範例最長的檔案（830 行），承擔了六大職責：

#### 2a. CLI 解析：`_build_parser()` 與 `parse_config()`

```python
def _build_parser() -> argparse.ArgumentParser:
    parser = argparse.ArgumentParser(
        description="LLM wiki (Deep Agents + LangSmith Hub CLI)"
    )
    parser.add_argument("--mode", required=True, choices=["init", "ingest", "query", "lint"])
    parser.add_argument("--repo", required=True, ...)
    parser.add_argument("--source", action="append", default=[], ...)
    parser.add_argument("--question", default=None, ...)
    parser.add_argument("--review", action="store_true", ...)
    # ... 其他旗標
    return parser
```

`parse_config()` 在解析後還會做 cross-field 驗證（ingest 模式強制 `--source`、query 模式強制 `--question`），並把 `--repo` 拆解成 `owner/repo` 兩個部分，最終組出不可變的 `RunnerConfig` dataclass。

#### 2b. 系統提示（System Prompt）的設計哲學

`_BASE_SYSTEM_PROMPT` 是 agent 的行為憲法，幾個設計決策值得注意：

```python
_BASE_SYSTEM_PROMPT = """You are an expert research synthesizer building a long-lived topic knowledge base.

Mission:
- Build an accurate, high-signal, source-grounded topic corpus in `/wiki/`.
- Treat `/raw/` as immutable evidence inputs.
...
Filesystem policy:
- Never write to `/raw/`.
- Never edit `/log.md`; the runner maintains append-only interaction entries.
- Write only under `/wiki/`.
"""
```

系統提示直接規定了檔案系統的存取政策：`/raw/` 不可寫、`/log.md` 由 runner 管理、只能在 `/wiki/` 下寫入。這些規則與後面的 `FilesystemPermission` 形成雙重保障——軟性的 prompt 指引加上硬性的程式碼防護。

#### 2c. FilesystemPermission 的雙模式政策

```python
def _permissions() -> list[FilesystemPermission]:
    """定義維基操作的檔案系統寫入政策。"""
    return [
        FilesystemPermission(operations=["write"], paths=["/raw/**"], mode="deny"),
        FilesystemPermission(operations=["write"], paths=["/AGENTS.md"], mode="deny"),
        FilesystemPermission(operations=["write"], paths=["/wiki/**"], mode="allow"),
        FilesystemPermission(operations=["write"], paths=["/log.md"], mode="deny"),
    ]

def _review_permissions() -> list[FilesystemPermission]:
    """定義 ingest review 階段的唯讀政策。"""
    return [
        FilesystemPermission(operations=["write"], paths=["/raw/**"], mode="deny"),
        FilesystemPermission(operations=["write"], paths=["/wiki/**"], mode="deny"),
        FilesystemPermission(operations=["write"], paths=["/log.md"], mode="deny"),
        FilesystemPermission(operations=["write"], paths=["/AGENTS.md"], mode="deny"),
    ]
```

這裡清楚地分出兩套政策：`_permissions()` 用於 apply 階段（允許 `wiki/**` 寫入），`_review_permissions()` 用於 review 階段（完全唯讀）。agent 的 sandbox 環境在程式碼層面就被鎖定，而不是靠 prompt 的「請你不要亂寫」。

#### 2d. Sandbox Backend 的建立：`_create_langsmith_sandbox_backend()`

```python
@contextmanager
def _create_langsmith_sandbox_backend() -> Iterator[SandboxBackendProtocol]:
    client = SandboxClient(api_key=env_key)
    snapshots = client.list_snapshots(name_contains=resolved_snapshot)
    has_ready_snapshot = any(
        snap.name == resolved_snapshot and snap.status == "ready" for snap in snapshots
    )
    if not has_ready_snapshot:
        client.create_snapshot(name=resolved_snapshot, docker_image=docker_image, ...)

    sandbox = client.create_sandbox(snapshot_name=resolved_snapshot)
    try:
        yield LangSmithSandbox(sandbox=sandbox)
    finally:
        with suppress(Exception):
            client.delete_sandbox(sandbox.name)
```

這個 context manager 負責建立和清理 LangSmith Sandbox：先查詢是否有已就緒的 snapshot（預設名稱 `deepagents-wiki`），沒有就建立一個；建立 sandbox 後 yield 出去；結束時無論如何都嘗試刪除 sandbox（`suppress(Exception)` 確保清理失敗不會掩蓋主錯誤）。

#### 2e. CompositeBackend：把 sandbox 與本地 filesystem 接在一起

```python
def _run_agent_mode(...) -> str:
    with _create_langsmith_sandbox_backend() as sandbox_backend:
        workspace_backend = FilesystemBackend(root_dir=workspace_dir, virtual_mode=True)
        backend = CompositeBackend(
            default=sandbox_backend,
            routes={
                "/raw/": workspace_backend,
                "/wiki/": workspace_backend,
                "/log.md": workspace_backend,
                "/AGENTS.md": workspace_backend,
            },
        )
        agent = create_deep_agent(
            model=model,
            backend=backend,
            permissions=permissions,
            system_prompt=_BASE_SYSTEM_PROMPT,
        )
        result = agent.invoke({"messages": [{"role": "user", "content": prompt}]})
```

這是本範例最精妙的設計之一：`CompositeBackend` 讓 agent 的「計算環境」跑在 LangSmith Sandbox（隔離、安全），但「知識存取」（`/raw/`、`/wiki/`、`/log.md`、`/AGENTS.md`）全部路由到本地 `FilesystemBackend`（`virtual_mode=True` 表示 agent 看到的路徑是虛擬化的，實際映射到 `workspace_dir` 下的對應子目錄）。這樣一來，知識可以持久存在於本地，而 agent 的執行環境則完全受 sandbox 保護。

#### 2f. 模式調度：`run()` 與 `_run_pull_mode()`

```python
def run(config: RunnerConfig, deps: CliDeps | None = None) -> RunResult:
    _ensure_mode_prerequisites(config.mode)
    resolved_deps = deps or CliDeps(
        run_langsmith_cli=_run_langsmith_cli,
        run_agent_mode=_run_agent_apply_mode,
        run_agent_review_mode=_run_agent_review_mode,
        ask_user=input,
        tempdir_factory=tempfile.TemporaryDirectory,
    )
    if config.mode == "init":
        return _run_init(config, resolved_deps)
    return _run_pull_mode(config, resolved_deps)
```

`run()` 是整個系統的調度樞紐。注意 `CliDeps` 的設計：所有外部相依（CLI 呼叫、agent 執行、使用者輸入、暫存目錄）都透過 dataclass 注入，讓測試可以輕鬆替換任何一個依賴——這是典型的 dependency injection 模式。

`_run_pull_mode()` 則實作了 ingest/query/lint 三種模式共用的骨架：先 `hub pull` 拉下最新版本到臨時目錄，執行選定的模式工作流程，視需要 `hub push` 更新，最後回傳 `RunResult`。

---

### 3. `models.py`：三個 frozen dataclass

```python
Mode = Literal["init", "ingest", "query", "lint"]

@dataclass(frozen=True)
class RunnerConfig:
    mode: Mode
    topic: str
    repo: str
    owner: str | None
    topic_dir: Path
    sources: tuple[Path, ...]
    note: str | None
    question: str | None
    model: str | None
    description: str | None
    review: bool

@dataclass(frozen=True)
class CliDeps:
    run_langsmith_cli: Callable[[Sequence[str]], subprocess.CompletedProcess[str]]
    run_agent_mode: Callable[[Path, str, str, str | None], str]
    run_agent_review_mode: Callable[[Path, str, str, str | None], str]
    ask_user: Callable[[str], str]
    tempdir_factory: Callable[[], tempfile.TemporaryDirectory[str]]

@dataclass(frozen=True)
class RunResult:
    answer: str | None
    hub_url: str | None
```

`models.py` 只有 51 行，卻是整個系統的「型別合約」。三個 dataclass 都是 `frozen=True`（不可變），確保 config 和結果在傳遞過程中不會被意外修改。

`CliDeps` 是最有設計感的一個：它把「如何呼叫 CLI」、「如何執行 agent」、「如何問使用者」、「如何建立暫存目錄」全部抽象成可注入的 Callable，讓 `helpers.py` 的核心邏輯與具體實作完全解耦，也讓單元測試可以不用真的啟動 sandbox 就能驗證流程邏輯。

---

### 4. `query.py`：讀維基 → 推理 → grounded 答案 → 選擇性存檔

`query.py` 展示了 Deep Agents 最典型的「分析-決策-存檔」三段式模式。

#### 4a. Prompt 工程：`build_query_prompt()`

```python
def build_query_prompt(topic: str, question: str) -> str:
    return (
        f"Answer this question about '{topic}': {question}\n\n"
        "Required workflow:\n"
        "1) Read `/wiki/index.md` first ...\n"
        "2) Read recent `/log.md` entries ... to understand what was ingested recently.\n"
        "3) Prefer checking relevant prior `/wiki/query/*.md` pages first as a discovery step.\n"
        "4) Use those query pages to identify likely canonical `/wiki/*.md` pages.\n"
        "5) Read the canonical wiki pages before final synthesis.\n"
        "6) Provide a grounded answer with wiki file path citations.\n"
        "7) Decide whether this answer should be filed as a durable wiki page.\n\n"
        "Output format (exact keys):\n"
        "ANSWER:\n<markdown answer with citations>\n\n"
        "FILING_DECISION: file|skip\n"
        "FILING_REASON: <one sentence>\n"
    )
```

這個 prompt 的七步工作流程設計得非常細緻：強制 agent 先讀 index（避免盲目遍歷）、再讀 log（取得近期脈絡）、再看舊的 query 頁面（discovery/routing），最後才讀正規 wiki 頁面做最終推理。輸出格式要求精確的 key（`FILING_DECISION: file|skip`），方便後續程式碼解析。

#### 4b. 決策解析：`parse_query_decision()`

```python
_QUERY_DECISION_PATTERN = re.compile(
    r"^FILING_DECISION:\s*(file|skip)\s*$", re.IGNORECASE | re.MULTILINE
)
_QUERY_REASON_PATTERN = re.compile(
    r"^FILING_REASON:\s*(.+)$", re.IGNORECASE | re.MULTILINE
)

def parse_query_decision(raw_response: str) -> QueryDecision:
    decision_match = _QUERY_DECISION_PATTERN.search(response)
    should_file = (
        decision_match is not None and decision_match.group(1).lower() == "file"
    )
    # 擷取 ANSWER: 之後、FILING_DECISION: 之前的文字
    answer_text = response[: decision_match.start()].strip()
    ...
    return QueryDecision(answer=answer_text, should_file=should_file, reason=reason)
```

用正規表達式從模型輸出中解析結構化決策，如果 marker 缺失則 fallback 到 `skip`（保守預設值）。這展示了一種務實的 LLM 輸出解析策略：要求結構化輸出，但優雅地處理格式不符的情況。

#### 4c. 兩階段執行：`run_query_workspace()`

```python
def run_query_workspace(config: RunnerConfig, workspace_dir: Path, deps: CliDeps) -> QueryResult:
    # 第一階段：唯讀分析
    review_response = deps.run_agent_review_mode(workspace_dir, ...)
    decision = parse_query_decision(review_response)
    helpers._append_log_entry(workspace_dir, "query.review", review_outcome, ...)

    if not decision.should_file:
        return QueryResult(answer=decision.answer, should_push=True, filed_path=None)

    # 第二階段：存檔（僅在決定 file 時執行）
    target_path = query_target_path(question)   # /wiki/query/<slug>.md
    deps.run_agent_mode(workspace_dir, ...)     # apply 模式，允許寫入
    helpers._refresh_index(config.topic, workspace_dir)
    helpers._append_log_entry(workspace_dir, "query.apply", "filed", ...)
    return QueryResult(answer=decision.answer, should_push=True, filed_path=target_path)
```

注意：即使 `should_file=False`，`should_push` 仍然是 `True`——因為 `query.review` 的 log 條目已經被寫入，需要 push 以保持 Context Hub 的時間線完整性。這個設計細節體現了「每次執行都留下可追蹤記錄」的系統性思維。

---

### 5. 其他模式摘要

#### `ingest.py`：來源展開 + review/apply 兩階段

`ingest` 的核心流程在 `run_ingest_workspace()` 中：

1. `expand_sources()` 把 `--source` 參數展開（支援單檔 + 目錄遞迴），去重後得到確定性的檔案清單。
2. `_stage_sources()` 把來源複製到 `workspace_dir/raw/`（處理同名衝突：自動加 `-2`、`-3` 後綴）。
3. 若有 `--review` 旗標，先執行 review 階段（唯讀）請 agent 輸出六段式分析報告（來源摘取 / 變更計劃 / 跨來源合成 / 矛盾分析 / index 更新 / 缺口建議），然後呼叫 `confirm_ingest_apply()` 等待使用者輸入 `y/N`。
4. Apply 階段才執行寫入：agent 把來源消化進正規的概念/實體頁面，回傳 apply 報告。
5. Runner 重建 index、寫入 `ingest.apply` log 條目、push。

值得注意的是，若沒有 `--review` 旗標，`build_ingest_apply_prompt()` 會在 review plan 欄位填入一段說明，要求 agent 自己先做 review-quality 分析再直接 apply——這讓批次模式依然保有分析品質，而不是直接魯莽地寫入。

#### `lint.py`：單階段健檢

`lint` 是三模式中最簡單的，`run_lint_workspace()` 只呼叫一次 agent（apply 模式）。`build_lint_prompt()` 要求 agent 先讀最近的 `log.md` 條目（取得近期脈絡），然後在 `/wiki/` 中執行：矛盾調和、過時論述更新、孤立頁面修補、跨引用補齊、關鍵概念頁面建立。結束後回傳固定格式的三節報告（Reconciled Changes / Remaining Gaps / Suggested Next Questions）。Runner 重建 index、寫入 `lint.apply` log、push。

#### `init.py`：建立 scaffold + 驗證 internal source

`init` 模式的獨特職責是確保 Context Hub repo 使用 `source=internal`。`ensure_internal_repo_default()` 先呼叫 `/api/v1/repos` 查詢 repo 是否存在：不存在就用 POST 建立（帶 `"source": "internal"`）；存在就檢查 `source` 欄位，若不是 `internal` 直接 fail fast（這個設計防止意外把 wiki 推進公開或其他類型的 repo）。之後呼叫 `hub init`、建立本地 scaffold（`AGENTS.md`、`raw/`、`wiki/index.md`、`log.md`），最後 push 第一個版本。

#### `index.py` / `log.py`：基礎設施層

`index.py` 的 `refresh_index()` 會遍歷 `wiki/*.md`（排除 `index.md` 本身），按目錄自動分類（`query/` → Queries、`entity/` → Entities 等），為每個頁面擷取標題、一行摘要、選擇性中繼資料（最新日期、來源數量），組出分類式的目錄頁。

`log.py` 的 `append_log_entry()` 建立格式固定的 log 條目（`## [YYYY-MM-DD] mode.phase | outcome=... key=value`），支援 `shell grep` 直接解析，設計上刻意不讓 agent 直接呼叫（只由 runner 呼叫），維持 append-only 不可竄改的審計特性。

---

## 示範的 Deep Agents 核心能力

| 能力 | llm-wiki 的體現 |
|---|---|
| **create_deep_agent** | 所有三種主模式（ingest/query/lint）底層都透過 `create_deep_agent` 建立 agent，不需要手動拼接 LangGraph 圖 |
| **LangSmith Sandbox** | agent 的計算環境完全隔離在 sandbox，透過 `SandboxClient` 管理 snapshot 生命週期 |
| **FilesystemPermission** | 在程式碼層面精確控制 agent 的讀寫邊界，review 模式和 apply 模式使用不同的 permission set |
| **CompositeBackend** | 把 sandbox（計算）與本地 filesystem（知識存取）無縫接合，讓知識持久化而執行環境隔離 |
| **script-first orchestration** | 整個知識庫生命週期（init → ingest → query → lint → push）都由 Python 腳本明確調度，工作流程透明可審計 |
| **Context Hub 同步** | 每次執行後 `langsmith hub push`，知識版本存在 hub 上，支援團隊協作與版本追蹤 |

---

## 怎麼跑：完整示範流程

```bash
# 1. 初始化維基，建立第一個 Context Hub revision
uv run --project examples/llm-wiki \
  python examples/llm-wiki/runner.py \
  --mode init \
  --repo "ada-lovelace-wiki"

# 2. 匯入來源素材（支援單檔 + 目錄，可重複 --source）
uv run --project examples/llm-wiki \
  python examples/llm-wiki/runner.py \
  --mode ingest \
  --repo "ada-lovelace-wiki" \
  --source ./notes/ada.md \
  --source ./notes/speeches/ \
  --review   # 可選：開啟兩階段 review/confirm

# 3. 查詢（自動判斷是否存檔）
uv run --project examples/llm-wiki \
  python examples/llm-wiki/runner.py \
  --mode query \
  --repo "ada-lovelace-wiki" \
  --question "What did Ada contribute to computing?"

# 4. 健檢與整理（修連結、去重、更新 index）
uv run --project examples/llm-wiki \
  python examples/llm-wiki/runner.py \
  --mode lint \
  --repo "ada-lovelace-wiki"

# 快速查看最近 5 筆 log 記錄
grep "^## \[" log.md | tail -5
```

---

## 延伸：這個設計能衍生出什麼？

1. **主題切換**：只需換 `--repo` 就能維護完全獨立的知識庫，每個 repo 有自己的版本歷史。
2. **自訂 AGENTS.md**：`AGENTS.md` 是 agent 的工作規則設定檔，可以針對特定 wiki 主題定制 ingest/query 的行為，例如指定哪類頁面優先、哪些來源可信度較高。
3. **CI 整合**：三種模式都是純 CLI 指令，可以輕鬆整合進 GitHub Actions，定期自動 ingest 新素材、定期 lint 維護知識品質。
4. **多人協作**：Context Hub 上的 revision 讓團隊成員可以追蹤知識庫的演進，透過 promote 操作管理哪個版本是「正式版」。
5. **擴展 backend**：`CompositeBackend` 的路由機制也可以擴充，例如把 `/raw/` 路由到 S3，讓大型素材不需要存在本地。

---

## 小結

**llm-wiki** 是一個展示 Deep Agents 實用性的精緻範例。它的核心洞見是：**LLM 最有價值的輸出不是一次性的回答，而是可以累積、可以版本控制、可以被未來 agent 重複使用的結構化知識**。

從程式碼設計上看，它示範了幾個值得借鑒的模式：

- **frozen dataclass** 作為跨模組的型別合約（`RunnerConfig` / `CliDeps` / `RunResult`）
- **dependency injection** 讓核心邏輯與 CLI / sandbox / filesystem 完全解耦，測試友善
- **雙層防護**（system prompt 規則 + `FilesystemPermission` 程式碼鎖定）確保 agent 不會越界寫入
- **CompositeBackend** 的路由設計，在不犧牲隔離性的前提下實現知識持久化
- **append-only log** 作為可稽核的互動時間線，格式設計成 `grep` 可直接解析

如果你想打造一個「越用越聰明」的 agent 知識庫，llm-wiki 提供了一個可以直接借鑒的完整藍圖。
