# Ch 5　Graph API：StateGraph、節點與條件邊

> **本章目標**
>
> - 用 `StateGraph` 把 Ch4 設計的 email agent 流程，寫成一張**可執行**的圖。
> - 掌握四件核心操作：定義 state、`add_node`、`add_edge` 與 `add_conditional_edges`、`compile`。
> - 理解 `START` / `END` 兩個特殊節點，以及「條件邊」如何實現動態路由。
> - 學會用 `invoke` 執行圖、用 `stream_mode="updates"` 觀察它一步步怎麼走。
>
> **使用版本**：`langgraph` 1.x、`langchain-google-genai`。

---

## 5.1 StateGraph 的五步骨架

LangGraph 主要的圖類別是 `StateGraph`，它由你自訂的 State 型別參數化。建一張圖永遠是同一套節奏，正好對應 Ch4 的五步：

```python
from langgraph.graph import StateGraph, START, END

builder = StateGraph(State)          # 1. 用你的 State 開一個 builder
builder.add_node("名字", 函數)        # 2. 加節點
builder.add_edge(START, "名字")       # 3. 加邊（固定或條件）
graph = builder.compile()            # 4. 編譯
graph.invoke({...})                  # 5. 執行
```

第 4 步的「編譯」做兩件事：對圖結構做基本檢查（例如沒有孤兒節點），以及讓你指定 runtime 參數（像 checkpointer、breakpoint——Ch8 起會用到）。**你必須先 compile 才能用這張圖。**

## 5.2 先定義 state

承接 Ch4 的設計，email agent 的 state 用一個 `TypedDict` 描述。先用最精簡的三個欄位起步：

```python
from typing_extensions import TypedDict

class State(TypedDict):
    email: str        # 原始信件內容（輸入）
    category: str     # 分類結果（中間產物）
    reply: str        # 草擬的回覆（輸出）
```

注意這完全呼應 Ch4 的原則：state 裡放的是**原始資料與中間產物**，不是組好的 prompt。`category` 是上游節點寫、下游節點讀的「中間白板欄位」。

## 5.3 寫節點：函數，讀 state、回傳更新

節點就是 Python 函數。它接收當前 state，回傳一個 dict 代表「要對 state 做的更新」——**只回傳要改的欄位即可，不必回傳整個 state**。

```python
from langchain_google_genai import ChatGoogleGenerativeAI

llm = ChatGoogleGenerativeAI(model="gemini-2.5-flash")

def classify(state: State) -> dict:
    """讀信件，分類成 simple / complex 兩類之一。"""
    prompt = (
        "把以下客服信件分類，只回一個詞：simple（單純問題）或 complex（複雜/需升級）。\n\n"
        f"信件：{state['email']}"
    )
    result = llm.invoke(prompt).content.strip().lower()
    category = "complex" if "complex" in result else "simple"
    return {"category": category}      # ← 只更新 category 這一個欄位

def draft_reply(state: State) -> dict:
    """對單純問題草擬回覆。"""
    prompt = f"用友善專業的語氣，草擬一封回覆這封信的中文 email：\n\n{state['email']}"
    return {"reply": llm.invoke(prompt).content}

def escalate(state: State) -> dict:
    """複雜案件：交給真人。"""
    return {"reply": "（此案已升級給真人客服專員處理。）"}
```

三個節點，分別對應 Ch4 的 LLM 步驟與動作步驟。每個都只關心自己要讀的欄位、只回傳自己要寫的欄位。

## 5.4 連邊：固定轉移 vs 條件分支

接下來把節點連起來。這裡會用到兩個特殊節點：`START`（代表「使用者輸入送進圖」的起點，用來指定哪個節點先跑）與 `END`（代表終點，標示某條路徑跑完了）。

email agent 的形狀是：先分類 → 再依分類**分岔**到草稿或升級 → 結束。其中「分類 → 草稿 / 升級」這一步是**條件分支**，要用 `add_conditional_edges`：

```python
def route(state: State) -> str:
    """依分類結果決定下一個節點的名字。"""
    return "draft" if state["category"] == "simple" else "escalate"

builder = StateGraph(State)
builder.add_node("classify", classify)
builder.add_node("draft", draft_reply)
builder.add_node("escalate", escalate)

builder.add_edge(START, "classify")                 # 固定邊：一開始一定先分類
builder.add_conditional_edges(                      # 條件邊：依 route 的回傳值分岔
    "classify",
    route,
    {"draft": "draft", "escalate": "escalate"},     # route 回傳 → 實際節點 的對應表
)
builder.add_edge("draft", END)                      # 草稿完 → 結束
builder.add_edge("escalate", END)                   # 升級完 → 結束

graph = builder.compile()
```

`add_conditional_edges` 的三個參數要看懂：第一個是「從哪個節點分岔」，第二個是「一個讀 state、回傳字串的路由函數」，第三個是「把路由函數的回傳值對應到實際節點名稱」的對照表。這就是 Ch4 說的——**箭頭畫出可能路徑，但實際走哪條由節點內邏輯（這裡是 `route`）在執行時決定**。

> ⏱ **三秒判斷** —— `add_edge` vs `add_conditional_edges`？
> - 「下一步**永遠是固定那個節點**」 → **`add_edge`**
> - 「下一步**依 state 決定**走哪條」 → **`add_conditional_edges`** + 路由函數
>
> 別用條件邊做「永遠走 A」這種事，會把該寫死的彈性化、增加除錯難度。

