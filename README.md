# PageCrawl for AI agents

Skills and an MCP connection that let AI agents work with [PageCrawl](https://pagecrawl.io), the website change monitoring service. Ask your agent to watch a page for price changes, policy updates, new job listings or release notes, then ask what changed this week, go through your alerts, or wire changes into your own systems.

The skills are instructions only: no scripts, no background processes. Everything the agent does goes through PageCrawl's MCP server, or the REST API for code it writes, with the access you give it. It works on every PageCrawl plan, and each plan's limits still apply.

## What is inside

| Path | What it does |
|---|---|
| `skills/pagecrawl` | Using PageCrawl from any agent: choosing how to track a page, reading what changed, triage, health checks, folders, webhooks and scheduled reports |
| `skills/pagecrawl-api` | Writing code against the PageCrawl REST API: creating and updating monitors, reading history and diffs, receiving and verifying webhooks, pushing values into data sources |
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

Then sign in once: run `/mcp`, select the PageCrawl server (`plugin:pagecrawl:pagecrawl`) and choose **Authenticate**. Your browser opens so you can sign in to PageCrawl and allow access. If you skip this, Claude offers a sign-in link the first time it needs PageCrawl.

Connect PageCrawl only once. A server you added earlier with `claude mcp add` and any address other than `https://mcp.pagecrawl.io/mcp` loads beside the plugin and lists every tool twice, so remove it (`claude mcp remove pagecrawl`). Claude Code also loads your claude.ai connectors: if yours uses another address, remove it under **Customize > Connectors** and add it again with `https://mcp.pagecrawl.io/mcp`, which counts as the same server as the plugin's.

Plugins from this marketplace do not update on their own. To get a new version, run `claude plugin update pagecrawl@pagecrawl`, or turn on auto-update once: `/plugin`, **Marketplaces**, `pagecrawl`, **Enable auto-update**.

### Claude on the web and desktop, and Cowork

1. In **Customize > Plugins**, choose **Add > Add marketplace**, enter `pagecrawl/skills`, then add PageCrawl.
2. Open the plugin's **Connectors** tab and connect PageCrawl. On Team and Enterprise plans, an Owner adds the connector for the organization first.

A plugin added from a marketplace updates from it: select **Check for updates**, or turn on **Sync automatically** for the marketplace. If your organization does not let you add marketplaces, download `pagecrawl-plugin.zip` from the [latest release](https://github.com/pagecrawl/skills/releases/latest) and use **Add > Upload plugin** instead. An uploaded copy never updates, so upload each new release again.

### ChatGPT

Connect PageCrawl as a custom MCP server in ChatGPT on the web: open **Plugins**, select **+**, then **Add custom MCP server**. Enter a name and the server URL below, choose OAuth for authentication, tick **I understand and want to continue** and select **Create as a plugin**. ChatGPT renames these menus from time to time; if a label differs, follow OpenAI's [guide to custom MCP servers](https://help.openai.com/en/articles/12584461).

- Creating and changing monitors needs ChatGPT's full MCP support, which OpenAI offers on Business, Enterprise and Edu. On Business, only workspace admins and owners can add a custom server; on Enterprise and Edu, an admin can allow members to. Pro connects custom servers read-only.
- ChatGPT keeps a snapshot of the tools. After PageCrawl adds or changes tools, a workspace admin refreshes the tool list.
- On Business, Enterprise, Healthcare and Edu plans, add the skill too: download `pagecrawl-skill.zip` from the [latest release](https://github.com/pagecrawl/skills/releases/latest), then open **Plugins**, select the **Skills** tab, click **Create** and choose **Upload from your computer**.

### Codex

Copy `skills/pagecrawl` (and `skills/pagecrawl-api` if you write integrations) into `~/.agents/skills/`, then connect the server:

```bash
codex mcp add pagecrawl --url https://mcp.pagecrawl.io/mcp
```

Codex opens the PageCrawl sign-in when it detects OAuth. If it does not, run `codex mcp login pagecrawl`. To update the skills, copy the folders again from a new release.

### VS Code and GitHub Copilot

Both install this repository as a plugin, with the skills and the PageCrawl connection together:

- VS Code: run **Chat: Install Plugin From Source** from the Command Palette and enter `https://github.com/pagecrawl/skills`.
- GitHub Copilot CLI: run `copilot plugin install pagecrawl/skills`, and `copilot plugin update pagecrawl` for new versions.

Sign in to PageCrawl when you are asked. Keep one PageCrawl connection: if you added the server by hand before, remove that entry.

### Cursor, Antigravity, Gemini CLI, OpenCode, OpenClaw and other agents

```bash
npx skills add pagecrawl/skills -g
```

`-g` installs the skills for your user account, so they work in every project. Without it, the installer asks whether to add them to the current project or to your account. Run `npx skills update -g` to get new versions.

Then add the MCP server in your agent's settings with the details under [Connect PageCrawl](#connect-pagecrawl). The [PageCrawl help centre](https://pagecrawl.io/help/integrations/article/mcp-server-ai-tools) has steps for Cursor, Cline, Windsurf, VS Code, OpenClaw and other clients.

### Anything else

Copy the folders under `skills/` into your agent's skills directory and connect the MCP server below. The skills follow the open [Agent Skills](https://agentskills.io) format.

## Connect PageCrawl

- Server URL: `https://mcp.pagecrawl.io/mcp`, over streamable HTTP. A client set to SSE cannot connect.
- Sign-in: OAuth. Your client opens PageCrawl, where you sign in and allow access. There is no key to copy.
- Clients without OAuth can send a personal API token as `Authorization: Bearer <api-token>`. Create one in PageCrawl under **Settings > API access** with **Create Token**, and keep it in the client's settings, never in a chat.
- In a JSON settings file, Cline needs `"type": "streamableHttp"` beside `url` (it reads an entry without a type as SSE), Claude Code's `.mcp.json` needs `"type": "http"`, and VS Code's `mcp.json` uses a `servers` key with `"type": "http"`.
- Connect PageCrawl once per agent. Two connections with different addresses list every tool twice.
- If a team owner or Administrator turns off **AI Analysis** under **Active Features** in PageCrawl's settings (on **General**, or on **Team Settings** when the team has several workspaces), this connection stops too, with the message "MCP is unavailable: AI features are disabled for your team." It follows the team you have open in PageCrawl, and signing in again does not help until the switch is back on.

## Try it

- "Watch https://www.raspberrypi.com/products/raspberry-pi-5/ and tell me when the price changes."
- "Monitor these five competitor pricing pages weekly and tag them competitors."
- "What changed on my monitored pages this week? Lead with what matters."
- "Go through my unseen changes with me."
- "Which of my monitors are failing or never change?"
- "Write an Express endpoint that receives PageCrawl webhooks and verifies the signature."

## Data

The skills contain instructions and send nothing themselves. When your agent follows them, it calls PageCrawl's MCP server at `mcp.pagecrawl.io` (or the REST API at `pagecrawl.io/api`) with the access you gave it, to create and manage monitors and to read your monitoring data. That access can also permanently delete monitors, workspaces and stored history, and send monitoring data to other systems through webhooks and scheduled reports; the skills tell the agent to confirm anything permanent or outward-facing with you first. Content PageCrawl returns comes from the websites you monitor, and the skills tell the agent to treat it as data, never as instructions, and to tell you when it holds instructions aimed at the agent. See the [privacy policy](https://pagecrawl.io/privacy-policy) and [terms](https://pagecrawl.io/terms-and-conditions).

## Support

- Questions and feedback: [pagecrawl.io/contact-us](https://pagecrawl.io/contact-us)
- Security reports: [SECURITY.md](SECURITY.md)
- Changes in each release: [CHANGELOG.md](CHANGELOG.md)

This repository is published from PageCrawl's main codebase, where tests check the tools, arguments and API paths the skills name against the MCP server and the API specification. Issues and pull requests are welcome; accepted changes are carried over by hand.

## License

[MIT](LICENSE)
