# mcp-project-template

Template และ boilerplate สำหรับสร้าง MCP Server project ใหม่ (`xxx-mcp-claude`) ให้ใช้กับ Claude Code

สรุป pattern จาก 9 MCP projects ที่ active อยู่:
chat, iot, line-oa, odoo, rag, samathi101, thudong, transcript, youtuber

## Files

| File | Description |
|------|-------------|
| `mcp-project-template.md` | Prompt template - ใช้เป็น spec สำหรับสั่ง Claude Code สร้าง project ใหม่ (port convention, tech stack, โครงสร้าง, build order, checklist) |
| `boilerplate-typescript.md` | Code boilerplate สำหรับ TypeScript project (10+ tools) - package.json, tsconfig, config, tools, index, server-sse, Dockerfile multi-stage, docker-compose, cache |
| `boilerplate-javascript.md` | Code boilerplate สำหรับ JavaScript project (2-10 tools) - single-stage Dockerfile, simpler structure |
| `boilerplate-python.md` | Code boilerplate สำหรับ Python project - pyproject.toml, Starlette + Uvicorn, StreamableHTTPSessionManager, service layer pattern |

## วิธีใช้

1. Copy `mcp-project-template.md` ไปเป็น prompt spec ของ project ใหม่
2. แก้ `xxx` เป็นชื่อ project, เติม tools, เลือก port ที่ว่าง
3. สั่ง Claude Code ให้อ่าน prompt spec แล้วสร้าง project ตาม template
4. Claude Code จะอ้างอิง boilerplate (TypeScript / JavaScript / Python) เพื่อสร้างไฟล์ตาม pattern เดียวกันทุก project

## Port Convention (ที่ใช้แล้ว)

| Port | Project | หมายเหตุ |
|------|---------|----------|
| 3000 | samathi101-mcp-claude | |
| 3001 | chat-mcp-claude | |
| ~~3010~~ | ~~youtube-mcp-claude~~ | *รวมเข้า transcript แล้ว (-obs)* |
| ~~3011~~ | ~~audio-mcp-claude~~ | *รวมเข้า transcript แล้ว (-obs)* |
| ~~3012~~ | ~~vdo-mcp-claude~~ | *รวมเข้า transcript แล้ว (-obs)* |
| 3013 | transcript-mcp-claude | All-in-One: YouTube+Audio+Video (19 tools) |
| 3020 | line-oa-mcp-claude | |
| 3030 | youtuber-mcp-claude | |
| ~~3100~~ | ~~esxi-mcp-claude~~ | *ย้ายไป obs แล้ว (-obs)* |
| 3200 | thudong-mcp-claude | |
| 3300 | iot-mcp-claude | |
| 8000 | odoo-mcp-claude | Python |
| 8100 | rag-mcp-claude | |

## Common Pattern

ทุก project ใช้ pattern เดียวกัน:
- **Dual transport**: stdio (`index.ts/js`) + Streamable HTTP (`server-sse.ts/js`)
- **Docker**: Streamable HTTP mode, unique port, healthcheck `/health`
- **Claude Code**: `.mcp.json` ชี้ไป `http://localhost:PORT/mcp`
- **Restart**: `unless-stopped`
- **Logging**: stderr (stdout สงวนสำหรับ MCP protocol)
