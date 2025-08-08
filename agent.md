# AGENT

## 一個可直接執行的簡易 Python 範例

展示 Agent 的核心 loop（plan → act → observe → reflect）。

```python
# simple_agent_demo.py
# 只需 Python 3.8+，無外部套件
import time
from typing import List, Dict, Any

# ---------- 假想工具（Tools） ----------
# 在真實系統裡，Tool 可能是 DB 查詢 / HTTP API / 檔案讀取 / 執行命令 等
# 這裡用簡單函式模擬工具行為與輸出（包含 metadata 以利追溯）

DOCS = [
    {"id": "spec-001", "text": "本系統支援 OAuth2 登入，token 有效期為 3600 秒。若需延長請使用 refresh flow。"},
    {"id": "guide-ops", "text": "啟動流程：先啟動 database，再啟動 backend。若 backend 無法連接 DB，檢查環境變數 DB_HOST。"},
    {"id": "sec-pol", "text": "使用者密碼策略：長度至少 12 字元，並啟用 MFA。"}
]

def tool_search_docs(query: str, top_k: int = 3) -> Dict[str, Any]:
    """
    簡單的關鍵字搜尋模擬（非 embedding）。
    回傳 top_k 份最相關的文件與相似度分數（模擬）。
    """
    q = query.lower()
    results = []
    for d in DOCS:
        score = 0
        text_lower = d["text"].lower()
        # very naive scoring: +1 per keyword match
        for token in q.split():
            if token in text_lower:
                score += 1
        if score > 0:
            results.append({"doc_id": d["id"], "text": d["text"], "score": score})
    # sort by score desc
    results.sort(key=lambda x: x["score"], reverse=True)
    return {"tool": "search_docs", "query": query, "results": results[:top_k]}

def tool_summarize(chunks: List[str]) -> Dict[str, Any]:
    """
    非常簡單的 summarize：把每段取第一句並合併。
    回傳 summary 與來源引用（簡略）。
    """
    summary_lines = []
    for c in chunks:
        # split by punctuation (very naive)
        first_sentence = c.split("。")[0]
        summary_lines.append(first_sentence.strip() + "。")
    summary = " ".join(summary_lines)
    return {"tool": "summarize", "summary": summary, "sources": len(chunks)}

def tool_get_server_time() -> Dict[str, Any]:
    return {"tool": "get_server_time", "time": time.strftime("%Y-%m-%dT%H:%M:%SZ", time.gmtime())}

# ---------- 一個簡單的 Agent class ----------
class SimpleAgent:
    def __init__(self):
        self.memory: List[Dict[str, Any]] = []  # short-term memory / observations
        # 可註冊工具表
        self.tools = {
            "search_docs": tool_search_docs,
            "summarize": tool_summarize,
            "get_server_time": lambda: tool_get_server_time()
        }

    def observe(self, observation: Dict[str, Any]):
        """把工具輸出或重要事件存進 memory（以便後續決策）"""
        self.memory.append(observation)

    def plan(self, goal: str) -> Dict[str, Any]:
        """
        最簡單的 rule-based planner：
        - 若 goal 包含 '彙整' 或 '摘要' -> 先 search_docs 再 summarize
        - 若 goal 包含 '時間' -> call get_server_time
        - 否則預設先 search_docs
        回傳 action dict: {'tool': name, 'args': {...}}
        """
        g = goal.lower()
        if "時間" in g or "現在時間" in g:
            return {"tool": "get_server_time", "args": {}}
        if "彙整" in g or "摘要" in g or "總結" in g or "整理" in g:
            # search first to gather data
            return {"tool": "search_docs", "args": {"query": goal, "top_k": 3}}
        # default
        return {"tool": "search_docs", "args": {"query": goal, "top_k": 2}}

    def execute(self, action: Dict[str, Any]) -> Dict[str, Any]:
        """執行工具並返回觀察"""
        tool_name = action.get("tool")
        args = action.get("args", {})
        tool_fn = self.tools.get(tool_name)
        if tool_fn is None:
            return {"error": f"Tool {tool_name} not found."}
        # call tool (support positionalless call)
        if isinstance(args, dict) and args:
            res = tool_fn(**args)
        else:
            res = tool_fn()
        # 包裝觀察（包含工具名、回傳與 timestamp）
        obs = {"tool": tool_name, "result": res, "timestamp": time.time()}
        self.observe(obs)
        return obs

    def is_goal_satisfied(self, goal: str) -> bool:
        """
        非常簡單的 goal check：如果 memory 裡出現 summarize 的結果，視為完成。
        生產系統通常會有更複雜的 success 判別器。
        """
        for m in reversed(self.memory):
            if m["tool"] == "summarize":
                return True
        return False

    def run(self, goal: str, max_steps: int = 6) -> Dict[str, Any]:
        """
        Agent loop:
          1) plan -> returns an action
          2) execute action -> tool returns observation
          3) optionally decide next action based on observations
          4) repeat until goal satisfied or max_steps reached
        """
        print(f"Agent start. Goal: {goal}")
        steps = 0
        last_search_results = None

        while steps < max_steps:
            steps += 1
            action = self.plan(goal)
            print(f"\n[Step {steps}] Planner => {action}")
            obs = self.execute(action)
            print("[Tool output]:", obs["result"])

            # 簡單的 decision logic：
            if action["tool"] == "search_docs":
                # 如果檢索到文件，就用 summarize 去總結
                hits = obs["result"].get("results", [])
                if hits:
                    last_search_results = [h["text"] for h in hits]
                    # 下一動作改成 summarize
                    print("Planner decides to summarize retrieved docs.")
                    action = {"tool": "summarize", "args": {"chunks": last_search_results}}
                    obs2 = self.execute(action)
                    print("[Tool output]:", obs2["result"])
                    # 總結後結束（示範）
                    break
                else:
                    print("No docs found. Agent may ask user for clarification (not implemented).")
                    break
            elif action["tool"] == "get_server_time":
                # 直接回應並結束
                return {"final": obs["result"]}
            else:
                # fallback end
                break

        # 最終回傳 agent memory 與最後的 summary（如果有）
        final_summary = None
        for m in reversed(self.memory):
            if m["tool"] == "summarize":
                final_summary = m["result"]["summary"]
                break
        return {"goal": goal, "summary": final_summary, "memory": self.memory}

# ---------- 示範運行 ----------
if __name__ == "__main__":
    agent = SimpleAgent()
    goal = "請彙整系統登入與啟動流程相關資訊"
    out = agent.run(goal)
    print("\n=== AGENT FINAL OUTPUT ===")
    print(out["summary"])
    # 印出 memory 便於檢視 agent 的行為歷程
    print("\nMemory (action trace):")
    for m in out["memory"]:
        print(m)
```

