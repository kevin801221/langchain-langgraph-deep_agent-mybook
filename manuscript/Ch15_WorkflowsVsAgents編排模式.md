# Ch 15　Workflows vs Agents：常見編排模式

> **本章目標**
>
> - 分清楚 **workflow**（預定路徑）與 **agent**（動態自決）兩種光譜，以及何時選哪個。
> - 掌握五種經典編排模式：prompt chaining、parallelization、routing、orchestrator-worker、evaluator-optimizer。
> - 看清每種模式如何用 Ch5 的 Graph API 與 Ch7 的平行執行落地。
> - 建立一個務實判準：能用 workflow 就別硬上 agent。
>
> **使用版本**：`langgraph` 1.x、`langchain-google-genai`。配套 notebook 用離線假節點示範結構，**不需 API key**。

---

## 15.1 一條光譜：workflow 與 agent

把前面學的機制收束成一個全局觀：agentic 系統大致落在一條光譜上。

- **Workflow（工作流）**：路徑是**預先決定**的，照固定順序跑。你（開發者）在編譯期就畫好了流程。
- **Agent（代理）**：路徑是**動態**的，由模型在執行期自己決定下一步、用哪個工具——也就是 Ch1 那個 agent loop。

LangGraph 兩者都能做，而且都享有 persistence、streaming、除錯、部署等好處。關鍵心法：**可預測、可拆成固定步驟的任務，用 workflow 最穩；非確定、需要模型臨機應變的任務，才交給 agent。** 多數真實系統是兩者的混合——固定骨架（workflow）裡，某幾個節點是小型 agent。

在組這些模式前，先記得 LLM 的幾個「強化」手段（前面章節都見過）：tool calling（給它手）、structured output（讓它吐結構化資料）、short-term memory（讓它記得脈絡）。模式就是把「強化過的 LLM」用不同拓樸接起來。

## 15.2 Prompt chaining：把任務拆成可驗證的小步

**prompt chaining** 是最單純的 workflow：每一次 LLM 呼叫處理上一次的輸出，像接力。適合「能拆成幾個明確、可逐步驗證的子步驟」的任務——例如「先把大綱寫出來 → 再依大綱擴寫 → 最後潤稿」，或「翻譯 → 檢查一致性」。

在 Graph API 裡，它就是一串線性節點：

```python
# 線性鏈：generate_outline → write_draft → polish
builder.add_edge(START, "generate_outline")
builder.add_edge("generate_outline", "write_draft")
builder.add_edge("write_draft", "polish")
builder.add_edge("polish", END)
```

有時你會在中間插一個「gate（閘門）」節點：用程式檢查上一步的輸出是否合格，不合格就提早結束或回頭重做。這把「可驗證」具體化了。

## 15.3 Parallelization：同時做、再彙整

當有多個**彼此獨立**的子任務，與其一個個串著做，不如**平行**跑、再把結果彙整。例如「同時從三個面向評估一份提案」「同時對一段文字做多種檢查」。

呼應 Ch7：**從 START 同時指向多個節點，它們就屬於同一個 super-step、平行執行**；再用一個彙整節點收斂。記得平行寫同一欄位要用可合併的 reducer（Ch7 的坑）：

```python
from operator import add
class State(TypedDict):
    aspects: Annotated[list[str], add]   # 平行節點各寫一筆，用 add 合併

builder.add_edge(START, "check_a")
builder.add_edge(START, "check_b")       # a、b 平行（同一 super-step）
builder.add_edge(START, "check_c")
builder.add_edge("check_a", "aggregate")
builder.add_edge("check_b", "aggregate")
builder.add_edge("check_c", "aggregate")  # 三路收斂到彙整節點
```

平行化的好處是延遲（latency）：三個各花 2 秒的檢查，平行只需約 2 秒而非 6 秒。

## 15.4 Routing：先分類，再走專門路徑

**routing** 你其實已經在 Ch5 寫過——先用一個節點分類輸入，再用條件邊導向專門的處理路徑。它讓你針對不同類型的輸入用不同的 prompt、工具、甚至模型，而不是用一個「萬用 prompt」勉強應付所有情況。

```python
# Ch5 的 email agent 就是 routing：classify → (simple→draft | complex→escalate)
builder.add_conditional_edges("classify", route, {"draft": "draft", "escalate": "escalate"})
```

routing 的價值在「分而治之」：把困難的「一個 prompt 處理全部」拆成「先判類型、再用最適合該類型的處理」。客服分流、難易度分級、語言分流都是它。

## 15.5 Orchestrator-worker：動態拆解 + 委派

當子任務的**數量與內容無法預先決定**（取決於輸入），就用 **orchestrator-worker**：一個 orchestrator（協調者）節點動態地把任務拆成若干子任務，分派給 worker 們去做，最後彙整。

跟 parallelization 的差別在「子任務是否預先固定」：parallelization 的分支是寫死的；orchestrator-worker 的分支是 orchestrator 在執行期**動態產生**的。例如「寫一份報告，但有幾個章節、各寫什麼，由 orchestrator 看了主題才決定」。

