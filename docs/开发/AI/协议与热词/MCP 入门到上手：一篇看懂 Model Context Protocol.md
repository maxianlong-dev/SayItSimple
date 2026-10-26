# MCP 入门到上手：一篇看懂 Model Context Protocol
面向所有想给 LLM 加工具的人 —— 不需要懂底层协议，跟着写就能跑。
---
## 一、什么是 MCP
**MCP（Model Context Protocol，模型上下文协议）** 是 Anthropic 在 2024 年提出、并迅速被 OpenAI、Google、社区跟进的一套**标准协议**，用来让大语言模型（LLM）调用外部工具、读取数据、访问服务。
它本身不是一个框架，而是一份"接口规范"。底层是 JSON-RPC，传输方式可以是：
- **stdio**（进程间标准输入输出，本地 IDE 启动子进程时用）
- **streamable-http**（最新推荐的远程协议）
- **SSE**（早期版本，正在淘汰）
### 一句话理解
**MCP 之于 AI 工具，就像 USB-C 之于硬件外设。**
### MCP 里有哪些角色
| 角色 | 是谁 | 干什么 |
| --- | --- | --- |
| **Host** | Claude Desktop / Cursor / 你写的 Agent | 把 LLM 和 MCP 客户端跑起来的程序 |
| **Client** | Host 内部一个 MCP 连接管理器 | 跟 Server 对话，转发工具调用 |
| **Server** | 你写的工具集合 | 暴露 tools / resources / prompts |
最常用的就是 **Tools**（工具）：一个带类型签名的函数，LLM 决定调它，Server 执行后把结果返回给 LLM。
---
## 二、为什么需要 MCP
没有 MCP 之前，给 LLM 加工具有几个老大难问题：
1. **工具协议碎片化**
LangChain 有自己的 `Tool`，OpenAI Function Calling 有自己的 schema，Anthropic 有自己的 tool use 格式，Cursor、Cline、Claude Desktop 各自又有插件机制。**同一个"查天气"工具，要给每家写一份。**
2. **工具和模型强耦合**
工具实现里写死 `import openai`，换 Claude 就要重写；工具实现写死 LangChain 类，换框架就要重写。
3. **能力难复用**
团队 A 做了个"查内部知识库"的工具，团队 B 想用，得把代码 fork 过去自己改。
4. **权限/部署混乱**
工具直接在应用进程里跑，密钥散落在各处客户端配置里，难以统一审计。
MCP 就解决这四件事：
| 旧问题 | MCP 给出的答案 |
| --- | --- |
| 协议碎片化 | 一份 JSON-RPC 规范，所有客户端共用 |
| 工具/模型耦合 | Server 不感知模型，纯工具实现 |
| 能力难复用 | Server 是独立进程/服务，URL 一发就能给别人 |
| 权限混乱 | 密钥在 Server 端，客户端只持 URL |
---
## 三、现成可用的 MCP Server（含 CLI 类）
很多场景根本不需要自己写。社区已经积累了大量开源 MCP，**绝大多数是 CLI 起服务、**`npx`** / **`uvx`** 一行命令就能拉起**，这是 MCP 生态最高频的形态。
### 3.1 官方仓库（modelcontextprotocol/servers）
| Server | 干什么 | 启动命令 |
| --- | --- | --- |
| `filesystem` | 读写本地文件、目录 | `npx -y @modelcontextprotocol/server-filesystem /path` |
| `git` | 本地 git 仓库操作 | `uvx mcp-server-git --repository /path` |
| `github` | GitHub API（PR / Issue / 搜索 / 评论） | `npx -y @modelcontextprotocol/server-github` |
| `sqlite` | SQLite 数据库读写 | `uvx mcp-server-sqlite --db-path ./db.sqlite` |
| `postgres` | Postgres 只读查询 | `npx -y @modelcontextprotocol/server-postgres "postgresql://..."` |
| `puppeteer` | 浏览器自动化截图 / 抓取 | `npx -y @modelcontextprotocol/server-puppeteer` |
| `fetch` | HTTP 抓取网页内容 | `uvx mcp-server-fetch` |
| `slack` | 收发 Slack 消息 | `npx -y @modelcontextprotocol/server-slack` |
| `memory` | 给 LLM 一个持久化知识图谱 | `npx -y @modelcontextprotocol/server-memory` |
| `time` | 时区 / 时间换算 | `uvx mcp-server-time` |
### 3.2 第三方常用
| Server | 干什么 |
| --- | --- |
| `@playwright/mcp` | 浏览器自动化（比 puppeteer 完整） |
| `@upstash/context7-mcp` | 实时检索最新版库文档，写代码神器 |
| `@cloudflare/mcp-server-cloudflare` | 操作 Cloudflare 资源 |
| `sentry-mcp` | 拉取 Sentry 报错 |
| `notion-mcp` | 操作 Notion 数据库与文档 |
| `linear-mcp` | Linear 工单管理 |
| `figma-developer-mcp` | 读 Figma 设计稿，配合 Cursor 直接生成代码 |
### 3.3 典型 CLI MCP 配置示例
让 AI 能读你的项目文件 + 操作 git + 调 GitHub —— `claude_desktop_config.json` / `~/.cursor/mcp.json` / 项目 `.mcp.json` 通用：
```plain text
{
  "mcpServers": {
    "fs": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/Users/me/projects"]
    },
    "git": {
      "command": "uvx",
      "args": ["mcp-server-git", "--repository", "/Users/me/projects/myrepo"]
    },
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": { "GITHUB_PERSONAL_ACCESS_TOKEN": "ghp_xxxxx" }
    },
    "playwright": {
      "command": "npx",
      "args": ["-y", "@playwright/mcp@latest"]
    }
  }
}
```
加上后就可以直接说："帮我看 myrepo 最近三个 commit 改了啥"、"在 GitHub 上把 issue #42 标题改成 …"、"打开 example.com 截个图"。
**小贴士**：CLI 类 MCP 第一次启动会下载，之后命中缓存秒起。零安装成本，删除即干净。
### 3.4 在哪找 MCP（市场 / 目录）
**官方**
- 官方仓库：https://github.com/modelcontextprotocol/servers
**社区市场**
- Awesome MCP Servers（社区精选清单）：https://github.com/punkpeye/awesome-mcp-servers
- **Smithery**（一键安装、托管）：https://smithery.ai
- **Glama MCP Directory**：https://glama.ai/mcp/servers
- **Pulse MCP**（带评分与分类）：https://www.pulsemcp.com
- **mcp.so**（中文友好）：https://mcp.so
- **MCP Servers Directory**：https://mcpservers.org
很多市场都提供"复制配置 → 粘进客户端 mcp.json"的一键功能，找到合适的服务就能用。
---
## 四、如何构建自己的 MCP（用 fastmcp）
### 4.1 fastmcp 简介
[fastmcp](https://github.com/jlowin/fastmcp) 是 Python 社区最流行的 MCP Server 框架，比官方 `mcp` SDK 更简洁：
- 一个 `@mcp.tool` 装饰器把普通函数变成 MCP 工具
- 参数 schema 从类型注解和 `pydantic.Field` 自动生成
- stdio / streamable-http / SSE 三种传输方式一行启动
- 支持挂到 FastAPI / Starlette 做生产部署
**注意区分两个名字相近的东西**：本文统一指后者。
### 4.2 装一下
```plain text
pip install fastmcp
# 或用 uv
uv add fastmcp
```
### 4.3 第一个 MCP Server：天气查询
```plain text
# weather_server.py
from fastmcp import FastMCP
from pydantic import Field

mcp = FastMCP("weather-server")


@mcp.tool(
    name="get_weather",
    description="查询某个城市的当前天气",
)
async def get_weather(
    city: str = Field(description="城市名称，例如：北京、上海"),
) -> str:
    # 真实场景里这里调天气 API；演示用就 mock
    return f"{city} 当前 22°C，晴。"


if __name__ == "__main__":
    mcp.run()                                    # 默认 stdio 传输
```
跑起来：
```plain text
python weather_server.py                          # 启动 stdio 服务
```
stdio 模式适合本地 IDE 用（Claude Desktop 会自己启动它）。如果要做远程服务，改一行：
```plain text
mcp.run(transport="http", host="0.0.0.0", port=8000)
```
之后客户端就能通过 `http://your-host:8000/mcp/\` 访问。
### 4.4 加多个工具：一个简易记事本
```plain text
# notes_server.py
from fastmcp import FastMCP
from pydantic import Field

mcp = FastMCP("notes-server")

_NOTES: list[str] = [ ]



@mcp.tool(name="add_note", description="添加一条笔记")
async def add_note(
    content: str = Field(description="笔记内容"),
) -> str:
    _NOTES.append(content)
    return f"已保存第 {len(_NOTES)} 条笔记。"


@mcp.tool(name="list_notes", description="列出所有笔记")
async def list_notes() -> list[str]:
    return _NOTES


@mcp.tool(name="clear_notes", description="清空全部笔记")
async def clear_notes() -> str:
    _NOTES.clear()
    return "已清空。"


if __name__ == "__main__":
    mcp.run(transport="http", host="0.0.0.0", port=8000)
```
到这里你已经能写出能用的 MCP Server 了。
### 4.5 写工具的几条经验
1. `name`** 和 **`description`** 要写清楚** —— LLM 选工具完全靠这两个字段判断。
2. **参数加 **`Field(description=...)` —— 帮 LLM 理解"这个参数填什么"。
3. **危险操作在 description 里标 **`⚠️` —— 例如删除、扣费等，便于客户端做拦截。
4. **返回值用字符串或简单结构** —— LLM 处理结构化 JSON 没问题，但太深的嵌套会增加幻觉。
5. **异步函数优先** —— `async def` 能让一个 server 同时处理多个请求。
---
## 五、写好的 MCP 怎么用
启动后，**有三种典型用法**，区别只在"谁来当客户端"。
### 5.1 三种方式速览
| 方式 | 客户端是谁 | 谁触发工具 | 适合场景 |
| --- | --- | --- | --- |
| **A. IDE / 桌面客户端** | Claude Desktop、Cursor、Cline、Claude Code、ChatGPT Desktop | 客户端内置的 LLM 自动选工具 | 个人使用，对话即工作 |
| **B. 程序化客户端** | 你写的 Python / Node 程序 | 你的代码 / 你的 Agent | 做产品、做后端、做自动化 |
| **C. 直接 HTTP** | curl / Postman / MCP Inspector | 人手动发请求 | 调试、协议验证 |
### 5.2 方式 A vs 方式 B 的本质区别
这两种方式最容易混淆，对比看就清楚了：
| 维度 | A. MCP 客户端（Claude Desktop / Cursor） | B. 程序化 MCP 客户端 |
| --- | --- | --- |
| **是谁在调工具** | LLM 自己根据对话内容决定调哪个工具、传什么参数 | 你写的代码显式`call_tool("xxx", {...})` |
| **使用场景** | 终端用户跟 LLM 聊天，LLM 自动用工具 | 自建 Agent / 后端服务 / 脚本 |
| **谁配 LLM** | 客户端内置（Claude/Cursor 自带模型） | 你自己接（OpenAI、Claude、Qwen、本地模型…甚至不接 LLM） |
| **是否要写代码** | 改 JSON 配置就行，不写代码 | 全程写代码 |
| **可控性** | LLM 黑盒决定 | 你完全控制何时调、怎么拼参数、怎么处理结果 |
| **典型产物** | "我在 Cursor 里聊天就能查数据" | "我的产品 Agent 里有这些能力" |
一句话：
- **方式 A** = 把 MCP 当成「Claude Desktop / Cursor 的插件」，让别人家的 LLM 用你的工具。
- **方式 B** = 把 MCP 当成「一个远程函数库」，让你自己的程序（不一定有 LLM）调用它。
两种用法**完全可以同时存在** —— 同一个 MCP Server 可以被 Claude Desktop、你的后端 Agent、curl 一起连，互不影响。
### 5.3 示例应用场景
| 场景 | 推荐方式 |
| --- | --- |
| 设计师在 Claude Desktop 让 AI 帮整理 Notion 笔记 | A |
| 程序员在 Cursor 让 AI 直接读项目数据库 | A |
| 自建客服机器人，需要查订单、查物流 | B |
| 运维做"自然语言查日志"的内部工具 | B |
| 写脚本批量调内部知识库做数据清洗 | B（甚至不接 LLM） |
| 验证一个新写的 MCP Server 接口对不对 | C |
---
## 六、方式 A：在 Claude Desktop / Cursor / Claude Code 里使用
### 6.1 Claude Desktop
编辑 `~/Library/Application Support/Claude/claude_desktop_config.json`（macOS）：
**stdio（本地启动）：**
```plain text
{
  "mcpServers": {
    "weather": {
      "command": "python",
      "args": ["/Users/me/projects/weather_server.py"]
    }
  }
}
```
**远程 HTTP：**
```plain text
{
  "mcpServers": {
    "notes": {
      "url": "http://127.0.0.1:8000/mcp/",
      "transport": "streamable-http"
    }
  }
}
```
重启 Claude Desktop，对话框右下角会出现工具图标，可以问"帮我看下东京天气"。
### 6.2 Cursor
打开 Settings → MCP → Add new MCP server，或编辑 `~/.cursor/mcp.json`：
```plain text
{
  "mcpServers": {
    "notes": {
      "url": "http://127.0.0.1:8000/mcp/"
    }
  }
}
```
Cursor 会自动识别 streamable-http。在 Cursor Chat 里直接说"帮我加一条笔记：明天提醒我开会"。
### 6.3 Claude Code
Claude Code 推荐用项目级 `.mcp.json`（提交到仓库可以让团队共享）：
```plain text
# 命令行添加
claude mcp add --transport http weather http://127.0.0.1:8000/mcp/

# 或手写 .mcp.json
```
`.mcp.json`：
```plain text
{
  "mcpServers": {
    "weather": {
      "type": "http",
      "url": "http://127.0.0.1:8000/mcp/"
    }
  }
}
```
进入 Claude Code 后用 `/mcp` 查看连接状态。
### 6.4 IDE / 桌面客户端最佳实践
| # | 实践 | 说明 |
| --- | --- | --- |
| 1 | **远程优先用 streamable-http**，本地工具用 stdio | SSE 已是 legacy；stdio 进程隔离最简单 |
| 2 | **危险工具开"每次确认"** | Claude Code 用`/permissions`，Cursor 在工具调用前会弹窗 |
| 3 | **密钥放 Server 端，不要进客户端配置** | 客户端只持 URL，便于轮换密钥 |
| 4 | **测试 / 生产用不同端口或不同前缀** | 避免误操作生产数据 |
| 5 | **项目级 **`.mcp.json`** 提交到仓库** | 团队同步配置，新人上手零成本 |
---
## 七、方式 B：在 Deep Agent / 自建后端中使用
### 7.1 最简方案：fastmcp 自带 Client
跑个自动化脚本只用 MCP 工具、不接 LLM 也行：
```plain text
import asyncio
from fastmcp import Client

async def main():
    async with Client("http://127.0.0.1:8000/mcp/") as client:
        tools = await client.list_tools()
        for t in tools:
            print(t.name, "-", t.description)

        result = await client.call_tool("get_weather", {"city": "上海"})
        print(result.content[0].text)

asyncio.run(main())
```
### 7.2 接 LangChain / LangGraph：`langchain-mcp-adapters`
把 MCP 工具批量转成 LangChain `BaseTool`，丢给任何 LangChain agent：
```plain text
pip install langchain-mcp-adapters langchain langgraph
```
```plain text
import asyncio
from langchain_mcp_adapters.client import MultiServerMCPClient

mcp_client = MultiServerMCPClient({
    "weather": {"url": "http://127.0.0.1:8000/mcp/", "transport": "streamable_http"},
    "notes":   {"url": "http://127.0.0.1:8001/mcp/", "transport": "streamable_http"},
})

tools = asyncio.run(mcp_client.get_tools())     # LangChain Tool 列表
```
### 7.3 Deep Agent 完整示例
[Deep Agents](https://github.com/langchain-ai/deepagents) 是 LangChain 出的"自带规划 + 子 agent + 文件系统"的 agent 框架。配合 MCP 工具的标准做法：
```plain text
import asyncio
from deepagents import create_deep_agent
from langchain_mcp_adapters.client import MultiServerMCPClient
from langgraph.checkpoint.memory import MemorySaver

# ① 进程级单例 —— 客户端持有长连接，不要每次请求 new
mcp_client = MultiServerMCPClient({
    "weather": {"url": "http://127.0.0.1:8000/mcp/", "transport": "streamable_http"},
    "notes":   {"url": "http://127.0.0.1:8001/mcp/", "transport": "streamable_http"},
})

async def build_agent():
    tools = await mcp_client.get_tools()

    # ② 工具按域分组，避免主 agent 选错
    weather_tools = [t for t in tools if t.name.startswith("get_weather")]
    notes_tools   = [t for t in tools if t.name in ("add_note", "list_notes", "clear_notes")]

    return create_deep_agent(
        model="claude-sonnet-4-5-20250929",

        tools=[ ],                              # 主 agent 只负责调度

        system_prompt="你是助理，按需要把任务委派给子 agent。",
        subagents=[
            {
                "name": "weather-bot",
                "description": "查询天气",
                "prompt": "你只负责查天气。",
                "tools": weather_tools,
            },
            {
                "name": "note-keeper",
                "description": "管理用户的笔记",
                "prompt": "你只管笔记的增删查。",
                "tools": notes_tools,
            },
        ],
        # ③ 危险工具走人工审批（必须配 checkpointer）
        interrupt_on={
            "clear_notes": True,               # 清空前问一句
        },
        checkpointer=MemorySaver(),
    )

async def main():
    agent = await build_agent()
    cfg = {"configurable": {"thread_id": "user-1"}}
    out = await agent.ainvoke(
        {"messages": [{"role": "user", "content": "记一条笔记：明早 9 点开会，再告诉我北京天气"}]},
        config=cfg,
    )
    print(out["messages"][-1].content)

asyncio.run(main())
```
### 7.4 Deep Agent + MCP 五条最佳实践
| # | 实践 | 原因 |
| --- | --- | --- |
| 1 | **传输统一用 **`streamable_http` | SSE 已是 legacy，新版 MCP 推荐 streamable-http |
| 2 | `MultiServerMCPClient`** 进程级复用** | 客户端持有长连接 + session，每次 new 会重复握手、堆 session |
| 3 | **危险工具必须 **`interrupt_on`** + **`checkpointer` | 没 checkpointer 时`interrupt_on` 不生效；HITL 是生产环境最低安全线 |
| 4 | **工具数 ≥ 10 个时拆 subagent** | 主 agent 选错率显著上升；按域拆，主 agent 用内置`task` 工具委派 |
| 5 | **领域知识放 SKILL.md，按需加载** | 比如"清空前必须二次确认"写成 skill，省 token 也防遗漏 |
### 7.5 在 FastAPI 里集成（生产部署）
```plain text
from contextlib import asynccontextmanager
from fastapi import FastAPI
from langchain_mcp_adapters.client import MultiServerMCPClient
from deepagents import create_deep_agent
from langgraph.checkpoint.memory import MemorySaver

mcp_client = MultiServerMCPClient({...})
state = {}

@asynccontextmanager
async def lifespan(app: FastAPI):
    tools = await mcp_client.get_tools()
    state["agent"] = create_deep_agent(
        model="claude-sonnet-4-5-20250929",
        tools=tools,
        checkpointer=MemorySaver(),
    )
    yield
    await mcp_client.close()

app = FastAPI(lifespan=lifespan)

@app.post("/chat")
async def chat(req: dict):
    out = await state["agent"].ainvoke(
        {"messages": [{"role": "user", "content": req["q"]}]},
        config={"configurable": {"thread_id": req["uid"]}},
    )
    return {"answer": out["messages"][-1].content}
```
---
## 八、方式 C：直接 HTTP 调用（调试用）
走标准 MCP JSON-RPC 三步握手：
```plain text
EP="http://127.0.0.1:8000/mcp/"

# ① initialize 拿 session id
SID=$(curl -sS -D - -o /dev/null \
  -X POST "$EP" \
  -H 'Content-Type: application/json' \
  -H 'Accept: application/json, text/event-stream' \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize",
       "params":{"protocolVersion":"2025-03-26","capabilities":{},
                 "clientInfo":{"name":"curl","version":"1"}}}' \
  | awk -v IGNORECASE=1 '/^mcp-session-id:/ {gsub(/\r/,""); print $2}')

# ② notifications/initialized
curl -sS -X POST "$EP" \
  -H 'Content-Type: application/json' \
  -H 'Accept: application/json, text/event-stream' \
  -H "Mcp-Session-Id: $SID" \
  -d '{"jsonrpc":"2.0","method":"notifications/initialized"}'

# ③ tools/list
curl -sS -X POST "$EP" \
  -H 'Content-Type: application/json' \
  -H 'Accept: application/json, text/event-stream' \
  -H "Mcp-Session-Id: $SID" \
  -d '{"jsonrpc":"2.0","id":2,"method":"tools/list"}'

# ④ tools/call
curl -sS -X POST "$EP" \
  -H 'Content-Type: application/json' \
  -H 'Accept: application/json, text/event-stream' \
  -H "Mcp-Session-Id: $SID" \
  -d '{"jsonrpc":"2.0","id":3,"method":"tools/call",
       "params":{"name":"get_weather","arguments":{"city":"北京"}}}'
