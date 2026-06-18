# Ch 1　為什麼需要 agent 框架：從一次 LLM 呼叫到 agent loop

> **本章目標**
>
> - 看清楚「一次 LLM 呼叫」的能力邊界，理解為什麼純粹的 prompt 不足以完成真實任務。
> - 建立全書最核心的心智模型：**agent loop**（reason → act → observe 的迴圈）。
> - 親手用純 Python + 一個 LLM SDK 刻一個最小可跑的 agent，藉此看清「沒有框架時，你得自己扛下哪些事」。
> - 理解從「能跑的 demo」到「可靠上線（production-reliable）」之間那道鴻溝——這正是後續 LangGraph、LangChain、Deep Agents 要替我們填平的東西。
>
> **使用版本**：`google-genai`（Google Gen AI SDK；本章不依賴任何 agent 框架，刻意如此）。模型以 `gemini-2.5-flash` 為例。

---

## 1.1 一次 LLM 呼叫能做什麼，不能做什麼

我們先把問題擺到最素的狀態。一次 LLM 呼叫，本質上是一個**純函數**：你丟一段文字（或一串訊息）進去，它吐一段文字出來。

```python
from google import genai

client = genai.Client()   # 讀環境變數裡的 GOOGLE_API_KEY

resp = client.models.generate_content(
    model="gemini-2.5-flash",
    contents="台北今天適合戶外活動嗎？",
)
print(resp.text)
```

這段程式會得到一個看起來很流暢的回答——但它幾乎一定是**錯的**，或者說，是「編的」。因為模型沒有今天的天氣資料，它只能根據訓練語料裡的統計規律，生出一段「聽起來合理」的文字。

這裡就點出了單次呼叫的三個根本限制，請記住它們，因為整本書都在處理這三件事：

第一，**模型沒有即時、私有或外部的資訊**。它不知道今天的天氣、你資料庫裡的訂單、這份 PDF 的內容。它的知識停在訓練截止日，且只涵蓋公開語料。

第二，**模型不能執行動作**。它無法真的去查 API、寫檔案、跑一段程式、發一封信。它只會「說」，不會「做」。

第三，**一次呼叫就是一個沒有記憶、不能修正的單步**。它無法「先看一下結果再決定下一步」。如果任務需要好幾個步驟、而且後面的步驟取決於前面的結果，單次呼叫就無能為力。

真實世界的任務幾乎都同時踩中這三點。「幫我查台北今天天氣再決定要不要安排路跑」需要外部資料（限制一）、需要實際去查（限制二）、而且「要不要安排」取決於查到什麼（限制三）。

> ⏱ **三秒判斷** —— 你的任務「需要 agent」嗎？看三個訊號：
> - 需要**即時／外部／私有資料**？
> - 需要**對世界做動作**（寫檔、發信、呼 API）？
> - 後面的步驟**取決於前面的結果**？
>
> **任一個 yes，就需要 agent loop**；三個都 no，一次 LLM 呼叫就夠了，別過度設計。

## 1.2 工具呼叫：讓模型「動手」

突破口叫做 **tool calling**（工具呼叫，早期也叫 function calling）。它的概念其實很簡單：我們在呼叫模型時，額外告訴它「你有哪些工具可用、每個工具長什麼樣」；模型如果判斷需要用工具，就不直接回答，而是回傳一個**結構化的工具呼叫請求**（要呼叫哪個工具、帶什麼參數）。

關鍵要釐清一件常被誤解的事：**模型自己並不會執行工具**。它只會「說：請幫我呼叫 `get_weather('Taipei')`」。真正去執行這個函數的，是**你的程式碼**。執行完，你再把結果回傳給模型，它才據此產生最終答案。

換句話說，模型負責「決策」，你的程式負責「執行」，兩邊一來一回。我們用 Google 的 Gen AI SDK 把這件事寫出來：

