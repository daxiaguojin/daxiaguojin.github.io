# Function Calling

## 程式
```python
from openai import OpenAI
import json

client = OpenAI()

# 模擬函數：查詢天氣
def get_weather(city: str):
    weather_data = {
        "Taipei": "Sunny, 30°C",
        "Tokyo": "Rainy, 22°C",
        "New York": "Cloudy, 25°C"
    }
    return weather_data.get(city, "Weather data not available.")

# 定義 Function Calling 的結構
tools = [
    {
        "type": "function",
        "function": {
            "name": "get_weather",
            "description": "查詢指定城市的天氣狀況",
            "parameters": {
                "type": "object",
                "properties": {
                    "city": {"type": "string", "description": "城市名稱，例如 Taipei"}
                },
                "required": ["city"]
            }
        }
    }
]

# 提示詞：指示 LLM 何時要使用 Function Calling
system_prompt = """
你是一個智慧助理。
- 如果使用者問到「某城市天氣」，請使用 get_weather 函數查詢。
- 如果問題不是關於天氣，請直接用自然語言回答，不要呼叫函數。
"""

# 發送請求
response = client.chat.completions.create(
    model="gpt-4o-mini",
    messages=[
        {"role": "system", "content": system_prompt},
        {"role": "user", "content": "台北的天氣如何？"}
    ],
    tools=tools
)

# 檢查模型是否要呼叫函數
if response.choices[0].message.tool_calls:
    tool_call = response.choices[0].message.tool_calls[0]
    if tool_call.function.name == "get_weather":
        args = json.loads(tool_call.function.arguments)
        result = get_weather(args["city"])
        print(f"查詢結果：{result}")
else:
    # 如果沒有 function call，直接回覆模型的自然語言回答
    print("LLM 回覆：", response.choices[0].message.content)
```
## 運作原理

LLM 不只是盲目呼叫，而是「根據任務需求」決定要不要 function calling。

1. System Prompt 引導

    - LLM 被告知「天氣問題 → 用工具查詢；其他問題 → 直接回答」
    - 這讓 LLM 知道該「何時」呼叫 function。

2. User 問台北天氣

    - LLM 依規則 → 呼叫 get_weather(city="Taipei")。

3. 如果問非天氣問題

    - LLM 就不會使用 function，而是直接輸出答案。