## 5.5 執行：invoke 與看見每一步

編譯好就能跑了。最直接的是 `invoke`，丟入符合 state schema 的輸入，拿回最終 state：

```python
result = graph.invoke({"email": "請問如何重設我的密碼？"})
print(result["category"])   # simple
print(result["reply"])      # （一段草擬好的回覆）
```

但對學習與除錯來說，更有價值的是用 `stream_mode="updates"` **看它一步步怎麼走**——每個節點跑完，就吐出它造成的 state 更新：

```python
for step in graph.stream(
    {"email": "你們的 API 整合會間歇性回 504，我們的服務一直中斷"},
    stream_mode="updates",
):
    print(step)
# {'classify': {'category': 'complex'}}
# {'escalate': {'reply': '（此案已升級給真人客服專員處理。）'}}
```

從輸出你能清楚看到：這封複雜信件，`classify` 把它標成 `complex`，`route` 因此把它導向 `escalate`，而 `draft` 完全沒被執行。動態路由成功了。（`stream` 的各種模式是 Ch11 的主題，這裡先用它來「看路徑」。）

## 5.6 那條「迴圈回邊」：把 agent loop 畫出來

到目前為止的 email agent 是「有分支但不迴圈」的工作流。但 Ch1 那個 agent loop 呢？在 Graph API 裡，它就是**一條從工具節點繞回模型節點的邊**：

```python
builder.add_node("model", call_model)
builder.add_node("tools", call_tools)
builder.add_edge(START, "model")
builder.add_conditional_edges("model", should_continue, {"tools": "tools", "end": END})
builder.add_edge("tools", "model")    # ← 這條回邊，就是 reason → act → observe 的迴圈
```

`model → tools → model → tools → …` 直到 `should_continue` 判斷模型不再要求工具、走向 `END`。Ch2 我們說「`create_agent` 底下是一張 LangGraph 圖」，指的就是這個結構。你現在不只能用那個 loop，還能親手把它畫出來、在中間插入自訂節點與分支——這就是下沉到 runtime 換來的掌控力。

> 🪞 **對照閱讀 —— 「依條件分支」在三層怎麼做：**
>
> | 層 | 寫法 | 路由由誰決定 |
> |---|---|---|
> | **LangGraph**（本章） | `add_conditional_edges` + 路由函數 | **你**親手寫的函數 |
> | **LangChain `create_agent`**（Ch19） | 模型 `bind_tools` 後自己決定要不要呼叫工具 | **模型**（隱式路由） |
> | **Deep Agents**（Ch29） | 模型 + 內建 harness 工具一起決定下一步 | 幾乎**全交給模型** |
>
> 愈往上，路由愈交給模型；愈往下，愈是你親手寫——又一次呼應 Ch2 的「黑盒 vs 控制」光譜。

---

## 常見坑（Pitfalls）

- **節點回傳了整個 state**：只需回傳「要更新的欄位」這個 dict。回傳整包不僅多餘，搭配 reducer 時（Ch6）還可能造成非預期的覆蓋。
- **忘了 `compile()` 就想 `invoke`**：圖一定要先編譯。沒編譯的是 builder，不是可執行的圖。
- **條件邊的對照表 key 對不上路由函數的回傳值**：`route` 回傳 `"draft"`，對照表卻寫 `"drafts"`，圖會找不到目的地。兩邊字串必須一致。
- **路由函數做了副作用或寫 state**：路由函數應該只「讀 state、回傳一個字串」。要更新 state 請在節點裡做。
- **忘了連 `START`**：沒有 `add_edge(START, ...)`，圖不知道從哪開始，compile 時或執行時會出問題。

## 本章小結

我們用 `StateGraph` 把 Ch4 的 email agent 從紙上的圖變成可跑的程式：定義 `TypedDict` state、用 `add_node` 加入 `classify`／`draft`／`escalate` 三個「只做一件事、只回傳要更新欄位」的節點、用 `add_edge` 連固定轉移、用 `add_conditional_edges` 搭配路由函數實現「依分類分岔」的動態路由，最後 `compile` 成圖。執行時，`invoke` 給你最終結果，而 `stream_mode="updates"` 讓你看見它實際走過哪些節點——這是建立直覺與除錯的利器。我們也看到 Ch1 的 agent loop 在 Graph API 裡，不過就是「一條從工具節點繞回模型節點的回邊」。你現在握有的，是 `create_agent` 給不了的結構掌控力。但有一件事我們還沒深究：當多個節點同時更新 state、或同一個欄位被寫很多次，更新到底是怎麼套用的？這就要談 state 的核心——reducer，也是下一章的主題。

## 延伸閱讀 / 練習

1. **跑起來**：把本章的 email agent 完整組起來跑，分別丟一封「簡單問題」與一封「複雜技術問題」，用 `stream_mode="updates"` 確認它們走了不同路徑。
2. **加一條分支**：在 `route` 裡多加一類 `billing`（帳務），新增一個 `billing_reply` 節點與對應的條件邊，讓三類信件走三條路。
3. **加一個前置節點**：在 `classify` 前插一個 `read_email` 節點（對應 Ch4 的「讀信」），讓它做一些前處理（例如去除簽名檔）再寫回 state。
4. **觀察未執行的節點**：丟一封 complex 信件，確認 `draft` 節點完全沒被呼叫——用一行 `print` 加在 `draft_reply` 裡驗證它真的沒跑。
