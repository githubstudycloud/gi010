# MCP Servers Collection

支持 Codex CLI、Claude Code、Gemini CLI 的 MCP 服务器合集，包含远程服务器和 Docker 部署方案。

## 目录

- [概述](#概述)
- [Codex CLI MCP](#codex-cli-mcp)
- [Claude Code MCP](#claude-code-mcp)
- [Gemini CLI MCP](#gemini-cli-mcp)
- [通用远程 MCP 服务器](#通用远程-mcp-服务器)
- [多 AI 协作编排](#多-ai-协作编排)
- [Claude Agent SDK 集成](#claude-agent-sdk-集成)
- [Docker 部署方案](#docker-部署方案)

---

## 概述

Model Context Protocol (MCP) 是一个开放标准协议，用于将 AI 模型连接到外部工具和数据源。支持的传输方式：

| 传输方式 | 说明 | 适用场景 |
|---------|------|---------|
| **stdio** | 标准输入输出 | 本地运行 |
| **SSE** | Server-Sent Events | 远程服务器（旧版） |
| **Streamable HTTP** | HTTP 流式传输 | 远程服务器（推荐） |

---

## Codex CLI MCP

### 官方支持

Codex CLI 内置 MCP 服务器支持，可以作为 MCP 服务器运行：

```bash
# 安装 Codex CLI
npm i -g @openai/codex

# 运行为 MCP 服务器
codex --mcp
```

配置文件位置：`~/.codex/config.toml`

```toml
# MCP 服务器配置示例
[mcp.servers.my-server]
command = "npx"
args = ["-y", "@modelcontextprotocol/server-filesystem", "/path/to/allowed/files"]
```

### 第三方 MCP 服务器

#### 1. tuannvm/codex-mcp-server
- **GitHub**: https://github.com/tuannvm/codex-mcp-server
- **功能**: 会话管理、模型选择、原生恢复支持
- **支持版本**: Codex CLI v0.50.0+

```bash
# 安装
npm install -g codex-mcp-server

# 运行
codex-mcp-server
```

#### 2. agency-ai-solutions/openai-codex-mcp
- **GitHub**: https://github.com/agency-ai-solutions/openai-codex-mcp
- **功能**: 让 Claude Code 调用 OpenAI Codex CLI
- **特点**: JSON-RPC 服务器包装

```bash
git clone https://github.com/agency-ai-solutions/openai-codex-mcp
cd openai-codex-mcp
npm install
npm start
```

---

## Claude Code MCP

### 官方远程 MCP 支持

Claude Code 原生支持远程 MCP 服务器（2025年6月18日起）。

配置方式：

```bash
# 添加 HTTP MCP 服务器
claude mcp add my-server --transport http --url https://your-mcp-server.com/mcp

# 添加 SSE MCP 服务器
claude mcp add my-server --transport sse --url https://your-mcp-server.com/sse
```

配置文件位置：`~/.claude/settings.json`

```json
{
  "mcpServers": {
    "my-remote-server": {
      "transport": "http",
      "url": "https://your-mcp-server.com/mcp"
    }
  }
}
```

### Docker MCP Toolkit

Docker Desktop 4.40+ 内置 MCP Toolkit，提供 200+ 预配置 MCP 服务器：

```bash
# 在 Docker Desktop 中一键启用
# Settings > Features in Development > Enable MCP Toolkit

# 配置文件添加 Docker MCP Gateway
{
  "mcpServers": {
    "docker": {
      "command": "docker",
      "args": ["mcp", "gateway", "run"]
    }
  }
}
```

### 推荐的 Claude Code MCP 服务器

#### 1. QuantGeekDev/docker-mcp
- **GitHub**: https://github.com/QuantGeekDev/docker-mcp
- **功能**: Docker 容器和 compose stack 管理

```bash
# Docker 运行
docker run -v /var/run/docker.sock:/var/run/docker.sock \
  ghcr.io/quantgeekdev/docker-mcp:latest
```

#### 2. majkonautic/NOVA_claude-code-mcp-guide
- **GitHub**: https://github.com/majkonautic/NOVA_claude-code-mcp-guide
- **功能**: 完整的 Claude Code MCP 集成指南
- **特点**: HTTP Bridge 云服务支持，本地 Docker MCP 模板

---

## Gemini CLI MCP

### 官方支持

Gemini CLI 内置 MCP 支持：

```bash
# 安装 Gemini CLI
npm install -g @anthropic-ai/gemini-cli

# 或使用 Homebrew
brew install gemini-cli
```

配置文件位置：`~/.gemini/settings.json`

```json
{
  "mcpServers": {
    "MCP_DOCKER": {
      "command": "docker",
      "args": ["mcp", "gateway", "run"],
      "env": {}
    },
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/path/to/files"]
    }
  }
}
```

### Gemini MCP 服务器项目

#### 1. vytautas-bunevicius/gemini-mcp-server
- **GitHub**: https://github.com/vytautas-bunevicius/gemini-mcp-server
- **功能**: Gemini API 交互
- **支持**: Docker 部署

```bash
# Docker 运行 (stdio)
docker run -i --rm \
  -e GEMINI_API_KEY=YOUR_API_KEY \
  gemini-mcp-server

# Docker 运行 (网络模式)
docker run -it --rm \
  -e GEMINI_API_KEY=YOUR_API_KEY \
  -p 127.0.0.1:3001:3001 \
  --name gemini-mcp \
  gemini-mcp-server
```

#### 2. bar0n-gemini-mcp (LobeHub)
- **地址**: https://lobehub.com/mcp/bar0n-gemini-mcp
- **功能**: Gemini 模型集成

---

## 通用远程 MCP 服务器

### 支持多客户端的 MCP 服务器

#### 1. Puppeteer MCP Server (远程 SSE)
- **GitHub**: https://github.com/sultannaufal/puppeteer-mcp-server
- **功能**: 浏览器自动化，16 种 Puppeteer 工具
- **传输**: Streamable HTTP + SSE
- **安全**: API Key 认证

```bash
# Docker 部署
docker pull sultannaufal/puppeteer-mcp-server
docker run -d \
  -p 3000:3000 \
  -e API_KEY=your-secret-key \
  sultannaufal/puppeteer-mcp-server
```

#### 2. Sequential Thinking MCP
- **功能**: 复杂问题分步推理
- **支持**: Codex CLI, Claude Code, Gemini CLI

#### 3. Filesystem MCP Server
- **官方**: @modelcontextprotocol/server-filesystem
- **功能**: 文件系统访问

```bash
npx -y @modelcontextprotocol/server-filesystem /allowed/path
```

#### 4. GitHub MCP Server
- **官方**: @modelcontextprotocol/server-github
- **功能**: GitHub API 操作

```bash
npx -y @modelcontextprotocol/server-github
```

---

## 多 AI 协作编排

### 核心理念

通过 MCP 服务器，可以让 Claude Code、Codex CLI、Gemini CLI 协同工作，各取所长：

- **Claude Code**: 强大的代码理解和生成，作为主编排器
- **Codex CLI**: OpenAI 模型的深度代码优化
- **Gemini CLI**: 大上下文窗口，适合分析大型代码库

### 🌟 推荐方案：PAL MCP Server

**PAL (Provider Abstraction Layer)** 是最强大的多 AI 协作 MCP 服务器：

- **GitHub**: https://github.com/BeehiveInnovations/pal-mcp-server
- **支持客户端**: Claude Code, Gemini CLI, Codex CLI, Cursor, VS Code
- **功能**: 在单个提示中使用多个模型

```bash
# 安装 PAL MCP Server
git clone https://github.com/BeehiveInnovations/pal-mcp-server
cd pal-mcp-server
npm install

# 配置 API Keys
cp .env.example .env
# 编辑 .env 添加各模型 API Key
```

Claude Code 配置 (`~/.claude/settings.json`):

```json
{
  "mcpServers": {
    "pal": {
      "command": "node",
      "args": ["/path/to/pal-mcp-server/index.js"],
      "env": {
        "OPENAI_API_KEY": "your-openai-key",
        "GOOGLE_API_KEY": "your-google-key",
        "ANTHROPIC_API_KEY": "your-anthropic-key"
      }
    }
  }
}
```

### 其他协作方案

#### 1. Gemini MCP (让 Claude 调用 Gemini)
- **GitHub**: https://github.com/RLabs-Inc/gemini-mcp
- **功能**: 直接查询、协作头脑风暴、代码分析

```json
{
  "mcpServers": {
    "gemini": {
      "command": "npx",
      "args": ["-y", "gemini-mcp"],
      "env": {
        "GOOGLE_API_KEY": "your-google-api-key"
      }
    }
  }
}
```

#### 2. Claude-Gemini 协作集成
- **LobeHub**: https://lobehub.com/mcp/jamiewonderchild-claude-gemini-mcp
- **功能**: Claude 和 Gemini 实时协作讨论

#### 3. Collaborative MCP Proxy
- **LobeHub**: https://lobehub.com/mcp/junteakim-collaborative-mcp-proxy
- **特点**: 无需 API Key，使用现有 CLI 登录配置
- **集成**: Ollama, Gemini CLI, Codex CLI

#### 4. Claude-Flow (企业级编排平台)
- **GitHub**: https://github.com/ruvnet/claude-flow
- **功能**:
  - 多 Agent Swarm 智能
  - 87+ 专用 MCP 工具
  - 持久化内存
  - 分布式工作流

```bash
# 安装 Claude-Flow
npm install -g claude-flow

# 初始化
claude-flow init

# 启动 swarm
claude-flow swarm start --agents 5
```

### 协作工作流示例

#### 示例 1: 代码审查流水线

```
Claude Code (编排器)
    ├── Codex CLI → 代码安全分析
    ├── Gemini → 大上下文代码理解
    └── Claude → 最终审查和建议
```

#### 示例 2: 多模型代码生成

```python
# 在 Claude Code 中使用 PAL MCP
# Claude 作为主编排器，调用其他模型

# 1. 让 Gemini 分析现有代码库（大上下文优势）
# 使用 MCP 工具: pal_analyze --model gemini --scope full

# 2. 让 Codex 生成优化建议
# 使用 MCP 工具: pal_generate --model codex --task optimize

# 3. Claude 整合并实施
```

---

## Claude Agent SDK 集成

### 概述

Claude Agent SDK（原 Claude Code SDK）提供一流的 MCP 支持，可以在 Python 应用中直接集成 MCP 服务器。

- **GitHub**: https://github.com/anthropics/claude-agent-sdk-python
- **文档**: https://docs.claude.com/en/docs/agent-sdk/mcp

### 安装

```bash
pip install claude-agent-sdk
```

### MCP 服务器集成方式

#### 方式 1: 项目配置文件 (`.mcp.json`)

```json
{
  "servers": {
    "codex": {
      "command": "codex",
      "args": ["--mcp"]
    },
    "gemini": {
      "command": "npx",
      "args": ["-y", "gemini-mcp"],
      "env": {
        "GOOGLE_API_KEY": "${GOOGLE_API_KEY}"
      }
    },
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "./workspace"]
    }
  }
}
```

#### 方式 2: Python 代码中集成

```python
from claude_agent_sdk import Agent, MCPServer

# 定义 MCP 服务器
codex_server = MCPServer(
    name="codex",
    command="codex",
    args=["--mcp"]
)

gemini_server = MCPServer(
    name="gemini",
    command="npx",
    args=["-y", "gemini-mcp"],
    env={"GOOGLE_API_KEY": os.getenv("GOOGLE_API_KEY")}
)

# 创建 Agent 并添加 MCP 服务器
agent = Agent(
    model="claude-sonnet-4-20250514",
    mcp_servers=[codex_server, gemini_server]
)

# 运行 Agent
result = agent.run("分析这段代码并优化性能")
```

#### 方式 3: 自定义工具（进程内 MCP）

```python
from claude_agent_sdk import Agent, tool

@tool
def call_codex(prompt: str, model: str = "gpt-5-codex") -> str:
    """调用 OpenAI Codex 进行代码分析"""
    # 实现 Codex API 调用
    import subprocess
    result = subprocess.run(
        ["codex", "query", "--model", model, prompt],
        capture_output=True, text=True
    )
    return result.stdout

@tool
def call_gemini(prompt: str, context: str = "") -> str:
    """调用 Gemini 进行大上下文分析"""
    import google.generativeai as genai
    model = genai.GenerativeModel('gemini-2.5-pro')
    response = model.generate_content(f"{context}\n\n{prompt}")
    return response.text

agent = Agent(
    model="claude-sonnet-4-20250514",
    tools=[call_codex, call_gemini]
)
```

### 多 Agent 工作流

```python
from claude_agent_sdk import Agent, Workflow

# 定义专门的子 Agent
analyst = Agent(
    name="analyst",
    model="claude-sonnet-4-20250514",
    system_prompt="你是代码分析专家"
)

implementer = Agent(
    name="implementer",
    model="claude-sonnet-4-20250514",
    system_prompt="你是代码实现专家"
)

reviewer = Agent(
    name="reviewer",
    model="claude-sonnet-4-20250514",
    system_prompt="你是代码审查专家"
)

# 创建工作流
workflow = Workflow(
    agents=[analyst, implementer, reviewer],
    pipeline=[
        ("analyst", "分析需求和现有代码"),
        ("implementer", "实现功能"),
        ("reviewer", "审查代码质量")
    ]
)

result = workflow.run("添加用户认证功能")
```

### 最佳实践

1. **最小权限原则**: MCP 工具调用使用最小必要权限
2. **健康检查**: 对远程 MCP 端点添加健康检查
3. **重试机制**: 添加重试和退避策略
4. **日志记录**: 记录所有 MCP 工具调用

```python
from claude_agent_sdk import Agent, MCPServer

agent = Agent(
    model="claude-sonnet-4-20250514",
    mcp_servers=[...],
    config={
        "mcp_timeout": 30,
        "mcp_retries": 3,
        "mcp_health_check": True,
        "tool_call_logging": True
    }
)
```

---

## Docker 部署方案

### 方案一：Docker MCP Gateway（推荐）

Docker MCP Gateway 提供统一的网关管理多个 MCP 服务器：

```yaml
# docker-compose.yml
version: '3.8'
services:
  mcp-gateway:
    image: docker/mcp-gateway:latest
    ports:
      - "8080:8080"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    environment:
      - MCP_CONFIG=/config/mcp.json
    volumes:
      - ./mcp-config.json:/config/mcp.json
```

客户端配置连接 Gateway：

```json
{
  "mcpServers": {
    "docker-gateway": {
      "transport": "sse",
      "url": "http://localhost:8080/sse"
    }
  }
}
```

### 方案二：独立 MCP 服务器容器

```dockerfile
# Dockerfile.mcp-server
FROM node:20-alpine

WORKDIR /app
COPY package*.json ./
RUN npm install

COPY . .

EXPOSE 3000

CMD ["node", "server.js"]
```

```yaml
# docker-compose.yml
version: '3.8'
services:
  filesystem-mcp:
    build: ./filesystem-mcp
    ports:
      - "3001:3000"
    volumes:
      - ./workspace:/workspace:ro

  github-mcp:
    build: ./github-mcp
    ports:
      - "3002:3000"
    environment:
      - GITHUB_TOKEN=${GITHUB_TOKEN}

  puppeteer-mcp:
    image: sultannaufal/puppeteer-mcp-server
    ports:
      - "3003:3000"
    environment:
      - API_KEY=${MCP_API_KEY}
```

### 方案三：云端部署

#### Azure Container Apps

```bash
# 部署到 Azure
az containerapp create \
  --name mcp-server \
  --resource-group myResourceGroup \
  --environment myContainerAppEnv \
  --image your-registry/mcp-server:latest \
  --target-port 3000 \
  --ingress external
```

#### Google Cloud Run

```bash
# 部署到 Cloud Run
gcloud run deploy mcp-server \
  --image gcr.io/your-project/mcp-server:latest \
  --platform managed \
  --allow-unauthenticated  # 或 --no-allow-unauthenticated 需要认证
```

---

## 快速对比表

| 特性 | Codex CLI | Claude Code | Gemini CLI |
|-----|-----------|-------------|------------|
| MCP 原生支持 | ✅ | ✅ | ✅ |
| 远程 MCP (HTTP/SSE) | ✅ | ✅ | ✅ |
| Docker MCP Toolkit | ✅ | ✅ | ✅ |
| 配置文件 | `~/.codex/config.toml` | `~/.claude/settings.json` | `~/.gemini/settings.json` |
| 添加命令 | `codex mcp add` | `claude mcp add` | 手动编辑配置 |

---

## 参考链接

### 官方文档
- [Codex CLI MCP 文档](https://developers.openai.com/codex/mcp/)
- [Claude Code MCP 文档](https://docs.anthropic.com/en/docs/claude-code/mcp)
- [Gemini CLI MCP 文档](https://google-gemini.github.io/gemini-cli/docs/tools/mcp-server.html)
- [Claude Agent SDK 文档](https://docs.claude.com/en/docs/agent-sdk/mcp)

### 多 AI 协作
- [PAL MCP Server](https://github.com/BeehiveInnovations/pal-mcp-server) - 多模型协作首选
- [Claude-Flow](https://github.com/ruvnet/claude-flow) - 企业级 Agent 编排
- [Gemini MCP for Claude](https://github.com/RLabs-Inc/gemini-mcp) - Claude 调用 Gemini
- [wshobson/agents](https://github.com/wshobson/agents) - 85+ 专用 Agent 和 47 技能
- [Claude Code 多模型集成指南](https://lgallardo.com/2025/09/06/claude-code-supercharged-mcp-integration/)

### Docker 相关
- [Docker MCP Toolkit](https://www.docker.com/blog/add-mcp-servers-to-claude-code-with-mcp-toolkit/)
- [Docker Hub MCP Server](https://www.docker.com/blog/introducing-docker-hub-mcp-server/)
- [Gemini CLI + Docker MCP](https://www.ajeetraina.com/how-to-setup-gemini-cli-docker-mcp-toolkit-for-ai-assisted-development/)

### 教程和指南
- [Azure Container Apps 部署](https://techcommunity.microsoft.com/blog/appsonazureblog/host-remote-mcp-servers-in-azure-container-apps/4403550)
- [Google Cloud Run 部署](https://cloud.google.com/blog/topics/developers-practitioners/build-and-deploy-a-remote-mcp-server-to-google-cloud-run-in-under-10-minutes)
- [Northflank MCP 服务器部署](https://northflank.com/blog/how-to-build-and-deploy-a-model-context-protocol-mcp-server)
- [Claude Agent SDK 最佳实践](https://skywork.ai/blog/claude-agent-sdk-best-practices-ai-agents-2025/)
- [Building agents with Claude Agent SDK](https://www.anthropic.com/engineering/building-agents-with-the-claude-agent-sdk)

---

*最后更新: 2025-12-10*
