# Ch 14　長期記憶與跨對話 store

> **本章目標**
>
> - 釐清 checkpointer（短期、綁 thread）與 store（長期、跨 thread）的分工。
> - 學會用 `InMemoryStore` 的 `put` / `search`，以 namespace 組織記憶，並讀懂回傳的 `Item`。
> - 把 store 接進圖，讓 agent 跨對話記住使用者偏好。
> - 認識 production 用的持久 store 與語意檢索。
>
> **使用版本**：`langgraph` 1.x。配套 notebook **不需 API key**。

---

## 14.1 短期 vs 長期：checkpointer 不夠用的時候

Ch8 我們用 checkpointer 得到「記憶」——但那是**短期記憶**，綁在 `thread_id` 上。同一個 thread 裡 agent 記得先前對話；換一個 thread，記憶就清空。

問題來了：一個聊天機器人，你會希望它**跨所有對話**都記得關於你的事——你叫什麼、你的偏好、你的專案慣例。這些資訊不該綁死在某一段對話（thread）裡。但**只靠 checkpointer，資訊無法跨 thread 共享**。

這就是 **store** 介面存在的理由：

| | checkpointer（Ch8） | store（本章） |
|---|---|---|
| 範圍 | 單一 thread 內 | 跨 thread（跨對話） |
| 用途 | 短期記憶、HITL、time travel、回復 | 長期記憶：使用者偏好、專案知識 |
| 綁定 | `thread_id` | namespace（可任意設計） |

兩者常常**一起用**：compile 圖時同時給 checkpointer（短期）與 store（長期）。

## 14.2 store 基礎：put 與 search

先脫離 LangGraph，單獨看 store 怎麼用。記憶用一個 **namespace（tuple）** 來分類，namespace 可以是任意長度、代表任何東西（不一定要跟使用者有關）：

```python
import uuid
from langgraph.store.memory import InMemoryStore

store = InMemoryStore()

user_id = "1"
namespace = (user_id, "memories")        # namespace 是個 tuple，這裡用 (使用者, "memories")

# 用 put 存一筆記憶：指定 namespace、一個唯一 key、和值（一個 dict）
memory_id = str(uuid.uuid4())
store.put(namespace, memory_id, {"food_preference": "我喜歡披薩"})

# 用 search 讀出某 namespace 下的記憶（回傳 list，預設上限 10 筆）
memories = store.search(namespace)
print(memories[-1].dict())
# {'value': {'food_preference': '我喜歡披薩'},
#  'key': '...', 'namespace': ['1', 'memories'],
#  'created_at': '...', 'updated_at': '...'}
```

`search` 回傳的每筆是一個 `Item`，可用 `.dict()` 轉成字典。它的屬性：`value`（記憶內容，本身是 dict）、`key`（此 namespace 下的唯一 key）、`namespace`（字串 tuple）、`created_at` / `updated_at`（時間戳）。

> 小提醒：`InMemoryStore` 依**插入順序**回傳（最新在最後）；不同後端排序可能不同（例如 `PostgresStore` 預設依 `updated_at` 倒序）。**別依賴跨後端的順序**，要排序就自己依 `item.updated_at` 排。

## 14.3 列舉與分頁

`search` 不帶 query 與 filter 時，會回傳某 namespace 前綴下的項目（до `limit`）。有三件事要記住：

- **namespace 是「前綴比對」**：`("alice",)` 也會回傳 `("alice", "memories")`、`("alice", "preferences")` 等底下的項目。要限定單一層，傳完整 namespace 或在 client 端用 `item.namespace` 過濾。
- **超過 `limit` 會被靜默截斷**：沒有溢位訊號。把 `limit` 設高於你預期的最大量，或用 `offset` 分頁。
- **預設排序依後端而定**：別跨後端假設順序。

分頁與列舉 namespace：

```python
# 分頁讀大量記憶
page_size, offset = 50, 0
while True:
    page = store.search(("alice", "memories"), limit=page_size, offset=offset)
    if not page:
        break
    for item in page:
        ...
    offset += page_size

# 列出存在哪些 namespace（例如先列出所有使用者）
namespaces = store.list_namespaces(prefix=("alice",), max_depth=2)
```

## 14.4 把 store 接進圖

實際用時，compile 圖同時給 checkpointer 與 store；節點就能存取跨 thread 的長期記憶：

