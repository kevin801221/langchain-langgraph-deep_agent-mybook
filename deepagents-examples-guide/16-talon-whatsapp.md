> 原始碼路徑：`docs/deepagents/examples/talon-whatsapp/`

# 讓 WhatsApp 訊息直接驅動 AI Agent：Deep Agents Talon WhatsApp 範例逐段導讀

**副標：同一容器跑 Talon host + WhatsApp bridge、QR 配對即上線、本機 ASR 語音轉文字——一篇讀懂所有關鍵設定**

---

## 引言：「傳一則訊息，agent 就動起來」的體驗

想像你拿起手機，在 WhatsApp 裡傳了一句「幫我查一下明天的行程」，幾秒後 AI agent 就回覆了整理好的答案——而且這個 agent 跑在你自己的機器上，沒有任何第三方服務介入聊天內容。這正是 Deep Agents **talon-whatsapp** 範例想示範的事。

這個範例把兩個角色塞進同一個 Docker container：
- **Talon host**：Deep Agents 的實驗性 runtime，負責排程、cron、工具執行。
- **WhatsApp bridge subprocess**：以 Node.js 實作的橋接程序，透過 QR 配對把 WhatsApp 帳號接進來。

兩者同居一個 container，bridge 只綁定容器內部的 loopback（`127.0.0.1`），完全不對外暴露埠口。主機上的 `~/agent-workspace/` 則透過 volume mount 掛進容器的 `/workspace`，讓 agent 產出的所有檔案都直接落在你的本機。

> ⚠️ **實驗性質（Experimental）**：Talon 是實驗性的 runtime，API 隨時可能變更或移除，請勿用於正式生產環境。

---

## 全貌一覽：架構的四個核心設計

在進入逐段導讀之前，先把整體設計講清楚：

| 設計決策 | 做法 | 原因 |
|---|---|---|
| **同容器雙程序** | Talon host + WhatsApp bridge 同跑於一個 container | 簡化部署，不需要額外的 service discovery |
| **bridge 只綁 loopback** | `BRIDGE_HOST=127.0.0.1`，不對外開放 | 安全隔離，bridge 只服務同容器的 Talon |
| **workspace mount** | `~/agent-workspace` → `/workspace` | 持久化 agent 輸出，重啟不遺失 |
| **QR 配對一次** | 掃碼後 session state 存在 `.deepagents/whatsapp-session` | 不需每次重啟都重掃 |

**Exposure mode** 控制哪些訊息能觸發 agent，三個選項：

- `self`（預設）：只有配對帳號**自己**發出的訊息才觸發——適合個人使用、安全測試。
- `allowlist`：只有 `DEEPAGENTS_TALON_WHATSAPP_ALLOWLIST_CHATS` 列出的聊天室，或符合 `DEEPAGENTS_TALON_WHATSAPP_MENTION_PATTERNS` 的訊息才觸發。
- `open`：所有入站 WhatsApp 訊息都觸發——請謹慎使用。

---

## 環境準備與執行流程

README 給出的啟動步驟只有四行，但每一行都有意義：

```bash
cp .env.example .env
mkdir -p ~/agent-workspace
# 填好 .env 裡的 AGENT_MODEL 對應憑證
docker compose build
docker compose up
```

1. **`cp .env.example .env`**：建立本地設定檔，所有機密（API key）留在 `.env`，不進 git。
2. **`mkdir -p ~/agent-workspace`**：預建主機側的 workspace 目錄；若不建立，docker compose 啟動時會因為 volume 來源不存在而出錯。
3. **填憑證**：至少要填 `AGENT_MODEL` 對應的 API key（預設範例是 `OPENAI_API_KEY`）。若 `AGENT_MODEL` 空白，Talon 會退回到 **echo runtime** 做基本煙霧測試，不會真正呼叫 LLM。
4. **`docker compose build`**：建置 image，這步會下載並安裝所有依賴（含 Python 的 `speech` extra），只需做一次。之後改 `.env` 不需重新 build。
5. **`docker compose up`**：啟動後，bridge 會在終端機印出 QR code，用 WhatsApp 手機 App 掃描完成配對。

---

## ★ 逐段導讀

### Dockerfile：每一層指令的意義

```dockerfile
FROM ghcr.io/astral-sh/uv:python3.13-bookworm
```

