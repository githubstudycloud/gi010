# MCP Servers Collection

支持 Codex CLI、Claude Code、Gemini CLI 的 MCP 服务器合集，包含远程服务器和 Docker 部署方案。

## 目录

- [概述](#概述)
- [Codex CLI MCP](#codex-cli-mcp)
- [Claude Code MCP](#claude-code-mcp)
- [Gemini CLI MCP](#gemini-cli-mcp)
- [通用远程 MCP 服务器](#通用远程-mcp-服务器)
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

### Docker 相关
- [Docker MCP Toolkit](https://www.docker.com/blog/add-mcp-servers-to-claude-code-with-mcp-toolkit/)
- [Docker Hub MCP Server](https://www.docker.com/blog/introducing-docker-hub-mcp-server/)

### 教程和指南
- [Azure Container Apps 部署](https://techcommunity.microsoft.com/blog/appsonazureblog/host-remote-mcp-servers-in-azure-container-apps/4403550)
- [Google Cloud Run 部署](https://cloud.google.com/blog/topics/developers-practitioners/build-and-deploy-a-remote-mcp-server-to-google-cloud-run-in-under-10-minutes)
- [Northflank MCP 服务器部署](https://northflank.com/blog/how-to-build-and-deploy-a-model-context-protocol-mcp-server)

---

*最后更新: 2025-12-10*
