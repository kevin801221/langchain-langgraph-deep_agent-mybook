> 原始範例路徑：`docs/deepagents/examples/ralph_mode/`

# Ralph Mode：用「每輪全新 Context」打造最純粹的自主 Agent 迴圈 🔄

**副標：一行 shell、一個無限 while loop、filesystem + git 當記憶——Geoff Huntley 的 Ralph 哲學如何被移植成 Deep Agents 的官方範例**

---

## 一、引言：Ralph 從哪裡來？

2025 年底，開發者社群突然瘋傳一段極簡的 shell 指令：

```bash
while :; do cat PROMPT.md | agent; done
```

這行指令出自 [Geoff Huntley](https://ghuntley.com/ralph/)，他把這個模式稱為 **Ralph**。概念簡單到讓人懷疑自己是不是少看了什麼：把同一份 prompt 無限餵給 agent，讓它一輪接一輪地跑，不把它叫停就不會停。

乍看之下這不就是「死循環」嗎？但 Ralph 的精妙在於**哲學立場**：

- **不管理 conversation history**：每一輪都從 token zero 開始，不用擔心 context window 被撐爆。
- **不設計記憶機制**：記憶就是 filesystem。agent 寫的每一個檔案、每一次 `git commit`，都是留給下一輪「自己」的訊息。
- **不寫控制邏輯**：終止條件就是你按下 `Ctrl+C`，對成果滿意的那一刻就是結束。

這是一種「宣告式 agent 編排」的極端形式——你只告訴 agent 你要什麼，它負責把自己不斷叫醒繼續做。

Deep Agents 官方把這個概念移植成一個範例，核心只有一支 249 行的 Python 檔：`ralph_mode.py`。本文將逐段拆解這支程式，看清楚它的每一個設計選擇。

---

## 二、核心概念：Fresh Context 為何是最單純的 Context Management？

在多輪 agent 對話中，context management 通常是最頭痛的問題：

- 要不要保留歷史訊息？保留多少？
- 要不要做 summarization？
- 超過 context window 怎麼辦？

Ralph 的回答是：**全部都不用管**。

每一輪迭代用的是一個全新的 thread（在 `deepagents-cli` 的術語裡就是全新的 checkpoint）。上一輪做了什麼、說了什麼，這一輪的 LLM 完全不知道——但它可以用工具去**讀取 filesystem**，看看有哪些檔案、`git log` 長什麼樣子，然後自己推斷出「前人留下了什麼，我接下來要做什麼」。

這個設計有幾個實際好處：

1. **token 成本固定**：每輪的 prompt 長度一樣，不會因為跑了 50 輪就突然爆掉。
2. **容錯性高**：某一輪出錯或輸出廢話，對下一輪沒有汙染。
3. **實作極簡**：不需要複雜的記憶模組或外部資料庫。

代價是 agent 必須自己去「重新理解現況」——但這反而訓練出更有自主性的 agent 行為。

---

## 三、環境與安裝

使用 ralph_mode 只需要三個步驟：

```bash
# 1. 建立虛擬環境
uv venv && source .venv/bin/activate

# 2. 安裝 deepagents-cli
uv pip install deepagents-cli

# 3. 執行
python ralph_mode.py "Build a Python programming course for beginners. Use git."
```

`deepagents-cli` 是本範例唯一的外部依賴（加上 `rich` 做終端機美化輸出）。它內建了模型解析、工具管理、checkpoint、streaming 輸出與 HITL（human-in-the-loop）核可等功能——`ralph_mode.py` 本身只負責最外層的「編排迴圈」，把複雜的 agent 基礎設施全部委派給 CLI。

---

## 四、★ ralph_mode.py 逐段導讀 ★

### 4.1 模組 docstring：三句話說清楚設計意圖

```python
"""Ralph Mode - 給 Deep Agents 使用的自主迴圈（Autonomous looping）。

每一輪迭代都委派給 `deepagents-cli` 的 `run_non_interactive` 處理，
它會負責模型解析（model resolution）、工具註冊、checkpoint、串流，
以及 HITL（human-in-the-loop）核可。本腳本只負責編排最外層的迴圈。
"""
```

這段 docstring 就是整支程式的設計書：ralph_mode.py **只做一件事**——維護最外層的 while loop；其他所有 agent 基礎設施都交給 `deepagents-cli`。這是單一職責原則的乾淨示範。

---

### 4.2 Import：極簡依賴

```python
from __future__ import annotations

import argparse
import asyncio
import contextlib
import json
import logging
import os
import warnings
from pathlib import Path
from typing import Any

from deepagents_cli.non_interactive import run_non_interactive
from rich.console import Console
```

標準庫用了 `argparse`（CLI 參數）、`asyncio`（非同步執行）、`contextlib`（優雅抑制 KeyboardInterrupt）、`json`（解析 model-params）、`os`（切換工作目錄）、`pathlib.Path`（跨平台路徑操作）。

外部依賴只有兩個：
- `deepagents_cli.non_interactive.run_non_interactive`：這是整個機制的引擎。
- `rich.Console`：讓終端機輸出帶顏色、有層次感。

---

### 4.3 `ralph()` coroutine：迴圈引擎的核心

```python
async def ralph(
    task: str,
    max_iterations: int = 0,
    model_name: str | None = None,
    model_params: dict[str, Any] | None = None,
    sandbox_type: str = "none",
    sandbox_id: str | None = None,
    sandbox_setup: str | None = None,
    *,
    stream: bool = True,
) -> None:
```

這是一個 `async` coroutine，最後由 `asyncio.run()` 在 `main()` 裡啟動。參數設計值得逐一關注：

- `task: str`：宣告式的任務描述，這個字串會被原封不動地嵌入每輪的 prompt。
- `max_iterations: int = 0`：**0 代表不限次數**。這是一個直覺的「魔法值」設計：0 = 無限，大於 0 = 有上限。
- `model_name: str | None = None`：格式為 `provider:model`（例如 `anthropic:claude-sonnet-4-6`）。傳 `None` 時讓 `deepagents-cli` 自動解析——它會依序查找設定檔的 `[models].default`、`[models].recent`，最後才看環境變數裡的 API 金鑰（`ANTHROPIC_API_KEY`、`OPENAI_API_KEY`、`GOOGLE_API_KEY`）。
- `model_params: dict[str, Any] | None = None`：額外模型參數，例如 `{"temperature": 0.5}`。
- `sandbox_type`, `sandbox_id`, `sandbox_setup`：支援在遠端沙箱（AgentCore、Modal、Daytona、Runloop）執行，`sandbox_id` 可重用既有實例。
- `stream: bool = True`（keyword-only）：是否串流輸出。用 `*` 強制成 keyword-only，防止位置參數誤傳。

---

### 4.4 啟動時的資訊面板

```python
work_path = Path.cwd()
console = Console()

console.print("\n[bold magenta]Ralph Mode[/bold magenta]")
console.print(f"[dim]Task: {task}[/dim]")
iters_label = (
    "unlimited (Ctrl+C to stop)" if max_iterations == 0 else str(max_iterations)
)
console.print(f"[dim]Iterations: {iters_label}[/dim]")
if model_name:
    console.print(f"[dim]Model: {model_name}[/dim]")
if sandbox_type != "none":
    sandbox_label = sandbox_type
    if sandbox_id:
        sandbox_label += f" (id: {sandbox_id})"
    console.print(f"[dim]Sandbox: {sandbox_label}[/dim]")
console.print(f"[dim]Working directory: {work_path}[/dim]\n")
```

程式一開始先印出一個資訊摘要：任務描述、迭代上限（或「unlimited」）、模型名稱（如果有指定）、沙箱設定、工作目錄。`work_path = Path.cwd()` 在這裡只做一次——因為 `main()` 已在 coroutine 啟動前就透過 `os.chdir()` 切好目錄了，之後不需要再更動。

---

### 4.5 主迴圈：while loop 的控制邏輯 ⚙️

```python
iteration = 1
try:
    while max_iterations == 0 or iteration <= max_iterations:
        separator = "=" * 60
        console.print(f"\n[bold cyan]{separator}[/bold cyan]")
        console.print(f"[bold cyan]RALPH ITERATION {iteration}[/bold cyan]")
        console.print(f"[bold cyan]{separator}[/bold cyan]\n")
```

迴圈條件是 `max_iterations == 0 or iteration <= max_iterations`：

- 當 `max_iterations == 0`：短路求值（short-circuit evaluation），右側永遠不被檢查，無限執行。
- 當 `max_iterations > 0`：檢查 `iteration <= max_iterations`，到達上限就自然跳出 while。

每一輪開頭印出醒目的分隔線和「RALPH ITERATION N」，讓你在長時間執行時能清楚看出每輪的邊界。

---

### 4.6 動態組裝每輪的 Prompt：「告訴 agent 它在哪裡」

```python
iter_display = (
    f"{iteration}/{max_iterations}"
    if max_iterations > 0
    else str(iteration)
)
prompt = (
    f"## Ralph Iteration {iter_display}\n\n"
    f"Your previous work is in the filesystem. "
    f"Check what exists and keep building.\n\n"
    f"TASK:\n{task}\n\n"
    f"Make progress. You'll be called again."
)
```

這段是整個設計最關鍵的地方之一。每輪的 prompt **不是靜態的**，而是動態生成，包含三個要素：

1. **迭代標頭** `## Ralph Iteration {iter_display}`：告訴 agent「你現在在第幾輪」；如果有上限，也會顯示 `N/M` 讓 agent 知道還有多少輪。
2. **語境提示** `"Your previous work is in the filesystem. Check what exists and keep building."`：這句話是 Ralph 模式的靈魂——它**明確指示** agent 去檢查 filesystem，而不是假設 agent 自己會想到。
3. **原始任務** + 收尾鼓勵 `"Make progress. You'll be called again."`：後者暗示 agent 不需要在這一輪做完所有事，可以專注於局部進展。

這個 prompt 結構巧妙地把「context 已重置」的限制轉化成 agent 行為的引導。

---

### 4.7 呼叫 `run_non_interactive`：把一輪委派給 CLI

```python
exit_code = await run_non_interactive(
    message=prompt,
    assistant_id="ralph",
    model_name=model_name,
    model_params=model_params,
    sandbox_type=sandbox_type,
    sandbox_id=sandbox_id,
    sandbox_setup=sandbox_setup,
    quiet=True,
    stream=stream,
)
```

`run_non_interactive` 是 `deepagents-cli` 提供的核心 async 函式，一次呼叫就是完整跑一輪 agent（包含工具呼叫、模型推理、streaming 輸出、checkpoint 記錄等）。幾個參數值得注意：

- `assistant_id="ralph"`：給這個 agent session 一個固定識別名稱，讓 CLI 知道這是 ralph 類型的 agent。
- `quiet=True`：抑制 CLI 自己的部分輸出，把顯示控制權交給 ralph_mode.py 的 `Console`。
- `stream=stream`：直接轉傳使用者的偏好。

`run_non_interactive` 回傳一個整數 exit code，這個值在下一段決定了迴圈的走向。

---

### 4.8 Exit Code 處理：退出、錯誤、繼續三條路

```python
if exit_code == 130:  # 130 代表使用者以 Ctrl+C 中斷
    break

if exit_code != 0:
    console.print(
        f"[bold red]Iteration {iteration} exited with code {exit_code}[/bold red]"
    )

console.print(f"\n[dim]...continuing to iteration {iteration + 1}[/dim]")
iteration += 1
```

exit code 130 是 Unix 慣例的「SIGINT 中斷」（即 Ctrl+C）。當 `run_non_interactive` 在 agent 執行中途偵測到使用者中斷，它會回傳 130，ralph_mode 收到後立即 `break` 跳出迴圈——這讓使用者可以在任意一輪中途按 Ctrl+C 乾淨地停止。

非 0 且非 130 的 exit code 表示某種錯誤（例如 API 失敗、工具執行異常）。**Ralph 的選擇是：印出錯誤，但繼續跑下一輪。** 這是刻意的容錯設計——某一輪出問題不代表整個任務失敗，下一輪的 agent 可能自行修正。

---

### 4.9 外層 KeyboardInterrupt 捕獲：雙重保險

```python
    except KeyboardInterrupt:
        console.print(
            f"\n[bold yellow]Stopped after {iteration} iterations[/bold yellow]"
        )
```

在 `try/except KeyboardInterrupt` 的外層還有一個 catch。這處理的是「在兩次 agent 呼叫之間」（例如在印分隔線的瞬間）按下 Ctrl+C 的情況，確保任何時機的中斷都能被乾淨捕獲並印出友善訊息。

---

### 4.10 結束時列出 Filesystem 快照 📁

```python
console.print(f"\n[bold]Files in {work_path}:[/bold]")
for path in sorted(work_path.rglob("*")):
    if path.is_file() and ".git" not in str(path):
        console.print(f"  {path.relative_to(work_path)}", style="dim")
```

迴圈結束後（不管是自然結束、達到上限，還是被中斷），程式會用 `Path.rglob("*")` 遞迴列出工作目錄下所有檔案，並過濾掉 `.git` 目錄內容（避免列出一堆 git 內部物件檔案）。這個「結案報告」讓你一眼看出 agent 在這幾輪裡總共建立或修改了哪些檔案。

---

### 4.11 `main()` 函式：CLI 入口點

```python
def main() -> None:
    """解析 CLI 參數並啟動 Ralph 迴圈。"""
    warnings.filterwarnings("ignore", message="Core Pydantic V1 functionality")
```

第一行用 `warnings.filterwarnings` 靜默掉 Pydantic V1 相容性警告——這是使用了某些仍依賴 Pydantic V1 API 的套件時常見的噪音，ralph_mode 選擇把它過濾掉以保持終端機輸出的整潔。

```python
parser = argparse.ArgumentParser(
    description="Ralph Mode - 給 Deep Agents 使用的自主迴圈",
    formatter_class=argparse.RawDescriptionHelpFormatter,
    epilog="""
Examples:
  python ralph_mode.py "Build a Python course. Use git."
  ...
    """,
)
```

`RawDescriptionHelpFormatter` 讓 `epilog` 的範例區塊保留原始換行格式，印出來的 `--help` 更易讀。

---

### 4.12 各個 CLI 參數的宣告

```python
parser.add_argument("task", help="要執行的任務（宣告式描述，說明你想要什麼）")
parser.add_argument("--iterations", type=int, default=0, ...)
parser.add_argument("--model", ...)
parser.add_argument("--work-dir", ...)
parser.add_argument("--model-params", ...)
parser.add_argument("--sandbox", default="none", ...)
parser.add_argument("--sandbox-id", ...)
parser.add_argument("--sandbox-setup", ...)
parser.add_argument("--no-stream", action="store_true", ...)
parser.add_argument("--shell-allow-list", ...)
```

`task` 是唯一的 positional argument（必填）。所有其他參數都是 optional flag。`--no-stream` 用 `action="store_true"` 而非 `type=bool`，這是 argparse 的最佳實踐——存在旗標就是 `True`，不存在就是 `False`。

---

### 4.13 工作目錄切換與 shell-allow-list 設定

```python
if args.work_dir:
    resolved = Path(args.work_dir).resolve()
    resolved.mkdir(parents=True, exist_ok=True)
    os.chdir(resolved)

if args.shell_allow_list:
    from deepagents_cli.config import parse_shell_allow_list, settings
    settings.shell_allow_list = parse_shell_allow_list(args.shell_allow_list)
```

`--work-dir` 處理有兩個細節：`.resolve()` 把相對路徑轉為絕對路徑，`.mkdir(parents=True, exist_ok=True)` 確保目錄存在（不存在就建立，存在也不會報錯）。接著 `os.chdir()` 在啟動 coroutine 之前就完成切換，讓 `ralph()` 裡的 `Path.cwd()` 能正確取到目標目錄。

`--shell-allow-list` 則是直接修改 `deepagents-cli` 的全域 `settings` 物件，把允許自動核可的 shell 指令清單注入進去，讓 agent 在指定指令上不需要等待使用者一一確認。

---

### 4.14 Model params 解析與啟動 coroutine

```python
model_params: dict[str, Any] | None = None
if args.model_params:
    model_params = json.loads(args.model_params)

with contextlib.suppress(KeyboardInterrupt):
    asyncio.run(
        ralph(
            args.task,
            args.iterations,
            args.model,
            model_params=model_params,
            sandbox_type=args.sandbox,
            sandbox_id=args.sandbox_id,
            sandbox_setup=args.sandbox_setup,
            stream=not args.no_stream,
        )
    )
```

`json.loads(args.model_params)` 把 CLI 傳入的 JSON 字串（如 `'{"temperature": 0.5}'`）解析成 Python dict。

`contextlib.suppress(KeyboardInterrupt)` 是最外層的一道防線——如果在 `asyncio.run()` 層面有任何未被捕獲的 `KeyboardInterrupt`（例如在事件迴圈初始化時），它會被靜默吸收，程式正常退出而不印出 traceback。搭配 `ralph()` 內部的 `except KeyboardInterrupt`，形成了雙重清潔退出機制。

---

## 五、Ralph 對應 Deep Agents 的哪些能力？

| Ralph 的機制 | Deep Agents 對應能力 |
|---|---|
| 每輪全新 thread | Fresh context / checkpoint 管理 |
| filesystem 持久化 | Agent 工具：讀寫檔案、執行 shell |
| git 當工作日誌 | 版本控制工具整合 |
| `run_non_interactive` | CLI agent loop 的封裝 |
| `--sandbox` 旗標 | 遠端沙箱（AgentCore、Modal 等）支援 |
| `--shell-allow-list` | HITL 核可策略設定 |

Ralph 本身就是 **agent loop + filesystem persistence + CLI orchestration** 這三個 Deep Agents 核心能力的最小可行展示。

---

## 六、怎麼跑？可以試什麼任務？

```bash
# 基本：無限跑，Ctrl+C 停
python ralph_mode.py "Build a REST API with FastAPI. Use git to track progress."

# 限定 5 輪
python ralph_mode.py "Write unit tests for the project" --iterations 5

# 指定工作目錄（不存在會自動建立）
python ralph_mode.py "Create a CLI todo app" --work-dir ./todo-project

# 在 Modal 沙箱執行
python ralph_mode.py "Build a data pipeline" --sandbox modal

# 預核可安全的 shell 指令，減少確認中斷
python ralph_mode.py "Refactor the codebase" --shell-allow-list recommended

# 指定模型與 temperature
python ralph_mode.py "Write creative docs" --model claude-sonnet-4-6 \
  --model-params '{"temperature": 0.7}'
```

**建議在任務描述裡加上 `"Use git."`**——這樣 agent 就會主動用 `git commit` 記錄每一步進展，而不只是把檔案寫在 filesystem 上，讓你之後可以用 `git log` 追蹤整個開發歷程。

---

## 七、什麼情境適合，什麼情境不適合用 Ralph？

### 適合 ✅

- **長期建構型任務**：寫一門程式課、建立一個 REST API、搭一個 CLI 工具——需要多輪累積的任務。
- **你不想一直盯著螢幕**的後台作業：讓 Ralph 跑著，你去做別的事，回來看成果。
- **容許部分失敗**的任務：某一輪出錯不影響整體，下一輪 agent 自行修正。
- **任務邊界模糊**的探索性工作：沒有明確的「完成定義」，靠你主觀判斷何時喊停。

### 不適合 ⚠️

- **需要在輪次間傳遞精確狀態**的任務：如果 agent 必須記住某個具體的對話脈絡（而不只是讀 filesystem），Ralph 的 fresh context 會讓它遺忘。
- **有明確終止條件且可自動判斷的任務**：若你想讓 agent 自動偵測「任務完成」並停止，需要在 prompt 工程上額外設計（或改用其他有 exit tool 的架構）。
- **需要快速迭代回饋**的對話型任務：Ralph 是「fire and forget」式的，不適合需要即時人機往返的工作流程。

---

## 八、小結

Ralph Mode 的程式碼只有 249 行，但它展示了一種強大的 agent 編排思路：**把複雜性推到 filesystem 和 git，而不是推到 context management**。

核心設計選擇可以用三句話總結：

1. **Fresh context every iteration**：讓 LLM 從不被歷史對話拖累，每輪都以最清晰的狀態思考。
2. **Filesystem is memory**：agent 寫下的每個檔案、每次 git commit，都是留給下一輪的「記憶膠囊」。
3. **Loop until you're satisfied**：終止邏輯交還給人類——`Ctrl+C` 是最誠實的 exit condition。

如果你曾被 agent context 管理搞得焦頭爛額，不妨試試 Ralph 的「reset everything, keep files」哲學——有時候，最單純的方案才是最強的方案。

---

*本文基於 `docs/deepagents/examples/ralph_mode/ralph_mode.py` 實際原始碼撰寫，所有函式名稱、參數名稱均與原始檔案一致。*
