# AnyRouter Transparent Proxy

一个基于 FastAPI 的轻量级透明 HTTP 代理服务，专为解决 AnyRouter 的 Anthropic API 在 Claude Code 中报错 500 的问题而设计。

## 目录

- [核心特性](#核心特性)
- [快速开始](#快速开始)
  - [Docker 部署（推荐）](#docker-部署推荐)
  - [本地运行](#本地运行)
- [配置说明](#配置说明)
- [核心功能](#核心功能)
- [架构与技术](#架构与技术)
- [安全性](#安全性)
- [贡献](#贡献)

## 核心特性

- **完全透明** - 支持所有 HTTP 方法，无缝代理请求
- **流式响应** - 基于 `httpx.AsyncClient` 异步架构，完美支持流式传输
- **标准兼容** - 严格遵循 RFC 7230 规范，正确处理 HTTP 头部
- **灵活配置** - 支持环境变量配置目标 URL 和 System Prompt 替换
- **高性能** - 连接池复用，异步处理，高效应对并发请求
- **智能处理** - 自动计算 Content-Length，避免请求体修改导致的协议错误

## 快速开始

### Docker 部署（推荐）

**使用 Docker Compose：**

```bash
# 克隆项目
git clone <repository-url>
cd AnyRouter-Transparent-Proxy

# 复制环境变量模板（可选）
cp .env.example .env

# 编辑 .env 文件修改配置（可选）
# 默认使用 https://anyrouter.top

# 启动服务
docker-compose up -d

# 查看日志
docker-compose logs -f

# 停止服务
docker-compose down

# 重启服务
docker-compose down && docker-compose up -d
```

**或使用 Docker 命令：**

```bash
# 构建并运行
docker build -t anthropic-proxy .
docker run -d --name anthropic-proxy -p 8088:8088 anthropic-proxy

# 自定义目标 URL
docker run -d --name anthropic-proxy -p 8088:8088 \
  -e API_BASE_URL=https://q.quuvv.cn \
  anthropic-proxy
```

服务将在 `http://localhost:8088` 启动，然后在 Claude Code 中配置此地址即可。

### 本地运行

**环境要求：** Python 3.7+

```bash
# 安装依赖
pip install -r requirements.txt

# 或手动安装
pip install fastapi uvicorn httpx python-dotenv

# 复制环境变量模板
cp .env.example .env

# 启动服务
python anthropic_proxy.py
```

服务将在 `http://0.0.0.0:8088` 启动。

## 配置说明

### 环境变量

创建 `.env` 文件或设置环境变量：

```bash
# API 目标地址（默认：https://anyrouter.top）
API_BASE_URL=https://anyrouter.top
# 或使用备用地址
# API_BASE_URL=https://q.quuvv.cn
```

> 注：配置完成后需要重新启动服务

## 核心功能

### System Prompt 替换

动态替换 Anthropic API 请求体中 `system` 数组第一个元素的 `text` 内容：

- 自定义 Claude Code CLI 行为
- 调整 Claude Agent SDK 响应
- 统一注入系统级提示词

配置 `SYSTEM_PROMPT_REPLACEMENT` 变量启用此功能。

### 智能请求头处理

- **自动过滤** - 移除 hop-by-hop 头部（Connection、Keep-Alive 等）
- **Host 重写** - 自动改写为目标服务器域名
- **Content-Length 自动计算** - 根据实际内容自动计算，避免长度不匹配
- **自定义注入** - 支持覆盖或添加任意请求头
- **IP 追踪** - 自动维护 X-Forwarded-For 链

## 使用示例

将原本指向 `anyrouter.top` 或其他 API 的请求改为指向代理服务：

```bash
# 原始请求
curl https://anyrouter.top/v1/messages \
  -H "x-api-key: $ANTHROPIC_API_KEY" \
  -H "content-type: application/json" \
  -d '{"model": "claude-3-5-sonnet-20241022", "messages": [...]}'

# 通过代理
curl http://localhost:8088/v1/messages \
  -H "x-api-key: $ANTHROPIC_API_KEY" \
  -H "content-type: application/json" \
  -d '{"model": "claude-3-5-sonnet-20241022", "messages": [...]}'
```

## 架构与技术

### 请求处理流程

```
客户端请求
    ↓
proxy() 捕获路由
    ↓
filter_request_headers() 过滤请求头
    ↓
process_request_body() 处理请求体（可选替换 system prompt）
    ↓
重写 Host 头 + 注入自定义头 + 添加 X-Forwarded-For
    ↓
httpx.AsyncClient 发起上游请求
    ↓
filter_response_headers() 过滤响应头
    ↓
StreamingResponse 流式返回给客户端
```

### 关键技术

- **路由处理** - `@app.api_route("/{path:path}")` 捕获所有路径和 HTTP 方法
- **生命周期管理** - 使用 FastAPI lifespan 事件管理 HTTP 客户端生命周期
- **连接池复用** - 全局共享 `httpx.AsyncClient` 实现连接复用
- **异步请求** - 60 秒超时，支持长时间流式响应
- **流式传输** - `StreamingResponse` + `aiter_bytes()` 高效处理大载荷
- **头部过滤** - 符合 RFC 7230 规范，双向过滤 hop-by-hop 头部

### 技术细节

**请求头过滤规则（RFC 7230）：**

移除的头部：Connection、Keep-Alive、Proxy-Authenticate、Proxy-Authorization、TE、Trailers、Transfer-Encoding、Upgrade、Content-Length（由 httpx 自动重新计算）

**System Prompt 替换逻辑：**

1. 检查 `SYSTEM_PROMPT_REPLACEMENT` 配置
2. 解析请求体为 JSON
3. 验证 `system` 字段存在且为非空数组
4. 替换 `system[0].text` 内容
5. 重新序列化为 JSON

失败时自动回退到原始请求体，确保服务稳定性。

**HTTP 客户端生命周期：**

- 启动时创建共享的 `httpx.AsyncClient` 实例
- 运行时所有请求复用同一客户端，享受连接池优势
- 关闭时优雅释放所有连接资源

### 日志输出示例

```
[Proxy] Original body (123 bytes): {...}
[System Replacement] Successfully parsed JSON body
[System Replacement] Original system[0].text: You are Claude Code...
[System Replacement] Replaced with: 你是一个有用的AI助手
[System Replacement] Successfully modified body (original size: 123 bytes, new size: 145 bytes)
```

## 安全性

- **防重定向攻击** - `follow_redirects=False`
- **请求超时** - 60 秒超时防止资源耗尽
- **错误处理** - 上游请求失败时返回 502 状态码
- **自动容错** - Content-Length 自动计算，避免协议错误
- **连接管理** - 共享客户端池确保连接正确关闭
- **日志记录** - 详细的调试信息（生产环境建议移除敏感信息）

## 贡献

欢迎提交 Issue 和 Pull Request！

---

**注意**：本项目仅供学习和开发测试使用，请确保遵守相关服务的使用条款。