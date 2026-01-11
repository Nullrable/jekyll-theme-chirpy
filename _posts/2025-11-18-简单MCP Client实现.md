---
title: 简单MCP Client实现
author: nhsoft.lsd
date: 2025-11-18
categories: [Python, AI]
pin: false
---

使用 uv 创建项目和依赖

```bash
uv init mcp-client-demo

uv add "mcp[cli]"

pip install mcp
```

编写 mcp_client.py

```python
from mcp import ClientSession, StdioServerParameters
from mcp.client.stdio import stdio_client

# Create server parameters for stdio connection
server_params = StdioServerParameters(
command="python", # Executable
args=[
"mcp_demo1.py"
],# Optional command line arguments
env=None # Optional environment variables
)

async def run():
async with stdio_client(server_params) as (read, write):
async with ClientSession(read, write) as session:
# Initialize the connection
await session.initialize()

            # List available tools
            tools = await session.list_tools()
            print("Tools:", tools)

            # call a tool
            score = await session.call_tool(name="get_weather",arguments={"city": "上海"})

            print("weather: ", score)

if __name__ == "__main__":
import asyncio
asyncio.run(run())
```

运行代码，可以看到输出

![weixin.png](/assets/img/nhsoft_lsd/weixin.png)

<div style="text-align: center;">公众号名称：怪味Coding</div>
<div style="text-align: center;">微信扫码关注或搜索公众号名称</div>