**Base image** 選用 Astral 官方的 `uv` image，搭配 Python 3.13 與 Debian Bookworm。`uv` 是高速 Python 套件管理工具，後續的依賴安裝都透過它完成，省去手動建 virtualenv 的步驟。

```dockerfile
ARG GCX_VERSION=0.4.0
ARG TEA_VERSION=0.14.1
```

兩個 build-time 參數：`gcx`（Grafana 的 CLI 工具）與 `tea`（Gitea 的 CLI 工具）的版本號，可在 `docker compose build` 時用 `--build-arg` 覆寫。

```dockerfile
ENV DEBIAN_FRONTEND=noninteractive \
    NODE_PATH=/opt/whatsapp-bridge/node_modules \
    PATH="/opt/deepagents-talon-venv/bin:${PATH}" \
    UV_LINK_MODE=copy \
    UV_PROJECT_ENVIRONMENT=/opt/deepagents-talon-venv
```

五個重要的環境變數：
- `DEBIAN_FRONTEND=noninteractive`：讓 apt-get 安裝時不跳互動問答。
- `NODE_PATH`：讓容器內的 Node.js 能找到 WhatsApp bridge 的 `node_modules`。
- `PATH` 前置 `/opt/deepagents-talon-venv/bin`：讓 `deepagents-talon` 指令直接可用，不需 `uv run`。
- `UV_LINK_MODE=copy`：`uv` 複製檔案而非使用 hardlink，避免跨 filesystem 的問題。
- `UV_PROJECT_ENVIRONMENT`：指定 virtualenv 路徑，讓 `uv sync` 安裝到固定位置。

```dockerfile
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
        ca-certificates chromium curl ffmpeg \
        fonts-liberation gh git nodejs npm \
        ripgrep xz-utils && \
    ...
```

系統套件安裝層，其中幾個值得特別點出：
- **`ffmpeg`**：音訊處理的核心工具。WhatsApp 語音訊息是 `.ogg` / `.opus` 格式，需要 ffmpeg 轉換後才能送進 ASR 模型。
- **`chromium`**：提供無頭瀏覽器能力，供 agent 的網頁工具使用。
- **`gh` / `git`**：GitHub CLI 與 git，供 coding-related 工具使用。
- **`ripgrep`**：高速文字搜尋，供 agent 在 workspace 裡搜尋檔案。
- **`nodejs` / `npm`**：執行 WhatsApp bridge（Node.js 實作）的必要執行環境。

同一個 `RUN` 層也依照主機架構（`amd64` / `arm64`）下載並驗證 `gcx` 與 `tea` 的二進位檔，透過 `sha256sum` 驗證完整性後安裝到 `/usr/local/bin`。

```dockerfile
WORKDIR /opt/deepagents-src

COPY libs/deepagents ./libs/deepagents
COPY libs/code ./libs/code
COPY libs/talon ./libs/talon
COPY examples/talon-whatsapp/AGENTS.md /opt/deepagents-talon/AGENTS.md
```

設定建置工作目錄，複製三個本地 library（`deepagents`、`code`、`talon`）進 image。注意 `AGENTS.md` 被複製到 `/opt/deepagents-talon/AGENTS.md`——這是 image 內的「預設版本」，之後 docker-compose 的 `command` 會在執行時再把它複製到正確的 agent 狀態目錄。

```dockerfile
RUN mkdir -p /opt/whatsapp-bridge && \
    cp libs/talon/deepagents_talon/channels/whatsapp_bridge/package.json \
        /opt/whatsapp-bridge/package.json && \
    cd /opt/whatsapp-bridge && \
    npm install --omit=dev
```

安裝 WhatsApp bridge 的 Node.js 依賴。只複製 `package.json`（不是整個 bridge 目錄），讓 `npm install` 建立 `node_modules` 於 `/opt/whatsapp-bridge/`——這樣 Node 套件快取層不會因為 bridge 原始碼改動而失效。`--omit=dev` 排除開發用依賴，縮小 image 體積。

```dockerfile
RUN cd libs/talon && uv sync --frozen --extra speech
```

安裝 Talon 的 Python 依賴，關鍵是 `--extra speech`：這個 extra 會拉進 `transformers`、ASR 相關套件（用於載入 NVIDIA Parakeet 模型做語音轉文字）。`--frozen` 確保使用 lockfile 的精確版本，讓 build 結果可重現。