```python
from langgraph.checkpoint.memory import InMemorySaver
from langgraph.store.memory import InMemoryStore

graph = builder.compile(
    checkpointer=InMemorySaver(),    # 短期：thread 內
    store=InMemoryStore(),           # 長期：跨 thread
)
```

節點函數可以拿到 store 來讀寫長期記憶（例如把使用者剛透露的偏好存起來，下次任何對話都讀得到）。一個典型流程：在對話中，agent 把「使用者說他喜歡披薩」`put` 進 `(user_id, "memories")`；之後**不同 thread** 的對話裡，agent `search` 同一個 namespace，就能讀回這個偏好。短期記憶（checkpointer）負責「這段對話的脈絡」，長期記憶（store）負責「關於這個人的恆久事實」。

## 14.5 上 production 與語意檢索

`InMemoryStore` 適合開發測試；上 production 要換持久後端，例如 `PostgresStore`、`MongoDBStore`、`RedisStore`。它們都繼承自 `BaseStore`——**在節點簽名裡用 `BaseStore` 當型別註記**，就能無痛切換後端。

```python
# 開發
from langgraph.store.memory import InMemoryStore
store = InMemoryStore()
# production（示意）
# from langgraph.store.postgres import PostgresStore
# store = PostgresStore(...)
```

跟 checkpointer 一樣，若你用 **Agent Server / LangGraph API** 部署，store 的儲存基礎設施會被自動處理，你不必手動配置。

進一步，store 還支援**語意檢索（semantic search）**：替記憶建立 embedding 後，你可以用「意思相近」而非「關鍵字相同」去找記憶——這對「找出和當前話題相關的歷史記憶」特別有用。基本用法是 `put` 一樣存、`search` 時帶上自然語言 query。需要時再深入即可。

---

## 常見坑（Pitfalls）

- **想跨對話記東西卻只用 checkpointer**：checkpointer 綁 thread，跨不了對話。跨 thread 的長期記憶要用 store。
- **依賴 store 的回傳順序**：不同後端排序不同（InMemory 插入序、Postgres 依 updated_at 倒序）。要順序就自己排。
- **忘了 namespace 是前綴比對**：`("alice",)` 會撈到底下所有子 namespace。要精確就傳完整 namespace。
- **以為超過 `limit` 會報錯**：它會靜默截斷。設高 `limit` 或用 `offset` 分頁。
- **production 還用 `InMemoryStore`**：一關機長期記憶就沒了。用 Postgres/Mongo/Redis，或交給 Agent Server。

## 本章小結

checkpointer 給的是綁 thread 的**短期記憶**；要跨對話、跨 thread 記住使用者偏好與恆久知識，需要 **store**。store 用 `put(namespace, key, value)` 存、`search(namespace)` 讀，namespace 是可任意設計的 tuple，回傳的 `Item` 帶 `value`/`key`/`namespace`/時間戳。注意 namespace 是前綴比對、`limit` 會靜默截斷、排序依後端而定。實務上 compile 圖時同時給 checkpointer（短期）與 store（長期）：前者管「這段對話的脈絡」，後者管「關於這個人的事實」。production 換 Postgres/Mongo/Redis（都繼承 `BaseStore`，用它當型別註記即可無痛切換），或交給 Agent Server 自動處理；store 還支援語意檢索。至此 **Part I 的 LangGraph 核心**全部到齊——你已經能從零打造一個有狀態、可回復、會串流、能暫停等人、模組化、且具長短期記憶的 agent。下一章起進入 Part II「銜接篇」，看這些低層能力如何被 LangChain 的 `create_agent` 與 middleware 包裝成更好用的抽象。

## 延伸閱讀 / 練習

1. **跨 thread 記憶**：用 `InMemoryStore` 在一段對話 `put` 一個偏好，再在「不同 `thread_id`」的對話裡 `search` 出來，親眼確認它跨 thread（配套 notebook 有可跑版本，免 API key）。
2. **namespace 前綴**：在 `("alice", "memories")` 與 `("alice", "preferences")` 各存一筆，用 `search(("alice",))` 觀察前綴比對撈到兩者。
3. **短長並用**：compile 一個同時帶 checkpointer 與 store 的圖，想一個「哪些資訊該放 thread（短期）、哪些該放 store（長期）」的劃分。
4. **思考題**：一個客服 agent，「本次工單的對話內容」與「這位客戶的長期偏好」分別該用 checkpointer 還是 store？為什麼？
