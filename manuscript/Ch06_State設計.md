# Ch 6　State 設計：schema、reducer 與 annotation

> **本章目標**
>
> - 搞懂 state 的兩個組成：**schema**（欄位結構）與 **reducer**（更新如何套用）。
> - 認識三種 schema 寫法（`TypedDict` / `dataclass` / Pydantic）的取捨，以及為什麼只有 Pydantic 會在 runtime 擋髒資料。
> - 把 reducer 演到底：覆蓋 vs 累加、平行衝突、`operator.add` 的 `None` 陷阱、自訂 reducer，以及對話專用的 `add_messages` 與 `MessagesState`。
> - 用多重 schema（input/output/private）把對外介面與對內工作通道乾淨分開。
>
> **使用版本**：`langgraph` 1.x、`langchain` 1.x。

---

## 6.1 state 的兩半：schema 與 reducer

Ch5 我們定義了 state，但只用到它的一半。完整的 state 由兩部分構成：

- **Schema**：state 的結構——有哪些欄位、各是什麼型別。它是所有節點與邊的輸入型別。
- **Reducer**：當節點回傳更新時，**這個更新該怎麼套用到現有 state**。每個欄位有自己獨立的 reducer。

第二部分是新手最容易忽略、卻最關鍵的。Ch5 裡 `classify` 回傳 `{"category": "simple"}`，這個 `"simple"` 怎麼進到 state 的？答案藏在 reducer 裡。先記一句口訣，後面整章都掛在它上面：**沒貼 reducer = 覆蓋（新值取代舊值）；貼了 reducer = 用你指定的方式合併。** Ch5 裡 `category`、`reply` 被覆蓋正是我們要的；但「對話歷史」「累積的檢索結果」這類**想累加、不想覆蓋**的欄位，覆蓋就是災難——這正是 6.3 要演到底的事。

> 💡 **冷知識** —— **reducer** 並非 LangGraph 發明：它來自 functional programming 的 **fold/reduce** 操作（Lisp 從 1960 年代就有），意思是「用一個二元函數把一串值合併成一個值」。Redux（2015 年前端狀態管理庫）把它推廣到 UI，所以你在 React 用的 `useReducer`、Python 的 `functools.reduce`、Spark 的 reduce、MapReduce 的 reduce——全是同一概念。LangGraph 的 reducer 就是這個老概念在「state channel 更新」上的再應用。

接下來三節依序展開：先談 schema 怎麼寫（6.2），再把 reducer 演到底（6.3），最後用多重 schema 把對外介面與對內工作通道分開（6.4）。我們先從「schema 有哪三種寫法、各自的取捨」開始。

## 6.2 三種 schema 寫法：TypedDict、dataclass 與 Pydantic 的取捨

Schema 不是只能用 `TypedDict`。LangGraph 支援三種寫法，各有適用場景：`TypedDict` 是首選、`dataclass` 要預設值才用、Pydantic 要驗證才用。但光「列清單」不夠——這一節要把**「各自痛在哪、怎麼寫、什麼時候用」**講完整，特別是一個初學者幾乎都會踩的地雷：**有些寫法看起來幫你檢查了型別，其實執行時根本不擋。**

先給一個能跟著你一整節的心智模型：

> 想像 state 是一張要交出去的**表單**。三種寫法，差別在「門口有沒有警衛」。
>
> - **`TypedDict` / `dataclass` = 沒人檢查的便利貼。** 你在便利貼上寫「心情：抓狂」，欄位明明只允許「開心 / 難過」，但沒有人攔你——便利貼上的「欄位說明」只是寫給你（和 IDE）看的提示，**填錯字也照樣貼上牆**。
> - **Pydantic `BaseModel` = 門口有警衛驗證的表單。** 你想填「心情：抓狂」，警衛當場攔下來：「這欄只收『開心』或『難過』，請重填」——**型別或值不對，當場擋下，根本進不了門。**

一句話記住：**`TypedDict` 和 `dataclass` 的型別標註是「給人看的提示」，不是「執行期的保全」；只有 Pydantic 會在 runtime 真的攔住髒資料。** 這節就圍著這句話展開。

### 一張圖看懂三者定位

把三種寫法放到兩條軸上，馬上看出它們各自卡在哪個位置——橫軸是「要不要欄位預設值」，縱軸是「要不要 runtime 驗證」：

```
                      runtime 會擋髒資料（有警衛）
                                 ▲
                                 │
                                 │      ┌──────────────┐
                                 │      │   Pydantic   │
                                 │      │  BaseModel   │
                                 │      └──────────────┘
                                 │      存取：state.x
                                 │      傳入：傳「實例」
                                 │
        ─────────────────────────┼─────────────────────────▶ 想要欄位預設值
                                 │
        ┌──────────────┐         │      ┌──────────────┐
        │  TypedDict   │         │      │  dataclass   │
        └──────────────┘         │      └──────────────┘
        存取：state['x']         │      存取：state.x
        傳入：傳 dict            │      傳入：傳「實例」
                                 │
                                 ▼
                      runtime 不擋（只有提示，便利貼）
```

讀這張圖只要抓三個重點：

- **縱軸**：只有 Pydantic 在上半部（會擋）；`TypedDict`、`dataclass` 都在下半部（不擋）。
- **橫軸**：要欄位**預設值**就往右——`dataclass` 與 Pydantic 都能寫 `mood: str = "happy"`，`TypedDict` 不行。
- **存取語法**（圖中已標）：`TypedDict` 用**下標** `state['x']`；`dataclass` 與 Pydantic 用**點存取** `state.x`。這個差異等一下會直接變成 bug 來源，先記著。

還有一個關鍵共通點，三者在這點上**完全一樣**：**它們對 `StateGraph` 來說一視同仁**——你照常 `StateGraph(MyState)`、節點照常回傳「更新 dict」、照常 `compile()`。唯一的差別在「**怎麼把初始值餵進去**」：`TypedDict` 餵 dict，`dataclass` / Pydantic 餵「實例」。

### TypedDict：預設首選，但只是型別提示

從最小形狀開始。`TypedDict` 就是「幫一個 dict 標上欄位名與型別」，輕量、零成本，是全書範例的預設首選。

```python
# 改寫自 langchain-academy module-2/state-schema.ipynb
from typing_extensions import TypedDict

class TypedDictState(TypedDict):
    foo: str
    bar: str
```

只加一個概念：用 `Literal` 把某個欄位的**取值範圍**鎖死。下面把 `mood` 限定成只能是 `"happy"` 或 `"sad"`：

```python
# 改寫自 langchain-academy module-2/state-schema.ipynb
from typing import Literal
from typing_extensions import TypedDict

class TypedDictState(TypedDict):
    name: str
    mood: Literal["happy", "sad"]      # ← 看起來像「只准填這兩個值」
```

把它接上 `StateGraph` 跑跑看。注意：`TypedDict` 的節點讀欄位用**下標** `state['name']`，呼叫時餵的是 **dict**：

```python
# 改寫自 langchain-academy module-2/state-schema.ipynb
import random
from typing import Literal
from typing_extensions import TypedDict
from langgraph.graph import StateGraph, START, END

class TypedDictState(TypedDict):
    name: str
    mood: Literal["happy", "sad"]

def node_1(state: TypedDictState) -> TypedDictState:
    print("---Node 1---")
    return {"name": state["name"] + " is ... "}      # ← TypedDict 用下標讀

def node_2(state: TypedDictState) -> TypedDictState:
    print("---Node 2---")
    return {"mood": "happy"}

def node_3(state: TypedDictState) -> TypedDictState:
    print("---Node 3---")
    return {"mood": "sad"}

def decide_mood(state: TypedDictState) -> Literal["node_2", "node_3"]:
    # 隨機在 node_2 / node_3 之間二選一
    return "node_2" if random.random() < 0.5 else "node_3"

builder = StateGraph(TypedDictState)
builder.add_node("node_1", node_1)
builder.add_node("node_2", node_2)
builder.add_node("node_3", node_3)
builder.add_edge(START, "node_1")
builder.add_conditional_edges("node_1", decide_mood)
builder.add_edge("node_2", END)
builder.add_edge("node_3", END)

graph = builder.compile()

print(graph.invoke({"name": "Lance"}))      # ← state 是 dict，所以餵 dict
# ---Node 1---
# ---Node 2---
# {'name': 'Lance is ... ', 'mood': 'happy'}
```

