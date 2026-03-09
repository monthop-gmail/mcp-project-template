# Boilerplate: Python MCP Server

สำหรับ project ที่ต้องใช้ Python ecosystem (ML, data processing, XML-RPC, etc.)
เปลี่ยน `xxx` เป็นชื่อ project จริง, `xxx_mcp` เป็น package name (underscore)

---

## 1. pyproject.toml

```toml
[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[project]
name = "xxx-mcp"
version = "0.1.0"
description = "MCP Server for [description]"
readme = "README.md"
requires-python = ">=3.10"
license = "MIT"
dependencies = [
    "mcp>=1.8.0",
    "python-dotenv>=1.0.0",
    "uvicorn>=0.30.0",
    "starlette>=0.38.0",
]

[project.scripts]
xxx-mcp = "xxx_mcp.server:main"

[tool.hatch.build.targets.wheel]
packages = ["src/xxx_mcp"]
```

**เพิ่มตามต้องการ:**
- HTTP Client: `"httpx>=0.27.0"`
- Database: `"sqlalchemy>=2.0.0"` หรือ `"aiosqlite>=0.20.0"`
- Data: `"pandas>=2.0.0"`, `"numpy>=1.26.0"`
- ML: `"faster-whisper>=1.0.0"`, `"torch>=2.0.0"`
- Web Scraping: `"beautifulsoup4>=4.12.0"`, `"lxml>=5.0.0"`

---

## 2. src/xxx_mcp/__init__.py

```python
```

(ไฟล์ว่าง - ใช้เป็น package marker)

---

## 3. src/xxx_mcp/server.py (Main Server)

```python
"""xxx MCP Server - Main server implementation."""

import argparse
import json
import logging
import os
from typing import Any

from dotenv import load_dotenv
from mcp.server import Server
from mcp.server.stdio import stdio_server
from mcp.types import TextContent, Tool

# Load environment variables
load_dotenv()

# Configure logging (stderr - stdout reserved for MCP protocol)
logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s [%(levelname)s] %(message)s",
    handlers=[logging.StreamHandler()],
)
logger = logging.getLogger(__name__)

# Initialize server
server = Server("xxx-mcp")


def format_result(result: Any) -> str:
    """Format result for MCP response."""
    if isinstance(result, str):
        return result
    return json.dumps(result, ensure_ascii=False, indent=2, default=str)


# ============================================================
# Tools
# ============================================================

@server.list_tools()
async def list_tools() -> list[Tool]:
    """List available tools."""
    return [
        Tool(
            name="example_search",
            description="Search for something",
            inputSchema={
                "type": "object",
                "properties": {
                    "query": {
                        "type": "string",
                        "description": "Search query",
                    },
                    "limit": {
                        "type": "integer",
                        "description": "Max results (default: 10)",
                        "default": 10,
                    },
                },
                "required": ["query"],
            },
        ),
        Tool(
            name="example_get",
            description="Get item by ID",
            inputSchema={
                "type": "object",
                "properties": {
                    "id": {
                        "type": "string",
                        "description": "Item ID",
                    },
                },
                "required": ["id"],
            },
        ),
        # ... more tools
    ]


@server.call_tool()
async def call_tool(name: str, arguments: dict) -> list[TextContent]:
    """Handle tool calls."""
    logger.info(f"Tool called: {name}")

    try:
        if name == "example_search":
            query = arguments.get("query", "")
            limit = arguments.get("limit", 10)
            result = await example_search(query, limit)

        elif name == "example_get":
            item_id = arguments["id"]
            result = await example_get(item_id)

        else:
            return [TextContent(type="text", text=f"Unknown tool: {name}")]

        return [TextContent(type="text", text=format_result(result))]

    except Exception as e:
        logger.error(f"Tool error: {name} - {e}")
        return [TextContent(type="text", text=f"Error: {str(e)}")]


# ============================================================
# Tool Implementations
# ============================================================

async def example_search(query: str, limit: int = 10) -> dict:
    """Search implementation."""
    # Business logic here
    return {"query": query, "limit": limit, "results": []}


async def example_get(item_id: str) -> dict:
    """Get item implementation."""
    # Business logic here
    return {"id": item_id, "data": {}}


# ============================================================
# Transport: Stdio
# ============================================================

async def run_stdio_server():
    """Run the MCP server with stdio transport."""
    async with stdio_server() as (read_stream, write_stream):
        await server.run(
            read_stream,
            write_stream,
            server.create_initialization_options(),
        )


# ============================================================
# Transport: Streamable HTTP
# ============================================================

async def run_streamable_http_server(host: str, port: int):
    """Run the MCP server with Streamable HTTP transport."""
    import contextlib
    from collections.abc import AsyncIterator

    from mcp.server.streamable_http_manager import StreamableHTTPSessionManager
    from starlette.applications import Starlette
    from starlette.requests import Request
    from starlette.responses import PlainTextResponse
    from starlette.routing import Mount, Route
    import uvicorn

    session_manager = StreamableHTTPSessionManager(app=server)

    async def handle_streamable_http(scope, receive, send):
        await session_manager.handle_request(scope, receive, send)

    async def health(request: Request):
        return PlainTextResponse("OK")

    @contextlib.asynccontextmanager
    async def lifespan(app: Starlette) -> AsyncIterator[None]:
        async with session_manager.run():
            logger.info(f"xxx MCP Server started at http://{host}:{port}")
            yield
            logger.info("xxx MCP Server shutting down...")

    starlette_app = Starlette(
        debug=False,
        routes=[
            Route("/health", health),
            Mount("/mcp", app=handle_streamable_http),
        ],
        lifespan=lifespan,
    )

    config = uvicorn.Config(starlette_app, host=host, port=port, log_level="info")
    server_instance = uvicorn.Server(config)
    await server_instance.serve()


# ============================================================
# Entry Point
# ============================================================

def main():
    """Main entry point."""
    import asyncio

    parser = argparse.ArgumentParser(description="xxx MCP Server")
    parser.add_argument(
        "--http",
        action="store_true",
        help="Run with Streamable HTTP transport instead of stdio",
    )
    parser.add_argument(
        "--host",
        default="0.0.0.0",
        help="Host to bind HTTP server (default: 0.0.0.0)",
    )
    parser.add_argument(
        "--port",
        type=int,
        default=3000,
        help="Port for HTTP server (default: 3000)",
    )

    args = parser.parse_args()

    if args.http:
        asyncio.run(run_streamable_http_server(args.host, args.port))
    else:
        asyncio.run(run_stdio_server())


if __name__ == "__main__":
    main()
```

