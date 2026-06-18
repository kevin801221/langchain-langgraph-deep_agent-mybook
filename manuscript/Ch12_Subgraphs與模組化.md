# Ch 12　Subgraphs 與模組化：把大圖拆成可重用元件

> **本章目標**
>
> - 理解 subgraph——「被當成一個節點來用的圖」——以及它解決什麼問題。
> - 掌握兩種父子圖溝通方式：**直接把編譯好的 subgraph 當節點加入**（共享 state key）vs **在節點函數內呼叫 subgraph**（schema 不同，需轉換）。
> - 學會用 `stream(..., subgraphs=True)` 觀察 subgraph 內部的執行，讀懂 namespace。
> - 用模組化思維把複雜 agent 拆成可獨立開發、可重用的部分。
>
> **使用版本**：`langgraph` 1.x。配套 notebook **不需 API key**。

---

## 12.1 什麼是 subgraph，為什麼需要它

一個 **subgraph 就是「被當成另一張圖裡的一個節點」來使用的圖**。當你的 agent 變大，subgraph 帶來三個好處：

- **建立多代理系統**：每個 agent 是一張 subgraph，各有自己的 state 與邏輯。
- **重用一組節點**：把「檢索 + 摘要」這種常見組合封裝起來，多處共用。
- **分散式開發**：不同團隊各自負責一張 subgraph，只要遵守介面（input/output schema），父圖不需要知道 subgraph 的內部細節就能組裝。

這呼應 Ch4 的精神——把流程拆成「只做一件事」的單元。subgraph 把這個單元從「一個節點」升級成「一整張可重用的圖」。

## 12.2 兩種父子溝通方式

加入 subgraph 時，要先決定父圖與 subgraph 怎麼溝通。關鍵看它們**有沒有共享的 state key**：

| 模式 | 何時用 | 怎麼做 |
|---|---|---|
| **直接當節點加入** | 父圖與 subgraph **共享 state key**（讀寫同樣的 channel） | 把編譯好的 subgraph 直接傳給 `add_node`，**不必寫包裝函數** |
| **在節點內呼叫** | 父子 schema **不同**（沒有共享 key），或需要在兩者間轉換 state | 你寫一個包裝函數，把父 state 映射成 subgraph 輸入，再把 subgraph 輸出映射回父 state |

選擇的判準很簡單：**共享 state 就直接加；schema 不同就包一層轉換。**

## 12.3 模式一：在節點內呼叫 subgraph（schema 不同）

當父圖與 subgraph 的 schema 沒有共享 key，就在節點函數裡 `invoke` subgraph，並負責「進去前轉換、出來後轉回」。這在多代理系統裡很常見——你想讓每個 agent 有自己私有的訊息歷史：

```python
from typing_extensions import TypedDict
from langgraph.graph.state import StateGraph, START

# --- subgraph：自己的 schema ---
class SubgraphState(TypedDict):
    bar: str

def subgraph_node_1(state: SubgraphState):
    return {"bar": "hi! " + state["bar"]}

subgraph_builder = StateGraph(SubgraphState)
subgraph_builder.add_node(subgraph_node_1)
subgraph_builder.add_edge(START, "subgraph_node_1")
subgraph = subgraph_builder.compile()      # ← 編譯成可呼叫的 subgraph

# --- 父圖：不同的 schema（用 foo，不是 bar）---
class State(TypedDict):
    foo: str

def call_subgraph(state: State):
    subgraph_output = subgraph.invoke({"bar": state["foo"]})   # 進去前：foo → bar
    return {"foo": subgraph_output["bar"]}                     # 出來後：bar → foo

builder = StateGraph(State)
builder.add_node("node_1", call_subgraph)
builder.add_edge(START, "node_1")
graph = builder.compile()
```

`call_subgraph` 這個節點扮演「翻譯官」：把父圖的 `foo` 翻成 subgraph 要的 `bar`，跑完再翻回 `foo`。父圖完全不需要知道 subgraph 內部長什麼樣——這就是介面隔離的威力。

## 12.4 模式二：直接把 subgraph 當節點（共享 state）