> 💡 **冷知識** —— 想看圖長怎樣可以呼叫 `graph.get_graph().draw_mermaid_png()`，但它要連網路產圖。本書範例一律以「執行結果註解」呈現輸出，繪圖當作選用，跑不出圖不影響你理解資料怎麼流。

#### 踩雷現場：`Literal` 不會在執行期擋值

上面 `mood` 標了 `Literal["happy", "sad"]`，看起來很安全。我們來試著塞一個非法值 `"mad"` 進去：

❌ **錯誤期待**：以為 `Literal` 會擋下非法值。

```python
def node_bad(state: TypedDictState) -> TypedDictState:
    return {"mood": "mad"}      # ← "mad" 不在 Literal["happy","sad"] 裡

# 把 node_2 換成 node_bad 再跑……
```

**實際會發生什麼**——程式**完全不報錯、照常跑完**，髒資料大搖大擺進到最終 state：

```python
# ---Node 1---
# {'name': 'Lance is ... ', 'mood': 'mad'}      # ← "mad" 就這樣留在 state 裡了！
```

**為什麼**：`TypedDict`（包含其中的 `Literal`）只是**型別提示**——它的工作對象是 IDE 和 mypy 這類靜態檢查工具，幫你在**寫程式時**標紅線。程式真正跑起來後，Python 根本不會拿這些提示去檢查任何值。便利貼上寫著「只准填開心或難過」，但門口沒有警衛，你寫「抓狂」一樣貼得上牆。

✅ **正確做法**：如果你**真的需要在執行期擋掉非法值**，`TypedDict` 辦不到——請往下看 Pydantic。(若只是想讓 IDE/mypy 在編輯階段提醒你，`Literal` 本身就夠了，這正是它的定位。)

### dataclass：要預設值時用它（順便換成點存取）

`dataclass` 相較 `TypedDict` 只多解一件事：**讓欄位可以有預設值**（`mood: str = "happy"` 這種）。代價是存取語法從下標換成**點存取**。其餘對 `StateGraph` 來說毫無差別。

```python
# 改寫自 langchain-academy module-2/state-schema.ipynb
from dataclasses import dataclass
from typing import Literal

@dataclass
class DataclassState:
    name: str
    mood: Literal["happy", "sad"] = "happy"      # ← dataclass 才能這樣給預設值
```

只改一個地方：節點裡讀欄位從 `state["name"]` 換成 **`state.name`**；呼叫時餵的是 **`DataclassState` 實例**而不是 dict。其他建圖步驟一模一樣：

```python
# 改寫自 langchain-academy module-2/state-schema.ipynb
import random
from dataclasses import dataclass
from typing import Literal
from langgraph.graph import StateGraph, START, END

@dataclass
class DataclassState:
    name: str
    mood: Literal["happy", "sad"] = "happy"

def node_1(state: DataclassState) -> DataclassState:
    print("---Node 1---")
    return {"name": state.name + " is ... "}      # ← dataclass 用點存取讀

def node_2(state: DataclassState) -> DataclassState:
    print("---Node 2---")
    return {"mood": "happy"}                       # ← 但回傳「更新」仍是 dict！

def node_3(state: DataclassState) -> DataclassState:
    print("---Node 3---")
    return {"mood": "sad"}

def decide_mood(state: DataclassState) -> Literal["node_2", "node_3"]:
    return "node_2" if random.random() < 0.5 else "node_3"

builder = StateGraph(DataclassState)
builder.add_node("node_1", node_1)
builder.add_node("node_2", node_2)
builder.add_node("node_3", node_3)
builder.add_edge(START, "node_1")
builder.add_conditional_edges("node_1", decide_mood)
builder.add_edge("node_2", END)
builder.add_edge("node_3", END)

graph = builder.compile()

print(graph.invoke(DataclassState(name="Lance", mood="sad")))      # ← 餵「實例」
# ---Node 1---
# ---Node 3---
# {'name': 'Lance is ... ', 'mood': 'sad'}
```

這裡有個第一眼會覺得矛盾、其實很合理的點：**節點「讀」state 用點存取 `state.name`，但「回傳更新」仍然回一個 dict `{"mood": "sad"}`，不是回一個 `DataclassState`。** 為什麼？因為 LangGraph 把 state 的**每個欄位當成獨立的通道**分開存——它只在乎你回傳的 dict「鍵名」對不對得上 state 的欄位，對得上就把那個欄位更新掉。所以無論你的 schema 是 `TypedDict` 還是 `dataclass`，**節點回傳的永遠是「只含變動欄位的更新 dict」**，這點三種寫法都一致。

#### 踩雷現場一：dataclass 同樣不驗證

跟 `TypedDict` 一樣，`dataclass` 的型別標註也只是提示。我們直接建一個非法 `mood`：

❌ **以為 dataclass 會擋**：

```python
bad = DataclassState(name="Lance", mood="mad")      # ← "mad" 不在 Literal 範圍
print(bad)
```

**實際**——**建立成功、不報錯**：

```python
# DataclassState(name='Lance', mood='mad')      # ← 一樣放行了
```

**為什麼**：`dataclass` 跟 `TypedDict` 在「驗不驗證」這件事上是同一掛的——**都只有型別提示、runtime 都不檢查**。它比 `TypedDict` 多的只是「預設值」與「點存取語法」，**不包含**「執行期驗證」。便利貼換了個樣式，門口還是沒警衛。

✅ **正確做法**：跟 `TypedDict` 一樣——要的若只是「IDE/mypy 編輯期提醒」，`Literal` 標註就夠了；但**要 runtime 真的擋掉 `"mad"` 這種髒值，`dataclass` 同樣辦不到，唯一選項是下一段的 Pydantic**。別被 `dataclass` 能寫預設值的「正式感」誤導，以為它順便也會驗證。

#### 踩雷現場二：點存取與下標不可混用

換成 `dataclass` 後最常見的手滑，是沿用 `TypedDict` 的下標習慣去讀欄位：

❌ **在 dataclass 節點誤用下標**：

```python
def node_1(state: DataclassState) -> DataclassState:
    return {"name": state["name"] + " is ... "}      # ← 對 dataclass 用了下標
```

**實際**——一跑就炸，丟出 `TypeError`：

```python
# TypeError: 'DataclassState' object is not subscriptable
#   （DataclassState 物件不支援下標存取）
```

**為什麼**：`dataclass` 實例不是 dict，沒有 `__getitem__`，自然不能 `state["name"]`。

✅ **正確**：記死這條對照——

- **`TypedDict`** → 讀欄位用**下標** `state["name"]`，呼叫餵 **dict**。
- **`dataclass` / Pydantic** → 讀欄位用**點存取** `state.name`，呼叫餵**實例**。

兩套語法**不可混用**：對 dict 用點存取會 `AttributeError`，對實例用下標會 `TypeError`。

### Pydantic：唯一會在 runtime 擋髒資料的選項

前兩種都是「沒警衛的便利貼」。如果你**真的需要在「入口」把關**——例如 state 的初始值來自外部輸入、不想讓髒資料一開始就溜進圖裡——就輪到 Pydantic `BaseModel` 上場。它是三者裡**唯一**會在 runtime 驗證的（先記住：它守的是「大門入口」，節點之間不再驗，這個邊界等一下會講清楚）。

寫法重點（Pydantic v2，本書鎖定版本）：

- 繼承 `BaseModel`。
- 要自訂驗證規則，用 **`@field_validator('欄位名')`**，而且**下一行必須緊接 `@classmethod`**（v2 規定）。
- validator 內部 `raise ValueError(...)`，Pydantic 會**自動把它包成 `ValidationError`**——你對外接的是 `ValidationError`。

```python
# 改寫自 langchain-academy module-2/state-schema.ipynb
from pydantic import BaseModel, field_validator, ValidationError

class PydanticState(BaseModel):
    name: str
    mood: str      # 之後用 validator 限制成 "happy" / "sad"

    @field_validator("mood")
    @classmethod                                   # ← v2：必須緊接 classmethod
    def validate_mood(cls, value):
        if value not in ["happy", "sad"]:
            raise ValueError("mood 只能是 'happy' 或 'sad'")      # ← 會被包成 ValidationError
        return value

# 試著塞髒資料——這次警衛會攔下來
try:
    state = PydanticState(name="John Doe", mood="mad")
except ValidationError as e:
    print("驗證失敗：", e)
# 驗證失敗： 1 validation error for PydanticState
# mood
#   Value error, mood 只能是 'happy' 或 'sad' [type=value_error, input_value='mad', input_type=str]
```