```dockerfile
WORKDIR /workspace

CMD ["/opt/deepagents-talon-venv/bin/deepagents-talon", "--whatsapp"]
```

**`WORKDIR /workspace`**：容器啟動後的預設工作目錄，對應 mount 進來的 `~/agent-workspace`，agent 建立的所有相對路徑檔案都會落在這裡。

**`CMD`**：預設啟動指令，直接呼叫 Talon 的 entry point 並帶上 `--whatsapp` flag 啟用 WhatsApp channel。實際使用時這個 CMD 會被 docker-compose 的 `command` 覆寫（見下節）。

---

### docker-compose.yml：服務、掛載、環境變數

```yaml
services:
  talon:
    build:
      context: ../..
      dockerfile: examples/talon-whatsapp/Dockerfile
    image: deepagents-talon-whatsapp:local
```

`build.context` 設為 `../..`（即專案根目錄），因為 Dockerfile 需要 `COPY libs/...` 複製根目錄下的 library——所以 build context 必須在根目錄，而不是在 example 子目錄。`image` 給本地 build 出來的 image 取名，方便後續識別。

```yaml
    working_dir: /workspace
    env_file:
      - .env
    environment:
      DEEPAGENTS_TALON_HOME: /workspace/.deepagents
      DEEPAGENTS_TALON_WORKSPACE: /workspace
      DEEPAGENTS_TALON_WHATSAPP_SESSION_DIR: /workspace/.deepagents/whatsapp-session
      DEEPAGENTS_TALON_VOICE_TRANSCRIPTION_ENABLED: ${DEEPAGENTS_TALON_VOICE_TRANSCRIPTION_ENABLED:-true}
      DEEPAGENTS_TALON_VOICE_TRANSCRIPTION_DEVICE: ${DEEPAGENTS_TALON_VOICE_TRANSCRIPTION_DEVICE:-cpu}
```

`env_file` 載入你的 `.env`（含 API keys 等機密），`environment` 區塊再疊加容器專屬的覆寫值：

- `DEEPAGENTS_TALON_HOME=/workspace/.deepagents`：把 Talon 的「家目錄」（存放 cron jobs、session state 等）指向 mount 進來的 workspace，確保重啟後資料不遺失。
- `DEEPAGENTS_TALON_WORKSPACE=/workspace`：明確告訴 Talon，agent 的工作目錄是 `/workspace`。
- `DEEPAGENTS_TALON_WHATSAPP_SESSION_DIR`：WhatsApp 連線的 session state 存放路徑，同樣指向 mount 目錄以持久化。
- `DEEPAGENTS_TALON_VOICE_TRANSCRIPTION_ENABLED`：使用 `${VAR:-default}` 語法，預設為 `true`；可在 `.env` 裡覆寫成 `false` 停用語音轉文字。
- `DEEPAGENTS_TALON_VOICE_TRANSCRIPTION_DEVICE`：預設 `cpu`；若主機有 NVIDIA GPU，在 `.env` 裡設成 `cuda` 即可切換 GPU 加速。

```yaml
    volumes:
      - ${HOME}/agent-workspace:/workspace
```

**唯一的 volume mount**：把主機的 `~/agent-workspace` 掛進容器的 `/workspace`。這是整個範例最關鍵的持久化機制——agent 的輸出檔、WhatsApp session、cron jobs，全都透過這個 mount 存活於容器生命週期之外。注意這裡沒有 `ports` 定義，因為 bridge 只綁 loopback，不需要對外開埠。

```yaml
    command: >
      bash -c "
      mkdir -p /workspace/.deepagents/$${AGENT_ASSISTANT_ID}/agent &&
      cp /opt/deepagents-talon/AGENTS.md /workspace/.deepagents/$${AGENT_ASSISTANT_ID}/agent/AGENTS.md &&
      /opt/deepagents-talon-venv/bin/deepagents-talon --whatsapp
      "
```

啟動時執行三步驟（注意 `$$` 是 docker-compose 對 `$` 的跳脫寫法）：

1. 建立 agent 的狀態目錄（以 `AGENT_ASSISTANT_ID` 為子目錄名）。
2. 把 image 裡預置的 `AGENTS.md` 複製到正確的 agent 狀態路徑——讓 Talon 能讀到 agent 的人格與指令。
3. 啟動 Talon（`--whatsapp`）。

