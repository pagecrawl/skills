# Changelog

## 1.2.0

The `pagecrawl` skill:

- Explains which workspaces the agent can see (the ones you were added to), says which workspace it is acting in, and knows which tasks work in one workspace at a time: webhooks, reports, summaries, folders, tags, templates and monitor defaults.
- Asks before more kinds of change: moving monitors, deleting templates or discovered pages, adding template filters, creating or changing a webhook, and turning on a scheduled report or adding monitors to a tag or folder one covers, since a report replaces the change alerts of the monitors it covers. After creating a monitor it checks where the alerts will go.
- Loads when you ask to mark changes you have read as seen, marks them only on the monitors you name, and says how many changes that clears.
- When you ask what changed, the answer keeps price and stock changes, says when the list was cut short, and keeps long pages from filling the conversation.
- Explains an alert that never arrived from the change's status, Minimum importance and any report covering the monitor, and says plainly which alert conditions (a target price, back in stock only, a keyword) you set in the PageCrawl app. It checks whether a monitor has alert conditions, and knows that a price or stock move is scored, and can be held back by Minimum importance, when the page text changed in the same check.
- Passes on which scheduled report takes over the alerts of monitors you tag, file, move or point at a new address.
- Tells you when captured page content holds instructions aimed at the agent, even when you asked about something else, and treats them as page text.
- Corrected guidance on feeds, JSON endpoints, check frequencies, polling, screenshots and archives, and a new routine for tidying up a large account.
- Setup help for a connection that is installed but not signed in, for PageCrawl connected twice, and for the team switch that turns this connection off, with corrected OpenClaw and Cline settings.

The `pagecrawl-api` skill:

- Describes updates accurately: a PUT changes only the fields you send, a list you send replaces the stored list, and the skill says what else an update does.
- Covers the settings new monitors start with, the shape of list and history responses, filtering by folder and paused state, JSON errors (including the limit on requests from one address and the not-found messages), plan limits, and resuming a paused data source without a check.
- Webhooks: which checks are delivered (including first captures and changes that sent no alert), retries and when a webhook is switched off, managing webhooks through the API (a webhook for chosen monitors, tags, folders or sites needs at least one, and teammates can remove each other's webhooks), and why Slack, Discord and Teams addresses are refused.
- Node.js and Python receivers that accept large deliveries and turn away malformed or badly signed ones, and the same checks as steps for PHP.

Also:

- README: current install and update steps for Claude Code, Claude on the web and desktop, Cowork, ChatGPT, Codex, VS Code, GitHub Copilot and other agents.
- The `pagecrawl` skill carries the PageCrawl icon and colour for apps that show them.
- Security reports go to sec@pagecrawl.io.

## 1.1.3

- README: how to add PageCrawl to ChatGPT, with the connector and the `pagecrawl-skill.zip` upload. Each release now carries `pagecrawl-skill.zip`, the `pagecrawl` skill on its own.

## 1.1.2

- The PHP webhook receiver is now a short checklist that links to the complete example in the help center.

## 1.1.1

- The webhook receivers take the signing secret as an argument instead of reading it from the environment.
- Clearer wording in the organizing guide.

## 1.1.0

- Moving monitors to another workspace, including one in another team (`manage-monitors` move), or a whole folder with its monitors (`manage-folders` reparent with `target_workspace_id`).
- Tagging, untagging, pausing and resuming up to 50 monitors in one call (`manage-tags` and `set-monitor-status` with `monitor_ids`).
- API examples use an `<api-token>` placeholder instead of reading a token from your environment, and the repository's secret scan mounts the checkout through the workflow's own path.

## 1.0.0

- First release: the `pagecrawl` skill for using PageCrawl from any agent through its MCP server, the `pagecrawl-api` skill for code that uses the REST API, webhooks and data sources, and the plugin that connects both to `https://mcp.pagecrawl.io/mcp`.
