# Ch 2　框架、runtime、harness：LangChain / LangGraph / Deep Agents 的分層地圖

> **本章目標**
>
> - 一次講清楚 **LangChain、LangGraph、Deep Agents** 三者的關係——這張地圖是本書的地基，後面每一章都掛在它上面。
> - 區分三個關鍵詞：agent **framework**（框架）、agent **runtime**（執行環境）、agent **harness**（外殼）。
> - 用同一個需求，分別用三層各寫一次，直觀感受抽象高度的差異。
> - 建立一套「何時該下沉到哪一層」的決策直覺。
>
> **使用版本**：`langchain` 1.x、`langgraph`、`deepagents` 0.5.x。

---

## 2.1 一句話地圖：誰蓋在誰上面

先給結論，後面再展開。這三者不是三個並列、互相競爭的選項，而是**同一個技術堆疊的三層**，下層被上層蓋住：

```
┌───────────────────────────────────────────────┐
│  Deep Agents（harness / 外殼）                   │
│  開箱即用：規劃、虛擬檔案系統、subagents、context 管理 │
├───────────────────────────────────────────────┤
│  LangChain（framework / 框架）                   │
│  好用的抽象：create_agent、middleware、跨 provider 模型 │
├───────────────────────────────────────────────┤
│  LangGraph（runtime / 執行環境）                  │
│  低層編排：state、persistence、durable execution、 │
│  streaming、human-in-the-loop                    │
└───────────────────────────────────────────────┘
            （三層跑的，都是 Ch1 那個 agent loop）
```

官方自己的定位很明確：**LangChain 1.0 是建立在 LangGraph 之上的**；而 **Deep Agents 又是建立在 LangGraph 之上的 harness**。也就是說，LangGraph 是地基，LangChain 與 Deep Agents 是蓋在它上面、面向不同需求的兩種上層建築。

這件事有一個對你非常實際的推論，請先記著，它是本書整本的價值主張：**因為上層都蓋在 LangGraph 上，所以當上層（特別是 Deep Agents）的行為不如預期時，懂 LangGraph 的人能拆開它、除錯它、客製它；不懂的人只能把它當黑盒子，卡住就動彈不得。** 這就是為什麼這本書要「從 LangGraph 講起，再往上到 Deep Agents」，而不是反過來。

## 2.2 三個詞：framework、runtime、harness

這三個詞常被混用，但官方文件把它們分得很清楚，值得我們也分清楚。

**Agent framework（框架，如 LangChain）** 提供「抽象」，目的是讓你**快速上手**又保有彈性。它替你標準化了一堆瑣事：模型的輸入輸出格式、agent loop、工具定義、middleware。LangChain 的代表抽象就是 `create_agent`。值得注意的是，**你不需要懂 LangGraph 也能用 LangChain**——框架的意義之一，就是讓你不必碰底層。其他同類還有 Vercel AI SDK、CrewAI、OpenAI Agents SDK、Google ADK、LlamaIndex 等。

**Agent runtime（執行環境，如 LangGraph）** 提供「在 production 跑 agent 所需的底層能力」。它不在乎你的 agent 邏輯多漂亮，它在乎的是：能不能 **durable execution**（撐過失敗、長時間執行、從斷點 resume）？能不能 **streaming**？能不能 **human-in-the-loop**（檢視、修改 agent 的中間狀態）？能不能做 thread 級與跨 thread 的 **persistence**？以及，能不能給你**低層、精細的編排控制**？LangGraph 就是這樣一個「低層編排框架兼 runtime」。同類概念還有 Temporal、Inngest 這些 durable execution 引擎。

**Agent harness（外殼，如 Deep Agents SDK）** 是「有主見、電池全包（batteries-included）」的一層，內建了打造**複雜、長時間運行 agent** 所需的常見能力：規劃（用 todo list 追蹤多任務）、任務委派（用 subagents 保持 context 乾淨）、檔案系統（在可插拔的 backend 上讀寫檔案）、token 管理（自動摘要與壓縮 context）。Deep Agents 專為「需要規劃與分解的多步驟任務」設計，例如研究、寫程式。同類還有 Claude Agent SDK、Manus 等 coding CLI 背後的外殼。

一個生活化的比喻：如果蓋房子，**LangGraph 是鋼筋水泥與水電管線**（你能完全掌控結構，但要自己設計）、**LangChain 是預鑄的建材模組**（牆面、門窗都做好了，組裝快）、**Deep Agents 是一間裝潢好、家電齊全可直接入住的精裝屋**（你只要搬進去，但想改格局就得懂前兩層）。

> 💡 **冷知識** —— 這三件套的釋出時間其實隔得不近：**LangChain** 在 **2022-10**（ChatGPT 前一個月）首次以 Python 套件釋出；**LangGraph** 在 **2024-02** 才獨立釋出，當時官方已經發現「高階抽象不夠、缺一個低層編排層」；**Deep Agents** 更晚到 **2026-03** 才以 v0.5.x 釋出。換句話說，**「LangGraph 在下、其他在上」不只是設計決定，也是歷史順序**——他們是在用 LangChain 的過程中先發現問題、才分別往下（runtime）與往上（harness）長出來的。