這個設計讓 `AGENTS.md` 能在 **不重新 build image** 的情況下被替換：只要在 `/workspace/.deepagents/<id>/agent/AGENTS.md` 放入新版本，下次啟動就會用新的。

---

### AGENTS.md：agent 的行為定義

```markdown
你是一個透過 Deep Agents Talon 運作的簡潔型個人助理。

當某件任務應該在稍後才執行時，請建立一個排程工作（cron job），而不是
要求使用者之後再提醒你一次。
```

`AGENTS.md` 是 Talon agent 的「人格說明書」，內容相當精簡——兩個要點：

1. **簡潔型個人助理**：定義 agent 的基本角色與風格，回應以簡潔為主。
2. **主動建立 cron job**：這是 Talon runtime 的核心能力展示。當使用者說「等一下再提醒我」，agent 不應該依賴使用者的第二次輸入，而應直接排程一個 cron job，到時自動觸發。

這份 `AGENTS.md` 刻意保持短小，讓使用者易於客製。要改變 agent 的行為，只需修改這個 Markdown 檔，不需改動任何程式碼或重建 image。

---

### .env.example：環境變數逐一解析

```env
AGENT_ASSISTANT_ID=whatsapp-local
AGENT_MODEL=openai:gpt-5.2
DEEPAGENTS_TALON_CONTEXT_SIZE=75000
OPENAI_API_KEY=
```

- `AGENT_ASSISTANT_ID`：assistant 的唯一識別名稱，決定 `~/.deepagents/` 底下的狀態子目錄名稱（在 Docker 模式下是 `/workspace/.deepagents/<id>/`）。
- `AGENT_MODEL`：格式為 `provider:model-name`。範例預設用 `openai:gpt-5.2`，若完全留空，Talon 退回 echo runtime——實際不呼叫 LLM，只把輸入原文回傳，適合快速驗證 pipeline 是否通。
- `DEEPAGENTS_TALON_CONTEXT_SIZE=75000`：傳給 LLM 的 context window 大小上限（token 數）。
- `OPENAI_API_KEY`：對應 `AGENT_MODEL` 中 `openai` provider 的憑證，需要填入真實 key。

```env
LANGSMITH_TRACING=false
LANGSMITH_API_KEY=
LANGSMITH_PROJECT=deepagents-talon
```

LangSmith 追蹤設定，預設關閉。`LANGSMITH_TRACING=true` 搭配有效的 `LANGSMITH_API_KEY`，即可追蹤每一次由 WhatsApp 訊息或 cron 排程觸發的 agent 執行，在 LangSmith 的 UI 上看到完整的 trace chain。

```env
DEEPAGENTS_TALON_WHATSAPP_ENABLED=true
DEEPAGENTS_TALON_WHATSAPP_EXPOSURE=self
DEEPAGENTS_TALON_WHATSAPP_OPERATOR_ID=
DEEPAGENTS_TALON_WHATSAPP_ALLOWLIST_CHATS=
DEEPAGENTS_TALON_WHATSAPP_MENTION_PATTERNS=
DEEPAGENTS_TALON_WHATSAPP_BRIDGE_HOST=127.0.0.1
DEEPAGENTS_TALON_WHATSAPP_BRIDGE_PORT=3000
DEEPAGENTS_TALON_WHATSAPP_START_BRIDGE=true
```

WhatsApp channel 的完整設定：
- `WHATSAPP_ENABLED=true`：開啟 WhatsApp channel。
- `WHATSAPP_EXPOSURE=self`：預設最安全模式，只有自己傳給自己的訊息才觸發。
- `WHATSAPP_OPERATOR_ID`：可設定特定 operator 的 WhatsApp ID，用於特殊路由邏輯（非必填）。
- `WHATSAPP_ALLOWLIST_CHATS`：`allowlist` 模式下，列出允許觸發 agent 的聊天室 ID（逗號分隔）。
- `WHATSAPP_MENTION_PATTERNS`：`allowlist` 模式下，訊息符合這些 pattern 也可觸發。
- `WHATSAPP_BRIDGE_HOST=127.0.0.1`：bridge 只綁 loopback，對外不可見。
- `WHATSAPP_BRIDGE_PORT=3000`：bridge 監聽的本地埠號。
- `WHATSAPP_START_BRIDGE=true`：讓 Talon 自動啟動 bridge subprocess；若設為 `false`，需要手動另行啟動 bridge。

