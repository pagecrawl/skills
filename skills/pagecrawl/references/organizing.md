# Organizing monitors

## Contents
- Tags and folders
- Editing, pausing and removing monitors
- Templates and page discovery
- Workspaces
- Showing monitors visually

## Tags and folders

Tags (Labels in the app) and folders are separate. A monitor sits in at most one folder but can carry many tags. Use folders for structure (a client, a competitor, a project) and tags for cross-cutting themes (pricing, legal, urgent).

Folders (`manage-folders`) belong to one workspace (`workspace_id`, else the current one; `move` uses the monitors' own workspace):

- Start with `manage-folders(action="list")` to see the tree, then act on folders by the `slug` or ID it returns. The same name can appear under several parents, and a name acts on the first match.
- `manage-folders(action="create", name="Competitors", parent="research")` nests a folder; omit `parent` for the top level.
- `manage-folders(action="move", folder="competitors", monitor_ids=[...])` moves up to 50 monitors, all from one workspace, into a folder of that workspace. A destination that does not exist is created at the top level. Leaving `folder` empty or out removes the monitors from any folder.
- `rename` and `reparent` change a folder. `manage-folders(action="reparent", folder="competitors", target_workspace_id=...)` moves the folder, its subfolders and every monitor in them to another workspace of the same team; confirm it like any move.
- `delete` removes a folder and its subfolders, and the monitors inside stay but lose their folder. Confirm before deleting.
- `list-monitors(folder="competitors")` includes monitors in subfolders.

Tags (`manage-tags`):

- `manage-tags(action="list")` shows the current workspace's tags, or another workspace's with `workspace_id`.
- `manage-tags(action="add", monitor_ids=[...], tags=["pricing"])` tags up to 50 monitors in one call, and `action="remove"` untags them. Use `monitor_id` for a single monitor. The monitors can be in different workspaces: each one gets the tag of its own workspace, created there when missing.
- Renaming, recolouring and deleting tags is done in the app, under Labels in the settings.

Adding a tag, or moving monitors into a folder (subfolders included), brings them under any enabled report that covers that tag or folder, and the report then takes over their change alerts. `manage-reports(action="list")` shows each report's `match_type`, `tag_ids` and `folder_ids`; when one covers the tag or folder, tell the user before you add the tag or move the monitors. Afterwards the response's `warnings` name each enabled report that now covers some of the monitors, and say when monitors are left with no report and send their own change alerts again: pass them on. `manage-tags` add and remove, `manage-folders` move, reparent and delete, and `manage-monitors` move and update (with a new `folder` or `url`) all return them. `reports` in `get-monitor-details` lists the reports that cover a monitor.

## Editing, pausing and removing monitors

- `manage-monitors(action="update", monitor_id="...", frequency=60)` changes one monitor's `name`, `url`, `frequency`, `notifications`, `screenshots`, `ai_page_focus`, `min_importance`, `folder`, or paused state (`disabled=true` or `disabled=false`). It cannot change what the monitor tracks: the user chooses Edit on the monitor in PageCrawl, which keeps its history, or you create a corrected monitor and offer to delete the old one.
- Read `get-monitor-details` before changing alert settings. `notifications` replaces the whole channel list, so pass every channel to keep.
- `min_importance` sets the monitor's own Minimum importance (0 to 100). `manage-monitors(action="update", monitor_id="...", min_importance=null)`, sent with no other field, makes it follow its template or workspace again. An update of `min_importance` alone resets nothing and queues no check. Any other update resets the failure count and moves an older monitor onto the current capture rules, so its next check can record a one-time change; changing the URL or frequency, or resuming, queues a check (`recheck_queued`).
- Changing `url` keeps the monitor's history and settings, including any custom headers and HTTP Basic credentials, which then go to the new address. The next check compares the new page with the old one's last capture and can alert. A report that covers a site matches its exact host name, so a new address on another host can bring the monitor under such a report or take it out of one; the response's `warnings` say so. For a different page or site, create a new monitor instead.
- On a monitor made from a template, `update` refuses a `frequency` or `notifications` the template sets. Change them for every monitor of the template with `manage-templates(action="update", template="...", frequency=1440)`, or the user chooses Customize this page in the monitor's settings to give it its own.
- If the response warns that screenshots are off for the team, pass it on: a team owner or Administrator turns on the Screenshots switch under Active Features in the PageCrawl settings.
- `set-monitor-status(monitor_ids=[...], enabled=false)` stops checks on up to 50 monitors (`monitor_id` for one) and keeps them with their history, freeing slots in the plan's active monitor count. Each monitor comes back with its own result. Turning monitors back on counts against the active monitor limit of the team each one belongs to, and enabling stops when that limit is reached.
- `manage-monitors(action="clear-history", monitor_id="...")` erases the stored checks, values, screenshots and archives but keeps the monitor. `manage-monitors(action="delete", monitor_ids=[...])` removes monitors with all their history. Both are permanent and accept up to 50 monitors: confirm first, naming the monitors.

## Templates and page discovery

A template holds shared settings and tracked elements for many monitors and can discover pages on a website. Templates, filters and discovered pages belong to one workspace (`workspace_id`, else the current one).

1. `list-templates` shows existing templates with their tracked elements and `monitor_count`.
2. `manage-templates(action="create", name="Example blog", url="https://example.com/blog", autoimport_enabled=true)` creates one with discovery on. `autoimport_mode` chooses how pages are found (for example `sitemap` or `combined`); some modes depend on the plan. `manage-templates(action="update", template="...", autoimport_enabled=true)` turns discovery on later. `update` also changes the name, URL, AI focus and discovery settings, and the `frequency` and `notifications` of every monitor made from the template.
3. Filters turn pages into monitors on their own, with no review step. Every discovery run adds each newly found page that matches as a monitor, within the plan's limits: past the active allowance it is created paused, and once the total page limit is reached none is added and discovery switches itself off. An exclude filter on its own lets through every page it does not match. Adding an include filter, such as `manage-templates(action="add-filter", template="...", filter_type="url", filter_action="include", filter_pattern="/blog/*", filter_match_type="wildcard")`, also adds the matching pages found so far, within the same limits (once the template has a title, text or element filter, that happens in the background and only while discovery is on). The count in the response includes the monitors created paused. While the template has no title, text or element filter, adding a URL exclude filter also pauses the template's own active monitors whose address matches it: the ones made from it and the ones its discovered pages link to, including any given their own settings with Customize this page. No other monitor in the workspace is touched. Agree the filters with the user before adding them, and say how many pages they may add or pause.
4. To let the user pick pages instead, run discovery before adding any filter: `manage-templates(action="run-discovery", template="...")` queues a crawl and returns once it is queued, not when pages are found. Later, `manage-discovered-pages(action="list", template_id=...)` shows what was found, and `manage-discovered-pages(action="import", ids=[...])` turns up to 50 chosen pages into monitors. Pages imported past the plan's active allowance are created paused, and importing depends on the plan.
5. `manage-templates(action="remove-filter", template="...", filter_id="...")` drops a filter; the filter IDs are in the `filters` list `manage-templates` returns (`list-templates` does not show them). Removing an exclude filter does not resume the monitors it paused: `set-monitor-status(monitor_ids=[...], enabled=true)` does. `manage-discovered-pages(action="summary")` counts discovered, monitored, paused and failed pages.

`pause` and `resume` stop and restart monitors created from discovered pages. `stop` deletes those monitors together with their history and keeps the discovered records; `delete` removes the records and leaves monitors already created. Confirm both.

Deleting a template keeps its monitors working: each one first gets its own copy of the template's tracked elements and settings, keeps its history, and keeps being checked unless it is paused. With monitors attached, the deletion finishes in the background; discovery stops at once and the discovered-page records are deleted. Confirm first, giving the template's `monitor_count`.

Help: https://pagecrawl.io/help/features/article/page-discovery

## Workspaces

- `list-workspaces` shows the teams and workspaces the user was added to, with IDs, monitor counts, the current one (`is_current`) and the user's `role` in each team: `owner`, `manager` (Administrator), `user` (Standard User) or `viewer` (Viewer). A Viewer can read and mark changes as seen but cannot change anything else.
- `manage-monitors(action="move", monitor_ids=[...], workspace_id=..., folder="clients")` moves up to 50 monitors to another workspace, including one in another of the user's teams; omit `folder` to place them at the top level. Their history, review status and tags go with them and saved logins are copied, while alert channels, AI settings, reports and webhooks come from the new workspace (only a webhook made for one monitor moves with it). Monitors made from a template move together with their template: select all of them, or move their folder with `manage-folders` reparent. Confirm first, and pass on the response's `warnings`.
- `manage-workspaces(action="create", name="Client: Example")` adds a workspace that only the user can open. `user_ids` sets exactly which team members can open it instead, so include the user's own ID. Others can be added later on the Members page in the PageCrawl settings (the Workspaces column).
- `manage-workspaces(action="update", workspace_id=..., timezone="Europe/London")` renames a workspace (`name`) or changes the time zone that schedules and report dates use.
- `manage-workspaces(action="delete", workspace_id=..., confirm=true)` permanently deletes every monitor in it with their history. Check what it contains and confirm with the user first. The last workspace in a team cannot be deleted.
- Creating a workspace, renaming it or changing its time zone, and deleting it need the team owner or an Administrator. Moving monitors needs at least the Standard User role in each team involved.

## Showing monitors visually

In clients that display interactive results, call `list-monitors` first and then `render-monitors-dashboard(monitor_ids=[...], title="...")` with 1 to 50 IDs from that result. It is for showing, not for reading data.
