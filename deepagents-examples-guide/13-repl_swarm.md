> 本文對應範例：`docs/deepagents/examples/repl_swarm/`

# 把並行派工變成一行 import：Deep Agents 的 repl_swarm 範例全解析

**副標：當 TypeScript skill 遇上 QuickJS REPL，semaphore pool 讓 subagent fan-out 變得像呼叫函式一樣自然**

---

## 一、引言：核心洞見

軟體工程師第一次看到「要並行派發多個 subagent」這個需求，通常的直覺是：在 Python 後端寫一個 `asyncio.gather`，或者用執行緒池包起來。這種做法沒問題，但它把並行邏輯死釘在基礎建設層，每次要修改排程策略就得動到 host 程式碼。

`repl_swarm` 這個官方範例反過來問一個問題：**如果把並行編排邏輯整個打包成一個 skill，讓 agent 在 REPL 裡用 `import` 動態拉進來，會發生什麼事？**

答案是：agent 只需要在某一次 `eval` 呼叫裡寫下：

```javascript
const { runSwarm } = await import("@/skills/swarm");
```

接著呼叫 `runSwarm({ tasks: [...] })`，就能同時派發多個 subagent、控制 bounded concurrency、收集結果，最後拿回一份結構化的 `SwarmSummary`。

Python 那側完全不知道這件事——它只是建立了一個帶有 skill 目錄的 agent，然後 `ainvoke` 一個使用者指令。**把編排邏輯放進 skill，而不是 Python**，這才是這個範例真正的核心洞見。

---

## 二、全貌速覽

執行流程可以用五個步驟描述：

| 步驟 | 主角 | 發生什麼事 |
|------|------|-----------|
| 1 | `swarm_agent.py` | 呼叫 `create_deep_agent`，掛上 `skills=["/skills/"]` 與 `CodeInterpreterMiddleware` |
| 2 | `SkillsMiddleware` | 解析 `SKILL.md` frontmatter，把 `SkillMetadata` 寫進 agent state |
| 3 | REPL `eval` | model 寫下 `await import("@/skills/swarm")`，middleware 動態載入 `index.ts` |
| 4 | `runSwarm()` | TypeScript worker-pool 透過 PTC 層呼叫 `tools.task(...)` 並行派工 |
| 5 | 結果回傳 | `SwarmSummary` 依輸入順序回傳，Python 側收到最終 messages |

整個範例的目錄結構極度精簡：

```
repl_swarm/
├── skills/
│   └── swarm/
│       ├── SKILL.md           # frontmatter + 散文說明（即 prompt）
│       └── scripts/index.ts   # runSwarm() 的真正實作
├── swarm_agent.py             # 薄 driver，不含任何 swarm 邏輯
└── pyproject.toml
```

三個主角，分工明確，彼此解耦。

---

## 三、環境與安裝

`pyproject.toml` 宣告了這個範例的三條依賴：

```toml
[project]
name = "repl-swarm-example"
version = "0.1.0"
requires-python = ">=3.11"
dependencies = [
    "deepagents>=0.6.8",
    "langchain-quickjs",
    "langchain-anthropic>=1.4.4",
]
```

- **`deepagents`**：提供 `create_deep_agent`、`SkillsMiddleware`、subagent 基礎建設。最低版本 `0.6.8` 才有 `module` / `entrypoint` 這組 frontmatter 鍵。
- **`langchain-quickjs`**：提供 `CodeInterpreterMiddleware`，內嵌 QuickJS 引擎，讓 agent 能在 REPL 裡執行 TypeScript/JavaScript（oxidase 在安裝時負責剝除 TypeScript 型別宣告）。
- **`langchain-anthropic`**：Anthropic 模型的 LangChain 整合。預設模型為 `claude-sonnet-4-6`。

`[tool.uv.sources]` 區段把兩個 lib 指向 monorepo 本地路徑（`editable = true`），這是開發期常見的本地覆蓋寫法，正式發布後這兩行會消失。

安裝：

```bash
cd docs/deepagents/examples/repl_swarm
uv sync
```

---

## 四、★ 逐段導讀

### 4.1 `swarm_agent.py`——薄 driver 怎麼把零件接起來

這份 Python 檔總共不到 100 行，而且如同它自己的 docstring 所說：「這份 Python 驅動程式（driver）裡沒有任何 swarm 專屬的東西。」讓我們逐段拆解。

**Backend 架構：CompositeBackend**

```python
skill_backend = FilesystemBackend(root_dir=SKILLS_DIR, virtual_mode=True)
backend = CompositeBackend(
    default=StateBackend(),
    routes={"/skills/": skill_backend},
)
```

這裡有個精心設計的路由策略：

