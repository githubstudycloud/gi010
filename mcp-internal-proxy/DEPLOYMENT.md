# 内网代理部署指南

完整的 Docker Compose 部署方案和配置文件。

## 目录结构

```
mcp-internal-proxy/
├── docker-compose.yml          # 主编排文件
├── .env                        # 环境变量
├── nginx/
│   └── nginx.conf              # Nginx 配置 (Headers 清洗)
├── litellm/
│   └── config.yaml             # LiteLLM 配置
└── data/
    ├── ollama/                 # Ollama 模型存储
    └── postgres/               # LiteLLM 数据库
```

---

## 配置文件

### 1. docker-compose.yml

```yaml
version: '3.8'

services:
  # ==================== 入口层 ====================

  nginx:
    image: nginx:alpine
    container_name: ai-gateway-nginx
    ports:
      - "443:443"
      - "80:80"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./nginx/certs:/etc/nginx/certs:ro  # 如需 HTTPS
    depends_on:
      - litellm
    restart: unless-stopped
    networks:
      - ai-gateway

  # ==================== LLM 网关层 ====================

  litellm:
    image: ghcr.io/berriai/litellm:main-stable
    container_name: ai-gateway-litellm
    ports:
      - "4000:4000"
    volumes:
      - ./litellm/config.yaml:/app/config.yaml:ro
    environment:
      - LITELLM_MASTER_KEY=${LITELLM_MASTER_KEY}
      - LITELLM_SALT_KEY=${LITELLM_SALT_KEY}
      - DATABASE_URL=postgresql://${POSTGRES_USER}:${POSTGRES_PASSWORD}@postgres:5432/litellm
      # 隐藏客户端信息的关键配置
      - LITELLM_DROP_PARAMS=true
    depends_on:
      postgres:
        condition: service_healthy
      ollama:
        condition: service_started
    command: ["--config", "/app/config.yaml", "--port", "4000", "--detailed_debug", "false"]
    restart: unless-stopped
    networks:
      - ai-gateway

  # ==================== 数据库 ====================

  postgres:
    image: postgres:15-alpine
    container_name: ai-gateway-postgres
    environment:
      - POSTGRES_USER=${POSTGRES_USER}
      - POSTGRES_PASSWORD=${POSTGRES_PASSWORD}
      - POSTGRES_DB=litellm
    volumes:
      - ./data/postgres:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER}"]
      interval: 5s
      timeout: 5s
      retries: 5
    restart: unless-stopped
    networks:
      - ai-gateway

  # ==================== 模型服务层 ====================

  ollama:
    image: ollama/ollama:latest
    container_name: ai-gateway-ollama
    ports:
      - "11434:11434"
    volumes:
      - ./data/ollama:/root/.ollama
    environment:
      - OLLAMA_HOST=0.0.0.0
      - OLLAMA_ORIGINS=*
      - OLLAMA_KEEP_ALIVE=24h
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]
    restart: unless-stopped
    networks:
      - ai-gateway

  # ==================== Redis (可选，用于限流和缓存) ====================

  redis:
    image: redis:7-alpine
    container_name: ai-gateway-redis
    ports:
      - "6379:6379"
    restart: unless-stopped
    networks:
      - ai-gateway

networks:
  ai-gateway:
    driver: bridge
```

### 2. .env

```bash
# ==================== LiteLLM 配置 ====================
LITELLM_MASTER_KEY=sk-internal-gateway-master-key-change-me
LITELLM_SALT_KEY=your-random-salt-key-for-encryption

# ==================== PostgreSQL 配置 ====================
POSTGRES_USER=litellm
POSTGRES_PASSWORD=your-secure-postgres-password

# ==================== 内部模型配置 ====================
# 如果有自定义内部 API
INTERNAL_API_BASE=http://your-internal-llm-api:8000
INTERNAL_API_KEY=your-internal-api-key

# ==================== 外部 API (可选，如需转发) ====================
# OPENAI_API_KEY=sk-xxx
# ANTHROPIC_API_KEY=sk-ant-xxx
# GOOGLE_API_KEY=xxx

# ==================== 网关标识 ====================
GATEWAY_USER_AGENT=Internal-AI-Gateway/1.0
```

### 3. nginx/nginx.conf