對照一下就懂 Pydantic 的價值了：同樣塞 `mood="mad"`，`TypedDict`／`dataclass` **悄悄放行**，Pydantic **當場拋 `ValidationError`**。這就是「有警衛」與「沒警衛」的差別。

接到圖上用法完全一樣——`StateGraph(PydanticState)`、節點點存取讀、呼叫餵實例：

```python
# 改寫自 langchain-academy module-2/state-schema.ipynb
import random
from typing import Literal
from pydantic import BaseModel, field_validator
from langgraph.graph import StateGraph, START, END

class PydanticState(BaseModel):
    name: str
    mood: str

    @field_validator("mood")
    @classmethod
    def validate_mood(cls, value):
        if value not in ["happy", "sad"]:
            raise ValueError("mood 只能是 'happy' 或 'sad'")
        return value

def node_1(state: PydanticState) -> PydanticState:
    print("---Node 1---")
    return {"name": state.name + " is ... "}      # ← 點存取（同 dataclass）

def node_2(state: PydanticState) -> PydanticState:
    print("---Node 2---")
    return {"mood": "happy"}

def node_3(state: PydanticState) -> PydanticState:
    print("---Node 3---")
    return {"mood": "sad"}

def decide_mood(state: PydanticState) -> Literal["node_2", "node_3"]:
    return "node_2" if random.random() < 0.5 else "node_3"

builder = StateGraph(PydanticState)
builder.add_node("node_1", node_1)
builder.add_node("node_2", node_2)
builder.add_node("node_3", node_3)
builder.add_edge(START, "node_1")
builder.add_conditional_edges("node_1", decide_mood)
builder.add_edge("node_2", END)
builder.add_edge("node_3", END)

graph = builder.compile()

print(graph.invoke(PydanticState(name="Lance", mood="sad")))      # ← 餵實例
# ---Node 1---
# ---Node 3---
# {'name': 'Lance is ... ', 'mood': 'sad'}
```

> ⚠️ **關鍵限制（很多人誤會的地方）** —— Pydantic 的警衛**只站在大門，不站在內部走廊**。也就是說：
> - **入口會驗**：`invoke` 把初始值送進「第一個節點」之前，Pydantic 會檢查一遍。連你餵 dict 也一樣——`graph.invoke({"name": "Lance", "mood": "mad"})` 會在進門那一刻就被 `ValidationError` 擋下。
> - **節點之間「不」會再驗**：一旦進了門，某個節點若回傳 `{"mood": "mad"}` 這種髒更新，**Pydantic 不會攔**——它會大搖大擺寫進 state，最終結果照樣是 `{'name': '...', 'mood': 'mad'}`。
> - **輸出也不是 Pydantic 實例**：圖跑完回給你的是 **dict**（看上面每個範例的輸出都是 `{...}`），不是 `PydanticState` 物件。
>
> 一句話：**Pydantic 在 LangGraph 裡只保「入口」，不保「節點間每一步」也不保「出口」。** 這是官方明列的已知限制（runtime validation only occurs on inputs to the first node, not on subsequent nodes or outputs）。所以別誤以為「換了 Pydantic 全程就免疫髒資料」——它擋的是「外面送進來的髒輸入」，擋不住「你自己節點寫出來的髒更新」。另外，Pydantic 的遞迴驗證有成本，對效能敏感的場景官方建議改用 `dataclass`。

#### 踩雷現場：別把 Pydantic v1 寫法帶進來

Pydantic v1 和 v2 的 validator API 不同，照舊版抄會出事。

❌ **沿用 v1 寫法**：

```python
from pydantic import validator      # ← v1 的舊 import

class PydanticState(BaseModel):
    mood: str

    @validator("mood")              # ← v1 用 @validator，且不配 @classmethod
    def validate_mood(cls, value):
        ...
```

**實際**——在 Pydantic v2 環境下，`@validator` 已是**過時 API**（會跳 deprecation 警告、行為與 v2 不一致），而 v2 正規的 `@field_validator` 若**漏掉 `@classmethod`** 也會出問題。

✅ **正確（本書鎖定 v2）**：用 **`@field_validator('欄位名')` + 緊接 `@classmethod`**，錯誤型別接 **`ValidationError`**，validator 內 `raise ValueError`（會被自動包成 `ValidationError`）。一句話：**看到 `@validator` 或 `from pydantic import validator` 就是 v1 味道，改掉。**

### 何時用哪一種

> ⏱ **三秒判斷** —— state schema 怎麼選？
> - **純結構、最常見、要搭 `create_agent`** → **`TypedDict`**（首選，輕量零成本）。
> - **要替欄位設「預設值」** → **`dataclass`**（記得改點存取、餵實例）。
> - **要在「入口」擋髒輸入／嚴格驗證** → **Pydantic `BaseModel`**（注意：只驗入口、不驗節點間，遞迴驗證較慢）。

實務上九成情境都是 `TypedDict`；`dataclass` 在「想讓某些欄位有預設值、少寫初始化樣板」時才划算；Pydantic 留給「外部輸入不可信、寧可在進門那一刻就炸、也不要髒輸入溜進圖」的場景——但要記得它只把守入口，圖內部各節點的更新仍要靠你自己的程式碼把關。

### 常見誤解清單

換寫法時，初學者最常栽在這四點：

1. **以為 `TypedDict` 的 `Literal` 會在執行期擋值。** 不會。`Literal`（連同整個 `TypedDict`、`dataclass` 的型別標註）只是給 IDE／mypy 的提示，runtime 一律放行。**要 runtime 擋值，只有 Pydantic。**
2. **以為換 Pydantic 只是把父類別改成 `BaseModel` 就好。** 不只——換到 Pydantic（或 dataclass）後，**讀欄位要改點存取 `state.x`**（不能再 `state['x']`），而且 **`invoke` 要餵「實例」而非 dict**。漏改任一個就會 `TypeError`／`AttributeError`。
3. **以為換了 Pydantic「全程」就擋得住髒資料。** 不會。Pydantic **只驗入口**（送進第一個節點之前），**節點之間的更新與最終輸出都不再驗**——某節點回傳 `{"mood": "mad"}` 照樣寫進 state，圖跑完拿到的還是 dict、不是 Pydantic 實例。圖內部的把關得靠你自己寫（例如在節點內檢查、或在寫回前自己 `PydanticState.model_validate(...)`）。
4. **想同時用 Pydantic state 又用 `create_agent`。** 不支援。**`create_agent`（Part II 的工廠）不接受 Pydantic state schema**（v1 已把 `AgentStatePydantic` 系列拿掉，官方說明 "no more pydantic state"）——一旦你打算用它，state 就得回退到 `TypedDict`。需要嚴格驗證又想用高階工廠時，這是你必須先知道的取捨，別等到接上去才發現。

## 6.3 reducer：白板 vs 帳本、平行衝突與自訂合併

6.1 開頭那句口訣建立了「覆蓋 vs 累加」的直覺：沒貼 reducer 就覆蓋，貼了 `operator.add` 就把 list 接起來。但「線性」地跑時——一個節點接一個節點——覆蓋頂多是把舊值換掉，看起來人畜無害。本節要把這件事**演到底**：為什麼某些情境**非貼 reducer 不可**（不是為了好看，是不貼就會直接報錯）、平行分支會怎麼炸、`operator.add` 遇到 `None` 又會怎麼炸、以及怎麼自己寫一個防呆的 reducer。最後把對話專用的 `add_messages` 與 `MessagesState` 的細節（覆寫訊息、刪除訊息、一次清空）補齊。

### 6.3.1 心智模型：白板 vs 帳本

先把一個對比講透，後面所有東西都掛在這個模型上。

把 state 的某個欄位想成團隊在開會時共用的一塊區域，每個節點跑完都會「交一份更新」回去。差別只在於：**這塊區域是一片白板，還是一本帳本？**

