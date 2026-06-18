# Ch 9　Durable execution 與 fault tolerance：長任務不怕中斷

> **本章目標**
>
> - 理解 durable execution：把進度存進耐久層，讓 workflow 能暫停、之後從斷點 resume——而**只要你用了 checkpointer，就已經有了它**。
> - 掌握 durable execution 的鐵則：resume 會「replay」，所以你的程式必須**確定性（deterministic）且冪等（idempotent）**，把副作用包進 task / node。
> - 認識三種 durability mode（`exit` / `async` / `sync`）的效能與安全取捨。
> - 用 `RetryPolicy`、per-node timeout、error handler 三件工具，組出節點層級的 fault tolerance。
>
> **使用版本**：`langgraph` 1.x（per-node timeout 與 node-level error handler 需 `langgraph>=1.2`）。

---

## 9.1 你其實已經有 durable execution 了

durable execution（耐久執行）是一種技術：process 在關鍵點存下進度，於是能暫停、稍後從**離開的地方**繼續，而不必重做先前的步驟——哪怕中間隔了一週。這對兩種情境特別重要：human-in-the-loop（讓人檢視、驗證、修改後再續跑），以及長時間任務（可能遇到中斷或錯誤，例如 LLM 呼叫 timeout）。

好消息：**LangGraph 的 persistence 層（Ch8）本身就提供了 durable execution。** 只要你用 checkpointer 編譯圖，你就已經能在任意點暫停、resume——即使中間發生了系統故障或人工介入。這也是為什麼 Ch8 是這一切的地基。

## 9.2 要件：三件事

要用好 durable execution，你需要：

1. **啟用 persistence**：指定一個 checkpointer（Ch8）。
2. **執行時帶 `thread_id`**：用它追蹤這個 workflow 實例的執行歷史。
3. **把不確定／有副作用的操作包進 task 或 node**：確保 workflow resume 時，這些操作不會被重複執行，而是從 persistence 層取回先前的結果。

前兩點你在 Ch8 已經會了。第三點是本章的關鍵心法，下一節展開。

## 9.3 鐵則：resume 會 replay，所以要確定性與冪等

這是最容易出事、也最該理解的一點：**當你 resume 一個 run，程式碼不是從「當初停下的那一行」繼續，而是會找到一個合適的起點，把從那裡到中斷點之間的步驟全部 replay 一遍。**

這個 replay 行為帶來兩個必須遵守的設計原則：

**原則一：把不確定性與副作用包進 task / node。** 想像一個節點裡同時做了「呼叫 API」和「寫檔案」兩件有副作用的事，如果中途崩潰、之後 replay，這兩件事可能被重做一次（重複扣款、重複寫入）。正確做法是**把每個有副作用的操作各包成一個 task**——這樣 resume 時，已完成的 task 會直接從 persistence 取回結果，不會重跑。同理，隨機數產生這類不確定操作也要包起來，確保 replay 時得到與當初相同的結果。

**原則二：盡量讓副作用冪等（idempotent）。** 冪等指「同一操作做一次和做多次，效果相同」。萬一某個 task 開始了卻沒成功完成，resume 會重跑它。用 idempotency key、或先檢查結果是否已存在，能避免非預期的重複——這對「會寫資料」的操作尤其關鍵。

一句話記住：**你寫的不是「跑一次」的腳本，而是「可能被 replay 很多次」的程式。** 把這個假設放在心裡，你就會自然地把副作用包好、做成冪等。

## 9.4 durability modes：效能與安全的旋鈕

LangGraph 提供三種 durability mode，讓你在「效能」與「資料一致性」之間調整。mode 愈耐久，額外開銷愈大。在任何執行方法上用 `durability=` 指定：

```python
graph.stream({"input": "test"}, durability="sync")
```

從最不耐久到最耐久：

- **`"exit"`**：只在圖執行**結束時**（成功、出錯、或因 HITL interrupt）才持久化。長任務效能最好，但中間狀態不存，**無法從執行中途的系統崩潰回復**。
- **`"async"`**：在下一步執行的同時**非同步**寫入。效能與耐久兼顧，但若 process 剛好在寫入途中崩潰，有小機率漏寫 checkpoint。
- **`"sync"`**：每一步開始前**同步**寫完 checkpoint。最高耐久（保證每個 checkpoint 都寫入），代價是一些效能開銷。

怎麼選？需要嚴格回復保證（金流、關鍵流程）用 `sync`；一般長任務用 `async`（多數情況的好預設）；極長、且能接受「崩潰就重來」的批次型任務可用 `exit` 換效能。

## 9.5 Fault tolerance：節點失敗時的三道防線

節點可能因為慢吞吞的外部 API、暫時性網路錯誤、或未處理的例外而失敗。LangGraph 給你三個可組合的機制：

- **Retries（重試）**：依例外型別與 backoff 設定，自動重跑失敗的嘗試。
- **Timeouts（逾時）**：限制單次嘗試最久能跑多久。
- **Error handling（錯誤處理）**：重試全部用盡後，跑一個回復函數。

它們的組合順序是固定的：節點拋出任何例外（包括 timeout 造成的 `NodeTimeoutError`）時，**先由 retry policy 決定是否重試；重試用盡後，才輪到 error handler**。