```python
from google import genai
from google.genai import types

client = genai.Client()   # 讀環境變數裡的 GOOGLE_API_KEY

# 宣告工具：給模型看的「說明書」（名稱、用途、參數 schema）。
# 用 FunctionDeclaration 手動宣告，模型就不會自動執行——把決定權留在我們眼前。
get_weather_decl = types.FunctionDeclaration(
    name="get_weather",
    description="查詢指定城市的目前天氣",
    parameters={
        "type": "object",
        "properties": {
            "city": {"type": "string", "description": "城市名稱，例如 Taipei"}
        },
        "required": ["city"],
    },
)
config = types.GenerateContentConfig(
    tools=[types.Tool(function_declarations=[get_weather_decl])]
)

resp = client.models.generate_content(
    model="gemini-2.5-flash",
    contents="台北現在天氣如何？",
    config=config,
)

print(resp.function_calls)   # 不為空 —— 模型決定要用工具，而不是直接回答
```

當 `resp.function_calls` 不為空，代表模型把球踢回來了：它想呼叫工具，等你把結果交給它。注意——到這裡任務還沒完成，模型只是「提出請求」。要讓它真的有用，我們得補上後半段：執行工具、回傳結果、讓模型繼續。而「執行 → 回傳 → 繼續」一旦需要重複多次，就形成了本書的主角——agent loop。

> 💡 **冷知識** —— **ReAct** 這名字來自 2022 年 Yao 等人的論文〈ReAct: Synergizing Reasoning and Acting in Language Models〉。在那之前，模型多半「只想不做」（Chain-of-Thought）或「只做不想」（直接生成 action）；ReAct 的洞見是讓兩者**交錯穿插**——`Thought → Action → Observation → Thought → ...`。所有現代 agent 框架的迴圈骨架（LangGraph 的圖、LangChain 的 `create_agent`、Deep Agents 的 harness）拆到最底層，都是這個想法的後代。

## 1.3 agent loop：reason → act → observe

把單次工具呼叫擴展成「可以連續做很多步、每步都看著上一步結果決定下一步」的迴圈，就是 **agent loop**。它源自 2022 年的 ReAct（**Rea**soning + **Act**ing）論文，核心是讓模型交錯地「推理」與「行動」：

```
        ┌─────────────────────────────────────────┐
        │                                           │
        ▼                                           │
   ┌─────────┐     ┌──────────┐     ┌───────────┐  │
   │ Reason  │────▶│   Act    │────▶│  Observe  │──┘
   │ 模型思考 │     │ 執行工具  │     │ 把結果回傳 │
   │ 下一步   │     │（你的程式）│     │ 給模型     │
   └─────────┘     └──────────┘     └───────────┘
        │
        │ 模型認為任務完成（不再要求用工具）
        ▼
   ┌─────────┐
   │  最終   │
   │  回答   │
   └─────────┘
```

一句話描述這個迴圈：**模型推理該做什麼（reason）→ 你的程式執行它要的動作（act）→ 把動作結果餵回給模型（observe）→ 回到第一步，直到模型不再要求動作、直接給出答案為止。**

這個迴圈看似簡單，卻是所有 agent 的共同骨架。不論是 LangGraph 裡你親手連的圖、LangChain 的 `create_agent`、還是 Deep Agents 的 harness，拆到最底層，跑的都是這同一個 loop。本書後面所有的機制——state、persistence、streaming、subagents——本質上都是在這個 loop 上「加東西」。先把這個骨架刻進腦子，後面所有抽象你都能還原回它。

## 1.4 動手刻一個最小 agent

光看圖不夠，我們把這個 loop 真的寫出來。下面這段程式**不依賴任何 agent 框架**，只用 Google GenAI SDK，目的是讓你親身感受「一個 agent loop 究竟要處理哪些細節」。

