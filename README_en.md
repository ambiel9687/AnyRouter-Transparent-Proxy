# AnyRouter Transparent Proxy

A lightweight transparent HTTP proxy service based on FastAPI, specifically designed to resolve the 500 error issue with AnyRouter's Anthropic API in the Claude Code for VS Code plugin.

## Demo Screenshot

<img width="3754" height="2110" alt="VS Code finally works with CC plugin" src="https://github.com/user-attachments/assets/6a492f30-43ed-4ba1-ad9f-15801b356f7a" />


## Table of Contents

- [Core Features](#core-features)
- [Quick Start](#quick-start)
  - [Docker Deployment (Recommended)](#docker-deployment-recommended)
  - [Local Setup](#local-setup)
- [Configuration](#configuration)
- [Core Functionality](#core-functionality)
- [Architecture & Technology](#architecture--technology)
- [Security](#security)
- [Contributing](#contributing)

## Core Features

- **Fully Transparent** - Supports all HTTP methods, seamlessly proxies requests
- **Streaming Response** - Asynchronous architecture based on `httpx.AsyncClient`, perfect streaming support
- **Standards Compliant** - Strictly follows RFC 7230 specification, correctly handles HTTP headers
- **Flexible Configuration** - Supports environment variable configuration for target URL and System Prompt replacement
- **High Performance** - Connection pool reuse, asynchronous processing, efficiently handles concurrent requests
- **Intelligent Processing** - Automatically calculates Content-Length, avoiding protocol errors caused by request body modifications

## Quick Start

### Docker Deployment (Recommended)

**Using Docker Compose:**

```bash
# Clone the project
git clone <repository-url>
cd AnyRouter-Transparent-Proxy

# Copy environment variable template (optional)
cp .env.example .env

# Edit .env file to modify configuration (optional)
# Default uses https://anyrouter.top

# Start the service
docker-compose up -d
# If you modified the source code after deployment, rebuild with --build --remove-orphans
docker-compose up -d --build --remove-orphans

# View logs
docker-compose logs -f

# Stop the service
docker-compose down

# Restart the service
docker-compose down && docker-compose up -d
```

**Or using Docker commands:**

```bash
# Build and run
docker build -t anthropic-proxy .
docker run -d --name anthropic-proxy -p 8088:8088 anthropic-proxy

# Custom target URL
docker run -d --name anthropic-proxy -p 8088:8088 \
  -e API_BASE_URL=https://q.quuvv.cn \
  anthropic-proxy
```

The service will start at `http://localhost:8088`, then configure this address in Claude Code.

### Local Setup

**Requirements:** Python 3.7+

```bash
# Install dependencies
pip install -r requirements.txt

# Or install manually
pip install fastapi uvicorn httpx python-dotenv

# Copy environment variable template
cp .env.example .env

# Start the service
python anthropic_proxy.py
```

The service will start at `http://0.0.0.0:8088`.

## Configuration

### Environment Variables

Create a `.env` file or set environment variables:

```bash
# API target address (default: https://anyrouter.top) Or use reverse address
API_BASE_URL=https://anyrouter.top

# System Prompt replacement (optional)
# Replaces the text content of the first element in the system array in the request body
SYSTEM_PROMPT_REPLACEMENT="You are Claude Code, Anthropic's official CLI for Claude."

# If you need to access through a proxy, please uncomment the two lines below and set the proxy address.
# HTTP_PROXY=http://127.0.0.1:7890
# HTTPS_PROXY=http://127.0.0.1:7890
```

### Custom Request Headers (Optional)

You can configure custom request headers through the `env/.env.headers.json` file, which will be injected into all proxied requests.

**Configuration Steps:**

1. Copy the template file to create configuration:
   ```bash
   cp .env.headers.json.example env/.env.headers.json
   ```

2. Edit the `env/.env.headers.json` file and add the headers you need:
   ```json
   {
     "User-Agent": "claude-cli/2.0.8 (external, cli)"
   }
   ```

3. Restart the service for the configuration to take effect

**Notes:**
- Configuration file uses standard JSON format
- Fields starting with `__` will be ignored (can be used for comments)
- If the file doesn't exist, defaults to empty configuration `{}`
- Custom request headers will override headers with the same name in the original request

> Note: Service restart required after configuration changes

## Core Functionality

### System Prompt Replacement

Dynamically replaces the `text` content of the first element in the `system` array in Anthropic API request bodies:

- Customize Claude Code CLI behavior
- Adjust Claude Agent SDK responses
- Uniformly inject system-level prompts

Enable this feature by configuring the `SYSTEM_PROMPT_REPLACEMENT` variable.

### Intelligent Request Header Handling

- **Automatic Filtering** - Removes hop-by-hop headers (Connection, Keep-Alive, etc.)
- **Host Rewriting** - Automatically rewrites to target server domain
- **Content-Length Auto-calculation** - Automatically calculates based on actual content, avoiding length mismatches
- **Custom Injection** - Supports overriding or adding arbitrary request headers
- **IP Tracking** - Automatically maintains X-Forwarded-For chain

## Usage Example

Change requests originally pointing to `anyrouter.top` or other APIs to point to the proxy service:

```bash
# Original request
curl https://anyrouter.top/v1/messages \
  -H "x-api-key: $ANTHROPIC_API_KEY" \
  -H "content-type: application/json" \
  -d '{"model": "claude-3-5-sonnet-20241022", "messages": [...]}'

# Through proxy
curl http://localhost:8088/v1/messages \
  -H "x-api-key: $ANTHROPIC_API_KEY" \
  -H "content-type: application/json" \
  -d '{"model": "claude-3-5-sonnet-20241022", "messages": [...]}'
```

## Architecture & Technology

### Request Processing Flow

```
Client Request
    ↓
proxy() catches route
    ↓
filter_request_headers() filters request headers
    ↓
process_request_body() processes request body (optional system prompt replacement)
    ↓
Rewrite Host header + Inject custom headers + Add X-Forwarded-For
    ↓
httpx.AsyncClient initiates upstream request
    ↓
filter_response_headers() filters response headers
    ↓
StreamingResponse streams back to client
```

### Key Technologies

- **Routing** - `@app.api_route("/{path:path}")` catches all paths and HTTP methods
- **Lifecycle Management** - Uses FastAPI lifespan events to manage HTTP client lifecycle
- **Connection Pool Reuse** - Globally shared `httpx.AsyncClient` for connection reuse
- **Asynchronous Requests** - 60-second timeout, supports long-running streaming responses
- **Streaming Transfer** - `StreamingResponse` + `aiter_bytes()` efficiently handles large payloads
- **Header Filtering** - RFC 7230 compliant, bidirectional filtering of hop-by-hop headers

### Technical Details

**Request Header Filtering Rules (RFC 7230):**

Removed headers: Connection, Keep-Alive, Proxy-Authenticate, Proxy-Authorization, TE, Trailers, Transfer-Encoding, Upgrade, Content-Length (automatically recalculated by httpx)

**System Prompt Replacement Logic:**

1. Check `SYSTEM_PROMPT_REPLACEMENT` configuration
2. Parse request body as JSON
3. Verify `system` field exists and is a non-empty array
4. Replace `system[0].text` content
5. Re-serialize to JSON

Falls back to original request body on failure, ensuring service stability.

**HTTP Client Lifecycle:**

- Creates shared `httpx.AsyncClient` instance on startup
- All requests reuse the same client during runtime, benefiting from connection pooling
- Gracefully releases all connection resources on shutdown

### Log Output Example

```
[Proxy] Original body (123 bytes): {...}
[System Replacement] Successfully parsed JSON body
[System Replacement] Original system[0].text: You are Claude Code...
[System Replacement] Replaced with: 你是一个有用的AI助手
[System Replacement] Successfully modified body (original size: 123 bytes, new size: 145 bytes)
```

## Security

- **Redirect Attack Prevention** - `follow_redirects=False`
- **Request Timeout** - 60-second timeout prevents resource exhaustion
- **Error Handling** - Returns 502 status code when upstream request fails
- **Automatic Fault Tolerance** - Content-Length auto-calculation avoids protocol errors
- **Connection Management** - Shared client pool ensures connections are properly closed
- **Logging** - Detailed debugging information (recommend removing sensitive info in production)

## Contributing

Issues and Pull Requests are welcome!

---

**Note**: This project is for learning and development testing purposes only. Please ensure compliance with the terms of service of related services.
