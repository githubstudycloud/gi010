# 内网 MCP 服务器部署指南

面向企业内网环境的 MCP 服务器部署方案，支持自定义 API 大模型和团队共享。

## 目录

- [架构概览](#架构概览)
- [为什么这样设计](#为什么这样设计)
- [怎么做](#怎么做)
- [最佳实践方案](#最佳实践方案)
- [快速开始](#快速开始)

---

## 架构概览

### 流程图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              内网环境                                        │
│                                                                             │
│  ┌──────────────┐   ┌──────────────┐   ┌──────────────┐                    │
│  │  开发者 A     │   │  开发者 B     │   │  开发者 C     │                    │
│  │  Claude Code │   │  Codex CLI   │   │  Gemini CLI  │                    │
│  └──────┬───────┘   └──────┬───────┘   └──────┬───────┘                    │
│         │                  │                  │                             │
│         └────────────┬─────┴─────┬────────────┘                             │
│                      │           │                                          │
│                      ▼           ▼                                          │
│         ┌────────────────────────────────────┐                              │
│         │     内网 MCP Gateway (Docker)       │                              │
│         │     http://192.168.x.x:8080        │                              │
│         │                                    │                              │
│         │  ┌─────────┐  ┌─────────┐         │                              │
│         │  │ SSE端点  │  │ HTTP端点 │         │                              │
│         │  │ /sse    │  │ /mcp    │         │                              │
│         │  └────┬────┘  └────┬────┘         │                              │
│         └───────┼────────────┼──────────────┘                              │
│                 │            │                                              │
│    ┌────────────┴────────────┴────────────┐                                │
│    │                                      │                                │
│    ▼                                      ▼                                │
│ ┌──────────────────────┐    ┌──────────────────────┐                       │
│ │   LiteLLM Proxy      │    │   MCP 工具服务器       │                       │
│ │   (统一 API 网关)     │    │                      │                       │
│ │                      │    │  ┌────────────────┐  │                       │
│ │  ┌────────────────┐  │    │  │ Filesystem MCP │  │                       │
│ │  │ OpenAI 兼容 API │  │    │  │ GitHub MCP     │  │                       │
│ │  │ /v1/chat/...   │  │    │  │ Puppeteer MCP  │  │                       │
│ │  └────────────────┘  │    │  │ 自定义 MCP...   │  │                       │
│ │                      │    │  └────────────────┘  │                       │
│ └──────────┬───────────┘    └──────────────────────┘                       │
│            │                                                                │
│            ▼                                                                │
│ ┌──────────────────────────────────────────────┐                           │
│ │           本地 LLM 服务层                      │                           │
│ │                                              │                           │
│ │  ┌─────────────┐  ┌─────────────┐           │                           │
│ │  │   Ollama    │  │  自定义 API  │           │                           │
│ │  │  (多模型)   │  │   大模型     │           │                           │
│ │  │             │  │             │           │                           │
│ │  │ - Qwen      │  │ - 公司内部   │           │                           │
│ │  │ - DeepSeek  │  │   部署模型   │           │                           │
│ │  │ - Llama     │  │             │           │                           │
│ │  └─────────────┘  └─────────────┘           │                           │
│ └──────────────────────────────────────────────┘                           │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 组件说明

| 组件 | 功能 | 端口 |
|------|------|------|
| **MCP Gateway** | 统一 MCP 入口，SSE/HTTP 传输 | 8080 |
| **LiteLLM Proxy** | 统一 LLM API 网关，OpenAI 兼容 | 4000 |
| **Ollama** | 本地 LLM 推理服务 | 11434 |
| **MCP 工具服务器** | 文件、GitHub、浏览器等工具 | 3001-300x |

---

## 为什么这样设计

### 1. 为什么用 LiteLLM Proxy？

```
问题：                          解决方案：
┌─────────────────────┐        ┌─────────────────────┐
│ 不同模型 API 格式不同  │   →   │ LiteLLM 统一为       │
│ - Ollama API        │        │ OpenAI 兼容格式      │
│ - 自定义 API        │        │ /v1/chat/completions │
│ - Azure OpenAI      │        │                     │
└─────────────────────┘        └─────────────────────┘
```

**核心优势：**
- **统一接口**: 100+ 模型提供商统一为 OpenAI 格式
- **负载均衡**: 多模型实例自动分配
- **成本追踪**: 记录每个 API Key 的使用量
- **访问控制**: 团队级别的 API Key 管理
- **MCP 集成**: v1.74+ 原生支持 MCP Gateway

### 2. 为什么用 Ollama？

```
问题：                          解决方案：
┌─────────────────────┐        ┌─────────────────────┐
│ 本地 LLM 部署复杂     │   →   │ Ollama 一键部署      │
│ - 模型下载管理       │        │ - docker pull       │
│ - GPU 资源配置       │        │ - ollama pull qwen  │
│ - API 服务暴露       │        │ - 自动 GPU 调度      │
└─────────────────────┘        └─────────────────────┘
```

**核心优势：**
- **简单部署**: Docker 镜像开箱即用
- **模型生态**: 支持 Qwen、DeepSeek、Llama 等主流模型
- **资源管理**: 自动 GPU 显存管理
- **API 标准**: RESTful API 易于集成

### 3. 为什么用集中式 MCP Gateway？

```
问题：                          解决方案：
┌─────────────────────┐        ┌─────────────────────┐
│ 每人本地装 MCP 服务器 │   →   │ 集中部署 + SSE 远程   │
│ - 版本不一致         │        │ - 统一版本管理       │
│ - 配置不同步         │        │ - 集中配置          │
│ - 维护成本高         │        │ - 一处部署，多人使用  │
└─────────────────────┘        └─────────────────────┘
```

**核心优势：**
- **版本一致**: 团队使用相同版本
- **配置集中**: 一处修改，全员生效
- **安全可控**: 统一的访问控制和审计
- **维护简单**: 运维只需维护一套服务

### 4. 为什么选择这个架构？

| 对比项 | 方案 A: 每人本地部署 | 方案 B: 集中式部署（推荐） |
|--------|---------------------|-------------------------|
| 部署复杂度 | 高（每人配置） | 低（一次部署） |
| 版本管理 | 难以统一 | 统一管理 |
| 资源利用 | 分散，利用率低 | 集中，利用率高 |
| 安全审计 | 无法统一 | 集中日志 |
| 维护成本 | N 倍（N=人数） | 1 倍 |
| 网络要求 | 无 | 内网可达 |

---

## 怎么做

### 整体实施步骤

```
┌────────────────────────────────────────────────────────────────┐
│                        实施路线图                               │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  第 1 步              第 2 步              第 3 步              │
│  ┌─────────┐         ┌─────────┐         ┌─────────┐          │
│  │ 部署 LLM │   →     │ 部署 MCP │   →     │ 配置客户 │          │
│  │ 服务层   │         │ 网关层   │         │ 端接入   │          │
│  └─────────┘         └─────────┘         └─────────┘          │
│       │                   │                   │                │
│       ▼                   ▼                   ▼                │
│  - Ollama            - MCP Gateway       - Claude Code        │
│  - LiteLLM Proxy     - MCP 工具服务器     - Codex CLI          │
│  - 自定义 API 模型    - 认证配置          - Gemini CLI         │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

### 技术选型决策树

```
                    ┌─────────────────┐
                    │ 你的模型来源？   │
                    └────────┬────────┘
                             │
            ┌────────────────┼────────────────┐
            │                │                │
            ▼                ▼                ▼
     ┌──────────┐     ┌──────────┐     ┌──────────┐
     │ 开源模型  │     │ 自定义 API │     │  混合     │
     │ (Ollama) │     │   大模型   │     │         │
     └────┬─────┘     └────┬─────┘     └────┬─────┘
          │                │                │
          ▼                ▼                ▼
    ┌──────────┐     ┌──────────┐     ┌──────────┐
    │ Ollama + │     │ LiteLLM  │     │ LiteLLM  │
    │ LiteLLM  │     │ 直连自定义 │     │ 路由多源  │
    └──────────┘     │   API    │     │  模型    │
                     └──────────┘     └──────────┘
```

---

## 最佳实践方案

### 方案一：轻量级（推荐入门）

适用场景：小团队（5人以下），单服务器

```
┌─────────────────────────────────────────┐
│          单机 Docker Compose            │
│                                         │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐ │
│  │ Ollama  │  │ LiteLLM │  │ MCP     │ │
│  │ :11434  │  │ :4000   │  │ Gateway │ │
│  │         │  │         │  │ :8080   │ │
│  └─────────┘  └─────────┘  └─────────┘ │
│                                         │
│      服务器: 192.168.x.100              │
└─────────────────────────────────────────┘
```

### 方案二：标准级（推荐生产）

适用场景：中型团队（5-30人），高可用需求

```
┌─────────────────────────────────────────────────────────────┐
│                    Docker Swarm / K8s                       │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │               负载均衡 (Traefik/Nginx)               │   │
│  │                   192.168.x.100                      │   │
│  └───────────────────────┬─────────────────────────────┘   │
│                          │                                  │
│     ┌────────────────────┼────────────────────┐            │
│     │                    │                    │            │
│     ▼                    ▼                    ▼            │
│  ┌──────┐            ┌──────┐            ┌──────┐         │
│  │ MCP  │            │ MCP  │            │ MCP  │         │
│  │ GW-1 │            │ GW-2 │            │ GW-3 │         │
│  └──────┘            └──────┘            └──────┘         │
│                          │                                  │
│                          ▼                                  │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                  LiteLLM Proxy                       │   │
│  │              (PostgreSQL + Redis)                    │   │
│  └───────────────────────┬─────────────────────────────┘   │
│                          │                                  │
│          ┌───────────────┼───────────────┐                 │
│          ▼               ▼               ▼                 │
│     ┌─────────┐    ┌─────────┐    ┌─────────┐             │
│     │ Ollama  │    │ Ollama  │    │ 自定义   │             │
│     │  GPU-1  │    │  GPU-2  │    │  API    │             │
│     └─────────┘    └─────────┘    └─────────┘             │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 方案三：企业级

适用场景：大型团队（30人+），多数据中心

```
┌─────────────────────────────────────────────────────────────────────┐
│                         Kubernetes 集群                              │
│                                                                     │
│  ┌───────────────────────────────────────────────────────────────┐ │
│  │                    Ingress Controller                          │ │
│  │              mcp.internal.company.com                          │ │
│  └───────────────────────────────────────────────────────────────┘ │
│                              │                                      │
│  ┌───────────────────────────┼───────────────────────────────────┐ │
│  │                      认证层 (OAuth2/OIDC)                      │ │
│  │                    接入公司统一身份认证                          │ │
│  └───────────────────────────────────────────────────────────────┘ │
│                              │                                      │
│     ┌────────────────────────┼────────────────────────┐            │
│     │                        │                        │            │
│     ▼                        ▼                        ▼            │
│  ┌──────────────┐     ┌──────────────┐     ┌──────────────┐       │
│  │  MCP Gateway │     │  LiteLLM     │     │  监控/日志    │       │
│  │  Deployment  │     │  Proxy       │     │  Prometheus  │       │
│  │  (HPA)       │     │  Deployment  │     │  Grafana     │       │
│  └──────────────┘     └──────────────┘     └──────────────┘       │
│                              │                                      │
│                              ▼                                      │
│  ┌───────────────────────────────────────────────────────────────┐ │
│  │                    Model Serving Layer                         │ │
│  │                                                                │ │
│  │   ┌─────────────┐   ┌─────────────┐   ┌─────────────┐        │ │
│  │   │ Ollama Pod  │   │ Ollama Pod  │   │ 自定义 API   │        │ │
│  │   │ (GPU Node)  │   │ (GPU Node)  │   │   Service   │        │ │
│  │   └─────────────┘   └─────────────┘   └─────────────┘        │ │
│  └───────────────────────────────────────────────────────────────┘ │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 快速开始

### 方案一实施：轻量级部署

#### 目录结构

```
mcp-internal/
├── docker-compose.yml
├── .env
├── litellm/
│   └── config.yaml
├── mcp-gateway/
│   └── config.json
└── data/
    └── ollama/
```

#### 1. 创建 docker-compose.yml

```yaml
version: '3.8'

services:
  # ==================== LLM 服务层 ====================

  # Ollama - 本地 LLM 推理
  ollama:
    image: ollama/ollama:latest
    container_name: ollama
    ports:
      - "11434:11434"
    volumes:
      - ./data/ollama:/root/.ollama
    environment:
      - OLLAMA_HOST=0.0.0.0
      - OLLAMA_ORIGINS=*
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]
    restart: unless-stopped
    networks:
      - mcp-network

  # LiteLLM Proxy - 统一 API 网关
  litellm:
    image: ghcr.io/berriai/litellm:main-stable
    container_name: litellm
    ports:
      - "4000:4000"
    volumes:
      - ./litellm/config.yaml:/app/config.yaml
    environment:
      - LITELLM_MASTER_KEY=${LITELLM_MASTER_KEY}
      - LITELLM_SALT_KEY=${LITELLM_SALT_KEY}
      - DATABASE_URL=postgresql://${POSTGRES_USER}:${POSTGRES_PASSWORD}@postgres:5432/litellm
    depends_on:
      - postgres
      - ollama
    command: ["--config", "/app/config.yaml", "--port", "4000"]
    restart: unless-stopped
    networks:
      - mcp-network

  # PostgreSQL - LiteLLM 数据存储
  postgres:
    image: postgres:15-alpine
    container_name: postgres
    environment:
      - POSTGRES_USER=${POSTGRES_USER}
      - POSTGRES_PASSWORD=${POSTGRES_PASSWORD}
      - POSTGRES_DB=litellm
    volumes:
      - ./data/postgres:/var/lib/postgresql/data
    restart: unless-stopped
    networks:
      - mcp-network

  # ==================== MCP 网关层 ====================

  # MCP Gateway - 统一 MCP 入口
  mcp-gateway:
    image: node:20-alpine
    container_name: mcp-gateway
    working_dir: /app
    ports:
      - "8080:8080"
    volumes:
      - ./mcp-gateway:/app
    command: ["node", "server.js"]
    environment:
      - PORT=8080
      - LITELLM_URL=http://litellm:4000
    depends_on:
      - litellm
    restart: unless-stopped
    networks:
      - mcp-network

  # ==================== MCP 工具服务器 ====================

  # Filesystem MCP Server
  mcp-filesystem:
    image: node:20-alpine
    container_name: mcp-filesystem
    working_dir: /app
    ports:
      - "3001:3000"
    command: ["npx", "-y", "@modelcontextprotocol/server-filesystem", "/workspace"]
    volumes:
      - ./workspace:/workspace:ro
    restart: unless-stopped
    networks:
      - mcp-network

  # GitHub MCP Server (可选)
  mcp-github:
    image: node:20-alpine
    container_name: mcp-github
    working_dir: /app
    ports:
      - "3002:3000"
    command: ["npx", "-y", "@modelcontextprotocol/server-github"]
    environment:
      - GITHUB_PERSONAL_ACCESS_TOKEN=${GITHUB_TOKEN}
    restart: unless-stopped
    networks:
      - mcp-network

networks:
  mcp-network:
    driver: bridge
```

#### 2. 创建 .env 文件

```bash
# LiteLLM 配置
LITELLM_MASTER_KEY=sk-your-master-key-here
LITELLM_SALT_KEY=your-salt-key-here

# PostgreSQL 配置
POSTGRES_USER=litellm
POSTGRES_PASSWORD=your-postgres-password

# GitHub Token (可选)
GITHUB_TOKEN=ghp_your_token_here

# 自定义 API (如果有)
CUSTOM_API_KEY=your-custom-api-key
CUSTOM_API_BASE=http://your-internal-api:8000
```

#### 3. 创建 LiteLLM 配置 (litellm/config.yaml)

```yaml
model_list:
  # Ollama 本地模型
  - model_name: qwen2.5-coder
    litellm_params:
      model: ollama/qwen2.5-coder:14b
      api_base: http://ollama:11434

  - model_name: deepseek-coder
    litellm_params:
      model: ollama/deepseek-coder-v2:16b
      api_base: http://ollama:11434

  - model_name: llama3.2
    litellm_params:
      model: ollama/llama3.2:latest
      api_base: http://ollama:11434

  # 自定义内部 API (如果有)
  - model_name: internal-model
    litellm_params:
      model: openai/internal-model
      api_key: os.environ/CUSTOM_API_KEY
      api_base: os.environ/CUSTOM_API_BASE

general_settings:
  master_key: os.environ/LITELLM_MASTER_KEY

  # 启用 MCP Gateway 功能
  enable_mcp_gateway: true

litellm_settings:
  # 设置默认模型
  default_model: qwen2.5-coder

  # 请求超时
  request_timeout: 300

  # 开启详细日志
  set_verbose: false
```

#### 4. 创建 MCP Gateway 服务器 (mcp-gateway/server.js)

```javascript
const http = require('http');
const { URL } = require('url');

const PORT = process.env.PORT || 8080;
const LITELLM_URL = process.env.LITELLM_URL || 'http://litellm:4000';

// 简单的 SSE MCP Gateway
const server = http.createServer((req, res) => {
  const url = new URL(req.url, `http://${req.headers.host}`);

  // CORS 头
  res.setHeader('Access-Control-Allow-Origin', '*');
  res.setHeader('Access-Control-Allow-Methods', 'GET, POST, OPTIONS');
  res.setHeader('Access-Control-Allow-Headers', 'Content-Type, Authorization');

  if (req.method === 'OPTIONS') {
    res.writeHead(204);
    res.end();
    return;
  }

  // 健康检查
  if (url.pathname === '/health') {
    res.writeHead(200, { 'Content-Type': 'application/json' });
    res.end(JSON.stringify({ status: 'healthy', timestamp: new Date().toISOString() }));
    return;
  }

  // SSE 端点
  if (url.pathname === '/sse') {
    res.writeHead(200, {
      'Content-Type': 'text/event-stream',
      'Cache-Control': 'no-cache',
      'Connection': 'keep-alive'
    });

    // 发送初始化消息
    res.write(`data: ${JSON.stringify({ type: 'init', message: 'MCP Gateway connected' })}\n\n`);

    // 保持连接
    const keepAlive = setInterval(() => {
      res.write(`: keepalive\n\n`);
    }, 30000);

    req.on('close', () => {
      clearInterval(keepAlive);
    });
    return;
  }

  // MCP 消息端点
  if (url.pathname === '/mcp' && req.method === 'POST') {
    let body = '';
    req.on('data', chunk => body += chunk);
    req.on('end', () => {
      try {
        const message = JSON.parse(body);
        // 处理 MCP 消息
        res.writeHead(200, { 'Content-Type': 'application/json' });
        res.end(JSON.stringify({
          success: true,
          received: message,
          timestamp: new Date().toISOString()
        }));
      } catch (e) {
        res.writeHead(400, { 'Content-Type': 'application/json' });
        res.end(JSON.stringify({ error: 'Invalid JSON' }));
      }
    });
    return;
  }

  // 服务信息
  if (url.pathname === '/') {
    res.writeHead(200, { 'Content-Type': 'application/json' });
    res.end(JSON.stringify({
      name: 'Internal MCP Gateway',
      version: '1.0.0',
      endpoints: {
        sse: '/sse',
        mcp: '/mcp',
        health: '/health'
      },
      litellm: LITELLM_URL
    }));
    return;
  }

  res.writeHead(404);
  res.end('Not Found');
});

server.listen(PORT, '0.0.0.0', () => {
  console.log(`MCP Gateway running on http://0.0.0.0:${PORT}`);
  console.log(`LiteLLM Proxy: ${LITELLM_URL}`);
});
```

#### 5. 启动服务

```bash
# 进入目录
cd mcp-internal

# 启动所有服务
docker-compose up -d

# 下载 Ollama 模型
docker exec -it ollama ollama pull qwen2.5-coder:14b
docker exec -it ollama ollama pull deepseek-coder-v2:16b

# 查看日志
docker-compose logs -f

# 测试服务
curl http://localhost:8080/health
curl http://localhost:4000/health
curl http://localhost:11434/api/tags
```

#### 6. 客户端配置

##### Claude Code 配置 (~/.claude/settings.json)

```json
{
  "mcpServers": {
    "internal-gateway": {
      "transport": "sse",
      "url": "http://192.168.x.100:8080/sse"
    },
    "internal-filesystem": {
      "transport": "sse",
      "url": "http://192.168.x.100:3001/sse"
    }
  }
}
```

如果要用内网 LLM 替代 Claude API，使用 claude-code-proxy：

```bash
# 安装 proxy
npm install -g claude-code-proxy

# 设置环境变量
export ANTHROPIC_BASE_URL=http://192.168.x.100:4000/v1
export ANTHROPIC_API_KEY=sk-your-litellm-key

# 启动 Claude Code
claude
```

##### Codex CLI 配置 (~/.codex/config.toml)

```toml
# 使用内网 LiteLLM 作为 API 端点
[api]
base_url = "http://192.168.x.100:4000/v1"
api_key = "sk-your-litellm-key"

# MCP 服务器配置
[mcp.servers.internal-gateway]
url = "http://192.168.x.100:8080/sse"
transport = "sse"
```

##### Gemini CLI 配置 (~/.gemini/settings.json)

```json
{
  "mcpServers": {
    "internal-gateway": {
      "transport": "sse",
      "url": "http://192.168.x.100:8080/sse"
    }
  }
}
```

---

## 验证清单

部署完成后，依次验证：

```bash
# 1. Ollama 服务
curl http://192.168.x.100:11434/api/tags
# 应返回已下载的模型列表

# 2. LiteLLM Proxy
curl http://192.168.x.100:4000/health
# 应返回 {"status": "healthy"}

# 3. LiteLLM 模型列表
curl http://192.168.x.100:4000/v1/models \
  -H "Authorization: Bearer sk-your-litellm-key"
# 应返回配置的模型列表

# 4. MCP Gateway
curl http://192.168.x.100:8080/health
# 应返回 {"status": "healthy"}

# 5. 测试 LLM 调用
curl http://192.168.x.100:4000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer sk-your-litellm-key" \
  -d '{
    "model": "qwen2.5-coder",
    "messages": [{"role": "user", "content": "Hello"}]
  }'
# 应返回模型响应
```

---

## 故障排查

| 问题 | 可能原因 | 解决方案 |
|------|----------|----------|
| Ollama 无法访问 | 防火墙/端口未开放 | `firewall-cmd --add-port=11434/tcp` |
| LiteLLM 连不上 Ollama | Docker 网络隔离 | 确保同一 network |
| GPU 不可用 | nvidia-docker 未安装 | 安装 nvidia-container-toolkit |
| MCP 连接超时 | SSE 长连接被代理中断 | 配置代理支持长连接 |

---

## 参考链接

- [LiteLLM Docker 部署](https://docs.litellm.ai/docs/proxy/deploy)
- [Ollama Docker 部署](https://github.com/ollama/ollama)
- [MCP 远程服务器部署](https://medium.com/@manojjahgirdar/deploy-a-remote-model-context-protocol-mcp-server-and-access-it-through-server-sent-events-sse-bdf6886c4534)
- [claude-code-proxy](https://github.com/fuergaosi233/claude-code-proxy)
- [Ollama MCP Server 示例](https://github.com/flexnst/ollama-mcp-example)

---

*最后更新: 2025-12-10*