## 2.3 同一個需求，三層各寫一次

抽象的高度差異，看程式碼最有感。我們用同一個需求——「一個會查天氣的 agent」——在三層各寫一次。

**(A) 用 LangChain：最高抽象，最快上手**

```python
from langchain.agents import create_agent

def get_weather(city: str) -> str:
    """查詢指定城市的天氣。"""
    return f"{city} 現在晴天，28°C。"

agent = create_agent(
    model="google_genai:gemini-2.5-flash",
    tools=[get_weather],
    system_prompt="你是個樂於助人的助理。",
)

result = agent.invoke(
    {"messages": [{"role": "user", "content": "台北天氣如何？"}]}
)
print(result["messages"][-1].content_blocks)
```

注意：Ch1 我們手刻了五十行才做到的 agent loop（維護對話、dispatch 工具、判斷停止），這裡 `create_agent` 一行就包好了。這就是 framework 的價值——**把通用 loop 抽象掉**。

**(B) 用 LangGraph：最低抽象，最大掌控**

同樣的 agent，若下沉到 LangGraph，你會親手定義 state、節點與邊（完整可跑版本留到 Part I，這裡先看「形狀」）：

```python
from langgraph.graph import StateGraph, START, END
from langgraph.graph.message import add_messages
from typing import Annotated, TypedDict

class State(TypedDict):
    messages: Annotated[list, add_messages]   # 你親手定義「狀態」長什麼樣

def call_model(state: State) -> dict: ...     # 你親手定義每個「節點」做什麼
def call_tools(state: State) -> dict: ...
def should_continue(state: State) -> str: ... # 你親手定義「分支」怎麼走

builder = StateGraph(State)
builder.add_node("model", call_model)
builder.add_node("tools", call_tools)
builder.add_edge(START, "model")
builder.add_conditional_edges("model", should_continue, {"tools": "tools", "end": END})
builder.add_edge("tools", "model")            # ← 這條回邊，就是 Ch1 的 agent loop
graph = builder.compile()
```

程式變多了，但你換得了**完全的控制權**：每個節點、每條分支、狀態裡放什麼，全是你說了算。那條 `tools → model` 的回邊，就是 Ch1 那個 reason→act→observe 迴圈，現在被你「畫」了出來。當你需要客製化的流程（分支、平行、固定順序與自由發揮混合），這層才給得了你需要的精度。

**(C) 用 Deep Agents：開箱即用，自帶超能力**

```python
from deepagents import create_deep_agent

agent = create_deep_agent(
    tools=[get_weather],
    model="google_genai:gemini-2.5-flash",
    system_prompt="你是個樂於助人的研究助理。",
)
```

看起來跟 LangChain 一樣簡潔，但這個 agent **天生就會規劃（`write_todos`）、有一整套虛擬檔案系統工具（ls/read/write/edit/glob/grep）、能派生 subagents、會自動管理過長的 context**。對「查個天氣」這種小任務，這些是殺雞用牛刀；但對「研究一個主題、查二十個來源、寫成一份十頁報告」這種長任務，這些內建能力就是天壤之別。

> 🪞 **對照閱讀** —— 三層的「呼叫」都一樣簡潔，但**取「最終答案」的方式不一樣**（配套 notebook 裡你會實際跑到）：
> - `create_agent` / 手刻 graph：messages 鏈單純，`result["messages"][-1]` 通常就是答案。
> - `create_deep_agent`：它會先規劃（`write_todos`）、調工具、收尾，**鏈更長，最後一條未必是帶文字的答案**——直接印 `result["messages"][-1].content_blocks` 可能得到空的 `[]`。要從後往前找**最後一條有 `content` 的 AI 訊息**才穩。
>
> 這不是 bug，是 harness「多做了規劃與編排」的副作用：你換到的是長任務能力，代價是輸出結構更複雜。

## 2.4 三層能力對照表

同樣一個概念，在三層有不同的實現方式。這張表幫你快速定位（後面各章會逐一展開）：

| 能力 | LangChain | LangGraph | Deep Agents |
|---|---|---|---|
| 短期記憶 | 內建於 agent | state | state |
| 長期記憶 | long-term memory | store | backend（StateBackend / StoreBackend / FilesystemBackend）|
| 多代理 | multi-agent（handoffs/subagents）| subgraphs | subagents（內建 `task` 工具）|
| 人類介入 | human-in-the-loop middleware | `interrupt` | `interrupt_on` 參數 |
| 串流 | agent streaming | streaming | streaming |
| 規劃 | 需自行設計 | 需自行設計 | 內建 `write_todos` |
| 檔案系統 | 需自行設計 | 需自行設計 | 內建虛擬檔案系統 |