- **白板（預設 reducer，覆蓋）**：新資料一來，**整塊擦掉、重寫**。白板上永遠只有「最後一個人寫的東西」。你寫 `['hi']`，下一個人寫 `['bye']`，白板上就只剩 `['bye']`，`'hi'` 蒸發了。
- **帳本（`operator.add` / `add_messages`，累加）**：新資料一來，**翻到最後一頁、往後添一筆**。前面的紀錄全部保留。你記 `['hi']`，下一個人記 `['bye']`，帳本上是 `['hi', 'bye']`，一筆都沒少。

一句話定錨，整節記這句就夠：

> **沒貼 reducer = 白板（覆寫）；貼了 reducer = 帳本（累加）或你自訂的合併規則。**

`reducer` 的本質就是一個雙參數函數 `(left, right) -> merged`——`left` 是現有的舊值、`right` 是節點剛交回來的新值，回傳「合併後的新狀態」。白板的合併規則是「回傳 right，無視 left」；帳本（`operator.add`）的合併規則是「回傳 `left + right`」。你也可以自己寫第三種規則，這就是 6.3.8 要做的事。

> 反過來也有一招：偶爾你會遇到「某個欄位平常想累加，但這一次想強制覆蓋」的情境。LangGraph 提供 `Overwrite` 型別讓你在單次更新時繞過該欄位的 reducer、直接覆寫，需要時再查即可。

### 6.3.2 文字圖解：reducer 怎麼把舊值＋新值合併

把「舊值 +(reducer)+ 新值 = 新狀態」畫出來，兩種規則並排對比：

```
白板（預設 reducer：覆寫）
   舊值 ['hi'] ──┐
                 ├─▶ [ reducer：只留 right ] ─▶ 新狀態 ['bye']   ← 'hi' 不見了
   新值 ['bye'] ─┘

帳本（operator.add：累加）
   舊值 ['hi'] ──┐
                 ├─▶ [ reducer：left + right ] ─▶ 新狀態 ['hi', 'bye']   ← 都留著
   新值 ['bye'] ─┘
```

中間那個方框就是 reducer 在做的事：它**不是**直接把新值塞進 state，而是先拿舊值和新值「過一手」，過完才寫回去。你貼什麼 reducer，這一手就用什麼規則。

### 6.3.3 文字圖解：平行 fan-out 為什麼會炸

線性流程下，白板（覆寫）其實沒問題——一個接一個寫，後面蓋掉前面，最終留下最後一筆，邏輯清楚。**真正的麻煩出在平行分支。**

當一個節點 fan-out（分岔）出多個節點，且這些節點落在**同一個 super-step**（同一執行步、並行跑）時，它們會**同時**交出對「同一個欄位」的更新：

```
            ┌─▶ node_2 ──(寫 foo=3)──┐
 node_1 ────┤                         ├──▶ ？？？
            └─▶ node_3 ──(寫 foo=3)──┘
            （node_2 與 node_3 在同一 super-step 並行）

 白板規則：「只留最後寫的那個」── 但兩個是同時寫的，沒有「最後」可言
                              │
                              ▼
                   圖無法決定要留誰 → InvalidUpdateError
```

關鍵在於 `InvalidUpdateError` 觸發**需要三個條件同時成立**，缺一個都不會炸：

1. **同一個 super-step**（兩個節點並行，不是先後）。
2. **多個節點寫同一個 key**。
3. **那個 key 沒有 reducer**（是白板）。

反過來說：如果是先後執行（不同 super-step），白板只是「依序覆蓋」，不會報錯；如果那個 key 貼了 reducer（帳本），平行的兩筆會各自走 reducer 合併進去，也不會報錯。**只有「並行 ＋ 同 key ＋ 沒 reducer」這個組合會觸發。** 記住這點，你日後看到 `InvalidUpdateError`，第一反應就該是「我是不是在平行分支裡對某個白板欄位多頭寫入了」。

### 6.3.4 漸進範例 1：線性下，白板沒問題

先建最小可動的形狀，確認「線性流程 + 白板」一切正常：

```python
# 改寫自 langchain-academy module-2/state-reducers.ipynb（Default overwriting state）
from typing_extensions import TypedDict
from langgraph.graph import StateGraph, START, END

class State(TypedDict):
    foo: int      # ← 沒貼 reducer：預設覆蓋（白板）

def node_1(state: State) -> State:
    return {"foo": state["foo"] + 1}      # 只回變動欄位，不回整個 state

builder = StateGraph(State)
builder.add_node("node_1", node_1)
builder.add_edge(START, "node_1")
builder.add_edge("node_1", END)
graph = builder.compile()

print(graph.invoke({"foo": 1}))
# {'foo': 2}
```

`node_1` 把 `foo` 從 1 寫成 2，白板被擦掉重寫成 `2`。線性、單一寫入者，覆寫剛好就是我們要的。到這裡都沒事。

### 6.3.5 踩雷現場 1：平行覆寫，當場炸給你看

現在把 `node_1` fan-out 到 `node_2`、`node_3`，三個節點都覆寫 `foo`：

❌ **錯誤寫法**——對白板欄位做平行寫入：

```python
# 改寫自 langchain-academy module-2/state-reducers.ipynb（Branching）
from typing_extensions import TypedDict
from langgraph.graph import StateGraph, START, END
from langgraph.errors import InvalidUpdateError

class State(TypedDict):
    foo: int      # ← 仍是白板，沒貼 reducer

def node_1(state: State) -> State:
    return {"foo": state["foo"] + 1}

def node_2(state: State) -> State:
    return {"foo": state["foo"] + 1}

def node_3(state: State) -> State:
    return {"foo": state["foo"] + 1}

builder = StateGraph(State)
builder.add_node("node_1", node_1)
builder.add_node("node_2", node_2)
builder.add_node("node_3", node_3)
builder.add_edge(START, "node_1")
builder.add_edge("node_1", "node_2")      # node_1 fan-out 到 node_2…
builder.add_edge("node_1", "node_3")      # …與 node_3（兩者同一 super-step 並行）
builder.add_edge("node_2", END)
builder.add_edge("node_3", END)
graph = builder.compile()

try:
    graph.invoke({"foo": 1})
except InvalidUpdateError as e:
    print(f"InvalidUpdateError occurred: {e}")
```

**實際會發生什麼**——程式不是回傳結果，而是被我們的 `try/except` 接住，印出：

```
InvalidUpdateError occurred: At key 'foo': Can receive only one value per step. Use an Annotated key to handle multiple values.
For troubleshooting, visit: https://docs.langchain.com/oss/python/langgraph/errors/INVALID_CONCURRENT_GRAPH_UPDATE
```

（langchain 1.x 會在錯誤訊息後多附一行 troubleshooting 連結，第一行才是重點，照著做就對了。）

**為什麼**——`node_2` 和 `node_3` 由 `node_1` 同時分岔出來，落在**同一個 super-step 並行執行**，兩者都對 `foo` 交出更新。`foo` 是白板（沒 reducer），白板的規則是「只留一個」，但同一步收到兩個值、沒有先後，圖根本不知道該留誰——三條件（並行 ＋ 同 key ＋ 無 reducer）全中，於是直接拋 `InvalidUpdateError`。注意錯誤訊息很貼心地告訴你解法：`Use an Annotated key`——也就是貼 reducer。

✅ **正確寫法**——給 `foo` 貼上 reducer，從白板換成帳本。這正是下一個範例。

### 6.3.6 漸進範例 2：`operator.add` 解平行衝突

只動一個地方：把 `foo` 的型別從 `int`（白板）改成 `Annotated[list[int], add]`（帳本）。`operator.add` 套在 list 上就是「串接」，平行交回來的兩筆會各自被 append 進去，衝突消失：

```python
# 改寫自 langchain-academy module-2/state-reducers.ipynb（Reducers）
from operator import add
from typing import Annotated
from typing_extensions import TypedDict
from langgraph.graph import StateGraph, START, END

class State(TypedDict):
    foo: Annotated[list[int], add]      # ← 貼 operator.add：list 串接（帳本）

def node_1(state: State) -> State:
    return {"foo": [state["foo"][-1] + 1]}      # 回傳包成 list

def node_2(state: State) -> State:
    return {"foo": [state["foo"][-1] + 1]}

def node_3(state: State) -> State:
    return {"foo": [state["foo"][-1] + 1]}

builder = StateGraph(State)
builder.add_node("node_1", node_1)
builder.add_node("node_2", node_2)
builder.add_node("node_3", node_3)
builder.add_edge(START, "node_1")
builder.add_edge("node_1", "node_2")
builder.add_edge("node_1", "node_3")
builder.add_edge("node_2", END)
builder.add_edge("node_3", END)
graph = builder.compile()

print(graph.invoke({"foo": [1]}))
# {'foo': [1, 2, 3, 3]}
```

