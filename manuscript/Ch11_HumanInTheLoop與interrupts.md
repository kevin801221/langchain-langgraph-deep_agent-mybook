# Ch 11　Human-in-the-loop、interrupts 與 time travel：暫停、審核、改寫

> **本章目標**
>
> - 學會用 `interrupt()` 在圖的任意位置暫停、等待外部輸入，再用 `Command(resume=...)` 繼續。
> - 理解 interrupt 為什麼一定要搭配 checkpointer 與 `thread_id`（呼應 Ch8）。
> - 掌握 HITL 的常見模式：審核關卡、review & edit、工具呼叫前攔截。
> - 學會 time travel 的兩招：**replay**（從過去 checkpoint 重跑）與 **fork**（改了狀態再分岔出另一條路）。
>
> **使用版本**：`langgraph` 1.x（v2 API 需 `langgraph>=1.1`）。配套 notebook **不需 API key**。

---

## 11.1 interrupt：在任意位置按下暫停鍵

Ch1 列的缺口裡，「人工介入」是 production agent 的關鍵一環：送一封信、刪一個檔、刷一筆款之前，最好先停下來等人點頭。LangGraph 用 `interrupt()` 做到這件事。

它的運作是這樣：**在節點裡呼叫 `interrupt(payload)`，圖會在那個確切位置暫停、用 persistence 層存下當前 state，然後無限期等待**，直到你用 `Command(resume=value)` 重新 invoke 它——這時 `value` 就成為節點內 `interrupt()` 的回傳值，節點接著往下跑。

跟「靜態 breakpoint」（只能停在某節點前/後）不同，interrupt 是**動態**的：可以放在程式碼任何地方、可以依條件觸發。

```python
from langgraph.types import interrupt

def approval_node(state: State):
    # 暫停，把問題拋給呼叫端，等待外部回應
    approved = interrupt("你核准這個動作嗎？")
    # resume 時，Command(resume=...) 的值會回到這裡
    return {"approved": approved}
```

用 `interrupt` 需要三樣東西，前兩樣你在 Ch8 已經會了：**一個 checkpointer**（存狀態，production 用持久後端）、**config 裡的 `thread_id`**（讓 runtime 知道要回復哪一段），以及**在你想暫停處呼叫 `interrupt()`**（payload 必須可 JSON 序列化）。

> 💡 **冷知識** —— `interrupt()` 能「在程式碼任意位置」暫停，靠的是 Python 的例外機制：呼叫它會丟出一個特殊例外，被 Pregel runtime（Ch7）的 super-step 邊界接住、寫進 checkpoint。也正因此，**只能從 super-step 邊界 resume**——這也就是 11.3 為什麼「節點會從頭重跑」的根源。HITL 不是另一套新東西，它是 Ch7 + Ch8 的延伸。

## 11.2 暫停與回復的完整流程

把暫停、檢視、回復串起來看（v2 API，`langgraph>=1.1`）：

```python
from langgraph.types import Command

config = {"configurable": {"thread_id": "thread-1"}}

# 第一次 invoke：撞到 interrupt 就暫停
result = graph.invoke({"input": "data"}, config=config, version="v2")

# result 是 GraphOutput，.interrupts 帶著你傳給 interrupt() 的 payload
print(result.interrupts)
# > (Interrupt(value='你核准這個動作嗎？'),)

# 拿到人的回應後，用 Command(resume=...) 回復
# resume 的值會成為節點裡 interrupt() 的回傳值
graph.invoke(Command(resume=True), config=config, version="v2")
```

幾個要點：**回復時必須用「當初暫停時的同一個 `thread_id`」**（checkpointer 靠它找回狀態）；`Command(resume=...)` 的值會成為 `interrupt()` 的回傳值；payload 與 resume 值都可以是任何 JSON-serializable 的東西。

> v1（預設）API 略有不同：暫停的 payload 在 `result["__interrupt__"]`，回復一樣用 `graph.invoke(Command(resume=True), config)`。本書範例用 v2，與 Ch10 的 streaming 一致。

## 11.3 最重要的一個坑：resume 會「重跑整個節點」

這是 HITL 最容易踩、且最反直覺的一點，請畫線：**resume 時，節點是「從頭」重新執行的，不是從 `interrupt()` 那一行接續。** 也就是說，`interrupt()` 之前的程式碼會再跑一次。

```python
def risky_node(state: State):
    log_to_db("about to ask")     # ⚠️ resume 時這行會再執行一次！
    answer = interrupt("approve?")
    do_something(answer)
    return {...}
```

如果 `interrupt()` 前面有副作用（寫 log、發請求、扣款），resume 時它們會重複發生。解法呼應 Ch9 的鐵則：**把副作用做成冪等，或移到 `interrupt()` 之後**。理解這點，你就不會被「為什麼我的 log 出現兩次」搞瘋。

