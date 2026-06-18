# Ch 7　執行模型內幕：Pregel、super-step 與 channels

> **本章目標**〔深入章〕
>
> - 揭開 LangGraph 的黑盒：你 `compile` 出來的圖，**實際上是怎麼被執行的**。
> - 理解 **Pregel** 執行模型——message passing、super-step、Bulk Synchronous Parallel（BSP）三相循環。
> - 認識 **channels**（`LastValue`、`Topic`、`BinaryOperatorAggregate`），並看清它們和 Ch6 的 reducer 其實是同一件事。
> - 用這層理解，解開幾個「不懂執行模型就會踩、且很難 debug」的真實坑——尤其是平行節點寫同一欄位的衝突。
>
> **使用版本**：`langgraph` 1.x。本章偏概念與心智模型，是「想精通」讀者的關鍵一章。
>
> > 為什麼要讀這章？因為 Ch2 說過：Deep Agents 蓋在 LangGraph 上，懂底層的人能拆開上層除錯。而 LangGraph 自己的底層，就是這章講的 Pregel。把它讀透，你對「state 為什麼這樣更新、平行為什麼會衝突、長對話為什麼 checkpoint 會膨脹」都會有第一性原理級的理解。

---

## 7.1 你 compile 出來的，其實是一個 Pregel

先講一個多數人不知道的事實：當你 `compile()` 一個 `StateGraph`（或建一個 `@entrypoint`），LangGraph 產生的是一個 **`Pregel` 實例**。Pregel 才是 LangGraph 真正的 runtime——它負責管理整個應用的執行。`StateGraph` 只是一個讓你**好寫**的高階介面，底下跑的引擎叫 Pregel。

這個名字來自 Google 的 Pregel 演算法，它描述了一種「用圖做大規模平行運算」的高效方法。LangGraph 借用了它的核心思想，但你不需要懂分散式系統——我們只取它對理解 agent 執行最有用的部分。

## 7.2 核心機制：message passing 與 super-step

LangGraph 底層用 **message passing（訊息傳遞）** 來驅動執行。機制是這樣：

> 當一個節點完成運算，它沿著一條或多條邊，把「訊息」送給其他節點。收到訊息的節點接著執行自己的函數，再把結果送往下一批節點，如此繼續。

而整個過程，被組織成一連串離散的 **super-step（超步）**。一個 super-step 可以理解為「對圖節點的一次迭代」。關鍵規則只有兩條，請記牢：

- **平行跑的節點，屬於同一個 super-step。**
- **依序跑的節點，屬於不同的 super-step。**

節點有「活躍／非活躍」狀態。執行開始時所有節點都非活躍；當一個節點在它的某條**入邊（也就是 channel）** 上收到新訊息（新的 state），它就變活躍、執行函數、回傳更新。每個 super-step 結束時，沒有收到任何入向訊息的節點會「投票停止」（把自己標為非活躍）。**當所有節點都非活躍、且沒有任何訊息在傳遞中，整個圖就執行結束。**

用 Ch5 的 email agent 走一遍，你會很有感：

```
super-step 1:  START 把輸入送進 classify  →  classify 活躍、執行，寫出 {category: ...}
super-step 2:  classify 沿條件邊把訊息送給 escalate（假設是 complex）
               →  escalate 活躍、執行，寫出 {reply: ...}；draft 沒收到訊息，不活躍
super-step 3:  escalate 指向 END，無人再收到訊息  →  全體非活躍  →  結束
```

注意 `draft` 從頭到尾沒變活躍，因為沒有訊息送到它的入邊。這就是 Ch5「未執行的節點」現象在執行模型層的解釋。

## 7.3 BSP 三相循環：Plan → Execution → Update

把 super-step 再放大來看，Pregel 遵循 **Bulk Synchronous Parallel（BSP）** 模型，每一個 step 由三個相位組成：

1. **Plan（規劃）**：決定這一步要執行哪些 actor（節點）。第一步選那些訂閱了輸入 channel 的節點；後續步驟選那些訂閱了「上一步被更新的 channel」的節點。
2. **Execution（執行）**：把選中的節點**平行**執行，直到全部完成、或其中一個失敗、或超時。**這一相位裡，channel 的更新對節點是不可見的**——大家讀到的都是上一步結束時的狀態。
3. **Update（更新）**：用這一步各節點寫出的值，更新 channels。

然後重複，直到沒有節點被選中、或達到最大步數。

第 2 相位那句話是整章最重要的一句，請畫線：**在同一個 super-step 內平行執行的節點，彼此看不到對方的寫入；所有寫入要等到 Update 相位才一起套用。** 這個「批次同步」特性，正是「平行節點為什麼會在更新同一欄位時衝突」的根源——下一節就會看到。