讀一下這個結果 `[1, 2, 3, 3]`：起始 `[1]`；`node_1` 看到最後一筆 `1`，加 1 得 `[2]`，串接成 `[1, 2]`；接著 `node_2`、`node_3` 並行，**兩者看到的最後一筆都是 `2`**（同一 super-step 開始時的快照），各自算出 `[3]`，兩個 `[3]` 都被 `add` 串進去，最終 `[1, 2, 3, 3]`。沒有 `InvalidUpdateError`——因為帳本的規則「left + right」對「同一步多筆」天生就有答案。

> ⚠️ 用 `operator.add` 有三個「都要是 list」的鐵則：**欄位型別**標成 `list[...]`、**初始輸入**就要傳 list（`{"foo": [1]}` 而非 `{"foo": 1}`）、**節點回傳也要包成 list**（`[...]` 而非裸值）。三者任一漏掉，`add` 的兩個運算元就會型別不合而報錯。

### 6.3.7 踩雷現場 2：`operator.add` 的 `None` 陷阱

`operator.add` 看起來很萬能，但它有個尖銳的死角：**它假設兩邊都是 list**。如果某一邊是 `None`，`list + None` 在 Python 裡是非法操作。

❌ **錯誤寫法**——對 `Annotated[list, add]` 欄位傳 `None`（沿用 6.3.6 的 `graph`）：

```python
# 改寫自 langchain-academy module-2/state-reducers.ipynb（None 陷阱）
try:
    graph.invoke({"foo": None})      # 初始值給 None，而非 list
except TypeError as e:
    print(f"TypeError occurred: {e}")
```

**實際會發生什麼**——印出：

```
TypeError occurred: can only concatenate list (not "NoneType") to list
```

**為什麼**——`add_messages` 那類聰明 reducer 會幫你做防呆，但 `operator.add` 不會。它就是老老實實做 `left + right`；當 `left` 是 `None`、`right` 是 `[2]` 時，等於要算 `None + [2]`，Python 直接拒絕，拋 `TypeError`。`operator.add` 完全不檢查、不容錯——它把「處理 `None`」這件事整個推給你。

✅ **正確寫法**——如果你的欄位有可能拿到 `None`（例如某些路徑沒填、外部輸入不保證給 list），就別再用裸的 `operator.add`，改寫一個會把 `None` 當成空 list 的**自訂 reducer**。

### 6.3.8 漸進範例 3：自訂 reducer，把 `None` 視為空 list

reducer 沒什麼魔法——它就是一個普通函數，簽名固定是兩個參數 `(left, right)`，回傳合併結果。我們自己寫一個 `reduce_list`，在相加前先把 `None`（或任何 falsy 值）替換成 `[]`：

```python
# 改寫自 langchain-academy module-2/state-reducers.ipynb（Custom Reducers）
from operator import add
from typing import Annotated
from typing_extensions import TypedDict
from langgraph.graph import StateGraph, START, END

def reduce_list(left: list | None, right: list | None) -> list:
    """安全合併兩個 list；任一邊為 None 都當成空 list。"""
    if not left:
        left = []           # ← None 或空都視為 []
    if not right:
        right = []
    return left + right     # 到這裡兩邊保證都是 list

class DefaultState(TypedDict):
    foo: Annotated[list[int], add]            # 裸 operator.add：遇 None 會炸

class CustomReducerState(TypedDict):
    foo: Annotated[list[int], reduce_list]    # 自訂 reducer：遇 None 安全

def node_1(state) -> dict:
    return {"foo": [2]}

# (A) 用 DefaultState：傳 None 仍然炸
builder = StateGraph(DefaultState)
builder.add_node("node_1", node_1)
builder.add_edge(START, "node_1")
builder.add_edge("node_1", END)
graph = builder.compile()
try:
    print(graph.invoke({"foo": None}))
except TypeError as e:
    print(f"TypeError occurred: {e}")
# TypeError occurred: can only concatenate list (not "NoneType") to list

# (B) 換成 CustomReducerState：同樣傳 None，這次成功
builder = StateGraph(CustomReducerState)
builder.add_node("node_1", node_1)
builder.add_edge(START, "node_1")
builder.add_edge("node_1", END)
graph = builder.compile()
print(graph.invoke({"foo": None}))
# {'foo': [2]}
```

對照很清楚：同一份節點、同一個 `{"foo": None}` 輸入，差別只在欄位貼的是 `add` 還是 `reduce_list`。`reduce_list` 把 `left=None` 補成 `[]`，於是 `[] + [2] = [2]`，安全過關。

寫自訂 reducer 記住三件事：**①簽名固定兩參數 `(left, right)`**——`left` 是舊值、`right` 是新值，順序不能反；**②寫成純函數**——只看輸入算輸出、不要有副作用（不要去改全域、不要 I/O），這樣它在重試、平行下行為才可預期；**③`None` 與型別要自己顧**——LangGraph 不會替你檢查，初始值、未填路徑都可能餵 `None` 進來。

### 6.3.9 漸進範例 4：`add_messages`，對話專用的聰明帳本

對話歷史是最常見的「需要累加」欄位——現代 LLM 的介面都吃一串 messages（`HumanMessage`、`AIMessage` 等），所以 agent 的 state 幾乎都會有一個存 message list 的欄位。你可能想：那用 `operator.add` 累加不就好了？不夠。問題出在 **human-in-the-loop**：有時你想**修改**對話裡某一則既有訊息（例如人工改寫 agent 的草稿），而不是「再附加一則」；用 `add` 的話，送進去的修改會被當成新訊息接在後面，造成重複。

> 🩸 **血淚教訓** —— 我曾經把 `messages` 的 reducer 寫成 `operator.add`，第一版很順。後來加了「人工改寫草稿」（用 `update_state` 改某則 AIMessage），結果**每改一次那則訊息就在歷史裡多一份重複**——三輪後對話長到爆 token。debug 半天才發現是 reducer 選錯：`add` 不認 ID、什麼都當新訊息接在後面。換成 `add_messages` 立刻乾淨。**對話欄位一律用 `add_messages`，沒有例外**。

`add_messages` 比 `operator.add` 多了三項聰明：**附加、依 ID 覆寫、依 ID 刪除**——它是一本「會認 ID 的帳本」。先看最基本的附加（注意 import 一律用 `langchain.messages`，不是 academy 的 `langchain_core.messages`）：

```python
# 改寫自 langchain-academy module-2/state-reducers.ipynb（Messages）
from langgraph.graph.message import add_messages
from langchain.messages import AIMessage, HumanMessage

initial = [
    AIMessage(content="Hello! How can I assist you?", name="Model"),
    HumanMessage(content="I'm looking for information on marine biology.", name="Lance"),
]
new = AIMessage(content="Sure, what specifically are you interested in?", name="Model")

result = add_messages(initial, new)
# result 為 3 則訊息：原本 2 則 + 新加的 1 則（new 被 append 到尾端）
```

**聰明之處一：相同 ID 會覆寫，而不是重複附加。** 這正是前面 human-in-the-loop「改寫某則歷史訊息」的底層機制——你不是再加一則，而是用同一個 `id` 把舊的那則替換掉：

```python
# 改寫自 langchain-academy module-2/state-reducers.ipynb（Re-writing）
from langgraph.graph.message import add_messages
from langchain.messages import AIMessage, HumanMessage

initial = [
    AIMessage(content="Hello! How can I assist you?", name="Model", id="1"),
    HumanMessage(content="I'm looking for information on marine biology.", name="Lance", id="2"),
]
# 帶相同 id="2" 的新訊息 → 覆寫既有 id="2" 那則，而非新增
rewrite = HumanMessage(content="I'm looking for information on whales, specifically", name="Lance", id="2")

result = add_messages(initial, rewrite)
# result 仍是 2 則：id="1" 不動，id="2" 的內容被改寫成 "...on whales, specifically"
```

**聰明之處二：用 `RemoveMessage` 依 ID 刪除。** 想砍掉某幾則歷史訊息（例如裁剪過長對話），就送出帶該 `id` 的 `RemoveMessage`：

