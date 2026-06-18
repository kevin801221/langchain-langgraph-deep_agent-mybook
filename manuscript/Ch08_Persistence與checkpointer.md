# Ch 8　Persistence 與 checkpointer：讓 agent 有記憶、可回復

> **本章目標**
>
> - 理解 LangGraph 的 persistence 層：用 checkpointer 在「每個 super-step」存下一份 state 快照（checkpoint）。
> - 學會最關鍵的兩件事：`compile(checkpointer=...)` 與在 config 裡帶 `thread_id`。
> - 用 `get_state` / `get_state_history` 檢視 thread 的當前與歷史狀態，讀懂 `StateSnapshot`。
> - 看見 persistence 如何一次解鎖四個能力：短期記憶、human-in-the-loop、time travel、fault tolerance。
>
> **使用版本**：`langgraph` 1.x。配套 notebook 的 checkpointer 範例**不需 API key 即可跑**。

---

## 8.1 一個 checkpointer，解鎖四個能力

Ch1 我們手刻 agent 時，列了一串「框架沒幫你做」的缺口，其中第一個就是 persistence。LangGraph 內建了 persistence 層：**當你用一個 checkpointer 編譯圖，它會在執行的每一個步驟存下一份 state 快照**。這一個機制，同時解鎖了四個原本要自己苦做的能力：

- **短期記憶（memory）**：同一個對話的後續訊息可以送進同一個 thread，agent 自動記得先前的往來。
- **Human-in-the-loop**：因為狀態被存著，人可以在任意時點檢視、中斷、核可，agent 之後再從那裡繼續（Ch12 詳談）。
- **Time travel**：可以重播（replay）過去的執行、或從任意 checkpoint 分岔（fork）出另一條路線來除錯（Ch12）。
- **Fault tolerance**：某個節點掛了，可以從上一個成功的 step 重啟，不必整個重跑（Ch9）。

換句話說，**persistence 是 Part I 後半很多功能的共同地基**。這章先把地基打好。

## 8.2 最小範例：checkpointer + thread_id

開啟 persistence 只要兩步：編譯時傳一個 checkpointer，執行時在 config 裡帶一個 `thread_id`。開發階段用 `InMemorySaver`（存在記憶體）最方便：

```python
from typing import Annotated
from typing_extensions import TypedDict
from operator import add
from langgraph.graph import StateGraph, START, END
from langgraph.checkpoint.memory import InMemorySaver

class State(TypedDict):
    foo: str
    bar: Annotated[list[str], add]

def node_a(state: State): return {"foo": "a", "bar": ["a"]}
def node_b(state: State): return {"foo": "b", "bar": ["b"]}

builder = StateGraph(State)
builder.add_node(node_a)
builder.add_node(node_b)
builder.add_edge(START, "node_a")
builder.add_edge("node_a", "node_b")
builder.add_edge("node_b", END)

checkpointer = InMemorySaver()                       # ← 1. 準備一個 checkpointer
graph = builder.compile(checkpointer=checkpointer)   # ← 編譯時掛上去

config = {"configurable": {"thread_id": "1"}}        # ← 2. 執行時帶 thread_id
graph.invoke({"foo": "", "bar": []}, config)
```

**`thread_id` 是這一切的主鍵。** checkpointer 用它來存取、回復狀態。少了它，checkpointer 無法存狀態、也無法在 interrupt 之後 resume。一個 thread 就是「一連串 run 累積下來的狀態」——你可以把它想成「一段對話的 ID」。

## 8.3 checkpoint 到底是什麼

**checkpoint 是某個時間點 thread 狀態的快照**，用一個 `StateSnapshot` 物件表示。關鍵規則承接 Ch7：**LangGraph 在每個 super-step 邊界各存一個 checkpoint。**

以 8.2 那個 `START → node_a → node_b → END` 的圖為例，跑完之後你會剛好得到 **4 個 checkpoint**：

1. 空 checkpoint，下一個要跑的是 `START`。
2. 帶使用者輸入 `{'foo': '', 'bar': []}`，下一個要跑 `node_a`。
3. 帶 `node_a` 的輸出 `{'foo': 'a', 'bar': ['a']}`，下一個要跑 `node_b`。
4. 帶 `node_b` 的輸出 `{'foo': 'b', 'bar': ['a', 'b']}`，沒有下一個節點（完成）。

注意第 4 個 checkpoint 的 `bar` 是 `['a', 'b']`——兩個節點的輸出都在，因為 `bar` 有累加 reducer（Ch6）。這個「每步一快照」的特性，正是 time travel 能「回到某一步」的基礎：**你只能從 checkpoint（也就是 super-step 邊界）resume**。

> 進階補充：除了 super-step 級的 checkpoint，LangGraph 還會在「節點（task）級」存 pending writes。同一 super-step 裡某個節點失敗時，已完成節點的寫入已經是持久的，resume 時不必重跑——這正是 Ch9 fault tolerance 的底層機制。

## 8.4 檢視狀態：get_state 與 get_state_history

存了狀態，當然要能讀。指定 `thread_id`，用 `get_state(config)` 取得**最新**的 `StateSnapshot`：

```python
config = {"configurable": {"thread_id": "1"}}
snapshot = graph.get_state(config)
```

`StateSnapshot` 的重要欄位（除錯時你會天天看）：

