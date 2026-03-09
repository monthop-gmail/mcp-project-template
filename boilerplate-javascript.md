# Boilerplate: JavaScript ES Modules MCP Server

สำหรับ project ขนาดเล็ก-กลาง (2-10 tools) ที่ไม่ต้องการ TypeScript
เปลี่ยน `xxx` เป็นชื่อ project จริง

---

## 1. package.json

```json
{
  "name": "xxx-mcp-claude",
  "version": "1.0.0",
  "description": "MCP Server for [description]",
  "type": "module",
  "main": "src/index.js",
  "scripts": {
    "start": "node src/index.js",
    "start:sse": "node src/server-sse.js"
  },
  "dependencies": {
    "@modelcontextprotocol/sdk": "^1.26.0",
    "dotenv": "^16.3.0"
  }
}
```

---

## 2. src/index.js (Stdio)

```javascript
#!/usr/bin/env node
import { Server } from '@modelcontextprotocol/sdk/server/index.js';
import { StdioServerTransport } from '@modelcontextprotocol/sdk/server/stdio.js';
import {
  ListToolsRequestSchema,
  CallToolRequestSchema,
} from '@modelcontextprotocol/sdk/types.js';
import { TOOLS, handleToolCall } from './tools.js';

const server = new Server(
  { name: 'xxx-mcp-claude', version: '1.0.0' },
  { capabilities: { tools: {} } }
);

server.setRequestHandler(ListToolsRequestSchema, async () => ({
  tools: TOOLS,
}));

server.setRequestHandler(CallToolRequestSchema, async (request) => {
  const { name, arguments: args } = request.params;
  try {
    const result = await handleToolCall(name, args || {});
    return {
      content: [{ type: 'text', text: JSON.stringify(result, null, 2) }],
    };
  } catch (error) {
    return {
      content: [
        { type: 'text', text: JSON.stringify({ error: error.message }, null, 2) },
      ],
      isError: true,
    };
  }
});

async function main() {
  const transport = new StdioServerTransport();
  await server.connect(transport);
  console.error(`xxx MCP Server running on stdio (${TOOLS.length} tools)`);
}

main().catch((err) => {
  console.error('Server error:', err);
  process.exit(1);
});
```

---

## 3. src/server-sse.js (Streamable HTTP/Docker)

