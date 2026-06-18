# Ch 10　Streaming 與事件流：即時回饋使用者

> **本章目標**
>
> - 理解為什麼 streaming 對 agent 的使用者體驗是必需品，而非加分項。
> - 掌握 `stream` / `astream` 與七種 stream mode：`values`、`updates`、`messages`、`custom`、`checkpoints`、`tasks`、`debug`。
> - 學會 v1.1 起的 **v2 統一輸出格式**（`version="v2"`），告別「依參數而變的輸出形狀」。
> - 用 `get_stream_writer` 從節點吐出自訂進度，並認識 v1.2 的新方向——event streaming。
>
> **使用版本**：`langgraph` 1.x（v2 格式需 `langgraph>=1.1`；event streaming 為 v1.2 引入）。配套 notebook 的非 LLM 範例**不需 API key**。

---

## 10.1 為什麼非串流不可

Ch1 那串「框架沒幫你做」的缺口裡，streaming 排第二。原因是體驗：**一個會自己跑好幾步、每步可能呼叫 LLM 的 agent，從送出到完成可能要十幾、幾十秒。** 使用者盯著一個沒有任何回饋的轉圈圈，很快就會以為當掉而離開。

streaming 解決這件事：把 agent 的中間步驟、甚至 LLM 逐 token 的輸出，**即時**吐給前端。使用者看到「正在查資料…」「正在草擬回覆…」乃至文字一個字一個字浮現，就知道它在動、也更信任它。對 production agent，這不是錦上添花，是基本盤。

## 10.2 stream / astream 與七種 mode

LangGraph 的圖提供 `stream`（同步）與 `astream`（非同步）兩個方法，回傳一個迭代器。你用 `stream_mode` 指定「想收到哪種資料」。七種 mode：

| Mode | 內容 |
|---|---|
| `values` | 每一步後的**完整 state**。 |
| `updates` | 每一步後的**狀態更新**（只有變動的 key）；同一步多個更新會分開串流。 |
| `messages` | LLM 呼叫吐出的 `(token, metadata)` 2-tuple——**逐 token 串流**。 |
| `custom` | 節點透過 `get_stream_writer` 發出的自訂資料。 |
| `checkpoints` | checkpoint 事件（格式同 `get_state()`）。需要 checkpointer。 |
| `tasks` | task 開始／結束事件，含結果與錯誤。需要 checkpointer。 |
| `debug` | 所有可用資訊，結合 checkpoints 與 tasks 再加 metadata。 |

最常用的是前四個。`updates` 適合「顯示它走到哪一步」，`messages` 適合「打字機效果」，`custom` 適合「自訂進度條」。

## 10.3 v2 統一格式：一種形狀，到處適用

這裡有個歷史包袱要講清楚，否則你會被輸出形狀搞瘋。在 **v1（預設）** 格式下，`stream` 的輸出形狀**會隨選項而變**：單一 mode 回傳原始資料、多個 mode 回傳 `(mode, data)` tuple、開了 subgraph 又回傳 `(namespace, data)` tuple。程式要寫一堆分支去拆。

`langgraph>=1.1` 引入了 **v2 格式**：傳 `version="v2"`，**不管幾個 mode、有沒有 subgraph，每個 chunk 都是同一種形狀**——一個 `StreamPart` dict：

```python
{
    "type": "values" | "updates" | "messages" | "custom" | "checkpoints" | "tasks" | "debug",
    "ns": (),       # namespace tuple，subgraph 事件才會有值
    "data": ...,    # 實際內容（型別依 mode 而定）
}
```

於是消費端變得乾淨且可做型別收斂（type narrowing）：

```python
for part in graph.stream(
    {"topic": "ice cream"},
    stream_mode=["values", "updates", "messages", "custom"],
    version="v2",
):
    if part["type"] == "values":
        print(f"完整 state：{part['data']}")
    elif part["type"] == "updates":
        for node_name, state in part["data"].items():
            print(f"節點 `{node_name}` 更新：{state}")
    elif part["type"] == "messages":
        msg, metadata = part["data"]          # (token chunk, metadata)
        print(msg.content, end="", flush=True)  # 打字機效果
    elif part["type"] == "custom":
        print(f"進度：{part['data']}")
```

**本書建議新程式一律用 `version="v2"`。** （Part 0、Ch5 的範例用的是 v1 預設格式 `{node: updates}`，那是為了一開始最低認知負擔；從這章起，正式做法用 v2。）

## 10.4 updates vs values：兩種看狀態的角度

最常搞混的是 `updates` 與 `values`：

- **`updates`**：每步後**只給變動的 key**。適合「日誌式」呈現——它走了哪個節點、改了什麼。
- **`values`**：每步後給**完整 state 快照**。適合「我每一步都想看到當前全貌」。

