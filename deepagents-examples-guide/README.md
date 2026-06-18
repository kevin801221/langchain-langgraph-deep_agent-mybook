# Deep Agents 官方範例 — 繁體中文 Medium 深度導讀系列

這個資料夾收錄 16 篇繁體中文 Medium 技術文章，逐一拆解 [LangChain 官方 `deepagents/examples/`](https://github.com/langchain-ai/deepagents/tree/main/examples) 底下的**每一個官方範例**。每篇都包含「核心檔案逐段（line-by-line）導讀」，所有程式碼片段與函式/欄位名稱皆取自實際原始碼、未經杜撰。

> 撰寫慣例：繁體中文正文、技術術語保留英文；每篇結構為「引言 → 全貌架構 → 環境安裝 → ★核心程式碼逐段導讀★ → Deep Agents 能力對照 → 怎麼跑 → 延伸 → 小結」。

## 索引（依官方分類）

### 🔬 Research（研究）

| # | 文章 | 範例 | 重點 |
|---|---|---|---|
| 01 | [deep_research](01-deep_research.md) | `deep_research/` | orchestrator + 平行 research subagents + 帶引用報告；`think_tool` 強制反思 |
| 02 | [deploy-mcp-docs-agent](02-deploy-mcp-docs-agent.md) | `deploy-mcp-docs-agent/` | 用 MCP 連官方文件、先查再答的宣告式部署 agent |

### 💻 Coding（程式）

| # | 文章 | 範例 | 重點 |
|---|---|---|---|
| 03 | [deploy-coding-agent](03-deploy-coding-agent.md) | `deploy-coding-agent/` | LangSmith sandbox 自主 coding agent；Plan→Implement→Review→Deliver |
| 04 | [nvidia_deep_agent](04-nvidia_deep_agent.md) | `nvidia_deep_agent/` | 多模型架構 + Modal GPU sandbox（RAPIDS）；Nemotron Super 當研究員 |

### ✍️ Content（內容）

| # | 文章 | 範例 | 重點 |
|---|---|---|---|
| 05 | [content-builder-agent](05-content-builder-agent.md) | `content-builder-agent/` | 用三種 filesystem primitive（memory / skills / subagents）定義 agent |
| 06 | [text-to-sql-agent](06-text-to-sql-agent.md) | `text-to-sql-agent/` | 自然語言轉 SQL；planning + filesystem + skills 工作流 |
| 07 | [llm-wiki](07-llm-wiki.md) | `llm-wiki/` | script-first 持久維基；ingest/query/lint 三模式 + Sandbox |

### 🚀 Deployable services（可部署服務）

| # | 文章 | 範例 | 重點 |
|---|---|---|---|
| 08 | [deploy-content-writer](08-deploy-content-writer.md) | `deploy-content-writer/` | per-user memory + Supabase auth，零程式碼隔離多使用者 |
| 09 | [deploy-gtm-agent](09-deploy-gtm-agent.md) | `deploy-gtm-agent/` | GTM 策略 agent；sync / async 子代理協作 |
| 10 | [async-subagent-server](10-async-subagent-server.md) | `async-subagent-server/` | Agent Protocol 自架 server + supervisor REPL（非同步子代理） |

### 🧪 Advanced patterns（進階模式）

| # | 文章 | 範例 | 重點 |
|---|---|---|---|
| 11 | [ralph_mode](11-ralph_mode.md) | `ralph_mode/` | Ralph 自主迴圈；每輪 fresh context，filesystem + git 當記憶 |
| 12 | [rlm_agent](12-rlm_agent.md) | `rlm_agent/` | `create_rlm_agent`：遞迴 REPL + PTC 一次扇出整批並行 |
| 13 | [repl_swarm](13-repl_swarm.md) | `repl_swarm/` | 把並行派工編排寫成 TypeScript skill，agent 在 REPL `import` 使用 |
| 14 | [downloading_agents](14-downloading_agents.md) | `downloading_agents/` | 「agent 就是資料夾」；下載 zip、解壓即跑 |
| 15 | [better-harness](15-better-harness.md) | `better-harness/` | eval 驅動的 harness 外層優化迴圈（agent 優化 agent） |
| 16 | [talon-whatsapp](16-talon-whatsapp.md) | `talon-whatsapp/` | 容器化 Talon runtime；WhatsApp 訊息驅動 agent + 本機語音轉文字 |

---

共 16 篇，對應 [LangChain 官方 deepagents](https://github.com/langchain-ai/deepagents) 的全部範例專案。原始 README 多為中英對照，本系列在其之上補齊「核心程式碼逐段導讀」，適合想真正讀懂這些範例怎麼寫成的開發者。
