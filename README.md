# TVCMall Customer MCP

A remote, read-only [Model Context Protocol](https://modelcontextprotocol.io/) server that lets AI coding assistants query your own TVCMall account: products, shipping estimates, orders, tracking, points, and balance.

**You do not install anything.** This is a hosted service. You point your MCP client at one HTTPS URL and supply a personal access token.

| | |
| --- | --- |
| **Endpoint** | `https://openai.tvc-mall.com/mcp` |
| **Transport** | Streamable HTTP |
| **Protocol revision** | `2026-07-28` |
| **Authentication** | Personal PAT in the `TVCMALL_API_KEY` request header |
| **Access** | Read-only. No writes, no file export. |

---

## What is MCP?

The Model Context Protocol is an open standard that lets AI assistants call external tools on your behalf. Instead of copying data into a chat window, you connect an MCP server once and the assistant can query that system directly.

An MCP **server** (this service) exposes a set of **tools**. An MCP **client** (Claude Code, Cursor, VS Code, Codex, …) connects to the server, discovers those tools, and lets the model call them.

Two connection styles exist:

- **Local (stdio)** — the client starts a program on your machine. Used for tools that need local filesystem access.
- **Remote (Streamable HTTP)** — the client talks to a hosted HTTPS endpoint.

This service is **remote**: TVCMall hosts and operates it. Your client needs only a URL and a token, and your PAT never leaves your machine except as a request header to TVCMall's own endpoint.

---

## Before you start

### 1. Get a personal access token (PAT)

Request a PAT from a TVCMall administrator through a secure channel. It looks like:

```text
tmcp_v1_{tokenId}.{secret}
```

**Use your own PAT.** Do not share it and do not configure a company-wide token: every request is authorized as *you*, and the TVCMall backend decides what you may read.

### 2. Check your client

This server implements protocol revision **`2026-07-28`** and does not serve older revisions. A client that speaks only `2025-11-25` or earlier will be rejected with:

```json
{"error":{"code":-32022,"message":"Unsupported protocol version: 2025-11-25",
          "data":{"supported":["2026-07-28"]}}}
```

If you see that, update the client.

---

## Client setup

| Client | Supported | How |
| --- | --- | --- |
| [Claude Code](#claude-code) | ✅ | CLI command |
| [Cursor](#cursor) | ✅ | `~/.cursor/mcp.json` |
| [VS Code / GitHub Copilot](#vs-code--github-copilot) | ✅ | `mcp.json` with an input variable |
| [Codex](#codex) | ✅ | `~/.codex/config.toml` |
| [Windsurf](#windsurf) | ⚠️ | Config path documented; verify the header syntax in your build |
| Claude Desktop | ❌ | Its connector UI cannot send the required header — use Claude Code instead |

Replace `<TVCMALL_PAT>` below with your own token, including removing the angle brackets. Never commit a file containing a real PAT.

### Claude Code

```bash
claude mcp add --transport http --scope user tvcmall \
  'https://openai.tvc-mall.com/mcp' \
  --header 'TVCMALL_API_KEY: <TVCMALL_PAT>'
```

`--scope user` makes the server available in every project. Then restart Claude Code, run `/mcp`, and confirm `tvcmall` shows as connected.

> ⚠️ The command line is recorded in your shell history. Do not run it in a shared terminal or while screen-recording.

To change the token later: `claude mcp remove tvcmall`, then add it again.

Reference: [Claude Code MCP documentation](https://code.claude.com/docs/en/mcp).

### Cursor

Edit your **user-level** file (not a project-level one):

- macOS / Linux: `~/.cursor/mcp.json`
- Windows: `%USERPROFILE%\.cursor\mcp.json`

```json
{
  "mcpServers": {
    "tvcmall": {
      "url": "https://openai.tvc-mall.com/mcp",
      "headers": {
        "TVCMALL_API_KEY": "<TVCMALL_PAT>"
      }
    }
  }
}
```

If the file already has an `mcpServers` object, merge just the `tvcmall` entry — do not replace your other servers. Save, quit Cursor completely, and restart it.

Reference: [Cursor MCP documentation](https://cursor.com/docs/mcp).

### VS Code / GitHub Copilot

VS Code can prompt for the token and store it securely instead of keeping it in the file. Add `.mcp.json` at your workspace root (portable format) **or** use `mcp.json` in your user profile via **MCP: Open User Configuration**:

```json
{
  "inputs": [
    {
      "type": "promptString",
      "id": "tvcmall-pat",
      "description": "TVCMall personal access token",
      "password": true
    }
  ],
  "servers": {
    "tvcmall": {
      "type": "http",
      "url": "https://openai.tvc-mall.com/mcp",
      "headers": {
        "TVCMALL_API_KEY": "${input:tvcmall-pat}"
      }
    }
  }
}
```

VS Code asks for the token the first time the server starts, then stores the value securely. Note the two container keys: the portable `.mcp.json` uses `mcpServers`, while VS Code's own `.vscode/mcp.json` uses `servers`.

Reference: [VS Code MCP configuration](https://code.visualstudio.com/docs/agent-customization/mcp-servers).

### Codex

Edit `~/.codex/config.toml` (Windows: `%USERPROFILE%\.codex\config.toml`):

```toml
[mcp_servers.tvcmall]
url = "https://openai.tvc-mall.com/mcp"
http_headers = { "TVCMALL_API_KEY" = "<TVCMALL_PAT>" }
```

If `[mcp_servers.tvcmall]` already exists, update it rather than adding a second entry. Restart Codex, then verify:

```bash
codex mcp list
```

Reference: [Codex MCP documentation](https://learn.chatgpt.com/docs/extend/mcp).

### Windsurf

Windsurf reads `~/.codeium/windsurf/mcp_config.json`. Its remote-server shape mirrors Cursor's, but confirm the `headers` field name in your build before relying on it:

```json
{
  "mcpServers": {
    "tvcmall": {
      "serverUrl": "https://openai.tvc-mall.com/mcp",
      "headers": {
        "TVCMALL_API_KEY": "<TVCMALL_PAT>"
      }
    }
  }
}
```

### Claude Desktop

Claude Desktop's custom connector interface accepts only a URL plus OAuth, so it **cannot send the `TVCMALL_API_KEY` header** this service requires. Use Claude Code, Cursor, VS Code, or Codex. Do not try to work around this by putting the PAT in the URL — the server rejects credentials in the query string.

---

## Verify the connection

Once your client reports the server as connected, try:

```text
Check whether the TVCMall MCP is configured, then show my 10 most recent orders
in descending creation-time order.
```

The client usually calls `tvcmall_auth_status` first. A result of `configured: true` means only that the request carried a PAT — it does **not** mean TVCMall has validated it. Your credentials are proven usable only when a read-only query actually succeeds.

This service is **stateless**: every request carries your PAT and no session is created. There is no session ID to save, and restarting your client or the service just means reconnecting.

---

## What you can query

| Tool | What it returns |
| --- | --- |
| `tvcmall_auth_status` | Whether the request carried a PAT |
| `tvcmall_search_products` | Product search by SKU, keyword, or listing-date range, including new-arrival counts |
| `tvcmall_get_product_filters` | The exact filter Codes available for a search result |
| `tvcmall_get_product_detail` | SKU, price, stock, and attributes for one product |
| `tvcmall_estimate_shipping` | Shipping estimate for products you have not ordered |
| `tvcmall_list_orders` | Order search by date, keyword, status, and sort order |
| `tvcmall_get_order_detail` | Items, amounts, and backend-masked delivery information |
| `tvcmall_get_tracking_info` | Tracking events and shipping cost for one order |
| `tvcmall_batch_get_tracking` | Tracking and shipping cost for several orders |
| `tvcmall_get_points` | Remaining points |
| `tvcmall_get_balance` | Current balance |
| `tvcmall_list_balance_records` | Balance transaction history |

Example prompts:

- "How many products were added today?"
- "Find black cases for iPhone 17 Pro" — the client resolves the real `Black` filter Code instead of guessing
- "Estimate shipping to the United States for 20 units of SKU 100100"
- "Find orders containing item 661100432A"
- "Show tracking and shipping costs for order V24011000008"
- "Show only my balance deductions"

Order-list queries expose these fields directly: `begindate`, `enddate`, `keywordinfo`, `pageindex`, `pagesize`, `sort`, `status`, and `withdetail`. In `keywordinfo.key`, `1` searches order number, `2` title, `3` item number, `4` receiver, `5` phone, `6` shipping address. A "most recent" request uses `sort=createddown`; set `withdetail=true` to include item details on every order.

**All capabilities are read-only.** The service cannot place orders, take payment, change addresses, cancel orders, redeem points, or export files.

---

## Security notes

- Send the complete PAT as the `TVCMALL_API_KEY` value. **Do not add a `Bearer ` prefix** — that is rejected.
- Do not configure an `Authorization` header, and do not send both credential types.
- Every user must use their own PAT. Do not share tokens or configure a server-wide credential.
- Do not use your TVCMall website password, website login token, or an OAuth token in place of a PAT.
- Connect only over HTTPS. Never send a PAT to a plaintext endpoint.
- Most clients store the PAT in a plain-text config file. Restrict access to that file and avoid shared devices. VS Code's `${input:...}` approach stores it more safely.
- Never put a PAT in a URL, chat message, log, screenshot, or Git commit.
- If you believe a PAT has leaked, contact a TVCMall administrator immediately to revoke and reissue it.

---

## Troubleshooting

| Symptom | Cause and fix |
| --- | --- |
| Client cannot find `tvcmall` | Check the URL ends in `/mcp`, the header name and value are complete, then fully restart the client. |
| `401 AUTH_REQUIRED` | No PAT on the request, or the PAT is malformed, expired, or revoked. Confirm the header is present and has no `Bearer ` prefix. |
| `400` with `-32022 Unsupported protocol version` | The client speaks an older MCP revision. This server serves `2026-07-28` only — upgrade the client. |
| `400` with `-32020 HeaderMismatch` | The client's request headers disagree with the request body. Update to a conforming `2026-07-28` client. |
| `403 ORIGIN_NOT_ALLOWED` | The request carried an `Origin` header the server does not allow. Non-browser MCP clients do not send `Origin`; check for a proxy that adds one. |
| `403 PERMISSION_DENIED` | Your PAT lacks the required permission, or the route is not enabled for it. Ask a TVCMall administrator to check your access. |
| `429 RATE_LIMITED` | Requests are being sent too frequently. Wait and retry. |
| `429 REQUEST_CAPACITY_REACHED` | The service is at its concurrent-request limit. Retry shortly. |
| `API_UNAVAILABLE` | The TVCMall backend is temporarily unavailable, the network failed, or the request timed out. Retry; contact an administrator if it persists. |
| Claude Desktop cannot connect | Its connector UI cannot send `TVCMALL_API_KEY`. Use Claude Code, Cursor, VS Code, or Codex. |

---

## About this repository

This repository contains **documentation only**. It is the public reference for the hosted TVCMall Customer MCP service.

- The server implementation is proprietary and is not published here.
- There is no source code to install, build, or audit in this repository.
- The service is discoverable in the [official MCP Registry](https://registry.modelcontextprotocol.io/) as `com.tvc-mall/customer-mcp`.

For access, a PAT, or a defect report, contact a TVCMall administrator.
