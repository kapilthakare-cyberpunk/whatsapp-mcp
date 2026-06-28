# WhatsApp MCP — Setup Guide

## Prerequisites

- Go (tested: 1.26.4)
- Python 3.6+
- uv — `curl -LsSf https://astral.sh/uv/install.sh | sh`

## Installation

### 1. Clone

```bash
git clone https://github.com/lharries/whatsapp-mcp.git ~/whatsapp-mcp
cd ~/whatsapp-mcp
```

### 2. Build the Go bridge

> **Note:** The upstream repo uses an older whatsmeow version. Update it first:

```bash
cd whatsapp-bridge
go get go.mau.fi/whatsmeow@latest
go get go.mau.fi/util/dbutil@latest
```

The following API calls in `main.go` also need a `context.Background()` argument (breaking change in newer whatsmeow):

| Line | Old | Fixed |
|------|-----|-------|
| `client.Download(...)` | `client.Download(downloader)` | `client.Download(context.Background(), downloader)` |
| `sqlstore.New(...)` | `sqlstore.New("sqlite3", ...)` | `sqlstore.New(context.Background(), "sqlite3", ...)` |
| `container.GetFirstDevice()` | `container.GetFirstDevice()` | `container.GetFirstDevice(context.Background())` |
| `client.GetGroupInfo(jid)` | `client.GetGroupInfo(jid)` | `client.GetGroupInfo(context.Background(), jid)` |
| `client.Store.Contacts.GetContact(jid)` | `...GetContact(jid)` | `...GetContact(context.Background(), jid)` |

Then build:

```bash
go build -o whatsapp-bridge .
```

### 3. Authenticate

Run the bridge — a QR code will appear on first run:

```bash
./whatsapp-bridge
```

On your phone: **WhatsApp → Settings → Linked Devices → Link a Device** → scan the QR code.

The bridge confirms auth and starts syncing messages. Keep this terminal open — the bridge must stay running.

> Re-authentication may be needed after ~20 days.

## Agent Configuration

The MCP server command for all agents:

```
command: /Users/kapilthakare/.local/bin/uv
args:    --directory /Users/kapilthakare/whatsapp-mcp/whatsapp-mcp-server run main.py
```

### Hermes (`~/.hermes/config.yaml`)

```yaml
mcp_servers:
  whatsapp:
    command: /Users/kapilthakare/.local/bin/uv
    args:
      - --directory
      - /Users/kapilthakare/whatsapp-mcp/whatsapp-mcp-server
      - run
      - main.py
```

### OpenCode (`~/.config/opencode/opencode.json`)

```json
"whatsapp": {
  "type": "local",
  "command": [
    "/Users/kapilthakare/.local/bin/uv",
    "--directory",
    "/Users/kapilthakare/whatsapp-mcp/whatsapp-mcp-server",
    "run",
    "main.py"
  ],
  "enabled": true
}
```

### Kiro (`~/.kiro/settings/mcp.json`)

```json
{
  "mcpServers": {
    "whatsapp": {
      "command": "/Users/kapilthakare/.local/bin/uv",
      "args": [
        "--directory",
        "/Users/kapilthakare/whatsapp-mcp/whatsapp-mcp-server",
        "run",
        "main.py"
      ]
    }
  }
}
```

## Running

Always start the Go bridge first, then your agent:

```bash
# Terminal 1 — keep running
cd ~/whatsapp-mcp/whatsapp-bridge && ./whatsapp-bridge

# Terminal 2 — your agent (hermes / opencode / kiro)
hermes
```

## Available MCP Tools

| Tool | Description |
|------|-------------|
| `search_contacts` | Search by name or phone number |
| `list_chats` | List all chats with metadata |
| `list_messages` | Retrieve messages with filters |
| `get_last_interaction` | Most recent message with a contact |
| `send_message` | Send a message to a number or group JID |
| `send_file` | Send image, video, document |
| `send_audio_message` | Send voice message (`.ogg` opus; FFmpeg auto-converts) |
| `download_media` | Download media from a message |

## Troubleshooting

**`client outdated (405)`** — whatsmeow version is stale. Run:
```bash
cd ~/whatsapp-mcp/whatsapp-bridge
go get go.mau.fi/whatsmeow@latest && go build -o whatsapp-bridge .
```

**Messages out of sync** — delete both DB files and re-authenticate:
```bash
rm whatsapp-bridge/store/messages.db whatsapp-bridge/store/whatsapp.db
./whatsapp-bridge
```

**Device limit reached** — remove a linked device in WhatsApp → Settings → Linked Devices.
