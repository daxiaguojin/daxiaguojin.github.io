# RAG
## 簡單的 RAG（Retrieval-Augmented Generation）範例

下面是一個用 Python 實作的**最小可運行範例**，展示 RAG 的完整流程：
**文件切分 → 向量化／索引 → 查詢 embedding → 檢索 top-k chunks → 組 prompt → LLM 生成答案**。範例使用 `sentence-transformers` 做 embeddings、`faiss` 當向量索引、以及 `openai` 作為 LLM（你也可以把 LLM 換成任何 Chat API）。

> **注意**：此範例是教學用，實務上請改用完善的 chunking／token 計數器、向量 DB（Pinecone、Milvus、Chroma 等），並加入授權、日誌、錯誤處理與資安控管。

```python
# requirements:
# pip install sentence-transformers faiss-cpu openai numpy

import os
import faiss
import numpy as np
from sentence_transformers import SentenceTransformer
import openai
import json
from typing import List, Dict

# ---------- 配置 ----------
openai.api_key = os.getenv("OPENAI_API_KEY")  # 或其它方式讀取 API KEY

EMBED_MODEL_NAME = "all-MiniLM-L6-v2"  # 教學用 embedding 模型（輕量、快速）
LLM_MODEL = "gpt-4o-mini"              # 這裡示範呼叫 OpenAI Chat Model（可換）
CHUNK_SIZE_WORDS = 180                # chunk 大小 (以詞數近似)
CHUNK_OVERLAP = 40                    # chunk 重疊詞數
TOP_K = 4                             # 檢索回傳的 chunk 數量

# ---------- 初始化 embedding 模型 ----------
embed_model = SentenceTransformer(EMBED_MODEL_NAME)

# ---------- 1) 文本分割（chunking） ----------
def split_text_into_chunks(text: str, chunk_size: int = CHUNK_SIZE_WORDS, overlap: int = CHUNK_OVERLAP) -> List[str]:
    """
    將長文本切成多個 chunk（以詞為單位的滑動窗口）。
    - chunk_size: 每 chunk 含大約多少個詞 (實務可基於 tokenizer 的 token 數)
    - overlap: 相鄰 chunk 之間重疊多少詞（保留上下文連貫性）
    """
    words = text.split()
    chunks = []
    start = 0
    n = len(words)
    while start < n:
        end = min(start + chunk_size, n)
        chunk = " ".join(words[start:end])
        chunks.append(chunk)
        if end == n:
            break
        start = end - overlap  # slide with overlap
    return chunks

# ---------- 2) 建索引（embed + faiss index） ----------
class SimpleRAGIndex:
    def __init__(self, embed_model):
        self.embed_model = embed_model
        self.ids_to_metadata: Dict[int, dict] = {}
        self.vectors = None
        self.index = None
        self.next_id = 0

    def build(self, docs: List[Dict[str, str]]):
        """
        docs: list of {"id": doc_id, "text": doc_text, "meta": {...}}
        對每個文件執行 chunking -> embed -> 建入 faiss index
        """
        all_chunks = []
        chunk_metas = []

        for doc in docs:
            doc_id = doc.get("id", f"doc{self.next_id}")
            text = doc["text"]
            meta = doc.get("meta", {})
            chunks = split_text_into_chunks(text)
            for i, chunk in enumerate(chunks):
                chunk_meta = {
                    "source_doc": doc_id,
                    "chunk_index": i,
                    "text": chunk,
                    "meta": meta
                }
                all_chunks.append(chunk)
                chunk_metas.append(chunk_meta)

        # embeddings (sentence-transformers 返回 numpy)
        embeddings = self.embed_model.encode(all_chunks, convert_to_numpy=True, show_progress_bar=True)
        # normalize embeddings for cosine similarity via inner product
        norms = np.linalg.norm(embeddings, axis=1, keepdims=True)
        norms[norms==0] = 1e-9
        embeddings = embeddings / norms

        # 建 FAISS index (Inner Product = cosine when normalized)
        dim = embeddings.shape[1]
        index = faiss.IndexFlatIP(dim)
        index.add(embeddings)

        # 保存 metadata 與 index
        self.index = index
        self.vectors = embeddings
        # map index id -> metadata
        for i, m in enumerate(chunk_metas):
            self.ids_to_metadata[i] = m

    def query(self, query_text: str, top_k: int = TOP_K):
        """
        1) 對 query embedding
        2) 在 index 搜尋 top_k
        3) 回傳 (score, metadata) 列表
        """
        q_emb = self.embed_model.encode([query_text], convert_to_numpy=True)
        q_emb = q_emb / np.linalg.norm(q_emb, axis=1, keepdims=True)
        D, I = self.index.search(q_emb, top_k)  # returns distances (inner product) and indices
        results = []
        for score, idx in zip(D[0], I[0]):
            meta = self.ids_to_metadata.get(int(idx), {})
            results.append({"score": float(score), "meta": meta})
        return results

# ---------- 3) 串接 LLM：組 prompt + 呼叫模型 ----------
def build_rag_prompt(query: str, retrieved: List[dict]) -> str:
    """
    將檢索到的 chunks 組成 prompt 給 LLM。
    我們會給 LLM 明確的 system 指令：*只能使用以下文件內容回答問題，若資料不足請誠實回覆 '我不知道'。*
    並把檢索到的 chunks 編號與來源附上，方便引用。
    """
    header = (
        "你是一個嚴謹的知識型助理。僅能使用下列的文件片段（DOCUMENTS）來回答使用者問題，"
        "不得憑空新增事實，若資料不足以回答請回覆：'我不知道' 並說明為何無法回答。\n\n"
    )
    docs_text = []
    for i, item in enumerate(retrieved, start=1):
        meta = item["meta"]
        chunk_text = meta["text"]
        src = meta.get("source_doc", "unknown")
        docs_text.append(f"[DOC {i} | source:{src}]\n{chunk_text}\n")
    docs_joined = "\n".join(docs_text)

    prompt = (
        header +
        "=== DOCUMENTS ===\n" +
        docs_joined +
        "\n=== END DOCUMENTS ===\n\n" +
        f"使用者問題：{query}\n\n" +
        "請根據上面 DOCUMENTS 的內容回答。若需要引用，請標註出處（例如 [DOC 1]）。\n"
    )
    return prompt

def call_llm(prompt: str, model: str = LLM_MODEL, max_tokens=512):
    """
    使用 OpenAI ChatCompletion（最簡版本）呼叫 LLM。
    注意：此處簡化成單次 system/user prompt，可依需求改成更複雜的 messages 結構。
    """
    # 這裡我們把整個 prompt 當作 user content，若要更嚴格可把 header 當 system message
    response = openai.ChatCompletion.create(
        model=model,
        messages=[
            {"role": "system", "content": "You are a helpful assistant that strictly uses provided documents."},
            {"role": "user", "content": prompt}
        ],
        temperature=0.0,
        max_tokens=max_tokens
    )
    # 取第一個候選回覆
    answer = response["choices"][0]["message"]["content"]
    return answer

# ---------- 4) 封裝一個 run_rag helper ----------
def run_rag(index: SimpleRAGIndex, query: str, top_k: int = TOP_K):
    # 1) 檢索
    retrieved = index.query(query, top_k=top_k)
    # 2) 若檢索得分很低，可視作沒有答案（這裡用一個簡單閾值示例）
    if retrieved and retrieved[0]["score"] < 0.2:
        return "我找不到與您問題相關的文件。"

    # 3) 構建 prompt 並呼叫 LLM
    prompt = build_rag_prompt(query, retrieved)
    answer = call_llm(prompt)
    # 4) 回傳答案與檢索來源（方便前端顯示或審核）
    return {
        "answer": answer,
        "retrieved": retrieved
    }

# ---------- 簡單示範如何使用 ----------
if __name__ == "__main__":
    # 範例文件（實務可從 PDF / DB / Confluence 等擷取）
    docs = [
        {"id": "spec-001", "text": "本系統支援 OAuth2 登入，token 有效期為 3600 秒。若需延長請使用 refresh flow。"},
        {"id": "guide-ops", "text": "啟動流程：先啟動 database，再啟動 backend。若 backend 無法連接 DB，檢查環境變數 DB_HOST。"}
    ]

    # 建立索引（較耗時：embedding）
    index = SimpleRAGIndex(embed_model)
    index.build(docs)

    # 用戶查詢
    q = "系統的登入 token 有效多久？"

    result = run_rag(index, q)
    print("=== Answer ===")
    print(result["answer"])
    print("\n=== Retrieved docs ===")
    for r in result["retrieved"]:
        print(r["meta"]["source_doc"], "score:", r["score"])
```

