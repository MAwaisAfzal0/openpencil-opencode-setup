# OpenPencil ⇄ OpenCode MCP (that stays connected)

A working, tested integration between the [OpenPencil](https://openpencil.dev) desktop app and [OpenCode](https://opencode.ai) — using OpenPencil's official **stdio bridge** instead of its HTTP endpoint.

Official OpenPencil MCP docs: https://openpencil.dev/programmable/mcp-server

## Why this exists

OpenPencil ships an MCP server with two transports: **stdio** (for MCP clients) and **Streamable HTTP** (for browsers, scripts, CI).

OpenCode's `type: "remote"` MCP client connects over **SSE**, which mismatches the Streamable-HTTP server. The connection works until the SSE stream drops once (app restart, machine sleep, close/reopen). Then OpenCode retries only twice, gives up, and marks the server as `failed` **for the rest of the session** — even while OpenPencil is running again. The integration looks permanently broken.

The fix: stop using the remote endpoint and let OpenCode spawn OpenPencil's official stdio bridge (`openpencil-mcp`). The bridge:

- retries the app connection every 2 seconds, forever — and reconnects automatically when OpenPencil comes back;
- never drops its MCP session, so the server never flips to `failed`;
- returns a clear per-call message when the app (or a document) is missing, instead of a transport error.

## Setup

1. Install the bridge:

   ```sh
   npm install -g @open-pencil/mcp
   ```

2. Add the server to your OpenCode config (`opencode.json(c)`) — see [`opencode.jsonc`](./opencode.jsonc) in this repo:

   ```jsonc
   "mcp": {
     "open-pencil": {
       "type": "local",
       "command": [
         "C:\\Program Files\\nodejs\\node.exe",
         "C:\\Users\\<you>\\AppData\\Roaming\\npm\\node_modules\\@open-pencil\\mcp\\dist\\stdio.mjs"
       ],
       "enabled": true
     }
   }
   ```

   Adjust the two paths for your machine (`where node` finds node.exe; the bridge lives under your global npm root — `npm root -g` prints it).

3. Start OpenCode (recent versions hot-reload the config), then open OpenPencil **with a document loaded**.

## Verify it works

- OpenCode's log shows a line like `mcp connected server=open-pencil tools=132` (tool count varies by app version)
- Ask the agent to run a tool, e.g. `list_documents` — it should return your open document.
- Close OpenPencil, run a tool again — you get a clean "OpenPencil app is not connected…" message and the MCP connection stays healthy.
- Reopen OpenPencil — tools work again **without restarting OpenCode**.

## Troubleshooting

- **"OpenPencil app is not connected"** — the app isn't running, or has no document open. Open a document and retry.
- **Port 7600 busy / stale server** — force-killing the app can orphan its HTTP server (`node ...\@open-pencil\mcp\dist\index.mjs`). It exits on its own; if not, end that node process and relaunch OpenPencil.
- **Keep the npm package stock** — the app spawns its HTTP server from the globally installed `@open-pencil/mcp`. Restore with `npm i -g @open-pencil/mcp@latest` if it ever gets modified.
- **File-scoped tools** are limited to `OPENPENCIL_MCP_ROOT` (defaults to your home directory on Windows). Set it in the server's `environment` if your designs live elsewhere.

## Files

- [`opencode.jsonc`](./opencode.jsonc) — example OpenCode config with the `open-pencil` stdio entry
- [`examples/mcp.json`](./examples/mcp.json) — shape reference for the discovery file OpenPencil writes. Your real discovery file can contain a plaintext auth token when auth is enabled — don't publish that one.

## Tested with

- OpenPencil 0.16.0 (Windows)
- `@open-pencil/mcp` 0.16.0
- OpenCode 2.0.32
