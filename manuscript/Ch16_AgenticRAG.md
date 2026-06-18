# Ch 16　Agentic RAG：用 LangGraph 打造檢索型 agent

> **本章目標**
>
> - 分清楚傳統 RAG 與 **agentic RAG** 的差別：讓 LLM 自己決定「要不要檢索、檢索夠不夠、要不要重查」。
> - 用 Ch15 的編排模式（routing + evaluator-optimizer）組出一個會評分、會重寫查詢的檢索型 agent。
> - 掌握關鍵零件：document loader / splitter / embeddings / vector store、retriever tool、文件相關性評分（structured output）。
> - 理解這條「檢索 → 評分 → 不夠就重寫再查」的迴圈，正是 production RAG 比 toy RAG 可靠的關鍵。
>
> **使用版本**：`langgraph` 1.x、`langchain`、`langchain-google-genai`。配套 notebook 用離線假檢索器示範**結構**，免 API key。

---

## 16.1 從 RAG 到 agentic RAG

**RAG（Retrieval-Augmented Generation）** 的基本想法：回答前先去知識庫檢索相關片段，把它塞進 prompt，讓模型「有依據地」回答，而非憑空編（呼應 Ch1 的限制一）。

傳統 RAG 是固定流程：**一定先檢索 → 一定把結果塞進 prompt → 生成**。但這有兩個問題：使用者只說「你好」也去檢索很浪費；檢索回來的東西如果不相關，模型還是會被誤導。

**agentic RAG** 把「決策」交給模型：

- **要不要檢索？** 「你好」就直接回；「Lilian Weng 對 reward hacking 的看法？」才去檢索。
- **檢索夠好嗎？** 對檢索結果做**相關性評分**；不相關就**重寫查詢**再檢索一次，而不是硬用爛結果生成。

換句話說，agentic RAG = RAG + Ch15 的 routing（要不要檢索）+ evaluator-optimizer（評分、不夠就重來）。這章就是把這些模式組起來。

## 16.2 準備知識庫：load → split → embed → index

先把文件變成可語意檢索的 vector store。四個步驟（每步都有對應的 LangChain 元件）：

```python
import bs4, requests
from langchain_core.documents import Document
from langchain_text_splitters import RecursiveCharacterTextSplitter
from langchain_core.vectorstores import InMemoryVectorStore
from langchain_google_genai import GoogleGenerativeAIEmbeddings

# 1. load：抓網頁內容（這裡用最小 helper 示範）
def load_web_page(url: str) -> list[Document]:
    r = requests.get(url); r.raise_for_status()
    text = bs4.BeautifulSoup(r.text, "html.parser").get_text()
    return [Document(page_content=text, metadata={"source": url})]

urls = ["https://lilianweng.github.io/posts/2024-07-07-hallucination/"]
docs = [d for url in urls for d in load_web_page(url)]

# 2. split：切成小 chunk 才好檢索
splitter = RecursiveCharacterTextSplitter.from_tiktoken_encoder(chunk_size=200, chunk_overlap=50)
doc_splits = splitter.split_documents(docs)

# 3. embed + 4. index：用 Gemini embeddings 建向量索引
vectorstore = InMemoryVectorStore.from_documents(
    documents=doc_splits,
    embedding=GoogleGenerativeAIEmbeddings(model="models/text-embedding-004"),
)
retriever = vectorstore.as_retriever()
```

`InMemoryVectorStore` 適合開發；production 換成持久的 vector store（Chroma、PGVector、Pinecone 等），介面相同。

## 16.3 把檢索包成一個工具

agentic RAG 的關鍵：檢索不是「一定會跑的步驟」，而是**一個工具**，讓模型自己決定要不要用。用 `@tool` 包起來：

```python
from langchain.tools import tool

@tool
def retrieve_blog_posts(query: str) -> str:
    """搜尋並回傳 Lilian Weng 部落格文章的相關內容。"""   # docstring = 給模型的工具說明
    docs = retriever.invoke(query)
    return "\n\n".join(d.page_content for d in docs)
```

## 16.4 節點一：generate_query_or_respond（要不要檢索？）

第一個節點讓模型看著對話，**自己決定**：直接回答，還是呼叫檢索工具。關鍵是 `.bind_tools` 把工具綁給模型——它會在需要時回傳一個 tool call（呼應 Ch1 的 tool calling）：

```python
from langgraph.graph import MessagesState
from langchain.chat_models import init_chat_model

response_model = init_chat_model("google_genai:gemini-2.5-flash", temperature=0)

def generate_query_or_respond(state: MessagesState):
    response = response_model.bind_tools([retrieve_blog_posts]).invoke(state["messages"])
    return {"messages": [response]}
```

問「你好」時，它直接回；問「reward hacking 有哪些類型」時，它回傳一個 `retrieve_blog_posts` 的 tool call。這就是 routing——但路由的決定是模型做的，不是寫死的條件。

## 16.5 節點二：grade_documents（檢索夠好嗎？）