```nginx
worker_processes auto;

events {
    worker_connections 4096;
}

http {
    # 日志格式 - 记录内部审计信息
    log_format audit '$remote_addr - $remote_user [$time_local] '
                     '"$request" $status $body_bytes_sent '
                     '"$http_referer" "$http_user_agent" '
                     'original_ip="$http_x_real_ip" '
                     'forwarded_for="$http_x_forwarded_for"';

    access_log /var/log/nginx/access.log audit;
    error_log /var/log/nginx/error.log warn;

    # 上游服务
    upstream litellm {
        server litellm:4000;
        keepalive 32;
    }

    server {
        listen 80;
        listen 443 ssl http2;
        server_name _;

        # SSL 配置 (如需 HTTPS)
        # ssl_certificate /etc/nginx/certs/server.crt;
        # ssl_certificate_key /etc/nginx/certs/server.key;

        # 客户端请求大小限制
        client_max_body_size 100M;

        # 超时配置 (SSE 流式响应需要长连接)
        proxy_connect_timeout 300s;
        proxy_send_timeout 300s;
        proxy_read_timeout 300s;

        # ==================== Headers 清洗 ====================

        # 移除所有客户端标识头
        proxy_set_header x-stainless-arch "";
        proxy_set_header x-stainless-os "";
        proxy_set_header x-stainless-runtime "";
        proxy_set_header x-stainless-runtime-version "";
        proxy_set_header x-stainless-package-version "";
        proxy_set_header x-stainless-lang "";
        proxy_set_header X-Real-IP "";
        proxy_set_header X-Forwarded-For "";
        proxy_set_header X-Forwarded-Proto "";
        proxy_set_header X-Forwarded-Host "";

        # 重写 User-Agent
        proxy_set_header User-Agent "Internal-AI-Gateway/1.0";

        # 保留必要的头
        proxy_set_header Host $host;
        proxy_set_header Connection "";
        proxy_http_version 1.1;

        # ==================== SSE 流式响应配置 ====================
        proxy_buffering off;
        proxy_cache off;
        chunked_transfer_encoding on;

        # ==================== 路由配置 ====================

        # Anthropic Messages API (Claude Code)
        location /v1/messages {
            proxy_pass http://litellm/v1/messages;

            # 额外清理 Anthropic 特有头
            proxy_set_header anthropic-beta $http_anthropic_beta;
            proxy_set_header anthropic-version $http_anthropic_version;
        }

        # Token 计数端点
        location /v1/messages/count_tokens {
            proxy_pass http://litellm/v1/messages/count_tokens;
        }

        # OpenAI Chat Completions API
        location /v1/chat/completions {
            proxy_pass http://litellm/v1/chat/completions;
        }

        # OpenAI Completions API
        location /v1/completions {
            proxy_pass http://litellm/v1/completions;
        }

        # 模型列表
        location /v1/models {
            proxy_pass http://litellm/v1/models;
        }

        # Embeddings
        location /v1/embeddings {
            proxy_pass http://litellm/v1/embeddings;
        }

        # 健康检查
        location /health {
            proxy_pass http://litellm/health;
        }

        # LiteLLM 管理界面 (可选，内网可开放)
        location /ui {
            proxy_pass http://litellm/ui;
        }

        # 默认路由到 LiteLLM
        location / {
            proxy_pass http://litellm;
        }
    }
}
```

### 4. litellm/config.yaml

