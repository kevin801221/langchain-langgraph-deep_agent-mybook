# 從 LangGraph 到 Deep Agents

### 用 Python 親手打造你自己的 Agent Harness — 最後做出一個你自己的 **Claude Code 式** coding agent

![GitHub Repo stars](https://img.shields.io/github/stars/kevin801221/langchain-langgraph-deep_agent-mybook?style=social)
![Code License: MIT](https://img.shields.io/badge/Code-MIT-blue.svg)
![Text License: CC BY-NC 4.0](https://img.shields.io/badge/Text-CC%20BY--NC%204.0-lightgrey.svg)
![Python](https://img.shields.io/badge/Python-3.11+-3776AB?logo=python&logoColor=white)
![LangChain](https://img.shields.io/badge/LangChain-1.x-1C3C3C)
![Deep Agents](https://img.shields.io/badge/deepagents-0.6.x-7C3AED)

> 市面上多數教材只教 agent 技術堆疊的「其中一層」,而且片段化。
> 這本書把 **LangChain → LangGraph → Deep Agents** 三層**縱向打通**:
> 讀完你不只會「用」harness,還能**拆開它、客製它、除錯它、把它送上線**。

---

## 🗺️ 這本書在講什麼

```
LangChain    ──  framework:抽象與整合
     │  建在
LangGraph    ──  runtime:低層編排、durable execution、streaming、HITL、persistence
     │  建在
Deep Agents  ──  harness:內建規劃、虛擬檔案系統、subagents、context 管理
```

會用 Deep Agents 的人很多,但**一旦行為不如預期就卡住**——因為不懂底下的 LangGraph。
本書帶你從 runtime 思維一路建到 harness,最後用 Part III 壓軸,做出一個會讀寫檔案、會跑指令、有 approval 關卡、帶持久記憶與 skills 的**終端 coding agent**(對應官方開源的 Deep Agents Code / `dcode`)。

## 👤 適合誰

- 用過 LLM / LangChain(會呼叫 API、做過簡單 chain 或 agent),想**真正精通、能做 production** 的中階開發者。
- 前置知識:熟 Python(type hints、async 概念)、懂 LLM / prompt 基本觀念。
- **不需要**先懂 LangGraph 或 Deep Agents——本書從 runtime 思維建起。

---

## ✅ 本 Repo 開源了什麼

這是全書的 **Part 0 + Part I（LangGraph 篇）**——agent 技術堆疊的「硬地基」,單獨拿來精通 LangGraph 也完全成立。

| 內容 | 說明 |
|---|---|
| 📖 [`manuscript/`](manuscript/) | **Ch01–18 正文**(繁體中文),每章:本章目標 → 概念圖解 → 最小可跑範例 → 常見坑 → 小結 → 練習 |
| 📓 [`notebooks/`](notebooks/) | **18 個配套 Jupyter notebook**,cell-by-cell 可跑、新手友好註解、附 `draw_ascii`/`draw_mermaid` 拓樸圖 |
| 🔬 [`deepagents-examples-guide/`](deepagents-examples-guide/) | **16 篇** LangChain 官方 Deep Agents 範例的**逐段(line-by-line)導讀** |

> 四種具名 callout 穿插全書:🩸 血淚教訓、💡 冷知識、⏱ 三秒判斷、🪞 對照閱讀(同件事在三層分別怎麼做)。

## 🚀 快速開始

```bash
# 1. 安裝 uv（本書一律用 uv 管套件）
curl -LsSf https://astral.sh/uv/install.sh | sh

# 2. 建環境並裝套件
uv venv && source .venv/bin/activate
uv pip install langgraph langchain "langchain[google-genai]" grandalf jupyter

# 3. 設金鑰（本書範例一律用 Gemini）
export GOOGLE_API_KEY="你的金鑰"

# 4. 開 notebook
jupyter lab notebooks/
```

> 機制類程式碼（state、reducer、拓樸圖…）**免金鑰即可跑**;需呼叫 LLM 的部分標明「設好金鑰即可跑」。

---

## 📚 全書目錄（✅ 開源 ／ 🔒 完整版）

#### Part 0 — 起步與全局地圖　✅
- ✅ [Ch01　為什麼需要 agent 框架](manuscript/Ch01_為什麼需要agent框架.md)
- ✅ [Ch02　框架、runtime 與 harness 的分層地圖](manuscript/Ch02_框架runtime與harness的分層地圖.md)
- ✅ [Ch03　開發環境、套件與版本策略](manuscript/Ch03_開發環境套件與版本策略.md)

#### Part I — LangGraph 篇:用低層 runtime 親手打造 agent　✅
- ✅ [Ch04　用 LangGraph 思考:node、edge、state](manuscript/Ch04_用LangGraph思考.md)
- ✅ [Ch05　Graph API:StateGraph、節點與條件邊](manuscript/Ch05_GraphAPI.md)
- ✅ [Ch06　State 設計:schema、reducer 與 annotation](manuscript/Ch06_State設計.md)
- ✅ [Ch07　執行模型內幕:Pregel、super-step 與 channels](manuscript/Ch07_執行模型內幕Pregel.md)
- ✅ [Ch08　Persistence 與 checkpointer](manuscript/Ch08_Persistence與checkpointer.md)
- ✅ [Ch09　Durable execution 與 fault tolerance](manuscript/Ch09_DurableExecution與FaultTolerance.md)
- ✅ [Ch10　Streaming 與事件流](manuscript/Ch10_Streaming與事件流.md)
- ✅ [Ch11　Human-in-the-loop、interrupts 與 time travel](manuscript/Ch11_HumanInTheLoop與interrupts.md)
- ✅ [Ch12　Subgraphs 與模組化](manuscript/Ch12_Subgraphs與模組化.md)
- ✅ [Ch13　Functional API](manuscript/Ch13_FunctionalAPI.md)
- ✅ [Ch14　長期記憶與跨對話 store](manuscript/Ch14_長期記憶與跨對話store.md)
- ✅ [Ch15　Workflows vs Agents:常見編排模式](manuscript/Ch15_WorkflowsVsAgents編排模式.md)
- ✅ [Ch16　Agentic RAG](manuscript/Ch16_AgenticRAG.md)
- ✅ [Ch17　SQL agent](manuscript/Ch17_SQLagent.md)
- ✅ [Ch18　測試 LangGraph agent](manuscript/Ch18_測試LangGraphAgent.md)

#### Part II — 銜接篇:從手刻 graph 到 LangChain agent 抽象　🔒
`create_agent` 與 ReAct loop · models/messages/structured output · Tools 與 MCP · RAG（LangChain 層） · **Middleware（理解 harness 的鑰匙）** · context engineering · guardrails · 多代理 handoffs/router/subagents/skills · voice agent　_(Ch19–28)_

#### Part III — Deep Agents 篇:harness、應用與 coding agent 壓軸　🔒
`create_deep_agent` · planning · 虛擬檔案系統與 backends · filesystem permissions · subagents 與 context 隔離 · token 管理 · sandboxes 與 interpreters · skills 與 memory · MCP/A2A/ACP 整合 · 前端 UI · 三大實戰(deep research / 資料分析 / 內容生成)　_(Ch29–42)_
- 🔒🔥 **Ch43–44　壓軸:打造你自己的 Claude Code** — 一個會讀寫檔案、跑指令、有 approval、帶記憶與 skills 的終端 coding agent

#### Part IV — 生產與交付　🔒
LangSmith tracing · LangGraph Studio · 評估(eval) · 監控與告警 · 部署(managed vs 自架) · 多租戶/安全/成本 · CI/CD · 遷移 · Deep Agents vs Claude Agent SDK 選型　_(Ch45–53)_ ＋ 附錄 A–D

---

## 📕 想要完整版?

開源的 18 章帶你**手刻 LangGraph**。
但要把它**打通到 Deep Agents harness、做出你自己的 Claude Code、再送上 production**——那是 Part II–IV(35 章 + 附錄,約 600 頁)的事。

完整版即將於 **[Leanpub](https://leanpub.com/)** 上線。
**⭐ Star ＋ 👀 Watch 本 repo,完整版開賣與每次更新第一時間通知你。**

<!-- TODO(kevin): 完整版上線後把 Leanpub 連結換上來 -->

---

## ✍️ 撰寫約定

- **語言**:繁體中文正文,技術術語保留英文(state、checkpointer、middleware、harness…)。
- **套件管理**:一律 `uv`(`uv add` / `uv sync`;notebook 用 `!uv pip install`)。
- **模型**:範例一律 **Gemini**(`gemini-2.5-flash`),並示範跨 provider 切換避免 lock-in。
- **版本鎖定**:`langchain` 1.x、`langgraph`、`deepagents` 0.6.x;每章開頭標註版本,程式碼可重現。

## 📜 授權（雙軌）

- **程式碼**(notebook、code snippet):[MIT](LICENSE-CODE) — 隨意使用、修改、散布。
- **書稿文字**(Markdown 正文、導讀):[CC BY-NC 4.0](LICENSE) — 可自由分享、翻譯,但**請勿商業利用**。完整版及其商業授權權利由作者保留。

## 🙋 作者

Kevin（[@kevin801221](https://github.com/kevin801221)）

> 如果這本書幫到你,給顆 ⭐ 是對作者最好的鼓勵,也讓更多中文開發者看見它。