另外一個邊界：`Command(resume=...)` 是**唯一**適合當作 `invoke()`／`stream()` 輸入的 Command 形式。其他 Command 參數（`update`、`goto`、`graph`）是設計給「節點函數的回傳值」用的，別拿來當 invoke 的輸入；要繼續多輪對話，傳普通的輸入 dict 即可。

> 🩸 **血淚教訓** —— 我曾經在 production 替一個審核 agent 加 interrupt，沒注意 `interrupt()` 前一行 `log_to_db("waiting for approval")` 是有副作用的。結果每次審核者點核准，DB 都收到**兩筆** waiting log——一筆來自初次執行、一筆來自 resume 時的 replay。一週後才被 DBA 抓到，補上 idempotency key 才止血。**寫 HITL 前，先把 `interrupt()` 之前的每一行掃過一次**，能冪等就冪等、不能冪等就移到 interrupt 之後。這個習慣後來救我很多次。

## 11.4 常見 HITL 模式

interrupt 解鎖的核心能力是「暫停等外部輸入」，常見用途有：

- **審核工作流（approval）**：在執行關鍵動作（API 呼叫、改資料庫、金流）前暫停。
- **review & edit**：讓人檢視、修改 LLM 的輸出或工具呼叫後再繼續。
- **攔截工具呼叫**：在執行工具前暫停，審視甚至改寫工具的參數。
- **驗證人類輸入**：在進入下一步前暫停，確認輸入合法。

把 Ch5 的 email agent 加上審核關卡，就是「在 `送出回覆` 節點前 `interrupt('這封回覆可以送嗎？')`」——人核准才送，否則改寫或退回。這正是 Ch4 五步思考法裡「使用者輸入步驟」的落地。

> ⏱ **三秒判斷** —— `interrupt` vs 條件邊：
> - 「下一步該不該等真人決定？」 → **是 → `interrupt`**。
> - 「只是依 state 路徑分岔、不必等人」 → **條件邊**（Ch5）。
> 常被混淆：「依分類分流」是條件邊的事，不該寫成 interrupt。

## 11.4.5 Review-and-edit：讓人不只能核准，還能直接改寫草稿

approve/reject 只是 HITL 最陽春的形式。production 更常見的是 **review-and-edit**——人看了 agent 草稿後，**直接改寫**再放行，而不是退回讓 agent 重做一次。實現很單純：讓 `Command(resume=...)` 帶結構化的決策，由節點依不同決策改寫 state：

```python
from langgraph.types import interrupt, Command

def review(state):
    decision = interrupt({
        "draft": state["draft"],
        "action": "approve | edit | reject",
    })
    # 三種決策：dict (改寫) / True (核准) / False (退回)
    if isinstance(decision, dict) and decision.get("action") == "edit":
        return {"draft": decision["new_draft"], "approved": True}
    if decision is True:
        return {"approved": True}
    return {"approved": False}
```

回復時三種輸入：
- `Command(resume=True)` —— 直接核准。
- `Command(resume={"action": "edit", "new_draft": "…"})` —— 改寫後核准。
- `Command(resume=False)` —— 退回。

這個小模式把 HITL 從「同意/否決」升級成「人類也是 agent 的協作者」——人可以親手改、agent 接著走。配套 notebook 有可跑版本，三種情境直接跑給你看。

> 🪞 **對照閱讀 —— HITL 在三層怎麼做：**
>
> | 層 | 寫法 | 抽象高度 |
> |---|---|---|
> | **LangGraph**（本章） | 節點裡 `interrupt(payload)` + `Command(resume=...)` 回復 | 最赤裸、最可控 |
> | **LangChain `create_agent`**（Ch23） | 內建 `human_in_the_loop` middleware，整段攔截可在工具呼叫前彈出審核——你不必手寫 interrupt | middleware-wrapped |
> | **Deep Agents**（Ch37） | `create_deep_agent(interrupt_on={"edit_file": True, ...})`——對指定工具自動加審核關卡 | 配置宣告、開箱即用 |
>
> 同一個能力，三層分別用 **raw → middleware-wrapped → 配置宣告** 三種抽象高度。愈往上內建愈多、愈往下控制愈精——Ch2 那張地圖在這裡又兌現一次。

## 11.5 串流中處理 interrupt

互動式 agent 通常要邊串流 AI 回應、邊偵測 interrupt。用多個 stream mode + `subgraphs=True`（若有 subgraph），就能同時做到即時回饋與暫停處理：