```python
from google import genai
from google.genai import types

client = genai.Client()   # 讀環境變數裡的 GOOGLE_API_KEY

# --- 真正會被執行的工具實作 ---
def get_weather(city: str) -> str:
    # 真實情境這裡會去打天氣 API；這裡先寫死方便示範
    fake_db = {"Taipei": "晴，28°C", "Tokyo": "陰，22°C"}
    return fake_db.get(city, "查無此城市資料")

TOOL_IMPLEMENTATIONS = {"get_weather": get_weather}

# --- 給模型的工具說明書 ---
get_weather_decl = types.FunctionDeclaration(
    name="get_weather",
    description="查詢指定城市的目前天氣",
    parameters={
        "type": "object",
        "properties": {"city": {"type": "string"}},
        "required": ["city"],
    },
)
config = types.GenerateContentConfig(
    tools=[types.Tool(function_declarations=[get_weather_decl])]
)

def run_agent(user_input: str, max_turns: int = 10) -> str:
    contents = [types.Content(role="user", parts=[types.Part(text=user_input)])]

    for turn in range(max_turns):            # ← 護欄：避免無窮迴圈
        resp = client.models.generate_content(
            model="gemini-2.5-flash", contents=contents, config=config,
        )
        calls = resp.function_calls          # 模型這一輪要求的工具呼叫（可能多個，或 None）

        if not calls:
            # 模型不再要求工具 → 任務結束，回傳最終文字
            return resp.text

        # 把模型這一輪的輸出（含 function_call）加回對話
        contents.append(resp.candidates[0].content)

        # 模型要求用工具：逐一執行，把結果包成 function_response 回傳
        tool_parts = []
        for fc in calls:
            impl = TOOL_IMPLEMENTATIONS[fc.name]
            output = impl(**dict(fc.args))   # ← 你的程式負責「執行」
            tool_parts.append(
                types.Part.from_function_response(name=fc.name, response={"result": output})
            )
        contents.append(types.Content(role="user", parts=tool_parts))

    return "（達到最大回合數仍未完成）"

print(run_agent("台北和東京現在天氣如何？哪個比較適合戶外活動？"))
```

執行它，模型會先要求查 Taipei、再要求查 Tokyo（甚至一次要求查兩個），你的程式把結果餵回去，最後它綜合兩地天氣給出建議。**這就是一個完整、能跑的 agent。**

請特別留意這段程式裡「框架沒幫你、你得自己做」的幾件事，它們之後都會變成框架的賣點：

- **手動維護 `contents` 對話歷史**：每一輪都要把模型輸出與 function_response 正確地接回去，順序、格式一錯，模型就壞掉。
- **手動 dispatch 工具**：用一個 dict 把工具名稱對應到實作，自己呼叫、自己塞參數。
- **手動設停止條件**：靠 `resp.function_calls` 是否為空判斷結束，還要加 `max_turns` 護欄防止無窮迴圈。
- **完全沒有**：錯誤重試、執行到一半當掉後的回復、把中間步驟即時串流給使用者、把長對話壓縮以免爆 token、人工審核關卡、可觀測性……

最後這一串「完全沒有」，就是下一節的主題。

> 🩸 **血淚教訓** —— 我第一次手刻 agent loop 沒加 `max_turns`，丟一個模糊指令就去開會。回來時 LangSmith 上累積了 200 多輪 tool_use（模型一直查不到自己要的、又不停重試），燒掉快兩百塊美金的 API 額度。從那次起我有條鐵則：**任何 agent loop，第一行先想停止條件**，再寫主體。`max_turns` 不是可選參數，是護欄。

## 1.5 從「能跑」到「可靠上線」的鴻溝

上面那 50 行能跑，但它離「能放上線給真實使用者用」還很遠。LangChain 官方有一句話講得很精準：*做一個 agentic 應用的 prototype 很容易，但要做到夠可靠、能放進 production，仍然非常難。* 這句話幾乎是整個 agent 工具生態存在的理由。

難在哪？把前一節那串「完全沒有」展開，就是一張 production 待辦清單：