**Retries** 用 `RetryPolicy`，傳給 `add_node` 的 `retry_policy=`：

```python
from langgraph.types import RetryPolicy

builder.add_node("call_api", call_api, retry_policy=RetryPolicy(max_attempts=3))
```

預設行為（`default_retry_on`）會對「大多數例外」重試，但**不**重試這些（視為你的程式 bug，重試也沒用）：`ValueError`、`TypeError`、`ArithmeticError`、`ImportError`、`LookupError`、`NameError`、`SyntaxError`、`RuntimeError` 等；對 `requests`、`httpx` 這類 HTTP 函式庫，只在 **5xx** 狀態碼時重試。`RetryPolicy` 的常用參數：`max_attempts`（預設 3）、`initial_interval`（首次重試前秒數，預設 0.5）、`backoff_factor`（每次重試的倍率，預設 2.0）、`max_interval`（重試間隔上限，預設 128）、`jitter`（加隨機抖動，預設 True）、`retry_on`（自訂哪些例外要重試）。

要自訂重試邏輯，傳一個 callable 給 `retry_on`，並可沿用預設行為：

```python
from langgraph.types import RetryPolicy, default_retry_on

def custom_retry_on(exc: BaseException) -> bool:
    if isinstance(exc, MyCustomError):
        return False                  # 我這個自訂錯誤不要重試
    return default_retry_on(exc)      # 其餘沿用預設判斷

builder.add_node("call_api", call_api,
                 retry_policy=RetryPolicy(max_attempts=3, retry_on=custom_retry_on))
```

**一次設定全部節點**：與其在每個 `add_node` 重複寫，可用 `set_node_defaults` 一次套用到所有節點。（per-node timeout 與 node-level error handler 需 `langgraph>=1.2`。）

## 9.6 偵測重試狀態：用 execution_info 做 fallback

有時你想在「主要 API 一直失敗」時切到備援。節點可以透過 `runtime.execution_info` 讀到目前是第幾次嘗試：

```python
from langgraph.graph import StateGraph, START, END
from langgraph.runtime import Runtime
from langgraph.types import RetryPolicy
from typing_extensions import TypedDict

class State(TypedDict):
    result: str

def my_node(state: State, runtime: Runtime) -> State:
    if runtime.execution_info.node_attempt > 1:   # 不是第一次了 → 改打備援
        return {"result": call_fallback_api()}
    return {"result": call_primary_api()}

builder = StateGraph(State)
builder.add_node("my_node", my_node, retry_policy=RetryPolicy(max_attempts=3))
```

這把「重試」從單純的「再試一次相同的事」升級成「換個策略再試」，是 production agent 很實用的韌性技巧。

---

## 常見坑（Pitfalls）

- **在節點裡做不冪等的副作用、又沒包成 task**：resume replay 時會重做，造成重複扣款／重複寫入這類災難。每個副作用包一個 task，並盡量冪等。
- **以為 resume 是「從停下的那一行」繼續**：它是 replay。沒有這個認知，你會寫出 resume 後行為錯亂的程式。
- **對程式 bug 類例外期待重試生效**：`ValueError`、`TypeError` 等預設不重試（重試也只是再錯一次）。retry 是給暫時性故障用的。
- **無腦用 `sync` durability**：它最安全但最慢。依任務的回復需求選 mode，別讓不需要強一致的長任務白白吃效能。
- **timeout / error handler 在舊版不存在**：per-node timeout 與 node-level error handler 需要 `langgraph>=1.2`，確認你的版本。

## 本章小結

只要用了 checkpointer，你就已經擁有 **durable execution**：能暫停、能從斷點 resume。但代價是一條鐵則——**resume 會 replay**，所以你的程式必須確定性、冪等，並把不確定／副作用操作包進 task 或 node，避免重跑造成重複。透過 `durability` 參數（`exit`/`async`/`sync`）你能在效能與一致性間調節。節點層級的韌性靠三道防線：`RetryPolicy`（重試，預設不重試程式 bug 類例外、HTTP 只重試 5xx）、timeout、error handler，依「重試→用盡才 error handler」的固定順序組合；還能用 `runtime.execution_info.node_attempt` 在多次失敗後切換到備援。你的 agent 現在能撐過崩潰與暫時性故障了。下一章，我們處理另一個 production 必需品：把 agent 的思考與進度即時 streaming 給使用者。

## 延伸閱讀 / 練習

1. **重試實測**：寫一個前兩次故意丟 `httpx`-style 5xx 例外、第三次成功的節點，配 `RetryPolicy(max_attempts=3)`，確認它最終成功（配套 notebook 有可跑版本）。
2. **不可重試的例外**：把上題的例外改成 `ValueError`，觀察它**不會**被重試而直接冒出來，體會「retry 是給暫時性故障、不是給 bug」。
3. **fallback 切換**：用 9.6 的 `node_attempt` 寫一個「第一次打主 API、之後打備援」的節點。
4. **思考題**：你的某個節點會「對外發一封 email」。為什麼這個操作特別需要冪等？若不冪等，replay 會發生什麼事？該怎麼用 idempotency key 解決？