```python
# updates：只看「這一步改了什麼」
for part in graph.stream(inputs, stream_mode="updates", version="v2"):
    print(part["data"])     # {"node_name": {"changed_key": ...}}

# values：看「這一步之後的完整 state」
for part in graph.stream(inputs, stream_mode="values", version="v2"):
    print(part["data"])     # 整個 state
```

## 10.5 messages mode：打字機效果

要做「文字一個字一個字浮現」的體驗，用 `messages` mode。它串流 LLM 呼叫產生的 `(token chunk, metadata)`：

```python
for part in graph.stream(inputs, stream_mode="messages", version="v2"):
    msg, metadata = part["data"]
    print(msg.content, end="", flush=True)   # 不換行、即時 flush = 打字機
```

`metadata` 裡帶有「這個 token 來自哪個節點、哪次 LLM 呼叫」等資訊，當你的圖有多個會呼叫 LLM 的節點時，可用它來分流（例如只顯示「草擬回覆」節點的 token，不顯示「分類」節點的）。

## 10.6 custom mode：自訂進度

有時你想吐的不是 state、也不是 token，而是**自訂的進度訊息**（「正在查第 3 個來源…」）。在節點裡用 `get_stream_writer` 拿到一個 writer，寫什麼就會以 `custom` mode 串流出來：

```python
from langgraph.config import get_stream_writer

def generate_joke(state):
    writer = get_stream_writer()
    writer({"status": "正在想笑話…"})          # ← 這會以 custom mode 串流出去
    return {"joke": f"為什麼{state['topic']}要上學？為了拿到聖代學位！"}

for part in graph.stream({"topic": "冰淇淋"},
                         stream_mode=["updates", "custom"], version="v2"):
    if part["type"] == "custom":
        print("狀態：", part["data"]["status"])
    elif part["type"] == "updates":
        print("更新：", part["data"])
```

這讓你能把 agent 內部「現在在忙什麼」精準地告訴使用者，而不必猜。

## 10.7 新方向：event streaming（v1.2）

最後提一個前瞻。`langgraph>=1.2` 推出了 **event streaming**——一套「typed-projection」API：它為每種投影（`messages`、`values`、`subgraphs`、`output`）各給一個**獨立的迭代器**，讓你能分別消費，而不必在一個迴圈裡用 `part["type"]` 分支。對新應用，官方推薦往這個方向走。本章先讓你掌握 stream-mode API（它仍是直接取用 runtime 事件的主力），event streaming 的細節等你需要分流多個投影時再深入即可。

---

## 常見坑（Pitfalls）

- **被 v1 的「輸出形狀隨選項而變」搞瘋**：單 mode、多 mode、subgraph 各一種形狀。新程式一律用 `version="v2"`，每個 chunk 都是統一的 `{type, ns, data}`。
- **`messages` 沒 flush**：忘了 `print(..., end="", flush=True)`，打字機效果會卡住不顯示。
- **`checkpoints` / `tasks` mode 沒掛 checkpointer**：這兩個 mode 需要 checkpointer，否則拿不到事件。
- **多 LLM 節點下不看 metadata**：`messages` mode 會把所有 LLM 節點的 token 混在一起，要靠 `metadata` 分流，否則畫面會亂。
- **混用 v1、v2 的消費寫法**：決定用 v2 就全程 v2，別一段用 tuple 拆、一段用 dict 拆。

## 本章小結

streaming 是 agent 的 UX 基本盤：把中間步驟與 LLM token 即時吐給使用者。LangGraph 用 `stream`／`astream` + 七種 mode（`values`、`updates`、`messages`、`custom`、`checkpoints`、`tasks`、`debug`）覆蓋各種需求；其中 `updates`（看走到哪步）、`values`（看完整全貌）、`messages`（打字機）、`custom`（自訂進度）最常用。務必採用 **`version="v2"`** 的統一格式——每個 chunk 都是 `{type, ns, data}`，乾淨又可型別收斂。自訂進度用 `get_stream_writer`。而 v1.2 的 event streaming 提供了「每種投影各一個迭代器」的新方向，需要時再深入。到這裡，你的 agent 已經能記憶、能回復、能即時回饋。下一章我們加入最關鍵的「人」——human-in-the-loop 與 interrupts，讓 agent 在關鍵時刻停下來等你點頭。

## 延伸閱讀 / 練習

1. **三種 mode 對照**：拿 Ch5 的 email agent（離線版即可），分別用 `updates`、`values`（都加 `version="v2"`）跑同一封信，比較兩者輸出的差異。
2. **自訂進度**：在某個節點裡用 `get_stream_writer` 發出「正在處理…」的訊息，用 `custom` mode 收它（配套 notebook 有可跑版本，不需 API key）。
3. **打字機**：若你有 API key，用 `messages` mode 對一個會呼叫 LLM 的節點做出逐字浮現效果，並用 `metadata` 印出 token 來自哪個節點。
4. **思考題**：什麼情況你會選 `values` 而非 `updates`？反過來呢？各舉一個實際場景。