這個模式是通往 Part III「Deep Agents subagents」的直接前身——Deep Agents 的 `task` 工具派生 subagent，本質上就是一種內建的 orchestrator-worker（Ch33 會接上）。

## 15.6 Evaluator-optimizer：生成 + 評估的迴圈

**evaluator-optimizer** 用兩個角色組成迴圈：一個 generator 產出結果，一個 evaluator 評估並給回饋；若不合格，帶著回饋回頭重生成，直到通過或達到上限。

```python
# generate → evaluate →（不合格則回 generate；合格則 END）
builder.add_edge(START, "generate")
builder.add_edge("generate", "evaluate")
builder.add_conditional_edges("evaluate", check, {"retry": "generate", "done": END})
```

它適合「有明確品質標準、且一次做不一定到位」的任務——例如「寫一段程式並通過測試」「翻譯到母語者覺得自然為止」。注意要加迴圈上限（呼應 Ch1 的 `max_turns` 精神），避免無限打磨。

## 15.7 完整 agent：把控制權交給模型

光譜的另一端是**完整的 agent**——就是 Ch1 的 agent loop：模型自己決定要不要用工具、用哪個、何時收尾，路徑完全動態。Ch5 我們看過它在 Graph API 裡就是「tools → model 的回邊」；Part II 的 `create_agent` 把它一行包好。

什麼時候才該用完整 agent？**當任務開放、步數不定、且你無法預先畫出合理的固定流程時。** 反之，如果你能畫出固定流程，用 workflow 幾乎總是更穩、更省、更好除錯。一個常見的反模式是：明明是「分類 → 回覆」這種能用 routing 解決的事，卻硬塞一個全自主 agent，結果又慢又難預測。

## 15.8 怎麼選：一個務實判準

把這幾個模式排成「自主度遞增」：

```
prompt chaining  →  routing  →  parallelization  →  orchestrator-worker  →  evaluator-optimizer  →  完整 agent
（最可預測、最好控）─────────────────────────────────────────────────────────→（最有彈性、最難控）
```

判準一句話：**從左邊開始，能解就別往右。** 自主度愈高，能力愈強，但也愈不可預測、愈難除錯、愈貴。先問「這任務能不能用固定流程表達？」能，就用 workflow 模式；真的不能，才往 agent 走。而且別忘了——你可以**混搭**：固定骨架裡放幾個小 agent 節點，取兩者之長。

---

## 常見坑（Pitfalls）

- **凡事都用全自主 agent**：能用 routing / chaining 解決的事硬上 agent，換來不可預測與高成本。能畫固定流程就用 workflow。
- **平行節點寫同欄位沒加 reducer**：parallelization 必踩的坑（Ch7）。平行寫入要用可合併的 reducer。
- **evaluator-optimizer 沒設上限**：可能無限打磨。一定要有迴圈次數上限。
- **混淆 parallelization 與 orchestrator-worker**：前者分支寫死、後者執行期動態產生。子任務數不固定才用 orchestrator-worker。
- **把 routing 的分類塞進「萬用 prompt」**：失去分而治之的好處。不同類型值得不同的處理路徑。

## 本章小結

agentic 系統落在「workflow（預定路徑）↔ agent（動態自決）」的光譜上。五種經典模式由可控到彈性排列：**prompt chaining**（線性接力、可逐步驗證）、**routing**（先分類再走專門路徑，Ch5 已見）、**parallelization**（獨立子任務平行跑再彙整，呼應 Ch7 的 super-step 與 reducer）、**orchestrator-worker**（執行期動態拆解委派，Deep Agents subagents 的前身）、**evaluator-optimizer**（生成+評估的迴圈，記得設上限），最右端則是 Ch1 的**完整 agent**。務實判準：從最可控的左端開始，能解就別往右；能畫固定流程就用 workflow，真的不能才交給 agent，並善用混搭。掌握了這些模式，你就能把 Part I 學的機制組裝成解決真實問題的形狀。接下來兩章（Ch16 Agentic RAG、Ch17 SQL agent）就是這些模式在具體應用上的落地。

## 延伸閱讀 / 練習

1. **離線搭五模式**：用假節點（不呼叫 LLM）把 chaining、parallelization、routing 各搭一個最小圖跑起來，觀察執行路徑（配套 notebook 有可跑版本，免 API key）。
2. **平行的延遲**：在三個平行節點裡各 `time.sleep(1)`，量總執行時間，確認接近 1 秒而非 3 秒，體會平行化的價值。
3. **evaluator 迴圈**：用一個「分數 < 門檻就回頭」的假 evaluator 組一個 evaluator-optimizer 迴圈，並加上次數上限。
4. **選型題**：對「把一篇英文新聞翻成中文並確保用詞自然」「依客戶問題類型給不同回覆」「研究一個你沒預設範圍的主題」三個任務，各選一個最適合的模式並說明理由。
