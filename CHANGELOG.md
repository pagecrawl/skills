# Changelog

## 1.1.1

- The webhook receivers take the signing secret as an argument instead of reading it from the environment.
- Clearer wording in the organizing guide.

## 1.1.0

- Moving monitors to another workspace, including one in another team (`manage-monitors` move), or a whole folder with its monitors (`manage-folders` reparent with `target_workspace_id`).
- Tagging, untagging, pausing and resuming up to 50 monitors in one call (`manage-tags` and `set-monitor-status` with `monitor_ids`).
- API examples use an `<api-token>` placeholder instead of reading a token from your environment, and the repository's secret scan mounts the checkout through the workflow's own path.

## 1.0.0

- First release: the `pagecrawl` skill for using PageCrawl from any agent through its MCP server, the `pagecrawl-api` skill for code that uses the REST API, webhooks and data sources, and the plugin that connects both to `https://mcp.pagecrawl.io/mcp`.