```python
from langchain.messages import AIMessageChunk
from langgraph.types import Command

for chunk in graph.stream(initial_input,
                          stream_mode=["messages", "updates", "values"],
                          subgraphs=True, config=config, version="v2"):
    if chunk["type"] == "messages":
        msg, _ = chunk["data"]
        if isinstance(msg, AIMessageChunk) and msg.content:
            display(msg.content)                       # 即時顯示 AI 文字
    elif chunk["type"] == "values" and chunk.get("interrupts"):
        info = chunk["interrupts"][0].value            # 偵測到暫停
        user_response = get_user_input(info)
        initial_input = Command(resume=user_response)  # 準備回復
        break
```

關鍵：v2 下 pending interrupt 出現在 `values` part 的 `chunk["interrupts"]`；偵測到 subgraph 內的 interrupt 需要 `subgraphs=True`。

## 11.6 Time travel：replay 與 fork

因為每個 super-step 都有 checkpoint（Ch8），LangGraph 能做 time travel——這在除錯與「探索另一種可能」時極有價值。兩招：

**Replay（重播）**：用某個過去 checkpoint 的 config 重新 invoke，從那點重跑。checkpoint 之前的節點不再執行（結果已存），之後的節點會**重跑**（LLM、API、interrupt 都會再次觸發，可能產生不同結果）。

```python
history = list(graph.get_state_history(config))   # 倒序
before_joke = next(s for s in history if s.next == ("write_joke",))
replay_result = graph.invoke(None, before_joke.config)   # 從該 checkpoint 重跑
# write_joke 重新執行，它前面的節點不會
```

**Fork（分岔）**：在過去某 checkpoint 上 `update_state` 改寫狀態，建立一條新分支，再 `invoke(None, fork_config)` 繼續。**注意 `update_state` 不是「回滾」——它建立一個新 checkpoint 從那點分岔，原本的歷史完整保留。**

```python
before_joke = next(s for s in history if s.next == ("write_joke",))
fork_config = graph.update_state(before_joke.config, values={"topic": "chickens"})
fork_result = graph.invoke(None, fork_config)   # 用新的 topic 重跑 write_joke
print(fork_result["joke"])   # 關於 chickens 的笑話，而非原本的 socks
```

replay 用來「重現問題」，fork 用來「如果當時換個輸入會怎樣」。兩者都建立在 checkpoint 之上——這再次說明 Ch8 的 persistence 是多少功能的共同地基。

---

## 常見坑（Pitfalls）

- **沒掛 checkpointer 就用 `interrupt()`**：interrupt 靠 persistence 存狀態才能暫停與回復。沒 checkpointer + `thread_id`，它無法運作。
- **以為 resume 從 `interrupt()` 那行接續**：節點會從頭重跑，`interrupt()` 前的副作用會重複。把副作用做成冪等或移到 interrupt 之後。
- **resume 時換了 `thread_id`**：那會找不到暫停的狀態（等於開一段新對話）。必須用同一個 thread_id。
- **把 `Command(update=...)` 當 invoke 輸入**：只有 `Command(resume=...)` 適合當 invoke 輸入；其餘 Command 是節點回傳用的。
- **以為 `update_state`（fork）會回滾歷史**：它是「另開分支」，原歷史保留。別期待它清掉舊 checkpoint。

## 本章小結

`interrupt()` 讓你在圖的任意位置動態暫停、等待外部輸入，再用 `Command(resume=value)` 回復——`value` 會成為 `interrupt()` 的回傳值。它必須搭配 checkpointer 與 `thread_id`（呼應 Ch8）。最關鍵的坑是：**resume 會從節點開頭重跑**，所以 interrupt 前的副作用要冪等或後移。HITL 的典型用途有審核、review & edit、攔截工具呼叫；串流時用 `subgraphs=True` 並從 `values` part 的 `interrupts` 欄位偵測暫停。最後，time travel 提供 **replay**（從過去 checkpoint 重跑）與 **fork**（改狀態後分岔），兩者都站在 checkpoint 這個地基上。你的 agent 現在能在關鍵時刻停下來等人、也能回到過去探索另一種可能。下一章，我們把大圖拆成可重用的 subgraph。

## 延伸閱讀 / 練習

1. **加審核關卡**：替 Ch5 的 email agent 在「送出回覆」前插一個 `interrupt('這封可以送嗎？')`，分別用 `Command(resume=True/False)` 回復，觀察兩種結果（配套 notebook 有可跑版本，免 API key）。
2. **重跑副作用**：在 interrupt 前放一行 `print("before interrupt")`，resume 後數一數它印了幾次，親身體會「節點重跑」。
3. **fork 探索**：用 11.6 的 fork 技巧，對同一個 checkpoint 改兩種不同的 `topic`，比較兩條分支的輸出，並確認原始歷史仍在。
4. **思考題**：你的「送出回覆」節點裡，`interrupt()` 之前若有「寫一筆送出紀錄到 DB」，resume 後會發生什麼？該怎麼改？