- `/skills/` 路徑 → `FilesystemBackend`（指向真實的 `skills/` 目錄）。`SkillsMiddleware` 在建立 agent 的階段就需要掃描 `SKILL.md`，而此時 graph state 尚未存在，所以 skills 必須落在真實檔案系統上。`virtual_mode=True` 讓它以虛擬路徑方式對外暴露。
- 其餘所有路徑 → `StateBackend`。model 在 REPL 裡寫出的任何檔案（例如 `/tmp_swarm/a`）都只寫進 agent state，不會碰到 host 磁碟。這能避開 macOS 的 SIP / 唯讀根目錄問題，也讓每次執行自我清理。

**建立 agent**

```python
return create_deep_agent(
    model=model,
    backend=backend,
    skills=["/skills/"],
    middleware=[
        CodeInterpreterMiddleware(
            ptc=["task"],
            skills_backend=backend,
            timeout=None,
        )
    ],
)
```

`skills=["/skills/"]` 告訴 `SkillsMiddleware` 去哪裡掃描 skill 目錄。`CodeInterpreterMiddleware` 的 `ptc=["task"]` 是關鍵——PTC（Pass-Through Call）層會把 `task` 工具暴露給 REPL 執行環境，`runSwarm` 才能在 TypeScript 裡呼叫 `tools.task(...)`。`skills_backend=backend` 讓 middleware 知道如何透過 `@/skills/*` 路徑解析並載入 ES module。

**非同步主程式**

```python
async def _amain() -> None:
    args = _parse_args()
    agent = _build_agent(args.model)
    result = await agent.ainvoke(
        {"messages": [{"role": "user", "content": args.task}]},
    )
```

注意這裡強制用 `ainvoke` 而不是 `invoke`。原因在程式碼注解裡說得很清楚：`quickjs_rs.Context` 是 `!Send` 的——如果從「建立該 context 的執行緒」以外的執行緒去呼叫 REPL 的 `eval`，就會 panic。`ToolNode` 的同步路徑會透過 `ThreadPoolExecutor` 分派，因此必須走非同步路徑讓工具呼叫留在原本的執行緒上。

**預設任務**

```python
default=(
    "Use the swarm skill to run these three tasks in parallel and "
    "report the results: (1) write the number 1 to /tmp_swarm/a, "
    "(2) write the number 2 to /tmp_swarm/b, "
    "(3) write the number 3 to /tmp_swarm/c."
)
```

三個相互獨立的寫檔任務，並行跑起來，正好驗證 fan-out 確實運作。

---

### 4.2 `skills/swarm/SKILL.md`——frontmatter 宣告 + 散文如何被當 prompt 載入

```yaml
---
name: swarm
description: 以受限的並行度（bounded concurrency）將一批任務並行派發給子代理。回傳一個摘要物件 {total, completed, failed, results[]} —— 走訪 `.results` 即可取得各任務的輸出。
metadata:
  entrypoint: scripts/index.ts
---
```

這段 frontmatter 有三個欄位：

- **`name`**：skill 的識別名，等一下 `await import("@/skills/swarm")` 裡的 `swarm` 就對應這裡。
- **`description`**：`SkillsMiddleware` 把這段文字注入 agent 的 system prompt，讓 model 知道有這個能力存在，以及回傳值的形狀。
- **`metadata.entrypoint`**：宣告 ES module 的進入點路徑，相對於 skill 目錄。`CodeInterpreterMiddleware` 看到 `await import("@/skills/swarm")` 時，就會去讀這個路徑、建立 `ModuleScope`、呼叫 `ctx.install`。

---

frontmatter 之下是散文說明，這部分同樣會被當作 prompt 載入，成為 model 的知識：

**載入方式**說明告訴 model「請用 `await import(...)` 而非把原始碼複製貼上」——這是反模式警告，防止 model 重複造輪子。

**用法範例**直接給出完整的 JavaScript 程式碼片段，包括如何解構 `results`：

```javascript
const { results, completed, failed } = await runSwarm({
  tasks: [
    { description: "Summarize /notes/alpha.md" },
    { description: "Summarize /notes/beta.md" },
    { description: "Summarize /notes/gamma.md" },
  ],
  concurrency: 3,
  subagentType: "general-purpose",
});
```

**介面約定**（Contract 區段）列出完整的 TypeScript 型別定義，讓 model 清楚知道參數與回傳值的結構，不需要猜測。

**設計筆記**解釋了三個設計決策：派工透過 `tools.task`（PTC 層）、失敗是逐任務捕捉（不中止整個 swarm）、並行度用 worker-pool 而非一次 `Promise.all`。

SKILL.md 的精妙之處在於：**它同時是機器讀的設定檔（frontmatter）和人/model 讀的說明書（散文）**。兩者合一，單一真相來源。

---

### 4.3 `skills/swarm/scripts/index.ts`——`runSwarm()` 的 semaphore pool、任務派發、結果收集