---

## 程式逐步說明（教學用重點）

### Agent 的主要組成部分（在範例中）

* **Goal（目標）**：使用者輸入的任務描述，例如：「請彙整系統登入與啟動流程相關資訊」。
* **Planner（計畫器）**：決定下一步要做什麼（在範例中是 `plan()` 函數，採簡單規則）。
* **Tools（工具）**：能被 Agent 呼叫的功能集合（例如 `search_docs`、`summarize`、`get_server_time`）。生產環境可能有：向量檢索、DB 查詢、API 呼叫、函數執行等。
* **Executor（執行器）**：負責呼叫工具並回收觀察結果（`execute()`）。
* **Memory（記憶 / 日誌）**：保存每次執行結果與觀察，供後續決策或稽核使用（`self.memory`）。
* **Observation（觀察）**：工具輸出的內容，會被記錄，並可能影響下一步行為。
* **Stop Criteria（終止條件）**：當達成目標或超過最大步數就停止（此範例以是否有 `summarize` 結果判定完成）。

---

### Agent 的運作原理（概念圖）

1. **接收 Goal**（User 指令）
2. **Planner** 決定 action（例如：`search_docs(query)`）
3. **Executor** 呼叫對應工具 → 取得 Observation（工具回傳結果）
4. **Agent 保存 Observation 到 Memory**（便於 trace、debug、後續決策）
5. **Planner 重新評估（可能基於 Memory + Observation）** → 決定下一步（例如：若找到了文件 → 呼叫 summarize）
6. **針對觀察進行產出（Generation）或繼續行動**，直到目標完成或超過限制