```yaml
# ==================== 模型配置 ====================

model_list:
  # ---------- Ollama 本地模型 ----------

  # Claude 请求映射到 Qwen
  - model_name: claude-sonnet-4-20250514
    litellm_params:
      model: ollama/qwen2.5-coder:32b
      api_base: http://ollama:11434
    model_info:
      description: "Internal Qwen model (mapped from Claude Sonnet)"

  - model_name: claude-3-5-sonnet-20241022
    litellm_params:
      model: ollama/qwen2.5-coder:32b
      api_base: http://ollama:11434

  - model_name: claude-3-haiku-20240307
    litellm_params:
      model: ollama/qwen2.5-coder:7b
      api_base: http://ollama:11434

  # GPT 请求映射到 DeepSeek
  - model_name: gpt-4o
    litellm_params:
      model: ollama/deepseek-coder-v2:16b
      api_base: http://ollama:11434

  - model_name: gpt-4o-mini
    litellm_params:
      model: ollama/deepseek-coder-v2:lite
      api_base: http://ollama:11434

  - model_name: gpt-5-codex
    litellm_params:
      model: ollama/qwen2.5-coder:32b
      api_base: http://ollama:11434

  - model_name: gpt-5-codex-mini
    litellm_params:
      model: ollama/qwen2.5-coder:14b
      api_base: http://ollama:11434

  # Gemini 请求映射
  - model_name: gemini-2.5-pro
    litellm_params:
      model: ollama/qwen2.5:72b
      api_base: http://ollama:11434

  - model_name: gemini-2.0-flash
    litellm_params:
      model: ollama/qwen2.5:14b
      api_base: http://ollama:11434

  # ---------- 直接可用的模型别名 ----------

  - model_name: qwen-coder-32b
    litellm_params:
      model: ollama/qwen2.5-coder:32b
      api_base: http://ollama:11434

  - model_name: deepseek-coder
    litellm_params:
      model: ollama/deepseek-coder-v2:16b
      api_base: http://ollama:11434

  # ---------- 自定义内部 API (如有) ----------

  # - model_name: internal-model
  #   litellm_params:
  #     model: openai/internal-model
  #     api_key: os.environ/INTERNAL_API_KEY
  #     api_base: os.environ/INTERNAL_API_BASE

# ==================== 通用设置 ====================

general_settings:
  master_key: os.environ/LITELLM_MASTER_KEY

  # 数据库配置
  database_url: os.environ/DATABASE_URL

  # 启用请求日志
  store_model_in_db: true

  # 禁用遥测
  disable_telemetry: true

# ==================== LiteLLM 行为设置 ====================

litellm_settings:
  # 默认模型
  default_model: qwen-coder-32b

  # 请求超时
  request_timeout: 300

  # 失败重试
  num_retries: 2

  # 禁用详细日志 (隐藏敏感信息)
  set_verbose: false

  # 丢弃不支持的参数 (兼容性)
  drop_params: true

  # 修改请求头
  modify_params: true

# ==================== 路由设置 ====================

router_settings:
  # 路由策略
  routing_strategy: simple-shuffle

  # 允许失败后降级
  allowed_fails: 2

  # 模型组别名
  model_group_alias:
    claude: qwen-coder-32b
    gpt: deepseek-coder
    gemini: qwen-coder-32b

# ==================== 环境变量 ====================

environment_variables:
  # 禁用 OpenAI 的遥测
  OPENAI_LOG: "error"
```

---

## 部署步骤

### 步骤 1: 准备环境

```bash
# 创建目录
mkdir -p mcp-internal-proxy/{nginx,litellm,data/ollama,data/postgres}
cd mcp-internal-proxy

# 复制配置文件
# (将上述配置文件保存到对应位置)

# 设置权限
chmod 600 .env
```

### 步骤 2: 启动服务

```bash
# 启动所有服务
docker-compose up -d

# 查看启动状态
docker-compose ps

# 查看日志
docker-compose logs -f
```

### 步骤 3: 下载模型

```bash
# 下载 Qwen Coder 模型
docker exec -it ai-gateway-ollama ollama pull qwen2.5-coder:32b
docker exec -it ai-gateway-ollama ollama pull qwen2.5-coder:14b
docker exec -it ai-gateway-ollama ollama pull qwen2.5-coder:7b

# 下载 DeepSeek Coder 模型
docker exec -it ai-gateway-ollama ollama pull deepseek-coder-v2:16b
docker exec -it ai-gateway-ollama ollama pull deepseek-coder-v2:lite

# 查看已下载模型
docker exec -it ai-gateway-ollama ollama list
```

### 步骤 4: 验证服务

```bash
# 1. 检查 Nginx
curl http://192.168.x.100/health

# 2. 检查 LiteLLM
curl http://192.168.x.100:4000/health

# 3. 检查 Ollama
curl http://192.168.x.100:11434/api/tags

# 4. 测试 Anthropic 格式 (Claude Code 使用)
curl -X POST http://192.168.x.100/v1/messages \
  -H "Content-Type: application/json" \
  -H "x-api-key: sk-internal-gateway-master-key-change-me" \
  -H "anthropic-version: 2023-06-01" \
  -d '{
    "model": "claude-sonnet-4-20250514",
    "max_tokens": 100,
    "messages": [{"role": "user", "content": "Hello"}]
  }'

# 5. 测试 OpenAI 格式
curl -X POST http://192.168.x.100/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer sk-internal-gateway-master-key-change-me" \
  -d '{
    "model": "gpt-4o",
    "messages": [{"role": "user", "content": "Hello"}]
  }'
```