檢索回來後，用一個**條件邊**評估文件相關性——這是 agentic RAG 的精髓。用 structured output 讓模型吐出乾淨的 yes/no 判斷（structured output 是 Ch20 主題，這裡先用）：

```python
from pydantic import BaseModel, Field
from typing import Literal

class GradeDocuments(BaseModel):
    """以二元分數評估文件相關性。"""
    binary_score: str = Field(description="'yes' 表示相關，'no' 表示不相關")

GRADE_PROMPT = (
    "你是評估『檢索到的文件與使用者問題是否相關』的評分員。\n"
    "檢索到的文件：\n\n {context} \n\n 使用者問題：{question}\n"
    "若文件含與問題相關的關鍵字或語意，評為相關。給 'yes' 或 'no'。"
)

def grade_documents(state: MessagesState) -> Literal["generate_answer", "rewrite_question"]:
    question = state["messages"][0].content
    context = state["messages"][-1].content      # 最後一則是檢索結果（tool message）
    prompt = GRADE_PROMPT.format(question=question, context=context)
    score = response_model.with_structured_output(GradeDocuments).invoke(prompt).binary_score
    return "generate_answer" if score == "yes" else "rewrite_question"
```

這個函數當條件邊用：相關 → 去生成答案；不相關 → 去重寫問題。

## 16.6 節點三、四：rewrite_question 與 generate_answer

- **rewrite_question**：檢索不夠好時，請模型把使用者的問題**改寫**成更利於檢索的版本，然後回到 `generate_query_or_respond` 再檢索一次——這就是 evaluator-optimizer 迴圈。
- **generate_answer**：文件夠相關時，把問題與檢索結果一起交給模型，生成有依據的最終答案。

## 16.7 把圖組起來

```
START → generate_query_or_respond → (要檢索?) ──tool──> retrieve
                  │ (直接回答)                              │
                  ▼                                  grade_documents
                 END                                  │         │
                                          相關→generate_answer  不相關→rewrite_question
                                                 │                      │
                                                END              （回 generate_query_or_respond 再來一次）
```

用 Graph API 把上面四個節點 + 一個 tools 節點接起來（`add_conditional_edges` 接 `tools_condition` 與 `grade_documents`）。整張圖跑起來，就是一個會「該查才查、查不好就改寫再查」的 agentic RAG——比「無腦先檢索再生成」的傳統 RAG 可靠得多。

> 銜接前後：這裡的 `bind_tools` + tools 節點 + 條件邊，正是 Ch5 的 Graph API；「評分後決定重來」是 Ch15 的 evaluator-optimizer；而 Part II 的 `create_agent` 會把「generate_query_or_respond + tools loop」這段一行包好——你現在是手刻它，之後會看到高階版。

---

## 常見坑（Pitfalls）

- **無腦每次都檢索**：那是傳統 RAG，不是 agentic。讓檢索成為「工具」，由模型決定要不要用。
- **不評分就用檢索結果**：檢索回不相關的東西照塞進 prompt，模型會被誤導。grade → 不相關就重寫再查，是可靠性的關鍵。
- **rewrite 沒有上限**：可能無限改寫。比照 Ch15 evaluator-optimizer，加重試上限。
- **chunk 切太大或太小**：太大檢索不精準、太小失去上下文。`chunk_size`/`chunk_overlap` 要依內容調。
- **開發用 `InMemoryVectorStore` 上 production**：一關就沒了。換持久 vector store。

## 本章小結

agentic RAG 把「決策」交還給模型：不再固定「先檢索再生成」，而是讓模型決定**要不要檢索**（routing），並對檢索結果做**相關性評分**、不夠好就**重寫查詢再檢索**（evaluator-optimizer）。我們準備了知識庫（load → split → embed → index）、把檢索包成 `@tool`、用 `bind_tools` 讓模型自己決定是否檢索、用 structured output 做文件評分，再用條件邊把「相關→生成 / 不相關→重寫」串成迴圈。這整套就是 Part I 機制（Graph API、條件邊）+ Ch15 編排模式的具體應用，也比 toy RAG 可靠得多。下一章我們做另一個經典應用——讓 agent 安全地查 SQL 資料庫。

## 延伸閱讀 / 練習

1. **離線結構版**：用一個 dict 當「假知識庫」、用關鍵字當「假檢索」與「假評分」，把 16.7 的圖結構搭起來跑，確認「相關→生成、不相關→重寫」的路徑（配套 notebook 有可跑版本，免 API key）。
2. **加重試上限**：在 rewrite 迴圈加一個 `rewrites` 計數與上限，避免無限改寫。
3. **線上版**：若你有 Gemini 金鑰，把假零件換成真的 embeddings + LLM，問一題需要檢索、一題不需要檢索的問題，觀察 routing。
4. **思考題**：為什麼「評分後重寫再查」比「一次檢索就用」可靠？用 Ch1 的「限制一（沒有外部資訊）」與「幻覺」風險回答。