---

## 生產環境中，常見的 Agent 變體（用 LLM 當 Planner）

在真實世界，**Planner** 很多時候會用 LLM 來生成下一步動作（structed output），例如使用 OpenAI 的 Function Calling、Anthropic 的工具呼叫、或 MCP。流程通常是：

1. **Agent 向 LLM 傳入**：Goal + 最近的 Memory + 可用 Tools 列表（含 schema/parameters）
2. **LLM 回應**：要麼直接給 final answer，要麼回傳一個「要呼叫哪個工具、帶哪些參數」的結構化回覆（例：`{ "tool": "search_docs", "args": {"query":"..."} }`）
3. **Agent 解析 LLM 的回覆，執行工具**，把結果觀察回饋給 LLM（或存 Memory）
4. **LLM 繼續規劃下一步或生成最終答案**（形成一個 iterative loop）

以下是一段「伪 code」示意（OpenAI function-calling 風格）：

```text
# 1) send messages to LLM:
system: "You are an agent. Available tools: search_docs(query), summarize(chunks). Always respond with JSON action if not final."
user: "請彙整系統登入與啟動流程相關資訊。"

# 2) LLM 回:
{"action":"call_tool","tool":"search_docs","args":{"query":"系統 登入 啟動 流程","top_k":3}}

# 3) Agent 執行 tool, 將結果 obs 回給 LLM:
user: "TOOL_RESULT: [ ...文本... ]"
# LLM 可能繼續回: {"action":"call_tool","tool":"summarize","args":{"chunks":[...]}}

# 4) 最終 LLM 回: {"action":"final_answer","content":"根據文件，... (並引用 DOC id)"}
```

---

## 優化與實務考量（Checklist）

* **工具設計要有明確 schema**（參數型別、必要欄位、返回格式） → 容易由 LLM 自動生成呼叫。
* **安全與授權**：Agent 呼叫的每個工具都要做權限檢查，避免 LLM 被用來操作敏感系統。
* **輸入驗證與匯流排**：Tool 的輸入要 sanitize，並有 timeout / retry 機制。
* **循環偵測與防爆炸**：限制最大步數、防止無限迴圈（planner 反覆呼叫同一工具）。
* **審計日誌（Audit trail）**：保留所有 planner 指令、tool inputs/outputs、LLM 回覆，便於稽核與錯誤追蹤。
* **Human-in-the-loop**：在高風險動作前加入人工確認（例如：修改 DB、發送郵件等）。
* **Fallback 策略**：當工具失敗或檢索分數過低時，向使用者提問或回報「無法回答」。

---

## 範例輸出解讀（用上面程式）

若執行上面的 `simple_agent_demo.py`，在 `goal = "請彙整系統登入與啟動流程相關資訊"` 的情況下，Agent 會：

1. 規劃先 `search_docs`（根據 goal 關鍵字）
2. 檢索到 `spec-001` 與 `guide-ops` 的相關文件（模擬）
3. 再呼叫 `summarize` 對檢索結果做合併摘要
4. 返回 summary（並在 memory 留下工具呼叫 trace）

這個過程展示了**搜尋到來源 → 摘要 → 回傳**的常見 RAG+Agent 工作流範式。

---

## 小結

* **Agent = Planner（大腦） + Tools（手腳） + Memory（短期/長期記憶） + Executor（執行器）**