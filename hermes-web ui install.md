---
name: install-hermes-web-ui
description: "Install and deploy Hermes Web UI (hermes-web-ui) — the official web dashboard for Hermes Agent"
version: 2.0.0
author: Hermes Agent
tags: [hermes, web-ui, dashboard, deployment, npm]
---

# Install Hermes Web UI

Full-featured web dashboard for Hermes Agent by EKKOLearnAI.  
GitHub: https://github.com/EKKOLearnAI/hermes-web-ui

## Quick Install

### Option A: npm (recommended)

```bash
npm install -g hermes-web-ui
hermes-web-ui start
```

### Option B: One-click auto-setup (auto-installs Node.js)

```bash
bash <(curl -fsSL https://cdn.jsdelivr.net/gh/EKKOLearnAI/hermes-web-ui@main/scripts/setup.sh)
hermes-web-ui start
```

### Option C: Docker

```bash
WEBUI_IMAGE=ekkoye8888/hermes-web-ui docker compose up -d
```

## Access

Default URL: `http://localhost:8648`

### Remote Access (Two-Layer Firewall)

Cloud servers typically have two firewalls. Both must be open:

1. **Cloud security group** (e.g., Alibaba Cloud / AWS / GCP):
   - Add inbound TCP rule for port 8648
   - Restrict source IP to your IP for security: `YOUR_IP/32`
   
2. **Server firewall (UFW)**:
   ```bash
   ufw allow 8648/tcp comment "Hermes Web UI"
   ```

The Web UI binds to `0.0.0.0:8648` by default (change with `BIND_HOST` env var).
SSH port forwarding alternative: `ssh -L 8648:localhost:8648 user@host`

## Authentication

On first start, an auth token is auto-generated and saved to `~/.hermes-web-ui/.token`.

```bash
# View current token
cat ~/.hermes-web-ui/.token

# Set custom token
echo -n "your-custom-token" > ~/.hermes-web-ui/.token
hermes-web-ui restart

# Disable auth (NOT recommended for public access)
export AUTH_DISABLED=true
hermes-web-ui restart
```

## Commands

| Command | Description |
|---------|-------------|
| `hermes-web-ui start` | Start in background (daemon) |
| `hermes-web-ui stop` | Stop background process |
| `hermes-web-ui restart` | Restart |
| `hermes-web-ui status` | Check running status |
| `hermes-web-ui update` | Update to latest version |
| `hermes-web-ui -v` | Show version |

## Chat Mode: Bridge vs API

The Web UI has two chat modes:

- **API mode** (default): Uses Hermes Gateway API Server (port 8642) — designed for external clients (Open WebUI, curl, etc.)
- **Bridge (beta) mode**: Uses in-process Python agent bridge — faster, real-time streaming, recommended for chatting in the Web UI itself

To switch: Click the ▼ arrow next to the "+" new chat button and select "Bridge (beta)".

## Troubleshooting

### Web UI starts but can't access from browser
- **Local access only:** Use SSH port forwarding: `ssh -L 8648:localhost:8648 user@host`
- **Server firewall:** Check UFW: `ufw status` → add rule: `ufw allow 8648/tcp`
- **Cloud firewall:** Check security group inbound rules for port 8648

### Chat messages not responding
Most likely using **API mode** instead of **Bridge (beta)** mode. Switch to Bridge mode (see above).  
If Bridge mode also fails, restart the Web UI: `hermes-web-ui restart`

### Python Socket.IO client can't send events
**Known issue**: `python-socketio` (any version) connects to `/group-chat` successfully but events (`join`, `message`) are silently rejected — the server sends a CLOSE packet immediately.  
**Fix**: Always use the **Node.js `socket.io-client`** library for external agents connecting to the Web UI group chat. Python is not compatible with this server.

### Node.js not installed
The setup script handles this automatically. Or install manually:
```bash
curl -fsSL https://deb.nodesource.com/setup_lts.x | bash -
apt-get install -y nodejs
```

## Group Chat Architecture

The Web UI has a built-in **group chat** module (Socket.IO namespace `/group-chat`) that supports multi-user + multi-Agent rooms.

### Core Concepts