如果父圖與 subgraph 共享 state key，你連包裝函數都不用寫，直接把編譯好的 subgraph 傳給 `add_node`：

```python
# 假設 subgraph 與父圖都用同一組 key（例如都讀寫 messages）
builder.add_node("my_subgraph", subgraph)   # ← 直接傳編譯好的 subgraph
builder.add_edge(START, "my_subgraph")
```

這時 subgraph 直接讀寫父圖的 channel，最省事。代價是兩者耦合在同一份 state 上——適合「同一團隊、緊密相關」的拆分；模式一則適合「需要隔離、各有私有狀態」的場景（例如多代理各自的對話歷史）。

## 12.5 看見 subgraph 內部：subgraphs=True

預設情況下，串流只會看到父圖層級的更新。想看 subgraph **內部**每一步，加上 `subgraphs=True`，輸出會帶 namespace（`ns`）告訴你事件來自哪張（子）圖：

```python
for chunk in graph.stream({"foo": "foo"}, subgraphs=True, version="v2"):
    if chunk["type"] == "updates":
        print(chunk["ns"], chunk["data"])
# () {'node_1': {'foo': 'hi! foo'}}                                  ← 父圖（ns 為空）
# ('node_2:<uuid>',) {'subgraph_node_1': {'baz': 'baz'}}             ← subgraph 內部
# ('node_2:<uuid>',) {'subgraph_node_2': {'bar': 'hi! foobaz'}}      ← subgraph 內部
# () {'node_2': {'foo': 'hi! foobaz'}}                               ← 回到父圖
```

`ns` 為空 tuple 代表父（根）圖，`"node_name:uuid"` 代表某個 subgraph。這也呼應 Ch8 提過的 `checkpoint_ns`——subgraph 的 checkpoint 有自己的 namespace。除錯多層圖時，`subgraphs=True` 是你看清「事件到底發生在哪一層」的關鍵開關。

---

## 常見坑（Pitfalls）

- **schema 不同卻直接把 subgraph 當節點加**：沒有共享 key 時直接 `add_node(subgraph)` 會對不上。schema 不同就用模式一（節點內呼叫 + 轉換）。
- **忘了 subgraph 要先 `compile()`**：傳給父圖或 `invoke` 的是「編譯好的」subgraph，不是 builder。
- **串流時看不到 subgraph 內部還以為它沒跑**：預設不顯示 subgraph 內部事件，加 `subgraphs=True` 才看得到。
- **過度拆 subgraph**：不是每組節點都值得變 subgraph。當它需要被「重用、隔離、或獨立開發」時才拆，否則徒增複雜度。

## 本章小結

subgraph 是「被當成節點使用的圖」，讓你建多代理系統、重用節點組、分散式開發。父子溝通有兩種：**共享 state key** 時直接把編譯好的 subgraph 傳給 `add_node`（最省事）；**schema 不同** 時在節點函數裡 `invoke` subgraph 並負責進出轉換（適合各有私有狀態的隔離場景）。用 `stream(..., subgraphs=True)` 可看見 subgraph 內部事件，`ns` 告訴你事件來自哪一層。模組化的判準是「需要重用、隔離或獨立開發」才拆，別為拆而拆。下一章，我們看 LangGraph 的另一條路——用 `@entrypoint` / `@task` 的 Functional API，以更貼近一般 Python 的方式寫出同樣具備 persistence、HITL、streaming 的 workflow。

## 延伸閱讀 / 練習

1. **封裝 subgraph**：把「檢索 + 摘要」兩個節點封成一張 subgraph，分別用模式一與模式二接進一個父圖，比較兩種寫法（配套 notebook 有可跑版本，免 API key）。
2. **觀察 namespace**：用 `subgraphs=True` 串流一個含 subgraph 的圖，把每個事件的 `ns` 印出來，標出哪些來自父圖、哪些來自 subgraph。
3. **多代理雛形**：用模式一讓兩個「各有私有 state」的 subgraph 當成兩個 agent，由父圖協調它們。
4. **思考題**：什麼情況你會選模式二（共享 state）而非模式一（隔離 state）？以「團隊協作」與「context 隔離」兩個角度各說一個理由。
