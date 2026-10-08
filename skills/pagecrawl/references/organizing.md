# Organizing monitors

## Contents
- Tags and folders
- Editing, pausing and removing monitors
- Templates and page discovery
- Workspaces
- Showing monitors visually

## Tags and folders

Tags (labels) and folders are separate. A monitor sits in at most one folder but can carry many tags. Use folders for structure (a client, a competitor, a project) and tags for cross-cutting themes (pricing, legal, urgent).

Folders (`manage-folders`):

- Start with `manage-folders(action="list")` to see the tree before changing it.
- `manage-folders(action="create", name="Competitors", parent="Research")` nests a folder; omit `parent` for the top level.
- `manage-folders(action="move", folder="Competitors", monitor_ids=[...])` moves up to 50 monitors. A destination that does not exist is created at the top level. An empty `folder` removes monitors from any folder.
- `rename` and `reparent` change a folder; `delete` removes it and its subfolders, and the monitors inside stay but lose their folder. Confirm before deleting.
- `list-monitors(folder="Competitors")` includes monitors in subfolders.

Tags (`manage-tags`):

- `manage-tags(action="list")` shows the workspace's tags.
- `manage-tags(action="add", monitor_id="...", tags=["pricing"])` and `manage-tags(action="remove", monitor_id="...", tags=["pricing"])` work on one monitor at a time.

## Editing, pausing and removing monitors

- `manage-monitors(action="update", monitor_id="...", frequency=60)` changes the name, URL, frequency, notifications, screenshots, `ai_page_focus`, folder or enabled state. It cannot change what the monitor tracks; for that, create a corrected monitor.
- `set-monitor-status(monitor_id="...", enabled=false)` stops checks and keeps the monitor and its history, freeing a slot in the plan's active monitor count. Turning it back on counts against that limit again.
- `manage-monitors(action="clear-history", monitor_id="...")` erases stored checks and values but keeps the monitor. `manage-monitors(action="delete", monitor_ids=[...])` removes monitors with all their history, screenshots and values. Both are permanent and accept up to 50 monitors: confirm first, naming the monitors.

## Templates and page discovery

A template holds shared settings for many monitors and can discover pages on a website.

1. `list-templates` shows existing templates.
2. `manage-templates(action="create", name="Example blog", url="https://example.com/blog", autoimport_enabled=true)` creates one with discovery on. `autoimport_mode` chooses how pages are found (for example `sitemap` or `combined`); some modes depend on the plan.
3. `manage-templates(action="add-filter", template="...", filter_type="url", filter_action="include", filter_pattern="/blog/*", filter_match_type="wildcard")` limits which discovered pages qualify.
4. `manage-templates(action="run-discovery", template="...")` queues a discovery crawl. It returns once queued, not when pages are found.
5. Later, `manage-discovered-pages(action="list", template_id=..., status="matches_filters")` shows candidates, and `manage-discovered-pages(action="import", ids=[...])` turns up to 50 of them into monitors. Pages imported past the plan's active monitor allowance are created paused.

`pause` and `resume` stop and restart monitors created from discovered pages. `stop` deletes those monitors but keeps the discovered records; `delete` removes the records. Confirm both. Deleting a template removes its discovered-page records and keeps monitors already created.

Help: https://pagecrawl.io/help/features/article/page-discovery

## Workspaces

- `list-workspaces` shows teams and workspaces with their IDs and monitor counts. Read and action tools work across all of them; only `add-page-monitor` needs `workspace_id`.
- `manage-workspaces(action="create", name="Client: Example")` adds one; `manage-workspaces(action="update", timezone="Europe/London")` changes the timezone that schedules and report dates use.
- `manage-workspaces(action="delete", workspace_id=..., confirm=true)` permanently deletes every monitor in it with their history. Check what it contains and confirm with the user first. The last workspace in a team cannot be deleted.
- Creating and deleting workspaces needs the team owner or manager role.

## Showing monitors visually

In clients that display interactive results, call `list-monitors` first and then `render-monitors-dashboard(monitor_ids=[...], title="...")` with 1 to 50 IDs from that result. It is for showing, not for reading data.