---

## 4. Dockerfile

```dockerfile
FROM python:3.12-slim

WORKDIR /app

# Install curl for healthcheck (เพิ่ม dependencies อื่นตามต้องการ)
RUN apt-get update && apt-get install -y --no-install-recommends curl \
    && rm -rf /var/lib/apt/lists/*

# Install dependencies
COPY pyproject.toml README.md ./
COPY src/ ./src/

RUN pip install --no-cache-dir .

# Create non-root user
RUN useradd -m -u 1000 mcp
USER mcp

EXPOSE 3000

# Default: run with Streamable HTTP transport for Docker
CMD ["python", "-m", "xxx_mcp.server", "--http"]
```

**Variations:**

```dockerfile
# สำหรับ ML workloads (เพิ่ม ffmpeg, build tools):
RUN apt-get update && apt-get install -y --no-install-recommends \
    curl ffmpeg build-essential \
    && rm -rf /var/lib/apt/lists/*

# สำหรับ project ที่มี data directory:
RUN mkdir -p /app/data && chown mcp:mcp /app/data
VOLUME ["/app/data"]
```

---

## 5. docker-compose.yml

```yaml
services:
  xxx-mcp:
    build: .
    container_name: xxx-mcp-claude
    restart: unless-stopped
    ports:
      - "XXXX:3000"    # เลือก port ที่ว่าง
    environment:
      - API_KEY=${API_KEY}
      # เพิ่ม env vars ตามต้องการ
    volumes:
      - ./data:/app/data
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 10s
```

---

## 6. .mcp.json

```json
{
  "mcpServers": {
    "xxx": {
      "type": "streamable-http",
      "url": "http://localhost:XXXX/mcp"
    }
  }
}
```

---

## 7. .env.example

```env
# === Required ===
API_KEY=your_api_key

# === Optional ===
# LOG_LEVEL=INFO
```

---

## 8. .gitignore

```
__pycache__/
*.pyc
*.pyo
*.egg-info/
dist/
build/
.env
data/*.db
*.log
.venv/
```

---

## 9. .dockerignore

```
__pycache__
*.pyc
*.pyo
*.egg-info
dist
build
.git
.gitignore
*.md
!README.md
*.db
.env
.DS_Store
.venv
```

---

## 10. Service Layer Pattern (Optional)

สำหรับ project ที่มี external API หรือ business logic ซับซ้อน แยก service ออกจาก server.py

```
src/xxx_mcp/
├── __init__.py
├── server.py          # MCP server + tools + transport
├── client.py          # External API wrapper
└── config.py          # Configuration
```

### src/xxx_mcp/config.py

```python
"""Configuration management."""

import os
from dotenv import load_dotenv

load_dotenv()


class Config:
    API_KEY: str = os.getenv("API_KEY", "")
    API_URL: str = os.getenv("API_URL", "https://api.example.com")
    CACHE_TTL: int = int(os.getenv("CACHE_TTL", "300"))

    @classmethod
    def validate(cls) -> None:
        if not cls.API_KEY:
            raise ValueError("API_KEY environment variable is required")
```

### src/xxx_mcp/client.py

```python
"""External API client."""

import logging
from typing import Any

import httpx

from .config import Config

logger = logging.getLogger(__name__)


class XXXClient:
    """Client for XXX API."""

    def __init__(self):
        self.base_url = Config.API_URL
        self.headers = {"Authorization": f"Bearer {Config.API_KEY}"}

    async def search(self, query: str, limit: int = 10) -> list[dict[str, Any]]:
        """Search the API."""
        async with httpx.AsyncClient() as client:
            response = await client.get(
                f"{self.base_url}/search",
                params={"q": query, "limit": limit},
                headers=self.headers,
            )
            response.raise_for_status()
            return response.json()

    async def get_item(self, item_id: str) -> dict[str, Any]:
        """Get item by ID."""
        async with httpx.AsyncClient() as client:
            response = await client.get(
                f"{self.base_url}/items/{item_id}",
                headers=self.headers,
            )
            response.raise_for_status()
            return response.json()
```

---

## Checklist ก่อน Deploy

- [ ] `pip install -e .` สำเร็จ
- [ ] `.env` ตั้งค่าครบ
- [ ] `python -m xxx_mcp.server` (stdio) ทำงานได้
- [ ] `docker compose up -d --build` สำเร็จ
- [ ] `curl http://localhost:XXXX/health` ตอบ `OK`
- [ ] `.mcp.json` port ตรงกับ docker-compose.yml, URL ชี้ `/mcp`
- [ ] เปิด Claude Code ใน project directory แล้วเห็น MCP tools
- [ ] ทดสอบเรียก tool อย่างน้อย 1 ตัว