```python
# 改寫自 langchain-academy module-2/state-reducers.ipynb（Removal）
from langgraph.graph.message import add_messages
from langchain.messages import AIMessage, HumanMessage, RemoveMessage

messages = [
    AIMessage("Hi.", name="Bot", id="1"),
    HumanMessage("Hi.", name="Lance", id="2"),
    AIMessage("So you said you were researching ocean mammals?", name="Bot", id="3"),
    HumanMessage("Yes, I know about whales. But what others should I learn about?", name="Lance", id="4"),
]
# 針對最前面兩則（id 1、2）各產一個 RemoveMessage
delete = [RemoveMessage(id=m.id) for m in messages[:-2]]

result = add_messages(messages, delete)
# id="1"、id="2" 被刪除，result 只剩 id="3"、id="4" 兩則
```

> 💡 **冷知識** —— langchain 1.x 新增了 `REMOVE_ALL_MESSAGES` sentinel（哨兵值），可以**一次清空整個 `messages` 通道**，不必逐則列 `RemoveMessage`。送出 `RemoveMessage(id=REMOVE_ALL_MESSAGES)` 即把該通道清乾淨——做「重置對話」「開新 session」時特別好用。它從 `from langgraph.graph.message import REMOVE_ALL_MESSAGES` 取得（academy notebook 尚未涵蓋，屬 1.x 新增）。

**聰明之處三：序列化／反序列化兩種格式通吃。** 你可以用兩種格式把訊息送進 `messages` 通道，`add_messages` 都接受——

```python
# 這樣可以（LangChain Message 物件）
{"messages": [HumanMessage(content="message")]}
# 這樣也可以（dict）
{"messages": [{"type": "human", "content": "message"}]}
```

但要注意：進到 `messages` 通道的更新一律會被**反序列化成 LangChain Message 物件**，所以讀內容要用點記法 `state["messages"][-1].content`，不要當 dict 去 `["content"]` 索引。

### 6.3.10 漸進範例 5：`MessagesState` 快捷類別

`messages: Annotated[list[AnyMessage], add_messages]` 這串你會在每個對話圖裡反覆寫。LangGraph 把它打包成 `MessagesState`——一個已經內建好 `messages` 欄位與 `add_messages` reducer 的現成 schema。下面兩種寫法**完全等價**：

```python
# 改寫自 langchain-academy module-2/state-reducers.ipynb（Messages / MessagesState）
from typing import Annotated
from typing_extensions import TypedDict
from langchain.messages import AnyMessage
from langgraph.graph.message import add_messages
from langgraph.graph import MessagesState

# 寫法 A：手寫 TypedDict + Annotated[list[AnyMessage], add_messages]
class CustomMessagesState(TypedDict):
    messages: Annotated[list[AnyMessage], add_messages]
    added_key_1: str
    added_key_2: str

# 寫法 B：繼承 MessagesState 再加欄位（不重複宣告 messages）
class ExtendedMessagesState(MessagesState):
    added_key_1: str
    added_key_2: str
```

寫法 B 是日後寫 agent 時最常見的 state 起手式：`messages` 與它的 reducer 由父類別 `MessagesState` 提供，**子類別只負責加自己的額外欄位、不要再重複宣告 `messages`**。重複宣告不會讓你更安全，只會讓 schema 更囉嗦、更容易寫錯。繼承 `MessagesState` 拿到對話通道、再依任務加欄位——這就是日後你最常見的 state 起手式。

### 6.3.11 何時用：三秒判斷

> ⏱ **三秒判斷** —— 這個欄位該配什麼 reducer？
> - **新值取代舊值**（分類結果、最新狀態旗標）→ **不貼 reducer**（白板）
> - **多節點平行寫同一欄位** → **必須貼 reducer**（`operator.add`），否則 `InvalidUpdateError`
> - **對話歷史** → **永遠用 `add_messages`**（附加 ＋ ID 覆寫 ＋ ID 刪除，一次到位）
> - **累積 list／dict 且保證非 None** → **`operator.add`**（最省事）
> - **累積的東西可能是 None**（外部輸入、未填路徑）→ **自訂 reducer 做防呆**（如 `reduce_list`）

### 6.3.12 取捨與常見誤解

新手在 reducer 上最常翻車的四點，逐一說清楚：

1. **以為「回傳更新」會自動幫你合併 list。** 不會。**預設是覆寫（白板）**，你回傳 `{"docs": [新的]}`，舊的整批被換掉。想累加，欄位一定要貼 reducer，沒有「自動偵測這是 list 就幫你接起來」這種事。
2. **以為 `operator.add` 會幫你處理 `None`。** 不會，會直接 `TypeError`。`operator.add` 是純粹的 `left + right`，不檢查、不容錯。欄位可能拿到 `None` 就改用自訂 reducer（6.3.8）。
3. **對話欄位用 `operator.add` 而不是 `add_messages`。** 這是 6.3.9 血淚教訓的現場：`operator.add` 不認 ID，human-in-the-loop 想「改寫某則訊息」時，它只會把改寫版**再 append 一遍**，於是同一則話出現兩次、對話歷史開始錯亂。對話一律 `add_messages`，讓「同 ID 覆寫」幫你做對的事。
4. **貼了 `add_messages` 後，忘了 messages 是 Message 物件、還想當 dict 索引。** 進通道後它們是 `AIMessage`／`HumanMessage` 物件，讀內容用 `state["messages"][-1].content`（點記法），寫成 `["content"]` 會直接報錯。

## 6.4 多重 schema：對外點餐單與廚房內部備料檯

預設情況下，圖的輸入與輸出 schema 相同——你 `invoke` 時給的、和拿回的，是同一個 state。但有時你想更精細地控制：不想讓內部中間欄位出現在輸出、或想讓輸入與輸出 schema 不同。這就是「多重 schema」要解的問題。很多人第一次接觸時會卡在兩個地方：**為什麼有些 key 會「自己消失」？** 以及 **PrivateState 到底有沒有把資料藏起來？** 這一節我們**拆成兩條線、從最小形狀開始、一步一步演給你看**——一條線講「私有中間狀態」，一條線講「輸入輸出過濾」——把「被過濾掉的原理」與一個很容易踩的版本坑一次講透。

> 💡 **冷知識** —— 一張 `StateGraph` 底下，所有 schema（`OverallState`、`InputState`、`OutputState`、`PrivateState`…）的欄位**會被 LangGraph 取「聯集」**，合併成一組底層通道（channels）。所以這些 schema 不是各自獨立的盒子，而是**對同一塊白板開的不同「視窗」**——每個視窗只露出它宣告的那幾個欄位。記住這句，本節所有現象都能解釋。

### 6.4.1 心智模型：對外點餐單 vs 廚房內部備料檯

把一張圖想成一家**餐廳**：

- **點餐單（input schema）**：客人只能在點餐單上填「想吃什麼」（`question`）。其他欄位客人寫不進來。
- **出餐口（output schema）**：客人最後只會拿到「一道菜」（`answer`）。廚房怎麼忙、用了哪些半成品，客人看不到也拿不到。
- **廚房內部備料檯（private state）**：切好的蔥花、調好的醬汁（`baz`、`notes`）——這些是**節點之間傳遞的中間料**，是工作流程必需的，但**不該出現在客人的帳單上**。

一句話定錨整節：**多重 schema 就是「把對外介面（點餐單／出餐口）與對內工作通道（備料檯）乾淨分開」。** 對外只露該露的，對內想用多少中間料都行。

> ⏱ **三秒判斷** ——
> - 對外只想收特定欄位、只想回特定欄位 → 用 `input_schema` / `output_schema`。
> - 只是節點間中繼、不該外露的暫存 → 用 `PrivateState`。
> - 一般小圖兩者都用不到 → 單一 schema 就好，知道有這招、設計大型圖時再拿出來。

文字圖解（把三層邊界畫出來）：

```text
        客人填的點餐單                                 客人拿到的帳單
       ┌──────────────┐                              ┌──────────────┐
進來 → │  InputState  │                              │ OutputState  │ → 出去
       │  question    │                              │  answer      │
       └──────┬───────┘                              └──────▲───────┘
              │  只有 question 能進                          │  只有 answer 能出
              ▼                                             │
   ┌───────────────────────────── OverallState ────────────┴──────────┐
   │  （整塊白板：所有 channel 都在這裡，節點愛讀愛寫哪個都行）          │
   │     question        answer        notes                          │
   └──────────────────────────────────────────────────────────────────┘
                                          ▲
                                   notes 不在 OutputState
                                   → 出餐口被擋下（過濾掉）

   廚房備料檯（PrivateState）：baz 只在 node_1 ──baz──► node_2 之間流動，
   因為 baz 不屬於 OverallState/OutputState，所以穿不過輸出邊界 → 客人看不到。
```

