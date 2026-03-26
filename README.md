# wechat-access-unqclawed

An [OpenClaw](https://openclaw.ai) channel plugin that connects WeChat (WeCom) to your AI agents via QR code login, persistent token management, and real-time AGP WebSocket messaging.

## Overview

`wechat-access-unqclawed` bridges WeChat enterprise messaging and OpenClaw's agent framework. After authenticating with a WeChat QR code scan, the plugin maintains a persistent WebSocket connection using the AGP (Agent Gateway Protocol) to route incoming WeChat messages to your OpenClaw agents and stream responses back in real time. An HTTP webhook fallback is also included.

## Features

- **QR code login** — displays a scannable QR code in the terminal or provides a browser link for WeChat OAuth2 authentication
- **Token persistence** — auth state is saved to `~/.openclaw/wechat-access-auth.json` and automatically loaded on restart
- **Automatic token refresh** — JWT tokens are refreshed transparently; no manual re-login required
- **AGP WebSocket gateway** — full-duplex, streaming communication with auto-reconnect and heartbeat
- **HTTP webhook fallback** — handles encrypted and plaintext WeChat callback messages via HTTP
- **Tool call support** — streams tool invocations and status updates back to WeChat users
- **Multi-environment** — switchable between `production` and `test` gateway endpoints
- **CLI & chat commands** — `openclaw wechat login/logout` and in-chat `/wechat-login` / `/wechat-logout`

## Requirements

- **OpenClaw** >= 2026.1.26
- **Node.js** with ES2022 support

## Installation

```bash
openclaw plugins install @henryxiaoyang/wechat-access-unqclawed
openclaw config set channels.wechat-access-unqclawed.enabled true
```

## First-Time Login

Run the login command:

```bash
openclaw channels login --channel wechat-access-unqclawed
```

The terminal will display a WeChat QR code (or a browser link). Scan it with WeChat and confirm. After confirmation, WeChat redirects your browser to a new URL.

In a **second terminal window**, write the full redirect URL (or just the `code` parameter) to the temporary auth file:

```bash
echo "PASTE_URL_OR_CODE_HERE" > ~/.openclaw/wechat-auth-code.tmp
```

The login flow detects this file automatically, completes authentication, and saves the token. Then restart the gateway:

```bash
openclaw gateway restart
```

## Token Resolution Order

1. `channels.wechat-access-unqclawed.token` in config — used directly if set
2. Saved state at `~/.openclaw/wechat-access-auth.json` — loaded automatically
3. Neither present — run `openclaw channels login --channel wechat-access-unqclawed`

## Configuration

Add the following block to your OpenClaw config file under `channels.wechat-access-unqclawed`:

```json
{
  "channels": {
    "wechat-access-unqclawed": {
      "enabled": true,
      "token": "",
      "wsUrl": "",
      "bypassInvite": false,
      "environment": "production",
      "authStatePath": ""
    }
  }
}
```

| Field | Type | Description |
|-------|------|-------------|
| `enabled` | `boolean` | Enable the channel (must be `true`) |
| `token` | `string` | Manually supply a channel token; skips QR login when set |
| `wsUrl` | `string` | Override the WebSocket gateway URL; uses the environment default if empty |
| `bypassInvite` | `boolean` | Skip invite-code verification |
| `environment` | `string` | `"production"` (default) or `"test"` |
| `authStatePath` | `string` | Custom path for token persistence file |

## Protocol

The plugin communicates over the **AGP (Agent Gateway Protocol)** — a JSON message protocol carried over WebSocket text frames. See [`websocket.md`](./websocket.md) for the full specification.

## Project Structure

```
index.ts                 # Plugin entry point — registers channel, CLI commands, WebSocket lifecycle
auth/
  types.ts               # Auth type definitions
  environments.ts        # Production / test endpoint configs
  device-guid.ts         # Device GUID generation and persistence
  qclaw-api.ts           # QClaw JPRX gateway API client
  state-store.ts         # Token persistence (0o600 permissions)
  wechat-login.ts        # Interactive QR login orchestration
  wechat-qr-poll.ts      # QR code generation and polling
websocket/
  types.ts               # AGP protocol types
  websocket-client.ts    # WebSocket client (connect, heartbeat, reconnect)
  message-handler.ts     # Message dispatch to OpenClaw agents
  message-adapter.ts     # AGP ↔ OpenClaw message format conversion
common/
  runtime.ts             # OpenClaw runtime singleton
  agent-events.ts        # Agent event subscription
  message-context.ts     # Message context builder
http/                    # HTTP webhook channel (fallback)
```

## Dependencies

| Package | Purpose |
|---------|---------|
| `ws` | WebSocket client |
| `undici` | HTTP client |
| `qrcode-terminal` | QR code rendering in terminal |
| `fast-xml-parser` | XML parsing for legacy services |
| `zod` | Schema validation |

## License

MIT