- **持久化與回復（persistence / durable execution）**：一個跑十分鐘的 agent，中途機器重啟怎麼辦？能不能從斷點續跑，而不是整個重來？
- **串流（streaming）**：使用者盯著一個轉圈圈等三十秒會跑掉。你得把模型的思考與中間步驟即時吐給前端。
- **人類介入（human-in-the-loop）**：agent 要送出一封信、要刪一個檔案之前，能不能先停下來等人核可？
- **context 管理**：對話一長，token 就爆。哪些要保留、哪些要摘要、哪些要丟到外部存起來？
- **可觀測性（observability）與評估（eval）**：它出錯時你怎麼知道是哪一步壞了？怎麼量化「這次改動讓 agent 變好還是變壞」？
- **編排的掌控力**：當流程不只是「一直 loop」，而是有分支、有平行、有「某些步驟要照固定順序、某些步驟才交給模型自由發揮」，你要怎麼精確控制？

你當然可以像 1.4 那樣，把這些全都自己手刻。但你會很快發現自己在重新發明一個框架——而且是一個沒人測過、只有你懂、出問題只能自己修的框架。這正是 LangChain 生態的三層分工要解決的：

- **LangGraph** 給你低層的編排與 runtime：state、persistence、durable execution、streaming、human-in-the-loop 都內建好。
- **LangChain** 在其上提供好用的抽象（`create_agent`、middleware、跨 provider 的模型介面），讓你不必每次都從 graph 砌起。
- **Deep Agents** 再往上，把「規劃、虛擬檔案系統、subagents、context 管理」這些長任務 agent 的常見需求打包成開箱即用的 harness。

這三層怎麼分工、何時該用哪一層，就是下一章的主題。但無論抽象疊多高，請記得：**它們跑的都還是 1.3 那個 agent loop。** 你已經親手刻過一次了，後面的一切都只是在這個骨架上加裝備。

---

## 常見坑（Pitfalls）

- **把工具呼叫當成「模型自己會執行」**：模型只會「請求」呼叫工具，執行永遠是你的程式的責任。沒接上後半段（執行 + 回傳），工具呼叫等於沒發生。
- **忘了把模型那一輪的輸出接回 `contents`**：很多人只把 function_response 加回去，卻漏掉模型那一輪含 `function_call` 的 content，導致對話錯亂或 API 報錯。function_call 與對應的 function_response 必須成對、依序出現。
- **沒有設迴圈上限**：agent loop 理論上可能一直要求用工具而不收尾。`max_turns` 這類護欄是必備，不是可選。
- **用一次性的 `prompt` 思維想 agent**：agent 的「智能」不在單一 prompt，而在「迴圈 + 工具 + 把結果餵回」的結構。提示工程重要，但結構更重要。

## 本章小結

一次 LLM 呼叫是個沒有外部資訊、不能行動、不能多步修正的純函數。**tool calling** 讓模型能「請求」行動，而把這些請求接成「reason → act → observe」的迴圈，就得到 **agent loop**——所有 agent 的共同骨架。我們用 50 行純 Python 親手刻了一個能跑的 agent，也因此看清：對話歷史、工具 dispatch、停止條件全都得自己扛，而持久化、串流、人工介入、context 管理、可觀測性這些 production 必需品則完全付之闕如。填平這道「能跑 vs. 可靠上線」的鴻溝，正是 LangGraph / LangChain / Deep Agents 三層工具的使命，也是本書接下來要帶你走完的路。

## 延伸閱讀 / 練習

1. **加一個工具**：替 1.4 的 agent 加上一個 `get_time(timezone)` 工具，並問它「現在台北幾點、適合打電話到東京嗎？」，觀察它如何串連兩個工具。
2. **製造錯誤**：把 `get_weather` 改成有機率丟出例外，看看現在的迴圈會怎麼壞掉——記下你「想加但還沒加」的重試邏輯，第 10 章談 fault tolerance 時我們會回來看它。
3. **數一數回合**：在迴圈裡印出每一輪的 `turn` 與 `stop_reason`，實際感受一次任務跑了幾趟 reason → act → observe。
4. **概念題**：用自己的話，向一個沒寫過 agent 的同事解釋「為什麼模型不能自己執行工具」。能講清楚這件事，你就抓到 agent loop 的精髓了。
