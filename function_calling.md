# Function Calling範例

```python
from openai import OpenAI
import json

client = OpenAI()

# 定義一個模擬函數：查詢天氣
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

# 發送請求給 LLM
response = client.chat.completions.create(
    model="gpt-4o-mini",
    messages=[{"role": "user", "content": "台北的天氣如何？"}],
    tools=tools
)

# 檢查模型是否要呼叫函數
if response.choices[0].message.tool_calls:
    tool_call = response.choices[0].message.tool_calls[0]
    if tool_call.function.name == "get_weather":
        args = json.loads(tool_call.function.arguments)
        result = get_weather(args["city"])
        print(f"查詢結果：{result}")
```