- **Room**: A chat room identified by UUID. Has configurable `triggerTokens` (default 100000), `maxHistoryTokens` (32000), `tailMessageCount` (10).
- **Member**: Human users connected via browser. Each has userId, userName, description.
- **Agent**: AI bot tied to a Hermes Profile. Runs as an independent process (Node.js) that maintains a persistent WebSocket connection.
- **@mention routing**: Only messages containing `@AgentName` trigger that agent to respond. Agents ignore messages not addressed to them.
- **Context management**: When `triggerTokens` is exceeded, the room's history is automatically summarized via LLM and stored as a compression snapshot in SQLite.

### Data Persistence (SQLite)

All group chat data stored in `~/.hermes-web-ui/hermes-web-ui.db`:

| Table | Purpose |
|-------|---------|
| `gc_rooms` | Room config (name, inviteCode, compression thresholds) |
| `gc_messages` | Chat messages indexed by room + timestamp |
| `gc_room_members` | Room members (user) |
| `gc_room_agents` | Agent config (bound Hermes profile) |
| `gc_context_snapshots` | Compressed context summaries |
| `gc_session_profiles` | Agent-to-Hermes-session bindings |

### External Agent Architecture

External AI agents connect to the group chat via **Socket.IO** as independent processes, NOT through the Web UI's built-in Agent management.

```
┌── Hermes Web UI ──────────────────┐
│  Group Chat Service (port 8648)    │
│    /group-chat namespace           │
└────────┬───────────────────────────┘
         │ WebSocket (Socket.IO v4)
    ┌────┴────┐
    │ External │  ← Any machine, any language
    │ Agent    │     with a Socket.IO client
    └─────────┘
```

**Critical compatibility note**: The Web UI uses `socket.io` v4.8.3 (Node.js). The `python-socketio` library (tested v5.16.1) connects successfully but its `emit` events (join, message) are silently rejected — the server sends a CLOSE packet immediately after receiving events from Python clients. **Always use the Node.js `socket.io-client` library** for external agents connecting to Hermes Web UI group chat.

### External Agent Flow

1. Agent connects to `http://<host>:8648/group-chat` via Socket.IO WebSocket
2. Authenticates with `auth: { token, name, description }`
3. Emits `join` event with callback: `sio.emit("join", { roomId, name }, callback)`
4. Listens for `message` events from the room
5. When a message contains `@AgentName`, the agent processes it and replies by emitting a `message` event back
6. Supports `typing` and `stop_typing` events for real-time indicators

### Reference: external-agent.md

See `references/external-agent.md` for the full deployment guide with a production-ready agent script. The template script is also available at `templates/external-agent.js` for quick customization.

### Template: external-agent.js

A ready-to-use Node.js script is available at `templates/external-agent.js` in this skill. 
Copy it, edit the CONFIG section, and run:

```bash
cd /path/to/your/project
cp <skill-dir>/templates/external-agent.js .
npm install socket.io-client
node external-agent.js
```

Key config values to set:
- `WEB_UI_URL` — Public URL of the Web UI (e.g., `http://47.98.123.49:8648`)
- `AUTH_TOKEN` — From `~/.hermes-web-ui/.token` on the Web UI server
- `AGENT_NAME` — What users type after `@` to summon this agent
- `ROOM_ID` — The target room UUID (from `sqlite3 ~/.hermes-web-ui/hermes-web-ui.db "SELECT id, name FROM gc_rooms"`)
- `BACKEND_API` — URL of the backend that generates responses (Hermes Gateway, OpenAI, etc.)

### Process Management

```bash
# Start in background
nohup node external-agent.js > /tmp/agent.log 2>&1 &

# View live logs
tail -f /tmp/agent.log

# Stop
pkill -f external-agent.js
```

## Env Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `PORT` | `8648` | Web UI listen port |
| `BIND_HOST` | `0.0.0.0` | Bind address |
| `HERMES_WEB_UI_HOME` | `~/.hermes-web-ui` | Data directory |
| `AUTH_TOKEN` | auto-generated | Bearer token |
| `AUTH_DISABLED` | unset | Set to `1` to disable auth |
| `PROFILE` | `default` | Hermes profile name |