---

## 客户端配置

### Claude Code 配置

```bash
# 方法 1: 环境变量
export ANTHROPIC_BASE_URL=http://192.168.x.100
export ANTHROPIC_API_KEY=sk-internal-gateway-master-key-change-me

# 启动 Claude Code
claude

# 方法 2: 写入 shell 配置
echo 'export ANTHROPIC_BASE_URL=http://192.168.x.100' >> ~/.bashrc
echo 'export ANTHROPIC_API_KEY=sk-internal-gateway-master-key-change-me' >> ~/.bashrc
source ~/.bashrc
```

### Codex CLI 配置

```bash
# Codex CLI 支持 HTTPS_PROXY 环境变量
export HTTPS_PROXY=http://192.168.x.100:80
export HTTP_PROXY=http://192.168.x.100:80

# 或者使用 config.toml
cat > ~/.codex/config.toml << 'EOF'
[network]
proxy = "http://192.168.x.100:80"
EOF

# 启动 Codex
codex
```

**注意**: Codex CLI 使用私有 ChatGPT Backend API，完全拦截替换需要更复杂的处理。上述配置会将 HTTPS 请求通过代理，但代理需要能处理 ChatGPT 的 OAuth 认证流程。

**推荐替代方案**: 使用 `ccproxy-api` 或直接使用支持 API Key 的 GPT-5-Codex。

### Gemini CLI 配置

```bash
# Gemini CLI 支持 GEMINI_API_HOST 环境变量
export GEMINI_API_HOST=http://192.168.x.100

# 配置 settings.json
cat > ~/.gemini/settings.json << 'EOF'
{
  "apiHost": "http://192.168.x.100"
}
EOF

# 启动 Gemini CLI
gemini
```

---

## 验证 Headers 清洗

```bash
# 在代理服务器上查看 Nginx 日志
docker exec -it ai-gateway-nginx tail -f /var/log/nginx/access.log

# 在另一个终端发送请求
curl -X POST http://192.168.x.100/v1/messages \
  -H "Content-Type: application/json" \
  -H "x-api-key: test-key" \
  -H "anthropic-version: 2023-06-01" \
  -H "User-Agent: claude-code/1.0.0 (Darwin; arm64)" \
  -H "x-stainless-os: MacOS" \
  -H "x-stainless-arch: arm64" \
  -H "X-Real-IP: 10.0.0.5" \
  -H "X-Forwarded-For: 10.0.0.5, 192.168.1.1" \
  -d '{
    "model": "claude-sonnet-4-20250514",
    "max_tokens": 10,
    "messages": [{"role": "user", "content": "test"}]
  }'

# 日志应该显示原始客户端信息被记录，但请求到 LiteLLM 时已清洗
```

---

## 故障排查

| 问题 | 可能原因 | 解决方案 |
|------|----------|----------|
| Claude Code 连接失败 | BASE_URL 格式错误 | 确保不带尾部斜杠 |
| 模型不存在 | Ollama 未下载模型 | `ollama pull qwen2.5-coder:32b` |
| 502 Bad Gateway | LiteLLM 未启动 | `docker-compose logs litellm` |
| SSE 中断 | Nginx 超时配置 | 增加 `proxy_read_timeout` |
| GPU 不可用 | nvidia-docker 未配置 | 安装 nvidia-container-toolkit |

---

## 监控和日志

```bash
# 查看所有服务状态
docker-compose ps

# 查看 LiteLLM 请求日志
docker-compose logs -f litellm

# 查看 Nginx 访问日志
docker exec ai-gateway-nginx cat /var/log/nginx/access.log

# 进入 LiteLLM UI (如开放)
# 浏览器访问: http://192.168.x.100:4000/ui
```

---

## 安全建议

1. **更改默认密钥**: 修改 `.env` 中的所有默认密钥
2. **网络隔离**: 只在内网暴露服务，不要暴露到公网
3. **访问控制**: 使用 LiteLLM 的 API Key 管理功能限制访问
4. **日志审计**: 定期检查 Nginx 审计日志
5. **HTTPS**: 生产环境建议启用 HTTPS

---

*最后更新: 2025-12-10*