看懂這張圖，本節就懂一半了：**進得來、出得去，由邊界 schema 決定；中間怎麼忙，由內部／私有 schema 決定。**

### 6.4.2 漸進範例 1：私有狀態（最小的「隔離」概念）

先講最小的東西——讓兩個節點之間傳一個**只有它們知道的中間值**。我們定義一個對外的 `OverallState`（只有 `foo`），和一個私有的 `PrivateState`（只有 `baz`）。`node_1` 讀 `foo`、寫 `baz`；`node_2` 讀 `baz`、寫回 `foo`。

```python
# 改寫自 langchain-academy module-2/multiple-schemas.ipynb
from typing_extensions import TypedDict
from langgraph.graph import StateGraph, START, END

class OverallState(TypedDict):
    foo: int                              # ← 對外欄位（預設覆蓋）

class PrivateState(TypedDict):
    baz: int                              # ← 只在節點間流動的中間料

def node_1(state: OverallState) -> PrivateState:
    print("---Node 1---")
    return {"baz": state["foo"] + 1}      # ← 寫入「備料檯」的 baz

def node_2(state: PrivateState) -> OverallState:
    print("---Node 2---")
    return {"foo": state["baz"] + 1}      # ← 讀備料檯、寫回對外的 foo

builder = StateGraph(OverallState)
builder.add_node("node_1", node_1)
builder.add_node("node_2", node_2)

builder.add_edge(START, "node_1")
builder.add_edge("node_1", "node_2")
builder.add_edge("node_2", END)

graph = builder.compile()

graph.invoke({"foo": 1})
# ---Node 1---
# ---Node 2---
# {'foo': 3}
```

注意輸出：`{'foo': 3}`。`baz` 明明被 `node_1` 寫進去、又被 `node_2` 讀出來用了，但**最終結果裡完全沒有 `baz`**。原因很單純——`baz` 只宣告在 `PrivateState`，而**圖的輸出只露出 `OverallState` 的欄位**，所以 `baz` 被自動排除在帳單之外。這就是「廚房備料檯」：用得到，但客人看不到。

這裡也順帶示範了一個常被誤解的點：`node_2` 的型別提示寫 `state: PrivateState`，它就**真的能讀到 `node_1` 寫的 `baz`**——因為 `OverallState` 和 `PrivateState` 的欄位被合併進同一塊白板，`baz` 這個 channel 就存在於整張圖裡，誰宣告了它、誰就讀得到。

### 6.4.3 踩雷現場 1：以為 PrivateState 是「真正隔離的命名空間」

很多人第一次看到 `PrivateState` 會以為：「喔，這是一個獨立的盒子，跟 `OverallState` 互不相干，所以我可以放心地用同名 key。」**這是誤會。**

❌ **錯誤心智模型**：讓 `PrivateState` 跟 `OverallState` 取同名 key，以為兩邊各有一份、互不干擾——

```python
class OverallState(TypedDict):
    foo: int

class PrivateState(TypedDict):
    foo: int          # ← 想當「私有的 foo」，以為跟上面那個是兩回事

def node_1(state: OverallState) -> PrivateState:
    return {"foo": 99}    # 以為這是寫進「私有空間」
```

**實際會發生什麼**：兩個 `foo` **共用同一個底層 channel**。`node_1` 寫的 `foo: 99` 不是寫進什麼私有空間，而是**直接覆寫了對外的 `foo`**——於是 `99` 會原封不動出現在最終輸出 `{'foo': 99}`，一點都沒被「藏起來」。你以為設了私有牆，其實只是把同一個變數又寫了一遍。

**為什麼**：回到 6.4.1 開頭那句冷知識——LangGraph 用**所有 schema 的 key 聯集**來建 channels。同名 key = 同一個 channel，不會因為「分別宣告在不同 class」就變成兩份。一個 key 是否出現在輸出，**只看它是否屬於 `OverallState`／`OutputState`**，跟它叫不叫 `PrivateState` 無關。

✅ **正確寫法**：私有的中間值**取一個不會跟對外欄位撞名的 key**（像範例 1 的 `baz`），它自然就不會出現在輸出——

```python
class OverallState(TypedDict):
    foo: int

class PrivateState(TypedDict):
    baz: int          # ← 不撞名，這才是真正「不會外露」的私有中間值
```

**口訣：私有不是靠「牆」，是靠「名字不在對外清單上」。**

### 6.4.4 漸進範例 2：對照組——不設過濾，全部曝光

換到第二條線：輸入輸出過濾。先看**沒有**過濾的樣子，當對照組。單一 `OverallState` 裝三個欄位，照常跑：

```python
# 改寫自 langchain-academy module-2/multiple-schemas.ipynb
from typing_extensions import TypedDict
from langgraph.graph import StateGraph, START, END

class OverallState(TypedDict):
    question: str
    answer: str
    notes: str

def thinking_node(state: OverallState):
    return {"answer": "bye", "notes": "... his name is Lance"}

def answer_node(state: OverallState):
    return {"answer": "bye Lance"}

builder = StateGraph(OverallState)
builder.add_node("thinking_node", thinking_node)
builder.add_node("answer_node", answer_node)
builder.add_edge(START, "thinking_node")
builder.add_edge("thinking_node", "answer_node")
builder.add_edge("answer_node", END)

graph = builder.compile()

graph.invoke({"question": "hi"})
# {'question': 'hi', 'answer': 'bye Lance', 'notes': '... his name is Lance'}
```

輸出含**全部三鍵**：`question`、`answer`、`notes` 通通端出來。`notes` 只是 `thinking_node` 的內部碎念（「他叫 Lance」），照理不該丟給呼叫端，但因為它在 `OverallState` 裡、又沒設任何過濾，就跟著一起曝光了。這就是**「單一 schema = 進出全暴露」**的預設行為。

### 6.4.5 漸進範例 3：加上 input_schema / output_schema

現在一次只加一個新概念——**過濾**。我們把對外介面拆出來：`InputState` 只收 `question`，`OutputState` 只回 `answer`，`OverallState` 仍然是內部那塊裝得下全部三鍵的大白板。關鍵在建圖那一行的 `input_schema=` / `output_schema=`：

```python
# 改寫自 langchain-academy module-2/multiple-schemas.ipynb
from typing_extensions import TypedDict
from langgraph.graph import StateGraph, START, END

class InputState(TypedDict):
    question: str                         # ← 對外只收這個

class OutputState(TypedDict):
    answer: str                           # ← 對外只回這個

class OverallState(TypedDict):
    question: str
    answer: str
    notes: str                            # ← 內部用的中間料

def thinking_node(state: InputState):
    return {"answer": "bye", "notes": "... his name is Lance"}

def answer_node(state: OverallState) -> OutputState:
    return {"answer": "bye Lance"}

builder = StateGraph(OverallState, input_schema=InputState, output_schema=OutputState)
builder.add_node("thinking_node", thinking_node)
builder.add_node("answer_node", answer_node)
builder.add_edge(START, "thinking_node")
builder.add_edge("thinking_node", "answer_node")
builder.add_edge("answer_node", END)

graph = builder.compile()

graph.invoke({"question": "hi"})
# {'answer': 'bye Lance'}
```

輸出只剩 **`{'answer': 'bye Lance'}`**。對比上一個範例的三鍵全曝光，這裡 `question` 和 `notes` 都被 `output_schema=OutputState` 擋在出餐口外了——`notes` 是內部碎念、`question` 是輸入回聲，呼叫端都不需要，於是乾乾淨淨只拿到一道菜。輸入端同理：就算呼叫方多塞了別的 key，`input_schema=InputState` 也只會放 `question` 進來。

> 💡 **冷知識** —— academy 那份 notebook 這個 cell 的「存檔輸出」其實是**錯的**（仍顯示 `{'question': 'hi', 'answer': 'bye Lance', 'notes': ...}` 三鍵，是上一個 cell 留下的舊輸出沒清乾淨）。實際重跑、套上 `output_schema=OutputState` 後，正確結果就是上面註解寫的單鍵 `{'answer': 'bye Lance'}`。照抄 notebook 輸出會被誤導，以這裡為準。

