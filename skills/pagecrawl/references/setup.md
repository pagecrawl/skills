# Connect PageCrawl

Read this when no PageCrawl tools are available, when they fail to connect, or when they appear twice.

## What to connect

- Server URL: `https://mcp.pagecrawl.io/mcp`, exactly as written (no trailing slash, no `/sse`).
- Transport: streamable HTTP. A client set to SSE fails to connect.
- Sign-in: OAuth is the default. The client opens PageCrawl, the user signs in and clicks Allow access (or Allow with the app's name), and no key is copied.
- Clients without OAuth can send a personal API token instead: `Authorization: Bearer <api-token>`. The user creates it in PageCrawl under Settings > API access > Create Token. Tokens work on every plan, and a token can be given an expiry date.

Never ask the user to paste a token into the conversation. They put it in their client's configuration, or in an environment variable the configuration reads.

## Common clients

Agents with plugin support can install the PageCrawl plugin, which includes this connection and these skills: see https://github.com/pagecrawl/skills. With the plugin installed, do not add the server again.

Command-line agents:

```bash
# Claude Code (skip this when the PageCrawl plugin is installed)
claude mcp add --scope user --transport http pagecrawl https://mcp.pagecrawl.io/mcp

# Codex: add the server; it opens the PageCrawl sign-in when it detects OAuth
codex mcp add pagecrawl --url https://mcp.pagecrawl.io/mcp
# Only if no sign-in opened:
codex mcp login pagecrawl

# OpenClaw (API token)
openclaw mcp add pagecrawl --url https://mcp.pagecrawl.io/mcp \
  --transport streamable-http \
  --header "Authorization=Bearer <api-token>"
```

Editors that read an `mcpServers` JSON file (Cursor, Windsurf, Cline and others):

```json
{
  "mcpServers": {
    "pagecrawl": {
      "url": "https://mcp.pagecrawl.io/mcp"
    }
  }
}
```

Add `"headers": {"Authorization": "Bearer <api-token>"}` beside `url` when the client does not support OAuth. Cline reads an entry without a `type` as SSE, so add `"type": "streamableHttp"` there. VS Code's `mcp.json` uses a `servers` key instead of `mcpServers`, with `"type": "http"`.

Chat apps (claude.ai, Claude Desktop, ChatGPT, Microsoft 365 Copilot, Notion) add PageCrawl as a custom connector with the server URL above. Step-by-step instructions for each are at https://pagecrawl.io/help/integrations/article/mcp-server-ai-tools.

## When the connection fails

- **Installed but not signed in**: the client offers a PageCrawl sign-in (for example an `authenticate` tool) instead of the PageCrawl tools. Start that sign-in and give the user the link it returns; in Claude Code the user can also run `/mcp`, select the PageCrawl server and choose Authenticate. Do not add the server a second time.
- **Every tool is listed twice**: PageCrawl is connected twice, for example through the plugin and an earlier `claude mcp add` entry or chat connector, or through a connection that still uses the old address `pagecrawl.io/mcp` (which keeps working). Call `list-workspaces` on each: if both reach the same account, use one for the whole task and tell the user the extra connection can be removed; if they reach different accounts, ask which one to use.
- **Cannot reach the server**: check the transport is streamable HTTP and the URL is exact.
- **"MCP is unavailable: AI features are disabled for your team"**: a team owner or Administrator turned off AI Analysis under Active Features in the PageCrawl settings, for the team the user has open there. Reconnecting does not help until it is switched back on.
- **Connected, but the tools are never used**: many clients need the connector turned on per conversation. Naming PageCrawl in the request also helps.
- **Authentication error**: the OAuth grant was revoked, or the token was deleted or has expired. Reconnect, or create a new token under Settings > API access.
- **Tools work but return nothing**: the monitors may be in a workspace the user was not added to, or the connection signed in with another PageCrawl account. Call `list-workspaces` to see which account and workspaces it reaches, and ask which ones the user means.
- **The client warns the server is unverified**: some clients show this for every custom server. PageCrawl's own sign-in page also marks command-line and editor clients as an unverified third-party application; continue only if the user started the connection.
