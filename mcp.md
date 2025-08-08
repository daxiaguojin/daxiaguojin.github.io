# MCP

## 簡單的 MCP（Model Context Protocol）程式片段


```python
# requirements:
# pip install mcp python-dotenv

import asyncio
import os
from mcp import StdioClient, connect_stdio
from dotenv import load_dotenv

# ---------- 1) 載入設定 ----------
load_dotenv()
NODE_PATH = os.getenv("NODE_PATH", "npx")   # 預設使用 npx 啟動
MCP_SERVER = os.getenv("MCP_SERVER", "my-mcp-server")  
# MCP Server 可以是你自己實作的工具服務，例如 "mcp-example-server"

# ---------- 2) 主要 async 流程 ----------
async def main():
    """
    這個範例會示範：
    1. 使用 stdio transport 啟動 MCP Server
    2. 初始化協議（初始化握手）
    3. 列出 Server 提供的 Tool
    4. 呼叫其中一個 Tool，並顯示結果
    """

    # 2.1 建立 stdio 連線
    # 相當於在終端機輸入: npx my-mcp-server
    transport = await connect_stdio(NODE_PATH, MCP_SERVER)

    # 2.2 初始化 MCP Client
    client = StdioClient(transport)
    await client.initialize()

    print("✅ MCP 連線成功！")

    # 2.3 列出 Server 可用的 Tool
    tools = await client.list_tools()
    print("🔍 Server 提供的工具：")
    for t in tools:
        print(f" - {t['name']}: {t['description']}")

    if not tools:
        print("⚠️ Server 沒有可用工具")
        return

    # 2.4 呼叫第一個工具（假設它接受一個 'query' 參數）
    tool_name = tools[0]['name']
    params = {"query": "查詢BPM資料庫中最新的流程數量"}
    print(f"⚡ 呼叫工具：{tool_name} 參數={params}")

    result = await client.call_tool(tool_name, params)
    print("📄 工具回傳結果：", result)

    # 2.5 關閉連線
    await client.close()
    print("🚪 MCP 連線已關閉")

# ---------- 3) 執行 ----------
if __name__ == "__main__":
    asyncio.run(main())
```

---

## 程式邏輯步驟解說

1. **啟動 MCP Server**

   * 使用 `npx my-mcp-server` 啟動（可以是任何符合 MCP 標準的 server，例如查詢資料庫、抓取 API、檔案搜尋等）。
   * 透過 **stdio**（標準輸入輸出）進行雙向通訊，適合本地或簡單網路情境。

2. **初始化協議**

   * `client.initialize()` 會進行握手，讓 Server 知道 Client 支援哪些功能，並同步 Server 提供的能力。

3. **列出可用 Tool**

   * `list_tools()` 會回傳 Server 提供的所有工具清單，每個工具包含 `name`、`description` 和 `parameters`。
   * 工具（Tool）就像是「LLM 可以直接呼叫的 API」。

4. **呼叫工具**

   * `call_tool(tool_name, params)` 呼叫 MCP Server 上的工具，傳入必要參數（例如查詢條件、檔案路徑等）。
   * 工具執行結果會回傳給 MCP Client，之後可以再餵給 LLM 使用。

5. **關閉連線**

   * MCP 是長連線協議，用完記得呼叫 `close()` 釋放資源。

---

## 這個範例的應用場景

* **企業內部資料存取**：
  LLM 想查詢 BPM 資料庫內容，但不直接連 DB，而是透過 MCP Server 提供的安全 API。
* **跨系統自動化**：
  MCP Server 可以整合 Jira、Confluence、ERP 等 API，LLM 可以用工具呼叫完成任務。
* **避免幻覺**：
  工具回傳的資料是真實系統返回的結果，LLM 僅需解讀與整理，不會憑空捏造。

---