這是這個範例技術含量最高的部分。全檔不到 100 行，但密度極高。

**型別定義（第 12–35 行）**

```typescript
export interface SwarmTask {
  description: string;
  subagentType?: string;
}

export interface SwarmResult {
  id: number;
  status: "completed" | "failed";
  output?: string;
  error?: string;
}

export interface SwarmSummary {
  total: number;
  completed: number;
  failed: number;
  results: SwarmResult[];
}
```

四個 interface 勾勒出整個資料流：`SwarmTask` 是輸入、`SwarmResult` 是單一任務的輸出、`SwarmSummary` 是最終回傳的摘要物件。`RunSwarmOptions` 把 `tasks`、`concurrency`（可選，預設 5，上限 10）、`subagentType`（可選）打包成一個參數物件。

**並行度限制（第 37–50 行）**

```typescript
const DEFAULT_CONCURRENCY = 5;
const MAX_CONCURRENCY = 10;

export async function runSwarm(opts: RunSwarmOptions): Promise<SwarmSummary> {
  const tasks = opts.tasks ?? [];
  const concurrency = Math.max(
    1,
    Math.min(opts.concurrency ?? DEFAULT_CONCURRENCY, MAX_CONCURRENCY),
  );
  const defaultSubagent = opts.subagentType ?? "general-purpose";

  const results: SwarmResult[] = new Array(tasks.length);
  let nextIndex = 0;
```

`Math.max(1, Math.min(..., MAX_CONCURRENCY))` 這個雙重限制確保 concurrency 永遠在 `[1, 10]` 之間，無論呼叫端傳什麼值。`results` 是一個固定大小的陣列，索引對應輸入任務的位置，這是保留輸入順序的關鍵設計。`nextIndex` 是一個共享計數器，worker 每次取任務時自增。

**Worker-Pool（semaphore 模式，第 58–82 行）**

```typescript
const worker = async (): Promise<void> => {
  while (true) {
    const idx = nextIndex++;
    if (idx >= tasks.length) return;
    const task = tasks[idx];
    const subagentType = task.subagentType ?? defaultSubagent;
    try {
      const out = await tools.task({
        description: task.description,
        subagent_type: subagentType,
      });
      results[idx] = { id: idx, status: "completed", output: String(out) };
    } catch (err: any) {
      results[idx] = {
        id: idx,
        status: "failed",
        error: err?.message ?? String(err),
      };
    }
  }
};

const workers: Promise<void>[] = [];
for (let i = 0; i < concurrency; i++) workers.push(worker());
await Promise.all(workers);
```

這段是整個 skill 的核心，實作了一個典型的 worker-pool 模式，效果等同 semaphore：

1. 建立 `concurrency` 個 worker coroutine（這裡是 5 個）。
2. 每個 worker 都在 `while (true)` 迴圈裡不斷取下一個任務（透過 `nextIndex++`，JavaScript 的單執行緒模型保證這個操作是原子的）。
3. 取到的任務呼叫 `tools.task(...)`——這是 PTC 層注入的全域工具，實際上會在 host 側派發一個 subagent。
4. 任務成功，把結果寫進 `results[idx]`；任務失敗，捕捉 error 同樣寫進 `results[idx]`。**失敗不傳播**，不會中斷其他 worker。
5. 當 `idx >= tasks.length`，worker 退出 `while` 迴圈，自然結束。
6. `Promise.all(workers)` 等到所有 worker 都跑完才 resolve。

為什麼不直接用 `Promise.all(tasks.map(...))`？因為那樣會瞬間啟動所有任務。100 個任務就是 100 個並發 subagent 呼叫，容易打爆速率限制和記憶體。Worker-pool 讓「在飛的呼叫」永遠不超過 `concurrency` 個，這才是真正的 bounded concurrency。

**統計與回傳（第 84–91 行）**

```typescript
let completed = 0;
let failed = 0;
for (const r of results) {
  if (r.status === "completed") completed++;
  else failed++;
}

return { total: tasks.length, completed, failed, results };
```

掃一遍 results 陣列統計數量，打包成 `SwarmSummary` 回傳。因為 `results` 是固定大小陣列且索引對應輸入順序，這裡直接走訪就能拿到有序結果。

**Ambient 宣告（第 97–99 行）**

```typescript
declare const tools: {
  task: (args: { description: string; subagent_type?: string }) => Promise<unknown>;
};
```

這段 `declare const` 只是給 TypeScript 編譯器看的型別提示。oxidase 在安裝時會把所有 TypeScript 型別宣告剝除，所以這段程式碼永遠不會真正進入 QuickJS 引擎。實際執行時，`tools` 是由 `CodeInterpreterMiddleware` 的 PTC 層注入為全域變數。

---

## 五、這個範例展示的 Deep Agents 能力