```env
DEEPAGENTS_TALON_VOICE_TRANSCRIPTION_ENABLED=true
DEEPAGENTS_TALON_VOICE_TRANSCRIPTION_DEVICE=cpu
```

語音轉文字（ASR）的兩個開關：
- `VOICE_TRANSCRIPTION_ENABLED=true`：預設開啟，收到 WhatsApp 語音訊息時自動轉文字。使用 NVIDIA Parakeet 模型，透過 HuggingFace Transformers 在本機執行，**不送到任何外部 API**。
- `VOICE_TRANSCRIPTION_DEVICE=cpu`：預設用 CPU 推理；若主機有 NVIDIA GPU，改成 `cuda` 可大幅縮短第一次之後的轉錄時間。注意：模型採**延遲下載**，第一則語音訊息送達時才會觸發下載，可能需要等待幾分鐘。

---

## 這個範例示範的三項核心能力

1. **Talon runtime 的 container 化部署**：把實驗性 runtime 封裝成可攜帶的 Docker image，透過 docker-compose 一鍵啟動，降低上手門檻。
2. **外部訊息來源驅動 agent**：WhatsApp 這類即時通訊平台成為 agent 的「輸入端」，使用者無需學習新介面，用熟悉的 app 就能操控 agent。
3. **本機 ASR（語音辨識）**：語音訊息直接在容器內轉文字，整個流程端到端都在本機或自有伺服器上執行，音訊內容不離開你的環境。

---

## 怎麼跑：快速啟動清單

```bash
# 1. 複製設定檔
cp .env.example .env

# 2. 填入 API key（至少填 OPENAI_API_KEY，或換成其他 provider）
# 編輯 .env 檔

# 3. 建立主機 workspace 目錄
mkdir -p ~/agent-workspace

# 4. 建置 image（只需一次）
docker compose build

# 5. 啟動
docker compose up

# 6. 掃 QR code 配對 WhatsApp
# 配對成功後，用配對帳號傳訊息給自己即可觸發 agent
```

配對完成後的 session state 會持久存在 `~/agent-workspace/.deepagents/whatsapp-session`，下次 `docker compose up` 不需重新掃碼。

---

## 延伸與注意事項

**實驗性，請勿生產使用**：README 明確標注 Talon 是實驗性 runtime，API 與行為可能在無預警的情況下改變或移除。目前適合用於學習、原型驗證與個人實驗。

**重新 build 的時機**：修改 `Dockerfile`、調整系統套件、升級 Talon Python 依賴，或變更 Node 套件時，都需要重新執行 `docker compose build`。只改 `.env` 或 `AGENTS.md` 不需 rebuild。

**ASR 模型初次下載**：NVIDIA Parakeet 模型會在第一則語音訊息觸發時才下載，在網路環境較慢的情況下，第一次可能需要等較長時間。建議先傳一則語音訊息做暖機。

**GPU 加速**：切換到 `VOICE_TRANSCRIPTION_DEVICE=cuda` 需要主機有 NVIDIA GPU 且 Docker 環境已設定 NVIDIA container runtime（`nvidia-docker2` 或 `nvidia-container-toolkit`），此部分 docker-compose.yml 未預設設定，需要使用者自行添加 `deploy.resources.reservations.devices` 設定。

**Exposure mode 的風險**：若將 exposure 設為 `open`，任何能傳訊息到你帳號的人都可能觸發 agent，請確認你理解其中的安全風險再使用。

---

## 小結

talon-whatsapp 範例是一個架構清晰、設定完整的「外部訊息源驅動 agent」藍圖。它的巧妙之處在於：

- **最小化對外暴露**：bridge 只綁 loopback，整個對話流量留在容器內。
- **持久化設計周全**：session state、cron jobs、agent 輸出全都透過單一 volume mount 得到持久化。
- **漸進式客製**：`AGENTS.md` 與 `.env` 分離，行為與設定都能在不 rebuild image 的前提下調整。
- **本機 ASR 閉環**：語音訊息全程在本機處理，不依賴外部語音 API。

對於想探索「如何把 AI agent 接上現實世界通訊工具」的開發者而言，這個範例提供了一個扎實的起點——雖然它仍標注為實驗性，但其展示的架構模式（同容器雙程序、loopback bridge、workspace mount）已足夠成熟，可以作為往後生產級設計的參考藍本。