### 6.4.6 踩雷現場 2：API 版本坑 + 「型別提示 ≠ 過濾」

這一節有兩個最常見的翻車點，一起演。

**坑 A：照抄舊版的 `input=` / `output=` 位置參數。**

❌ 舊教材、舊 notebook 常見這種寫法——

```python
# 舊版寫法，langgraph 1.x 已不該再用
builder = StateGraph(OverallState, input=InputState, output=OutputState)
```

**實際會發生什麼**：在 `langgraph` 1.x 上，`input=` / `output=` 是**已棄用（deprecated）的舊參數名**。它目前還能跑、過濾結果也正確，但每次都會丟出一條 deprecation 警告（實測訊息：`` `input` is deprecated and will be removed. Please use `input_schema` instead. ``），而且官方明講「Deprecated in LangGraph V0.5 to be removed in V2.0」——也就是**到 V2.0 會被整個移除，屆時這行直接報錯**。所以照抄是埋了一顆未來會爆的雷。

✅ **正確寫法**：用關鍵字 **`input_schema=` / `output_schema=`**——

```python
builder = StateGraph(OverallState, input_schema=InputState, output_schema=OutputState)
```

**坑 B：以為節點上的型別提示 `state: InputState` 本身會「強制過濾」。**

❌ 錯誤期待：在 `thinking_node(state: InputState)` 標了 `InputState`，就以為「這個節點只看得到 / 只能寫 `question`」。

**實際會發生什麼**：型別提示**不會**幫你過濾任何東西。節點實際讀到的、能寫進去的，是整張圖（聯集）的 channels；`thinking_node` 照樣能寫出 `notes`（它確實寫了），也照樣能讀到 `answer`。型別提示在這裡**純粹是文件／語意用途**——告訴讀程式的人「這個節點主要關心 `InputState` 的欄位」，但它不是一道門。

**為什麼**：真正決定「什麼能進、什麼能出」的，是 `StateGraph(...)` 的 `input_schema=` / `output_schema=` 參數，**不是節點簽名上的型別提示**。

✅ **正確心智模型**：過濾的開關在**建圖那一行**，型別提示只是註解。想真正限制進出，就把 `input_schema=` / `output_schema=` 設對；型別提示拿來幫讀者理解資料流就好。

### 6.4.7 何時用多重 schema

- **對外要乾淨的 API 邊界**（只收特定欄位、只回特定欄位、不想把內部 `notes` 之類的東西外洩）→ 用 `input_schema=` / `output_schema=`。
- **只是節點間中繼、不該外露的暫存**（半成品、計分、debug 註記）→ 用 `PrivateState`，並給它**不撞名**的 key。
- **一般小圖**：兩者都用不到，單一 schema 最省事。**知道它存在就好，等到設計大型圖、要對外暴露穩定介面時再拿出來。**

> 🩸 **血淚教訓** —— 我曾經以為把欄位塞進 `PrivateState` 就等於「對外隱藏」，結果因為跟對外欄位同名，那個我以為藏起來的中間值**原封不動印在 API 回應裡**送出去，還夾帶了內部評分邏輯。真正讓欄位不外露的，從來不是它宣告在哪個 class，而是**它的名字到底有沒有出現在 `OverallState` / `OutputState` 上**。

### 6.4.8 常見誤解清單

1. **「節點只能讀寫建圖時傳入的那個 schema。」** —— 錯。因為所有 schema 的欄位被取**聯集**成同一組 channels，節點其實能讀寫**任何已定義 schema 的欄位**（範例 1 的 `node_2` 標 `PrivateState` 卻寫得回 `OverallState` 的 `foo` 就是證明）。型別提示只是文件，不是權限。
2. **「`PrivateState` 是真正獨立隔離的命名空間。」** —— 錯。是否外露**只看 key 名是否屬於 `OverallState` / `OutputState`**；同名 key 會共用同一個底層 channel，根本沒有「私有牆」。要私有，就取不撞名的 key。
3. **「照抄 notebook 的 `input=` / `output=` 與它的存檔輸出。」** —— 兩個都會害你：舊參數名 `input=` / `output=` 在 1.x 雖然還能跑，但已標記 deprecated、會丟警告，而且**到 V2.0 會被移除**，該換成 `input_schema=` / `output_schema=`；而那份 notebook 過濾範例的存檔輸出是錯的（仍顯示三鍵），正確結果只剩單一 `answer` 鍵。

---

## 常見坑（Pitfalls）

- **以為「回傳更新」會自動合併 list**：預設是覆蓋。想累加（對話、累積結果）一定要貼 reducer（`add_messages` 或 `operator.add`）。
- **對話欄位用 `operator.add` 而非 `add_messages`**：human-in-the-loop 改寫既有訊息時會變成重複附加。對話一律用 `add_messages`。
- **想用 Pydantic state 又想用 `create_agent`**：`create_agent` 不支援 Pydantic schema。要嚴格驗證就留在純 LangGraph 層用 `TypedDict` 以外的選擇。
- **直接用索引存取 message 的內容卻拿到 dict**：`add_messages` 會把更新反序列化成 Message 物件，請用 `.content` 等點記法存取，而非當成 dict。
- **state 欄位過多、什麼都塞**：呼應 Ch4——能推導的別存。state 愈精簡，reducer 行為愈好掌握、checkpoint 愈小。

## 本章小結

State 有兩半：**schema**（結構）與 **reducer**（更新如何套用）。Schema 可用 `TypedDict`（首選）、`dataclass`（要預設值）或 Pydantic（要驗證，但 `create_agent` 不支援），三者裡只有 Pydantic 會在「入口」擋髒資料（6.2）。reducer 方面，預設是「覆蓋」，這對 `category`、`reply` 這類欄位剛好；但對話、累積結果這類要「累加」的欄位，得用 `Annotated[type, reducer]` 指定自訂 reducer——而且平行 fan-out 對白板欄位寫入會直接 `InvalidUpdateError`（6.3）。對話的標準作法是 `add_messages`——它會附加新訊息、但對帶既有 ID 的訊息做覆蓋（支援 human-in-the-loop 改寫），還順手處理序列化；而 `MessagesState` 把這個常見模式打包好，繼承它再加欄位是日後最常見的起手式。最後，input/output/private 多重 schema 讓你把對外介面與對內工作通道分開，但要記得「私有」靠的是「名字不在對外清單上」而非牆（6.4）。你現在已經完全理解「節點回傳的更新，是怎麼變成新 state 的」——下一章，我們再往下挖一層，看 LangGraph 的執行引擎 Pregel 究竟如何排程這一切。

## 延伸閱讀 / 練習

1. **reducer 對照實驗**：寫一個有兩個欄位的 state，一個不貼 reducer、一個貼 `operator.add`，各跑兩個會更新它們的節點，印出最終 state，親眼確認「覆蓋 vs 累加」的差異。
2. **改造 email agent**：把 Ch5 的 email agent 的 state 改成繼承 `MessagesState`，讓信件往來以 message list 形式累積，觀察 `add_messages` 的行為。
3. **ID 覆蓋**：建構一個帶相同 ID 的訊息更新，驗證 `add_messages` 是覆蓋既有訊息而非重複附加。
4. **多重 schema**（呼應 6.4）：把 6.4 的範例改成「輸出只回傳一個欄位、輸入只接受一個欄位」，確認中間的 `notes` 不會出現在最終輸出裡。
5. **Literal 不擋值親測**（呼應 6.2）：用 `TypedDict` 給某欄位標 `Literal["happy","sad"]`，在節點裡硬塞 `"mad"`，跑完印出最終 state，確認程式不報錯、髒值照樣留在 state；再換成 Pydantic `@field_validator` 版本，確認這次 `ValidationError` 當場攔下。
6. **平行衝突重現**（呼應 6.3）：對一個白板（無 reducer）欄位做平行 fan-out 寫入，重現 `InvalidUpdateError`；接著把它改成 `Annotated[list[int], add]`，確認衝突消失、兩筆都被串進去。
7. **私有不是牆**（呼應 6.4）：讓 `PrivateState` 跟 `OverallState` 取同名 key，驗證它「不會被藏起來」原封不動出現在輸出；再改成不撞名的 key，確認這次才真的不外露。
