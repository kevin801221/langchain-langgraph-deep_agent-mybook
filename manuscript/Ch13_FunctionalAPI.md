# Ch 13　Functional API：用裝飾器寫 graph 的另一條路

> **本章目標**
>
> - 認識 Functional API：用 `@entrypoint` 與 `@task` 兩個裝飾器，把 persistence、memory、HITL、streaming 加進「長得像一般 Python」的程式碼。
> - 看清 Functional API 與 Graph API 的差異與取捨——兩者共用同一個 runtime，可混用。
> - 用一個「寫文章 → 暫停請人審核」的範例，體會 `@task` 的結果如何被 checkpoint 快取。
>
> **使用版本**：`langgraph` 1.x。配套 notebook **不需 API key**。

---

## 13.1 另一種風格：不畫圖，寫函數

到目前為止，我們都用 Graph API：宣告 State、加節點、連邊、編譯。但有時你的邏輯用一般 Python 的 `if`、`for`、函數呼叫來表達更自然，不想被迫拆成節點與邊。這時 **Functional API** 是另一條路。

它的承諾是：**用最少的改動，把 LangGraph 的關鍵能力（persistence、memory、human-in-the-loop、streaming）加進你既有的、用一般語言原語寫的程式碼。** 不必把程式重構成顯式的 pipeline 或 DAG。

兩個核心積木：

- **`@entrypoint`**：標記一個函數為 workflow 的起點，封裝邏輯、管理執行流程（包括長任務與 interrupt）。
- **`@task`**：代表一個離散的工作單元（一次 API 呼叫、一段資料處理），可在 entrypoint 內非同步執行，回傳一個 future-like 物件，可 `.result()` 取值。

## 13.2 一個完整範例：寫文章 + 請人審核

下面這個 workflow 寫一篇文章，然後暫停請人審核——把 `@task`、`@entrypoint` 與 interrupt 一次串起來：

```python
import time
from langgraph.func import entrypoint, task
from langgraph.types import interrupt
from langgraph.checkpoint.memory import InMemorySaver

@task
def write_essay(topic: str) -> str:
    """寫一篇關於指定主題的文章。"""
    time.sleep(1)   # 假裝這是個長時間任務
    return f"一篇關於「{topic}」的文章"

@entrypoint(checkpointer=InMemorySaver())
def workflow(topic: str) -> dict:
    """寫文章，然後暫停請人審核。"""
    essay = write_essay(topic).result()       # 呼叫 task，.result() 取回結果
    is_approved = interrupt({                   # 暫停，把文章交給人審核
        "essay": essay,
        "action": "請核准 / 退回這篇文章",
    })
    return {"essay": essay, "is_approved": is_approved}
```

執行它：

```python
from langchain_core.utils.uuid import uuid7

config = {"configurable": {"thread_id": str(uuid7())}}
for item in workflow.stream("貓", config):
    print(item)
# > {'write_essay': '一篇關於「貓」的文章'}
# > {'__interrupt__': (Interrupt(value={'essay': ..., 'action': '請核准 / 退回這篇文章'}），）}
```

文章寫好、停在審核點。拿到人的回應後回復（跟 Ch11 一樣用 `Command(resume=...)`）：

```python
from langgraph.types import Command

for item in workflow.stream(Command(resume=True), config):
    print(item)
# > {'workflow': {'essay': '一篇關於「貓」的文章', 'is_approved': True}}
```

## 13.3 關鍵細節：task 結果會被 checkpoint 快取

這個範例藏著 Functional API 一個重要、且呼應 Ch9 的特性。Ch11 我們學到：interrupt 回復時，**節點會從頭重跑**。Functional API 也一樣——workflow 從頭執行——**但因為 `write_essay` 是一個 `@task`，它的結果已經存進 checkpoint，resume 時會直接從 checkpoint 載回，而不會重算。**

這正是 Ch9「把不確定／副作用包進 task」鐵則的體現：在 Functional API 裡，`@task` 就是你包裝副作用的單位。包了，resume 才不會重跑那個昂貴或有副作用的步驟。所以上面那篇文章不會在審核回復後又被重寫一次。

## 13.4 Functional API vs Graph API：怎麼選

兩者共用同一個底層 runtime（都是 Pregel，Ch7），可以在同一個應用裡混用。差異在風格：

- **控制流**：Functional API 不需要你思考「圖結構」，用標準 Python 的 `if`/`for`/函數呼叫即可，通常程式更短。Graph API 則是顯式的節點與邊。
- **短期記憶**：Graph API 要宣告 State、可能要定義 reducer；`@entrypoint`/`@task` 不需要顯式 state 管理，狀態 scope 在函數內、不跨函數共享。
- **checkpointing**：兩者都會產生 checkpoint。Graph API 在**每個 super-step 後**產生新 checkpoint；Functional API 則是**把 task 結果存進 entrypoint 對應的既有 checkpoint**，而非每次都新建。
- **可視化**：Graph API 容易把 workflow 畫成圖（利於除錯、溝通）；Functional API 因為圖是執行時動態產生的，不支援可視化。

實務上的判準：流程**動態、分支多、想用一般 Python 控制流、程式想短** → Functional API；流程**需要顯式結構、想畫圖、需要明確的 state/reducer 管理、要做複雜編排** → Graph API。兩者能混搭，不是二選一。

---

## 常見坑（Pitfalls）

- **副作用沒包進 `@task`**：interrupt/resume 或失敗重跑時，沒包的副作用會重複執行。把昂貴或有副作用的步驟都包成 `@task`（呼應 Ch9）。
- **忘了 `.result()`**：`@task` 回傳的是 future-like 物件，要 `.result()` 才拿到值。
- **期待 Functional API 能畫圖**：它的圖是動態產生的，不支援可視化。要可視化請用 Graph API。
- **以為兩種 API 不能混**：它們共用同一個 runtime，可以在同一應用裡並用。

## 本章小結

Functional API 用 `@entrypoint`（workflow 起點）與 `@task`（離散工作單元）兩個裝飾器，讓你用「貼近一般 Python」的方式取得 persistence、memory、HITL、streaming——不必把程式重構成顯式的圖。它與 Graph API 共用同一個 Pregel runtime，可混用。最重要的特性是：**`@task` 的結果會存進 checkpoint，resume 時直接載回不重算**，這正是 Ch9「把副作用包進 task」鐵則的落地。選擇上，動態控制流、想用原生 Python、程式想短就用 Functional API；要顯式結構、要可視化、要複雜編排就用 Graph API。至此 Part I 的核心機制（state、執行模型、persistence、durable execution、streaming、HITL、subgraph、functional）都到齊了。下一章我們補上最後一塊：跨對話、跨 thread 的長期記憶——store。

## 延伸閱讀 / 練習

1. **跑審核流程**：把 13.2 的 workflow 跑起來，分別用 `Command(resume=True/False)` 回復，觀察 `is_approved` 的差異（配套 notebook 有可跑版本，免 API key）。
2. **驗證快取**：在 `write_essay` 裡加一行 `print("writing...")`，resume 後確認它**沒有**再印一次——證明 task 結果是從 checkpoint 載回的。
3. **改寫對照**：把 Ch5 的 email 分類流程用 Functional API 重寫一次（`@task` 做分類、`@entrypoint` 串流程），跟 Graph API 版比較程式長度與可讀性。
4. **思考題**：同一個應用裡，哪些部分你會用 Functional API、哪些用 Graph API？用「是否需要可視化」與「控制流是否動態」兩個角度說明。
