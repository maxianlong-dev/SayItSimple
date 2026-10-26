# MCP HTTP 鉴权教程：从固定 Header 到自定义 OAuth
本文用一个可在本地运行的最小示例，说明下面两种 MCP HTTP 鉴权方式的区别，并从零模拟 OAuth 2.0 Client Credentials 流程：
```python
# 固定凭证
"headers": {"Authorization": "Bearer YOUR_TOKEN"}

# 动态鉴权逻辑
"auth": auth  # httpx.Auth 实例
```
示例包含三个角色：
```plain text
OAuth 客户端 ──client_id/client_secret──> 模拟授权服务器
      │                                  │
      │<────────短期 access_token─────────┘
      │
      └──Bearer access_token────────────> 受保护的 MCP 风格服务
```
> 为了聚焦鉴权，受保护接口只模拟 MCP 的 HTTP 入口，并没有实现 MCP JSON-RPC 协议。最后一节会展示如何把同一个 `httpx.Auth` 对象接入真正的 `MultiServerMCPClient`。
## 1. 两种方式怎么选
`headers` 只是给每次请求附加固定值。客户端不知道 token 何时过期，也不会获取或刷新 token：
```python
"headers": {
    "Authorization": "Bearer fixed-token",
    "X-Trace-ID": "demo-001",
}
```
`auth` 则是一段可执行的认证逻辑。它可以在发送请求之前获取 token，在 token 即将过期时刷新，并在服务器返回 `401` 后重试：
```python
"auth": OAuthClientCredentialsAuth(...)
```
经验法则：已有且长期有效的 API Key 使用 `headers`；涉及 OAuth、短期 token、自动刷新或逐请求签名时使用 `auth`。
## 2. 创建项目
创建一个空目录，并准备虚拟环境：
```bash
mkdir mcp-oauth-demo
cd mcp-oauth-demo
python -m venv .venv
source .venv/bin/activate
pip install fastapi 'uvicorn[standard]' httpx
```
项目最终包含：
```plain text
mcp-oauth-demo/
├── auth_server.py
├── resource_server.py
└── client.py
```
## 3. 实现模拟 OAuth 授权服务器
新建 `auth_server.py`：
```python
import secrets
import time
from urllib.parse import parse_qs

from fastapi import FastAPI, HTTPException, Request

app = FastAPI(title="Mock OAuth Authorization Server")

CLIENTS = {
    "demo-client": "demo-secret",
}

# 演示环境中，资源服务器和授权服务器会在同一进程内共享该字典。
TOKENS: dict[str, float] = {}
TOKEN_TTL_SECONDS = 5


@app.post("/oauth/token")
async def issue_token(request: Request):
    # OAuth token endpoint 通常接收 application/x-www-form-urlencoded。
    form = parse_qs((await request.body()).decode())
    grant_type = form.get("grant_type", [""])[0]
    client_id = form.get("client_id", [""])[0]
    client_secret = form.get("client_secret", [""])[0]

    if grant_type != "client_credentials":
        raise HTTPException(status_code=400, detail="unsupported_grant_type")

    if CLIENTS.get(client_id) != client_secret:
        raise HTTPException(status_code=401, detail="invalid_client")

    access_token = secrets.token_urlsafe(24)
    expires_at = time.time() + TOKEN_TTL_SECONDS
    TOKENS[access_token] = expires_at

    return {
        "access_token": access_token,
        "token_type": "Bearer",
        "expires_in": TOKEN_TTL_SECONDS,
    }
```
这里模拟了 OAuth 2.0 的 Client Credentials 授权：客户端用自己的身份换取 access token，不涉及最终用户或浏览器跳转。为了便于观察刷新行为，token 只有 5 秒有效期。
## 4. 实现受保护的资源接口
将受保护接口追加到同一个 `auth_server.py`，以便这个纯内存示例共享 token 状态：
```python
from fastapi import Header


def validate_bearer(authorization: str | None) -> str:
    if not authorization or not authorization.startswith("Bearer "):
        raise HTTPException(status_code=401, detail="missing_bearer_token")

    token = authorization.removeprefix("Bearer ")
    expires_at = TOKENS.get(token)

    if expires_at is None or time.time() >= expires_at:
        raise HTTPException(status_code=401, detail="invalid_or_expired_token")

    return token


@app.get("/mcp")
async def protected_mcp_endpoint(authorization: str | None = Header(default=None)):
    validate_bearer(authorization)
    return {
        "ok": True,
        "message": "你已经通过 OAuth access token 访问受保护资源",
    }
```
此时 `auth_server.py` 同时扮演授权服务器和资源服务器。真实系统通常会把两者拆开，并通过 JWT 签名验证或 token introspection 验证 token，而不是共享内存字典。
启动服务：
```bash
uvicorn auth_server:app --port 8000 --reload
```
## 5. 先观察固定 Header 的局限
手动申请 token：
```bash
curl -s -X POST http://127.0.0.1:8000/oauth/token \
  -H 'Content-Type: application/x-www-form-urlencoded' \
  --data 'grant_type=client_credentials&client_id=demo-client&client_secret=demo-secret'
```
响应类似：
```json
{
  "access_token": "临时生成的字符串",
  "token_type": "Bearer",
  "expires_in": 5
}
```
复制 `access_token` 后访问资源：
```bash
curl http://127.0.0.1:8000/mcp \
  -H 'Authorization: Bearer 复制到这里的_token'
```
5 秒内请求会成功；等待 token 过期后，相同 Header 会得到 `401`。这正是固定 `headers` 无法自行解决的问题。
## 6. 用 `httpx.Auth` 自动获取和刷新 token
新建 `client.py`：
```python
import asyncio
import time

import httpx


class OAuthClientCredentialsAuth(httpx.Auth):
    """用于异步 httpx 客户端的最小 Client Credentials 实现。"""

    def __init__(self, token_url: str, client_id: str, client_secret: str):
        self.token_url = token_url
        self.client_id = client_id
        self.client_secret = client_secret
        self.access_token: str | None = None
        self.expires_at = 0.0
        self._lock = asyncio.Lock()

    async def _refresh_token(self) -> None:
        # 使用独立客户端，避免申请 token 的请求再次触发当前 Auth。
        async with httpx.AsyncClient() as token_client:
            response = await token_client.post(
                self.token_url,
                data={
                    "grant_type": "client_credentials",
                    "client_id": self.client_id,
                    "client_secret": self.client_secret,
                },
            )
            response.raise_for_status()
            payload = response.json()

        self.access_token = payload["access_token"]
        # 提前 1 秒视为过期，减少 token 在网络传输过程中失效的概率。
        self.expires_at = time.monotonic() + payload["expires_in"] - 1
        print("取得了新的 access token")

    async def _ensure_token(self, force: bool = False) -> None:
        async with self._lock:
            expired = time.monotonic() >= self.expires_at
            if force or self.access_token is None or expired:
                await self._refresh_token()

    async def async_auth_flow(self, request: httpx.Request):
        await self._ensure_token()
        request.headers["Authorization"] = f"Bearer {self.access_token}"
        response = yield request

        # 服务端仍可能提前撤销 token。收到 401 时强制刷新并重试一次。
        if response.status_code == 401:
            await self._ensure_token(force=True)
            request.headers["Authorization"] = f"Bearer {self.access_token}"
            yield request


async def main():
    auth = OAuthClientCredentialsAuth(
        token_url="http://127.0.0.1:8000/oauth/token",
        client_id="demo-client",
        client_secret="demo-secret",
    )

    async with httpx.AsyncClient(auth=auth) as client:
        first = await client.get("http://127.0.0.1:8000/mcp")
        print("第一次请求：", first.status_code, first.json())

        print("等待 token 过期……")
        await asyncio.sleep(6)

        second = await client.get("http://127.0.0.1:8000/mcp")
        print("第二次请求：", second.status_code, second.json())


if __name__ == "__main__":
    asyncio.run(main())
```
运行：
```bash
python client.py
```
预期输出：
```plain text
取得了新的 access token
第一次请求： 200 {'ok': True, ...}
等待 token 过期……
取得了新的 access token
第二次请求： 200 {'ok': True, ...}
```
认证对象完成了四件事：
1. 第一次请求前，通过 `client_id` 和 `client_secret` 获取 token。
2. 给业务请求添加 `Authorization: Bearer ...`。
3. 缓存 token，并根据 `expires_in` 判断何时刷新。
4. 收到 `401` 时强制刷新，并将请求重试一次。
## 7. 接入 `MultiServerMCPClient`
当 `http://127.0.0.1:8000/mcp` 换成真正实现 MCP 协议的 Streamable HTTP 服务后，可以直接复用认证对象：
```python
from langchain.agents import create_agent
from langchain_mcp_adapters.client import MultiServerMCPClient

auth = OAuthClientCredentialsAuth(
    token_url="https://auth.example.com/oauth/token",
    client_id="your-client-id",
    client_secret="your-client-secret",
)

client = MultiServerMCPClient(
    {
        "weather": {
            "transport": "http",
            "url": "https://mcp.example.com/mcp",
            "auth": auth,
            "headers": {
                "X-Trace-ID": "demo-001",
            },
        }
    }
)

tools = await client.get_tools()
agent = create_agent("openai:gpt-5.5", tools)
response = await agent.ainvoke(
    {"messages": "what is the weather in nyc?"}
)
```
这里可以同时使用两者，但职责应分开：
- `auth` 管理动态的 `Authorization`。
- `headers` 传递固定的追踪信息或业务元数据。
- 不要同时在两处设置 `Authorization`，否则容易产生覆盖和排查困难。
## 8. 这个示例省略了什么
这是教学用模拟，不应直接用于生产环境。生产实现还需考虑：
- 使用 HTTPS，绝不通过明文 HTTP 传输 client secret 或 token。
- 从密钥管理系统或环境变量读取凭证，不将其提交到代码仓库。
- 使用成熟 OAuth/OIDC 服务，不自己实现密码学和完整协议。
- 校验 token 的签名、`issuer`、`audience`、`scope` 和有效期。
- 并发刷新去重、超时、退避重试和错误分类。
- 服务端撤销 token，以及客户端安全销毁缓存凭证。
- 多用户场景下按用户隔离 token，避免跨租户复用认证对象。
如果是需要用户登录和授权的 MCP 应用，应采用 Authorization Code + PKCE，而不是本教程的 Client Credentials。Client Credentials 表示“应用以自己的身份访问服务”，并不代表某个最终用户。
## 9. 小结
固定 `headers` 的本质是“我已经有凭证，请原样带上”；自定义 `httpx.Auth` 的本质是“请执行一套凭证生命周期管理流程”。OAuth 的价值不只是设置一个 Bearer Header，而是安全地签发、缓存、刷新和撤销短期凭证。