`repl_swarm` 一次示範了三種 Deep Agents 的核心能力，而且三者彼此疊加：

### 🧩 Skills as Code
Skill 不只是自然語言的 prompt 描述，它可以附帶可執行的 TypeScript 模組（透過 `metadata.entrypoint` 宣告）。這讓 skill 從「描述 agent 能做什麼」升級為「真正擴充 agent 的能力」——agent 在 REPL 裡可以用 `import` 拉進一段經過測試、可重複使用的程式碼，就像引入 npm 套件一樣。

### 🖥️ REPL / Code Interpreter
`CodeInterpreterMiddleware` 給了 agent 一個內嵌的 QuickJS 執行環境。Agent 可以在 `eval` 工具裡執行 TypeScript/JavaScript，這讓它能做複雜的資料操作、流程控制，甚至——如這個範例——呼叫並行編排邏輯，而不需要把這些邏輯寫死在 Python 裡。

### ⚡ Subagent 並行（Fan-Out）
透過 PTC 層把 `task` 工具暴露給 REPL，`runSwarm()` 可以在純 TypeScript 裡以 worker-pool 模式並行派發多個 subagent。每個 subagent 各自執行自己的任務，彼此獨立，結果收集後依序回傳。

---

## 六、怎麼跑

```bash
# 安裝依賴
cd docs/deepagents/examples/repl_swarm
uv sync

# 執行預設任務（並行寫三個數字到三個路徑）
uv run python swarm_agent.py

# 自訂任務
uv run python swarm_agent.py "Use the swarm skill to summarize these files: ..."

# 指定不同模型
uv run python swarm_agent.py --model claude-opus-4-5
```

預設任務會要求 agent「並行地把數字 1、2、3 分別寫進 `/tmp_swarm/a`、`/tmp_swarm/b`、`/tmp_swarm/c`」。因為 backend 走 `StateBackend`，這些檔案只存在於 agent state 裡，執行完畢自動清理，不會污染 host 磁碟。

執行成功後，終端輸出會顯示各 message 的型別與內容，可以看到 agent 的思考過程、REPL eval 的程式碼，以及最終的 `SwarmSummary`。

---

## 七、延伸方向

理解了 `repl_swarm` 之後，有幾個自然的延伸方向：

**調整並行度**：把 `concurrency` 從預設的 5 改到 10（上限），適合子任務輕量且彼此完全獨立的場景。記住：subagent 的呼叫不是免費的，每次呼叫都會消耗 token 和 API quota。

**豐富任務結構**：目前每個任務只有 `description` 字串和選填的 `subagentType`。若子任務需要傳遞結構化的上下文（例如一個 JSON 物件），可以修改 `SwarmTask` interface，把額外資訊序列化進 `description` 字串，或者擴充 `tools.task` 的參數。

**錯誤重試**：目前 `runSwarm` 的失敗是一次性捕捉，沒有重試機制。可以在 worker 的 `try/catch` 裡加入指數退避重試邏輯，讓短暫性錯誤（例如 rate limit）能自動恢復。

**進度回報**：`runSwarm` 目前在所有任務完成後才回傳。對於長時間執行的 swarm，可以加入 streaming callback（例如每完成一個任務就呼叫一次回呼函式），讓呼叫端能即時顯示進度。

**組合多個 skill**：這個範例只掛載了 `swarm` 一個 skill，但 `skills=["/skills/"]` 接受的是目錄，可以放多個 skill。Agent 可以在同一次 REPL 會話裡同時 `import` 多個 skill，組合出更複雜的編排模式。

---

## 八、小結

`repl_swarm` 用不到 200 行程式碼示範了一個違反直覺但非常優雅的架構決策：**把並行編排邏輯從 Python 後端移進 TypeScript skill，讓 agent 在需要時動態載入並執行**。

這帶來三個好處：

1. **Python driver 保持薄**——`swarm_agent.py` 不含任何 swarm 邏輯，只是把零件接在一起。要換掉 swarm 策略，只需更新 skill，不需要動 host 程式碼。
2. **Skill 可版本化、可測試、可重用**——`index.ts` 是純 TypeScript，可以單獨寫單元測試，可以被多個不同的 agent 共享。
3. **Agent 的能力是可組合的**——今天用 `runSwarm`，明天可能需要一個 `runPipeline`（有依賴順序的串行+並行混合）。只要寫成 skill，掛上去就行。

從更大的視角看，這正是 Deep Agents 架構的精神所在：**agent 不是一個固定功能的黑盒，而是一個可以動態擴充能力的執行環境**。Skill 就是這個擴充機制的實體——程式碼、說明、型別定義，全部打包在一起，需要時拉進來，不需要時佔用零資源。

---

*本文基於 `docs/deepagents/examples/repl_swarm/` 下的實際程式碼，所有函式名稱、參數與行為均與原始碼一致。*
