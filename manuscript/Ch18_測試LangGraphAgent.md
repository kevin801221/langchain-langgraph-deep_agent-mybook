# Ch 18　測試 LangGraph agent

> **本章目標**
>
> - 學會替 LangGraph agent 寫單元測試：整體執行、個別節點、部分路徑。
> - 掌握三個關鍵技巧：每個測試前重建並編譯圖、用 `graph.nodes["x"].invoke` 測單一節點、用 persistence 模擬「從中途開始、到中途停止」。
> - 學會 mock 掉 LLM 與工具，讓測試**快速、確定、不花錢**。
> - 建立「改了不會壞」的安全網——這是把 agent 維護下去的基本功。
>
> **使用版本**：`langgraph` 1.x、`pytest`。配套 notebook **不需 API key**（測試本來就該把 LLM mock 掉）。

---

## 18.1 為什麼 agent 特別需要測試

agent 是「會自己決定走幾步」的非確定系統，又常常依賴 state、外部工具、LLM。這讓它**特別容易在你改一個 prompt、加一個節點後悄悄壞掉**。測試就是你的安全網：每次改動後跑一遍，確保既有行為沒退化。

本章聚焦 LangGraph 特有的測試模式（自訂圖結構）。若你用的是 Part II 的 `create_agent` 高階抽象，另有對應的測試方式（Ch19 起）。

先裝 pytest：

```bash
uv add --dev pytest      # 或 pip install -U pytest
```

## 18.2 起手式：每個測試前重建並編譯圖

因為 agent 依賴 state，一個好習慣是：**把建圖寫成一個函數，每個測試裡重新建、用全新的 checkpointer 編譯。** 這確保測試之間互不汙染。

```python
import pytest
from typing_extensions import TypedDict
from langgraph.graph import StateGraph, START, END
from langgraph.checkpoint.memory import MemorySaver

def create_graph() -> StateGraph:
    class MyState(TypedDict):
        my_key: str
    g = StateGraph(MyState)
    g.add_node("node1", lambda s: {"my_key": "hello from node1"})
    g.add_node("node2", lambda s: {"my_key": "hello from node2"})
    g.add_edge(START, "node1")
    g.add_edge("node1", "node2")
    g.add_edge("node2", END)
    return g

def test_basic_agent_execution():
    graph = create_graph().compile(checkpointer=MemorySaver())
    result = graph.invoke({"my_key": "initial"}, config={"configurable": {"thread_id": "1"}})
    assert result["my_key"] == "hello from node2"   # 跑完整條，驗證最終 state
```

`create_graph()` 回傳「未編譯的 builder」，每個測試自己 `compile` 並帶一個新的 `MemorySaver`——乾淨、隔離。

## 18.3 測單一節點：graph.nodes["x"].invoke

整體測試之外，你常想**單獨測一個節點**的邏輯。編譯後的圖透過 `graph.nodes` 暴露每個節點，可以直接 invoke（這會**繞過 checkpointer**，純測該節點的函數）：

```python
def test_individual_node():
    graph = create_graph().compile(checkpointer=MemorySaver())
    result = graph.nodes["node1"].invoke({"my_key": "initial"})   # 只跑 node1
    assert result["my_key"] == "hello from node1"
```

這對「節點裡有複雜邏輯」特別有用——你不必跑整條流程，就能精準驗證單一節點對各種輸入的行為。

## 18.4 測部分路徑：用 persistence 模擬中途

大型圖你常只想測「中間某一段」，而不是每次都從頭跑到尾。LangGraph 的 persistence 機制讓你模擬「agent 暫停在某節點前、跑到某節點後再停」。步驟：

1. 用 checkpointer（`InMemorySaver` 即可）編譯。
2. 用 `update_state` 並帶 `as_node=` 設成「你想開始那段的前一個節點」，把狀態擺到那個位置。
3. 用同一個 `thread_id` invoke，並帶 `interrupt_after=` 設成「你想停下的節點」。

```python
def test_partial_execution():
    graph = create_graph().compile(checkpointer=MemorySaver())
    config = {"configurable": {"thread_id": "1"}}
    # 假裝 node1 已經跑完了：把狀態設成「node1 的輸出」
    graph.update_state(config, {"my_key": "set by node1"}, as_node="node1")
    # 只往下跑到 node2 就停
    result = graph.invoke(None, config, interrupt_after=["node2"])
    assert result["my_key"] == "hello from node2"
```