---

## 每個步驟的詳細說明

1. **Chunking（分割）**

   * 把長文件切成小段（chunk），以避免一次把整本文件塞入 embedding 或 prompt 而超過 token 限制。
   * 例中以「詞數」近似 chunk\_size（實務上建議用 tokenizer 真實計 token 數）。

2. **Embedding（向量化）**

   * 把每個 chunk 轉成向量（dense vector），語意相近的文本其向量在空間上距離接近。
   * 本範例用 `sentence-transformers`（本地模型）作示範；也可用 OpenAI / Anthropic 的 embedding API（商用 / 精度不同）。

3. **Indexing（索引）**

   * 把 embeddings 存入向量 DB（例：FAISS、Pinecone、Milvus、Chroma）。
   * 查詢時會把 query 同樣 embedding，做相似度搜尋（nearest neighbors / top\_k）。

4. **Retrieval（檢索）**

   * 從向量資料庫取回 top\_k 最相似的 chunk，並回傳分數（score）表示相似度。
   * 可加入閾值判斷（score 太低 → 回傳「找不到」），避免把無關 chunk 帶給 LLM。

5. **Prompt Construction（組 prompt）**

   * 將檢索到的 chunk 放入 prompt（或 system message 的 context）並告訴 LLM：「只能使用這些文件來回答」。
   * 同時要求 LLM 在回答中標註引用來源（例如 `[DOC 1]`），能幫助審核與可追溯性。

6. **Generation（生成）**

   * 將組好的 prompt 送入 LLM（例如 ChatGPT、Claude、Gemini），取得最終答案。
   * 爲降低 hallucination，`temperature=0.0`、並明確 instruct LLM 僅使用文件內容回答很重要。

7. **後處理與來源回傳**

   * 將生成的答案與實際檢索到的文件片段一併回傳給前端或審核者，確保能追溯與稽核。

---

## 實務注意事項（Checklist）

* **Chunk 大小 & overlap**：依任務調整（查詢精準 → 小 chunk；主題理解 → 大 chunk）。
* **Embedding 模型一致性**：做 query/embed 時務必使用同一 embedding 模型。
* **向量 DB 選擇**：FAISS 適合本地原型；Pinecone/Chroma/Milvus 適合生產與資料量大情況。
* **Prompt 約束**：明確告訴 LLM「只能使用給定文件回答」，並設定低 temperature，降低幻覺。
* **可信度閾值**：檢索分數太低時回傳「無法回答」或觸發 fallback（如向使用者請求更多上下文）。
* **審核與版本控制**：保存檢索片段與 LLM 輸出，便於事後追蹤與修正。
* **資安/隱私**：機敏資料需做 redaction、資料存取權限控管與加密。

---