> 💡 **冷知識** —— Pregel 的設計初衷不是為了 agent，而是 Google 在 2010 年要對「網頁圖（PageRank、social graph）」做兆級規模的迭代計算。BSP 最大賣點不是「快」，而是 **deterministic（同樣輸入跑一千次結果一樣）**。LangGraph 借用這個特性，讓你的 agent 圖**天生可重播、可 time travel**（Ch11 的 replay/fork 全靠這層）——**非確定的是模型，不是 runtime**。這個區分對除錯很重要：runtime 給你「不變」，模型給你「智能」。

## 7.4 Channels：reducer 的真面目

前面一直提到 channel。**Channel 是 actor（節點）之間溝通的管道。** 每個 channel 有三樣東西：一個值的型別、一個更新的型別、以及一個**更新函數**（接收一串更新、修改所存的值）。

讀到「更新函數」你應該要有既視感——**這不就是 Ch6 的 reducer 嗎？** 沒錯。Ch6 你用 `Annotated[list, add_messages]` 替某個 state 欄位指定 reducer，底層做的事，就是**把那個欄位變成一個帶特定更新函數的 channel**。`StateGraph` 把你的 state schema 編譯成一組 channels，把你的 reducer 編譯成 channels 的更新函數。Ch6 講的是「面向使用者的 reducer」，這章講的是「它在引擎裡的真身——channel」。

LangGraph 內建幾種 channel 型別：

- **`LastValue`（預設）**：只存「最後寫入的值」，覆蓋先前的值。這正是 Ch6 那個「預設 reducer = 覆蓋」的真身。用於輸入、輸出、或把資料從一步傳到下一步。
- **`Topic`**：一個可設定的 PubSub channel，適合在節點間送多個值、或跨步驟累積輸出。可設定去重或累積。
- **`BinaryOperatorAggregate`**：存一個持續累積的值，每次更新時用一個二元運算子（如 `operator.add`）把現值與新值合併。用於跨步驟計算 running aggregate——Ch6 你貼 `operator.add` 累加 list，底層就是它。

```python
import operator
from langgraph.channels import LastValue, BinaryOperatorAggregate

last: LastValue[int] = LastValue(int)                       # 覆蓋
total = BinaryOperatorAggregate(int, operator.add)          # 累加
```

絕大多數時候你不會直接碰 channel——你透過 `StateGraph` + reducer 間接使用它們。但知道「reducer 就是 channel 的更新函數」，你對 state 行為的理解就從「背 API」升級成「懂原理」。

> ⏱ **三秒判斷** —— 你的圖有沒有平行？
> - 從同一個節點（如 `START`）**指向多個**節點 → **有平行**
> - 多個節點都從同一上游分岔、且都**指向同一彙整節點** → **有平行**
> - 線性 `A → B → C → D` → **沒有平行**
>
> 一旦有平行寫同欄位，**要嘛改順序、要嘛貼 reducer**（呼應 Ch6）。

## 7.5 用執行模型解開三個真實坑

理解 7.3 的 BSP 與 7.4 的 channel，下面這些原本「玄學」的問題就變得理所當然。

**坑一：平行節點寫同一個欄位，報 `InvalidUpdateError`。** 假設你讓兩個節點在同一個 super-step 平行跑，且都寫 `summary` 這個用預設 `LastValue` 的欄位。Update 相位收到「兩個對同一 channel 的寫入」，但 `LastValue` 不知道該留哪個（它只能存一個），於是報錯。**解法**：給這個欄位一個能合併多個寫入的 reducer（例如 `operator.add` 或 `add_messages`），也就是換一個能吃多筆更新的 channel。一旦你知道「平行寫入在 Update 相位一起套用、由 channel 的更新函數決定怎麼合併」，這個錯就一目了然。

**坑二：平行節點「讀不到」彼此剛寫的值。** 你以為 A、B 平行跑時，B 能讀到 A 寫的東西——讀不到。因為 7.3 第 2 相位：同一 super-step 內，寫入互不可見，要到下一步才生效。**解法**：若 B 真的依賴 A 的輸出，它們就不該平行——讓 B 在 A 的**下一個** super-step 跑（用邊把 A→B 串成順序）。

**坑三：長對話的 checkpoint 越來越肥、越來越慢。** 每一步都把整個對話 list 重新序列化進 checkpoint，thread 一長，checkpoint 大小就線性成長。針對這個，`langgraph>=1.2` 提供了 beta 的 **`DeltaChannel`**：它每一步只存「增量 delta」而非完整累積值，特別適合「寫得頻繁、又會長很大」的欄位（例如長 thread 的訊息 list）。判斷訊號很簡單——**如果你發現某個 channel 的 checkpoint 大小隨 thread 長度線性增長，它就是 `DeltaChannel` 的好對象**。（`DeltaChannel` 用的是「bulk reducer」，且 reducer 在「重建時」而非「寫入時」執行，設計上有些講究，需要時再深入。）