```javascript
#!/usr/bin/env node
import http from 'http';
import crypto from 'crypto';
import { Server } from '@modelcontextprotocol/sdk/server/index.js';
import { StreamableHTTPServerTransport } from '@modelcontextprotocol/sdk/server/streamableHttp.js';
import {
  ListToolsRequestSchema,
  CallToolRequestSchema,
  isInitializeRequest,
} from '@modelcontextprotocol/sdk/types.js';
import { TOOLS, handleToolCall } from './tools.js';

const PORT = parseInt(process.env.PORT || '3000', 10);
const HOST = process.env.HOST || '0.0.0.0';

function createMCPServer() {
  const mcpServer = new Server(
    { name: 'xxx-mcp-claude', version: '1.0.0' },
    { capabilities: { tools: {} } }
  );

  mcpServer.setRequestHandler(ListToolsRequestSchema, async () => ({
    tools: TOOLS,
  }));

  mcpServer.setRequestHandler(CallToolRequestSchema, async (request) => {
    const { name, arguments: args } = request.params;
    try {
      const result = await handleToolCall(name, args || {});
      return {
        content: [{ type: 'text', text: JSON.stringify(result, null, 2) }],
      };
    } catch (error) {
      return {
        content: [
          { type: 'text', text: JSON.stringify({ error: error.message }, null, 2) },
        ],
        isError: true,
      };
    }
  });

  return mcpServer;
}

const transports = new Map();

const httpServer = http.createServer(async (req, res) => {
  res.setHeader('Access-Control-Allow-Origin', '*');
  res.setHeader('Access-Control-Allow-Methods', 'GET, POST, DELETE, OPTIONS');
  res.setHeader('Access-Control-Allow-Headers', 'Content-Type, mcp-session-id');
  res.setHeader('Access-Control-Expose-Headers', 'mcp-session-id');

  if (req.method === 'OPTIONS') { res.writeHead(204); res.end(); return; }

  const url = new URL(req.url || '/', `http://${req.headers.host}`);

  if (url.pathname === '/health') {
    res.writeHead(200, { 'Content-Type': 'application/json' });
    res.end(JSON.stringify({ status: 'ok', server: 'xxx-mcp-claude', tools: TOOLS.length }));
    return;
  }

  if (url.pathname === '/mcp') {
    if (req.method === 'POST') {
      let body = '';
      req.on('data', (chunk) => { body += chunk; });
      req.on('end', async () => {
        try {
          const jsonBody = JSON.parse(body);
          const sessionId = req.headers['mcp-session-id'];
          let transport;

          if (sessionId && transports.has(sessionId)) {
            transport = transports.get(sessionId);
          } else if (!sessionId && isInitializeRequest(jsonBody)) {
            transport = new StreamableHTTPServerTransport({
              sessionIdGenerator: () => crypto.randomUUID(),
              onsessioninitialized: (sid) => { transports.set(sid, transport); },
            });
            transport.onclose = () => {
              const sid = transport.sessionId;
              if (sid) transports.delete(sid);
            };
            const server = createMCPServer();
            await server.connect(transport);
          } else {
            res.writeHead(400, { 'Content-Type': 'application/json' });
            res.end(JSON.stringify({ error: 'Bad request: no session ID or not an initialize request' }));
            return;
          }
          await transport.handleRequest(req, res, jsonBody);
        } catch (err) {
          if (!res.headersSent) {
            res.writeHead(500, { 'Content-Type': 'application/json' });
            res.end(JSON.stringify({ error: String(err) }));
          }
        }
      });
      return;
    }

    if (req.method === 'GET') {
      const sessionId = req.headers['mcp-session-id'];
      const transport = sessionId && transports.get(sessionId);
      if (transport) {
        await transport.handleRequest(req, res);
      } else {
        res.writeHead(400, { 'Content-Type': 'application/json' });
        res.end(JSON.stringify({ error: 'Invalid or missing session ID' }));
      }
      return;
    }

    if (req.method === 'DELETE') {
      const sessionId = req.headers['mcp-session-id'];
      const transport = sessionId && transports.get(sessionId);
      if (transport) {
        await transport.handleRequest(req, res);
      } else {
        res.writeHead(400, { 'Content-Type': 'application/json' });
        res.end(JSON.stringify({ error: 'Invalid or missing session ID' }));
      }
      return;
    }
  }

  res.writeHead(404); res.end(JSON.stringify({ error: 'Not found' }));
});

httpServer.listen(PORT, HOST, () => {
  console.log(`xxx MCP Server (Streamable HTTP) at http://${HOST}:${PORT}`);
  console.log(`  MCP: /mcp | Health: /health | Tools: ${TOOLS.length}`);
});
```

---

## 4. src/tools.js (Tools + Dispatcher)

```javascript
export const TOOLS = [
  {
    name: 'example_tool',
    description: 'Does something useful',
    inputSchema: {
      type: 'object',
      properties: {
        query: { type: 'string', description: 'Input query' },
        limit: { type: 'number', description: 'Max results (default: 10)' },
      },
      required: ['query'],
    },
  },
  // ... more tools
];

export async function handleToolCall(name, args) {
  switch (name) {
    case 'example_tool':
      return exampleTool(args);
    default:
      throw new Error(`Unknown tool: ${name}`);
  }
}

async function exampleTool({ query, limit = 10 }) {
  // Business logic
  return { query, limit, results: [] };
}
```

---

## 5. Dockerfile (Single-stage)

```dockerfile
FROM node:22-slim

# เพิ่ม dependencies ตามต้องการ
RUN apt-get update && apt-get install -y wget && rm -rf /var/lib/apt/lists/*

WORKDIR /app

COPY package*.json ./
RUN npm install --production

COPY src/ ./src/

RUN mkdir -p /app/data

EXPOSE 3000

CMD ["node", "src/server-sse.js"]
```

---

## 6. docker-compose.yml

```yaml
services:
  xxx-mcp:
    build: .
    container_name: xxx-mcp-claude
    restart: unless-stopped
    ports:
      - "XXXX:3000"
    volumes:
      - ./data:/app/data
    environment:
      - PORT=3000
      - HOST=0.0.0.0
    healthcheck:
      test: ["CMD", "wget", "-q", "--spider", "http://localhost:3000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
```

---

## 7. .mcp.json

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

## 8. .gitignore

```
node_modules/
.env
data/*.db
*.log
```

---

## 9. .dockerignore

```
node_modules
npm-debug.log
.git
*.md
!README.md
*.db
.env
.DS_Store
```