讀這張表的正確方式不是背它，而是體會一件事：**愈往右，內建愈多、你要寫的愈少，但「黑盒」成分也愈高**；愈往左，愈是赤裸的積木，你要自己組，但組什麼、怎麼組你完全清楚。這也再次呼應本書的策略——先學會左邊（LangGraph 的積木），右邊（Deep Agents 的成品）對你就不再是魔法。

> 🩸 **血淚教訓** —— 我看過好幾個團隊一開始就跳到 Deep Agents 用 `create_deep_agent` 起手，前兩週做研究 demo 跑得很爽；第三週開始遇到 agent 的 todo 規劃不對、context 自動摘要把關鍵資訊摘掉，就**整個卡死**——因為他們連底下的 LangGraph 圖長什麼樣都不知道，找不到除錯入口。最後還是只能回頭學 Part I 全部。這就是為什麼本書刻意「由下而上」：先吃 LangGraph 的苦頭，後面用 Deep Agents 才會踩在地上。

## 2.5 何時該用哪一層

沒有「最好的一層」，只有「對這個任務最對的一層」。給你一組可操作的判斷：

**選 LangChain，當**：你想快速做出 agent 與自主應用；你需要模型、工具、agent loop 的標準抽象；你要的是好上手又保有彈性的框架；你的應用是相對直接的 agent，沒有複雜的編排需求。

**下沉到 LangGraph，當**：你需要對編排做精細的低層控制；你需要 durable execution 來跑長時間、有狀態的 agent；你在組合「確定性步驟 + agentic 步驟」的複雜工作流；你需要 production 等級的部署基礎設施。

**選 Deep Agents，當**：你的 agent 要跑很長時間；任務複雜、多步驟、需要規劃與分解；你想直接用現成的工具（檔案操作、bash 執行、自動 context 工程）；你想要現成的 prompt 與 subagents，不想從零砌。

> ⏱ **三秒判斷** —— 該起手於哪一層？
> - **能畫出固定流程、需要精細掌控分支／平行** → **LangGraph**
> - **相對直接的 agent loop、想快速做出可用版本** → **LangChain `create_agent`**
> - **長任務、要規劃、要派 subagent、要虛擬檔案系統** → **Deep Agents**
>
> 闔上書能記得的就這三條。

還有一個務實的現實：這三層**可以混用、可以下沉**。你大可從 LangChain 的 `create_agent` 開始，遇到它包太死、你需要更多控制時，再下沉到 LangGraph 重寫關鍵部分；或反過來，發現自己在 LangGraph 裡重造 Deep Agents 已經內建的東西時，就該往上換成 harness。**懂得在三層之間自由移動，正是「精通」的標誌**，也是這本書要送你的能力。

---

## 常見坑（Pitfalls）

- **以為三者是競品，要「二選一」**：它們是同一堆疊的三層。問題不是「用 LangChain 還是 LangGraph」，而是「這個任務該在哪一層解」。
- **一開始就跳到 Deep Agents，把它當黑盒**：對長任務它很香，但出問題時若不懂底下的 LangGraph，你會無從下手。這也是本書刻意「由下而上」的原因。
- **在 LangGraph 裡重造輪子**：如果你發現自己在手刻規劃、檔案系統、subagent 委派——停下來，那些 Deep Agents 已經做好了。
- **被 `create_agent` 的簡潔迷惑，以為它「只是個 wrapper」**：它底下是一張完整的 LangGraph 圖，享有 persistence、streaming、HITL 等 runtime 能力。簡潔不代表簡單。

## 本章小結

LangChain、LangGraph、Deep Agents 不是三個競爭者，而是同一堆疊的三層：LangGraph 是低層 **runtime**（編排、持久執行、串流、人類介入），LangChain 是蓋在其上的 **framework**（好用的抽象如 `create_agent`、middleware），Deep Agents 是再往上、開箱即用的 **harness**（規劃、虛擬檔案系統、subagents、context 管理）。我們用同一個「查天氣 agent」在三層各寫一次，看到抽象愈高、程式愈少、黑盒愈多。選層的關鍵在任務特性，而三層之間能自由下沉與上移，正是精通的標誌。記住那條貫穿全書的主軸：**無論抽象疊多高，跑的都還是 Ch1 那個 agent loop**；先把最底層學透，上層自然不再神秘。

## 延伸閱讀 / 練習

1. **三寫對照**：把本章 (A)(C) 兩段程式實際跑起來（LangChain 與 Deep Agents），對同一個問題比較它們的輸出與「自帶行為」差異。(B) 的完整版我們會在 Part I 補齊，這裡先讀懂它的形狀。
2. **歸位練習**：列出你手上一個真實想做的 agent 需求，套用 2.5 的判斷，說出它「應該起手於哪一層」與理由。
3. **對照表填空**：闔上書，憑記憶重畫一次 2.4 的能力對照表，標出哪些能力是 Deep Agents「內建、其他兩層要自己做」的——這幾格正是 harness 的賣點，也是 Part III 的重點。
4. **概念題**：為什麼說「懂 LangGraph 的人能除錯 Deep Agents，反之不然」？用 2.1 的「誰蓋在誰上面」回答。