| 欄位 | 說明 |
|---|---|
| `values` | 此 checkpoint 的 state channel 值。 |
| `next` | 接下來要執行的節點名稱（tuple）。空的 `()` 代表圖已完成。 |
| `config` | 含 `thread_id`、`checkpoint_ns`、`checkpoint_id`。 |
| `metadata` | 執行 metadata：`source`（`input`/`loop`/`update`）、`writes`（節點輸出）、`step`（super-step 計數）。 |
| `created_at` | 此 checkpoint 建立的時間戳。 |
| `parent_config` | 上一個 checkpoint 的 config，第一個 checkpoint 為 `None`。 |
| `tasks` | 此步要執行的 task，含 `id`、`name`、`error`、`interrupts` 等。 |

要拿**整段歷史**，用 `get_state_history(config)`，它回傳一串 `StateSnapshot`，**最新的在最前面**：

```python
history = list(graph.get_state_history(config))
```

有了歷史，你能精準定位特定 checkpoint，這在除錯與 time travel 時極有用：

```python
# 找「node_b 還沒跑」的那個 checkpoint
before_node_b = next(s for s in history if s.next == ("node_b",))
# 用 step 編號找
step_2 = next(s for s in history if s.metadata["step"] == 2)
# 找出 interrupt 發生的 checkpoint
interrupted = next(s for s in history if s.tasks and any(t.interrupts for t in s.tasks))
```

## 8.5 短期記憶：同一個 thread 累積對話

persistence 最日常的用途，就是**對話記憶**。把後續訊息送進**同一個 `thread_id`**，agent 就會記得先前的往來——因為每一輪的 state（含 message 歷史）都被存進那個 thread 了：

```python
config = {"configurable": {"thread_id": "user-42"}}

graph.invoke({"messages": [{"role": "user", "content": "我叫 Kevin"}]}, config)
# … 同一個 thread 的下一輪 …
graph.invoke({"messages": [{"role": "user", "content": "我剛剛說我叫什麼？"}]}, config)
# agent 能答出 "Kevin"，因為前一輪的對話留在這個 thread 的 checkpoint 裡
```

換一個 `thread_id`，就是換一段全新、互不相干的對話。這就是「短期記憶」——它**綁在 thread 上**。注意：這還不是「跨對話、跨 thread」的長期記憶（那要用 store，留到 Ch14）。

## 8.6 上 production：換一個持久的 checkpointer

`InMemorySaver` 很適合開發與測試，但它存在記憶體裡，程式一關狀態就沒了。上 production 要換成持久後端，例如 `SqliteSaver`（小型、單機）或 `PostgresSaver`（正式環境）。好消息是：**換 checkpointer 不用動你的圖邏輯**，只換編譯時傳進去的那個物件。

```python
# 開發
from langgraph.checkpoint.memory import InMemorySaver
graph = builder.compile(checkpointer=InMemorySaver())

# production（示意；PostgresSaver 來自 langgraph-checkpoint-postgres）
# from langgraph.checkpoint.postgres import PostgresSaver
# graph = builder.compile(checkpointer=PostgresSaver(...))
```

還有一個更省事的選項：如果你用 **Agent Server**（Part IV 的部署主題）部署，它會**自動**幫你處理 checkpointing，你完全不必手動配置 checkpointer。

---

## 常見坑（Pitfalls）

- **掛了 checkpointer 卻忘了帶 `thread_id`**：沒有 `thread_id`，checkpointer 不知道要存／讀哪一段，記憶與 resume 都會失效。
- **以為 `InMemorySaver` 能撐 production**：它一關機就清空。正式環境用 SqliteSaver / PostgresSaver，或交給 Agent Server。
- **想用一個 `thread_id` 服務所有使用者**：那會把所有人的對話混在一起。一個使用者（或一段對話）對應一個 thread。
- **混淆短期記憶與長期記憶**：checkpointer 給的是「同一 thread 內」的短期記憶。要跨 thread 記住使用者偏好，需要 store（Ch14）。
- **以為能從任意一行 resume**：你只能從 checkpoint（super-step 邊界）resume，不是任意程式碼行。這影響 Ch9 的設計原則。

## 本章小結

LangGraph 的 persistence 層靠 **checkpointer** 在**每個 super-step** 存下一份 `StateSnapshot`（checkpoint），組織成 **thread**。開啟它只要：`compile(checkpointer=...)` + 執行時帶 `thread_id`（這是存取狀態的主鍵）。你能用 `get_state` 看最新狀態、`get_state_history` 看整段歷史，並讀懂 `StateSnapshot` 的 `values`／`next`／`metadata`／`tasks` 等欄位。一個 checkpointer 同時解鎖短期記憶（同 thread 累積對話）、human-in-the-loop、time travel 與 fault tolerance 四個能力。開發用 `InMemorySaver`，production 換 Sqlite/Postgres 或交給 Agent Server——而圖邏輯一行都不用改。地基打好了，下一章我們就用它來做「長任務不怕中斷」的 durable execution，以及節點失敗時的 retry、timeout、error handling。

## 延伸閱讀 / 練習

1. **數 checkpoint**：把 8.2 的圖跑起來，用 `list(graph.get_state_history(config))` 印出歷史，確認剛好 4 個 checkpoint，並對照每個的 `next` 與 `values`。
2. **兩段對話**：用兩個不同的 `thread_id` 各跑一段對話，確認它們互不干擾；再用同一個 `thread_id` 連續兩輪，確認記憶被保留。
3. **定位 checkpoint**：用 8.4 的 `next(...)` 技巧，找出「`node_b` 即將執行」的那個 snapshot，印出它的 `values`。
4. **思考題**：為什麼「只能從 super-step 邊界 resume」這件事，對 Ch9 要談的「把副作用包進 task」很重要？先想想，下一章驗證你的直覺。
