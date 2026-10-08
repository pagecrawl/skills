# PageCrawl for AI agents

Skills and an MCP connection that let AI agents work with [PageCrawl](https://pagecrawl.io), the website change monitoring service. Ask your agent to watch a page for price drops, policy changes, new job listings or release notes, then ask what changed this week, triage your alerts, or wire changes into your own systems.

The skills are instructions only: no scripts, no background processes. Everything the agent does goes through PageCrawl's MCP server, with the access you approve when you connect it. It works on every PageCrawl plan, including Free.

## What is inside

| Path | What it does |
|---|---|
| `skills/pagecrawl` | Using PageCrawl from any agent: choosing how to track a page, reading what changed, triage, health checks, folders, webhooks and scheduled reports |
| `skills/pagecrawl-api` | Writing code against the PageCrawl REST API: creating monitors, reading history and diffs, verifying webhook signatures, pushing values into data sources |
| `.mcp.json` | The connection to PageCrawl's MCP server, `https://mcp.pagecrawl.io/mcp` (OAuth) |
| `.claude-plugin/` | Plugin and marketplace manifests |

## Install

### Claude Code

Inside a session:

```
/plugin install pagecrawl --marketplace pagecrawl/skills
```

Or from your shell:

```bash
claude plugin marketplace add pagecrawl/skills
claude plugin install pagecrawl@pagecrawl
```

The first time a PageCrawl tool runs, a browser window opens to sign in and approve access. If you added PageCrawl earlier with `claude mcp add`, remove that entry (`claude mcp remove pagecrawl`) so the tools are not listed twice.

### Claude on the web and desktop, and Cowork

Download `pagecrawl-plugin.zip` from the [latest release](https://github.com/pagecrawl/skills/releases/latest), then go to **Customize > Plugins > Add > Upload plugin**. Connect PageCrawl from the plugin's **Connectors** tab.

### Codex

Copy `skills/pagecrawl` (and `skills/pagecrawl-api` if you write integrations) into `~/.agents/skills/`, then connect the server:

```bash
codex mcp add pagecrawl --url https://mcp.pagecrawl.io/mcp
codex mcp login pagecrawl
```

### Cursor, GitHub Copilot, Gemini CLI, OpenCode and other agents

```bash
npx skills add pagecrawl/skills
```

Then add the MCP server in your agent's settings. Instructions for each client are in the [PageCrawl help centre](https://pagecrawl.io/help/integrations/article/mcp-server-ai-tools).

### Anything else

Copy the folders under `skills/` into your agent's skills directory and connect the MCP server below. The skills follow the open [Agent Skills](https://agentskills.io) format.

## Connect PageCrawl

- Server URL: `https://mcp.pagecrawl.io/mcp` (streamable HTTP)
- Sign-in: OAuth. Your client opens PageCrawl, you sign in and click **Approve**.
- Clients without OAuth can send a personal API token as `Authorization: Bearer <token>`. Create one in PageCrawl under **Settings > API**.

## Try it

- "Watch https://www.apple.com/shop/buy-iphone/iphone-17-pro and tell me when the price changes."
- "Monitor these five competitor pricing pages weekly and tag them competitors."
- "What changed on my monitored pages this week? Lead with what matters."
- "Go through my unreviewed changes with me."
- "Which of my monitors are failing or never change?"
- "Write an Express endpoint that receives PageCrawl webhooks and verifies the signature."

## Data

The skills contain instructions and send nothing themselves. When your agent follows them, it calls PageCrawl's MCP server at `mcp.pagecrawl.io` (or the REST API at `pagecrawl.io/api`) with the access you approved, to create monitors and to read your monitoring data. Content PageCrawl returns comes from the websites you monitor, and the skills tell the agent to treat it as data, never as instructions. See the [privacy policy](https://pagecrawl.io/privacy-policy) and [terms](https://pagecrawl.io/terms-and-conditions).

## Support

- Questions and feedback: [pagecrawl.io/contact-us](https://pagecrawl.io/contact-us)
- Security reports: [SECURITY.md](SECURITY.md)
- Changes in each release: [CHANGELOG.md](CHANGELOG.md)

This repository is published from PageCrawl's main codebase, where the skills are tested against the live MCP server's tool definitions. Issues and pull requests are welcome; accepted changes are carried over by hand.

## License

[MIT](LICENSE)
