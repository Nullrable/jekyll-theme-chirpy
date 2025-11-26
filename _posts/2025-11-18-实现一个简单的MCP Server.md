---
title: 实现一个简单的MCP Server
author: nhsoft.lsd
date: 2025-11-18
categories: [Python, AI]
pin: false
---

安装 uv && 初始化项目
```shell
pip install uv==0.5.24

uv init mcp_demo1
```

安装依赖 FastMCP

```shell
cd mcp_demo1

uv add "mcp[cli]"
```

mcp_server 代码实现，使用标准输出

```python
# server.py
from mcp.server.fastmcp import FastMCP

# 创建一个 MCP 服务器
mcp = FastMCP("Demo")


# 添加一个加法工具
@mcp.tool()
def add(a: int, b: int) -> int:
"""将两个数字相加"""
return a + b

    # 添加一个加法工具
@mcp.tool()
def get_weather(city: str) -> str:
"""获取天气"""
return f"{city}的天气是暴雨，温度 20 度"


if __name__ == "__main__":
mcp.run(transport='stdio')

```

在MCP Host 中配置该server。这里用到的是 Cursor 中 Roo Code， 没有的可以在插件市场中下载. 编辑 Project Config

```
{
"mcpServers": {
"elicit-mcp": {
"command": "C:\\Users\\nhsof\\PyCharmMiscProject\\.venv\\Scripts\\python.exe",
"args": [
"C:\\Users\\nhsof\\PyCharmMiscProject\\mcp_logger.py", //这个可以不需要
"C:\\Users\\nhsof\\PyCharmMiscProject\\.venv\\Scripts\\uv.exe",
"--directory",
"C:\\Users\\nhsof\\PyCharmMiscProject",
"run",
"mcp_demo1.py"
]
}
}
}
```

![img.png](/assets/img/nhsoft_lsd/img_mcp_server.png)

配置完成后，配置成功的会有个小绿点，注意不是开关按钮，那个开关按钮是控制 MCP 是否启用。这样一切准备就绪后就可以在聊天框里进行问答，比如：

![img_1.png](/assets/img/nhsoft_lsd/img_mcp_server_1.png)

![img_2.png](/assets/img/nhsoft_lsd/img_mcp_server_2.png)


![weixin.png](/assets/img/nhsoft_lsd/weixin.png)

公众号名称：怪味Coding
微信扫码关注或搜索公众号名称

