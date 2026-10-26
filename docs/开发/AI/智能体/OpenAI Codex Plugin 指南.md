# OpenAI Codex Plugin 指南
#
## 概述
Agent Plugins 1.0 是 OpenAI 联合 Microsoft、Google、Amazon、Cursor 等推出的开放标准，用于将 AI Agent 的能力打包成可移植、可安装的扩展包。
一个 Plugin 的目录结构：
```plain text
my-plugin/
├── plugin.json      # 清单文件（名称、版本、描述）
├── skills/          # 技能文件夹（SKILL.md 文件）
└── mcp.json         # MCP 服务器配置（可选）
```
**Plugin = Skills + MCP Servers 的容器。**
---
## Skills vs MCP Server 核心区别
|  | **Skills（技能）** | **MCP Server** |
| --- | --- | --- |
| 本质 | 静态知识/指令文档（Markdown） | 运行中的服务进程，暴露可调用的工具 |
| 形式 | `SKILL.md` 文件 | 通过 MCP 协议通信的 server 进程 |
| 能力 | 给 AI 提供**领域知识和指导**（"知道怎么做"） | 给 AI 提供**可执行的动作**（"能够去做"） |
| 交互方式 | AI 读取并遵循指令 | AI 通过协议调用 tool，获取结构化结果 |
| 运行时开销 | 无（纯文本） | 需要启动进程、维护连接 |
| 类比 | 操作手册 | 工具箱 |
---
## 实例
### 实例 1：纯 Skill 插件 — AWS 部署指南
```plain text
aws-deploy-plugin/
├── plugin.json
└── skills/
    └── deploy-to-ecs/
        └── SKILL.md
```
**plugin.json**
```json
{
  "name": "aws-deploy",
  "version": "1.0.0",
  "description": "AWS ECS deployment best practices"
}
```
**skills/deploy-to-ecs/SKILL.md**
```markdown
---
name: deploy-to-ecs
description: Guide for deploying services to AWS ECS
---

## Steps

1. Build Docker image: `docker build -t myapp .`
2. Push to ECR: `aws ecr get-login-password | docker login ...`
3. Update task definition in `task-def.json`
4. Deploy: `aws ecs update-service --cluster prod --service myapp --force-new-deployment`

## Notes

- Always use immutable image tags (git SHA, not `latest`)
- Health check must pass within 60s or rollback triggers
- Use `--capacity-provider-strategy` for Fargate Spot
```
> AI 读取这个 Skill 后，知道该按什么步骤部署，但它需要通过 shell 命令来实际执行。
---
### 实例 2：纯 MCP Server 插件 — 数据库查询工具
```plain text
db-query-plugin/
├── plugin.json
└── mcp.json
```
**plugin.json**
```json
{
  "name": "db-query",
  "version": "1.0.0",
  "description": "Query production database safely"
}
```
**mcp.json**
```json
{
  "servers": {
    "postgres": {
      "command": "npx",
      "args": ["@mcp/postgres-server", "--connection", "postgresql://readonly@prod-db:5432/main"],
      "transport": "stdio"
    }
  }
}
```
> AI 可以直接调用 MCP server 提供的 `query` tool 来执行 SQL，无需手动拼命令。
---
### 实例 3：Skill + MCP Server 组合 — Jira 项目管理
```plain text
jira-plugin/
├── plugin.json
├── skills/
│   └── jira-workflow/
│       └── SKILL.md
└── mcp.json
```
**plugin.json**
```json
{
  "name": "jira-project-mgmt",
  "version": "1.0.0",
  "description": "Manage Jira issues with team workflow knowledge"
}
```
**skills/jira-workflow/SKILL.md**
```markdown
---
name: jira-workflow
description: Team's Jira workflow conventions
---

## Issue Types

- `bug`: Production issues, must include reproduction steps
- `story`: Feature work, must link to design doc
- `task`: Technical debt, must have size estimate (S/M/L)

## Workflow Rules

- Bug Priority: P0 = 4h response, P1 = 1 day, P2 = sprint
- All stories must have acceptance criteria before moving to "In Progress"
- Use label `needs-review` when PR is ready

## Transitions

- To Do → In Progress: assign to yourself first
- In Progress → In Review: link the PR
- In Review → Done: requires 2 approvals
```
**mcp.json**
```json
{
  "servers": {
    "jira": {
      "command": "npx",
      "args": ["@mcp/jira-server", "--site", "myteam.atlassian.net"],
      "transport": "stdio"
    }
  }
}
```
> Skill 告诉 AI 团队的工作流规范（什么时候该设什么状态、标签），MCP Server 让 AI 能直接调用 Jira API 创建/更新 issue。两者结合 = AI 既知道规则，又能执行操作。
---
### 实例 4：远程 MCP Server — Web 搜索能力
```plain text
web-search-plugin/
├── plugin.json
└── mcp.json
```
**mcp.json**
```json
{
  "servers": {
    "web-search": {
      "url": "https://api.search-provider.com/mcp",
      "transport": "streamable-http",
      "headers": {
        "Authorization": "Bearer ${SEARCH_API_KEY}"
      }
    }
  }
}
```
> 与实例 2 的区别：这是一个**远程** MCP Server（通过 HTTP），不需要在本地启动进程。
---
## 总结
```plain text
┌─────────────────────────────────────┐
│            Plugin (容器)             │
├──────────────────┬──────────────────┤
│   Skills (知识)   │  MCP Server (能力) │
├──────────────────┼──────────────────┤
│ "怎么做"          │ "去做"             │
│ 静态 Markdown     │ 运行时进程/服务     │
│ 零开销            │ 需要资源            │
│ 指导 AI 的行为    │ 扩展 AI 的工具集    │
└──────────────────┴──────────────────┘
```
- **只需要知识指导** → 用 Skill
- **需要调用外部系统** → 用 MCP Server
- **既要知道规则又要能操作** → 两者组合打包为 Plugin