這讓你能對「只有在某些前置狀態下才會走到的路徑」寫精準測試，而不必費力地從頭把 agent 推到那個狀態。（若某段邏輯複雜到值得獨立，也可考慮把它重構成 subgraph（Ch12）單獨測。）

## 18.5 Mock 掉 LLM 與工具：快、穩、不花錢

測試的鐵律：**別在測試裡真的呼叫 LLM 或外部 API。** 理由有三——慢、不確定（同樣輸入可能不同輸出，測試會 flaky）、花錢。做法是把 LLM／工具換成「假的、回傳固定值」的替身。

呼應 Ch5 的設計：因為節點只是「讀 state、回傳更新」的函數，你可以在測試裡注入假的 model/tool。例如把分類節點的 LLM 換成「永遠回 complex」的假函數，就能穩定測「complex 會走 escalate」這條路徑：

```python
def test_routing_to_escalate(monkeypatch):
    # 把會呼叫 LLM 的分類，換成回傳固定值的假實作
    def fake_classify(state):
        return {"category": "complex"}      # 假裝模型判定為 complex
    # …用 fake_classify 建圖，斷言它走到 escalate、reply 是升級訊息
```

實務上常用 `monkeypatch`（pytest 內建）或依賴注入（把 model 當參數傳進建圖函數）來替換。重點是：**測試驗證的是「你的圖邏輯」（路由、state 流動、條件分支），不是「LLM 聰不聰明」。** 後者交給 Ch36 的 eval。

> 區分清楚：**測試（test）** 驗證確定性的程式邏輯（這章）；**評估（eval）** 衡量非確定性的 LLM 輸出品質（Ch36）。兩者互補，別混為一談。

---

## 常見坑（Pitfalls）

- **測試裡真的呼叫 LLM**：慢、flaky、花錢。一律 mock 掉，測你的邏輯而非模型。
- **測試間共用同一個 compiled graph / checkpointer**：狀態互相汙染。每個測試重建圖、用新的 checkpointer。
- **只做端到端測試**：難定位是哪個節點壞了。善用 `graph.nodes["x"].invoke` 測單一節點、用 `update_state`+`interrupt_after` 測局部。
- **拿測試當 eval 用**：測試是「行為對不對」（確定性）；LLM 輸出「好不好」是 eval 的事（Ch36）。
- **忘了 `graph.nodes` 繞過 checkpointer**：單節點 invoke 不經過 persistence，這正是它能純測節點邏輯的原因——別期待它有記憶。

## 本章小結

agent 是非確定、依賴 state 的系統，特別容易悄悄壞掉，所以測試是維護它的基本功。三個關鍵模式：**每個測試前用函數重建圖、以新 checkpointer 編譯**（隔離）；用 **`graph.nodes["x"].invoke`** 單獨測一個節點（繞過 checkpointer）；用 **`update_state(as_node=...)` + `interrupt_after=...`** 模擬中途開始、中途停止，測部分路徑。最重要的鐵律是 **mock 掉 LLM 與工具**——測試驗證的是你的圖邏輯（路由、state、分支），快、穩、不花錢；LLM 輸出的品質是 eval（Ch36）的事，別混為一談。至此 **Part I 完結**：你已經能用純 LangGraph 從零打造一個有狀態、可回復、會串流、能暫停等人、模組化、具長短期記憶、能做 RAG 與 SQL、且寫好測試的 production-grade agent。下一章進入 Part II，看 LangChain 的 `create_agent` 與 middleware 如何把這些能力包裝成更好用的抽象——而你因為懂底層，將能隨時拆開它。

## 延伸閱讀 / 練習

1. **三種測試各一**：對 Ch5 的 email agent（離線版），各寫一個「端到端測試」「單節點測試」「路由測試（mock 分類）」（配套 notebook 有可跑版本，免 API key）。
2. **部分路徑**：用 `update_state(as_node=...)` + `interrupt_after` 對一個多節點圖，測「只跑中間兩個節點」。
3. **mock 工具**：把一個會打外部 API 的工具換成假替身，測「工具失敗時」你的錯誤處理（接 Ch9 的 retry/error handler）。
4. **思考題**：「測試 agent 走對了路徑」和「評估 agent 回答得好不好」差在哪？為什麼前者該用 test、後者該用 eval？