```
调试推荐 **MCP Inspector**：`npx @modelcontextprotocol/inspector`，可视化点工具，比 curl 方便得多。
---
## 九、CLI + Skill vs MCP：什么时候选哪个
Claude Code 用户经常面临一个选择：**给 agent 写一个 SKILL.md 让它自己用 Bash 调命令，还是把命令包成 MCP Server？** 两条路都能走，但场景完全不同。
### 9.1 两条路径的工作方式
```plain text
路径 A：CLI + Skill                    路径 B：MCP Server
─────────────────────                  ──────────────────────
SKILL.md 教 LLM 怎么用                 server.py 写工具实现
       ↓                                       ↓
LLM 直接 Bash 调 git/gh/jq             LLM 通过协议调 typed tool
       ↓                                       ↓
解析文本输出（stdout）                  返回结构化 JSON 结果
```
### 9.2 全维度对比
| 维度 | CLI + Skill | MCP Server |
| --- | --- | --- |
| **搭建成本** | 写一个 markdown 即可，零代码 | 至少要写一个 server，再发布 / 部署 |
| **能力发现** | 靠 SKILL.md 文档描述 | `tools/list` 自动暴露名字 + schema |
| **参数校验** | 无 schema，靠提示词约束 | Pydantic / Zod 类型校验 |
| **跨客户端** | 仅限当前客户端（如 Claude Code） | 任意 MCP 客户端（CC / Cursor / Desktop / 自建） |
| **跨用户分享** | 拷贝 markdown 文件 | 发个 URL 或 npm 包 |
| **运行环境** | 本机 shell（受 LLM 当前权限） | server 进程（可远程、可隔离） |
| **密钥管理** | 容易散落到 SKILL / env | 集中放 server 端 |
| **状态保持** | 难（每次 Bash 是独立子进程） | 容易（server 长驻，可持有连接 / 缓存） |
| **错误处理** | 解析 stderr，混在文本里 | 结构化 error 返回 |
| **工具选择准确率** | 依赖 LLM 读文档判断 | LLM 看到 tool name + schema，更精准 |
| **观测 / 审计** | 散落在 shell history | 集中在 server 端日志 |
| **适合场景** | 个人提效、一次性任务、本地探索 | 多人共享、产品化、生产环境 |
### 9.3 决策清单
**写 Skill 当且仅当：**
- 工具已经是好用的 CLI（`git`、`gh`、`jq`、`kubectl`、`aws`、`docker`、`ffmpeg`...）
- 只你自己用、只在 Claude Code 里用
- 输出是给人看的文本，不是结构化数据
- 不想维护一个长跑进程
- 行为可能经常变（改 markdown 比改代码快）
**写 MCP 当：**
- 要在 Claude Desktop / Cursor / 多个 Agent 里共用
- 工具有复杂参数、严格类型
- 涉及私有 API、需要密钥集中管理
- 操作有状态（数据库连接、登录态、会话）
- 团队 / 客户也要用
- 高频调用，性能 / 鉴权要可控
### 9.4 同一件事的两种实现
**让 AI 操作 GitHub：**
| 路径 | 实现 |
| --- | --- |
| Skill | 写一份`~/.claude/skills/github-ops/SKILL.md`，告诉 LLM "查 PR 用 `gh pr list`，评论用 `gh pr comment`"，LLM 自己拼命令 |
| MCP | 装`@modelcontextprotocol/server-github`，所有客户端共用 |
**让 AI 查询本地 SQLite：**
| 路径 | 实现 |
| --- | --- |
| Skill | SKILL.md 里写"用`sqlite3 db.sqlite ''`，列名要带 `.headers on`"，LLM 自己拼 |
| MCP | 直接装`mcp-server-sqlite`，得到 `read_query` / `list_tables` 等结构化工具 |
### 9.5 强组合：MCP 提供能力 + Skill 引导用法
两者**并不互斥**，最强的组合是：
**底层封装成 MCP**（保证类型与跨客户端复用）
**上层写 SKILL.md** 教 LLM "什么时候该调哪个 tool、参数有什么约定、怎么解读结果"
特别适合工具数量多、领域知识重的场景。比如：
- MCP 暴露 `query_orders` / `cancel_order`
- SKILL.md 告诉 LLM："取消订单前必须先 `query_orders` 看状态，已发货的拒绝取消"
这样 MCP 负责"能做什么"，Skill 负责"该怎么做"，配合得当能显著提升 agent 表现。
---
## 十、速查表
| 我想做的事 | 选哪种方式 | 关键点 |
| --- | --- | --- |
| 在 IDE/桌面对话里用工具 | A | 配`.mcp.json` / `claude_desktop_config.json` |
| 自建产品后端，要 LLM + 工具 | B + Deep Agents | `MultiServerMCPClient` + `interrupt_on` |
| 写自动化脚本，不一定要 LLM | B + fastmcp Client | `Client(url).call_tool(...)` |
| 验证 server 接口 | C | curl 三步握手 / MCP Inspector |
| 给个人 Claude Code 加能力 | 优先 Skill；多客户端再上 MCP | 见第九节决策清单 |
| 找现成工具 | 看市场 | 见 3.4 节市场链接 |
| 换模型 / 换框架 | 任意 | MCP 协议不变，无需改 server |
| 加新工具 | 改 server | `@mcp.tool` 装饰新函数即可，自动出现在所有客户端 |
---
## 十一、常见坑
1. **stdio 模式下不能 **`print()`** 任何东西**
stdio 是协议通道，多余输出会破坏 JSON-RPC 帧。日志请用 `loguru` / `logging` 写到 stderr 或文件。
2. **多 fastmcp server 挂到同一 FastAPI 时，必须合并各自的 **`lifespan`
每个 fastmcp `http_app` 都有独立的 session manager 启动逻辑，只挂一个 lifespan 会导致其余 server 报 "Task group is not initialized"。
3. **streamable-http 必须保留响应头里的 **`Mcp-Session-Id`
后续所有请求都要回填，否则 server 找不到会话。
4. **Deep Agent 的 **`interrupt_on`** 没 **`checkpointer`** 不生效**
一定要同时传 `checkpointer=MemorySaver()` 之类的实例。
5. **不要把密钥写进客户端 **`mcp.json`
全部放 server 端环境变量，客户端只持 URL，便于轮换和审计。
6. **CLI 类 MCP 第一次启动慢**`npx -y` / `uvx` 第一次会下载，可能有几秒延迟；之后命中缓存就秒起。客户端如果有超时设置，注意调长。
---
## 十二、进一步阅读
**官方与协议**
- MCP 官方文档：https://modelcontextprotocol.io
- 协议规范：https://spec.modelcontextprotocol.io
- 官方 Server 集合：https://github.com/modelcontextprotocol/servers
**MCP 市场 / 目录**
- Awesome MCP Servers（社区精选）：https://github.com/punkpeye/awesome-mcp-servers
- Smithery（一键安装、托管）：https://smithery.ai
- Glama MCP Directory：https://glama.ai/mcp/servers
- Pulse MCP（带评分与分类）：https://www.pulsemcp.com
- mcp.so（中文友好）：https://mcp.so
- MCP Servers Directory：https://mcpservers.org
**框架与桥接**
- fastmcp：https://github.com/jlowin/fastmcp
- langchain-mcp-adapters：https://github.com/langchain-ai/langchain-mcp-adapters
- Deep Agents：https://github.com/langchain-ai/deepagents
**调试工具**
- MCP Inspector：`npx @modelcontextprotocol/inspector`
---
**结语**：MCP 真正的价值是**把"工具"从"框架/模型"里解耦出来**。今天写一个 fastmcp server，明天 Claude、Cursor、ChatGPT 都能用，后天换个模型也不用改一行工具代码。这才是值得投入的抽象。但也别为了 MCP 而 MCP —— **个人用、CLI 现成的、变化频繁的，先写 Skill；要分享、要类型、要状态、要鉴权的，才上 MCP。**