> 🩸 **血淚教訓** —— 我曾經給一個有 5 個平行 node 的圖設 `recursion_limit=10`，覺得「10 步應該夠了」。結果一直撞牆——因為 **`recursion_limit` 算的是 super-step 數，不是節點總數**。我把「跑 5 個節點」想成 5 步，但它們**屬於同一 super-step**；加上前後的路由節點、彙整節點，實際是 8 步左右。一個圖的 step 數要算「**依序執行的層數**」，不是節點總數。這個小誤解花了我半天追真因——也是為什麼 7.2 那兩條規則我特別請你「牢牢記住」。

## 7.6 你也能直接用 Pregel（但多半不必）

雖然多數人透過 `StateGraph` 或 `@entrypoint` 間接使用 Pregel，但你也可以直接操作它——用 `NodeBuilder` 宣告節點訂閱哪些 channel、寫到哪些 channel：

```python
from langgraph.channels import EphemeralValue
from langgraph.pregel import Pregel, NodeBuilder

node1 = NodeBuilder().subscribe_only("a").do(lambda x: x + x).write_to("b")

app = Pregel(
    nodes={"node1": node1},
    channels={"a": EphemeralValue(str), "b": EphemeralValue(str)},
    input_channels=["a"],
    output_channels=["b"],
)
print(app.invoke({"a": "foo"}))   # {'b': 'foofoo'}
```

這段程式把 7.2~7.4 的概念赤裸地攤開：節點明確「訂閱 channel a、寫到 channel b」。你幾乎永遠不會這樣寫產品程式（`StateGraph` 好用太多），但讀懂它，等於確認你真的理解了底層在做什麼。把它當成一面驗證自己理解的鏡子。

---

## 常見坑（Pitfalls）

- **以為平行節點能即時看到彼此的寫入**：BSP 模型下，同一 super-step 的寫入互不可見，下一步才生效。需要順序依賴就用邊串起來。
- **平行寫同一個 `LastValue` 欄位**：必然衝突報錯。要嘛改成順序執行，要嘛給該欄位一個能合併多筆寫入的 reducer/channel。
- **把「節點數」當「super-step 數」**：平行的節點屬同一 super-step。step 數對應的是「依序的層數」，不是節點總數——這會影響你對 `recursion_limit`（最大步數）的估算。
- **長 thread 不管 checkpoint 膨脹**：放著不管會越來越慢、越來越貴。留意線性成長的 channel，評估 `DeltaChannel`（或在更上層做 context 壓縮，見 Ch34）。

## 本章小結

你 `compile` 出來的 `StateGraph` 底下是一個 **Pregel** runtime。它用 **message passing** 驅動執行，把過程切成一連串 **super-step**——平行的節點同屬一步、依序的節點分屬不同步；節點收到入向訊息才活躍，全體非活躍且無訊息在途時結束。每個 step 遵循 **BSP** 三相：Plan（選節點）、Execution（平行跑，期間寫入互不可見）、Update（套用寫入）。而你在 Ch6 用的 reducer，真身就是 **channel 的更新函數**：`LastValue` 是覆蓋、`BinaryOperatorAggregate` 是累加、`Topic` 是 PubSub。理解這層，你就能秒解「平行寫衝突、平行讀不到、長對話 checkpoint 膨脹」這些原本玄學的問題。這正是「精通」與「會用」的分水嶺——也是你日後能拆開 Deep Agents 除錯的底氣。接下來，我們把目光轉到 runtime 的另一個支柱：persistence，讓 agent 真正擁有記憶與可回復的能力。

## 延伸閱讀 / 練習

1. **數 super-step**：給 Ch5 的 email agent 跑一封信，對照 7.2 的範例，自己數出它經歷了幾個 super-step，並說明 `draft` 為何沒活躍。
2. **製造衝突**：故意建兩個平行節點都寫同一個 `LastValue` 欄位，跑跑看那個 `InvalidUpdateError`，再用「加 reducer」與「改順序」兩種方式各修一次。
3. **驗證可見性**：建 A、B 兩個平行節點，B 嘗試讀 A 在同一步寫的值，印出來確認它讀到的是「上一步的舊值」。
4. **channel ↔ reducer 對照**：寫一句話解釋「為什麼 `Annotated[list, operator.add]` 等價於把該欄位設成 `BinaryOperatorAggregate`」。能講清楚，你就真的懂 channel 了。
