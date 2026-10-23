# Automating with PageCrawl

## Contents
- Webhooks
- Scheduled reports
- Polling for new changes
- Push data sources
- No-code tools

## Webhooks

A webhook makes PageCrawl POST a JSON payload to another system for every change it records on the monitors the webhook covers, the first capture included, whether or not that change sent an alert (only a stock level saved without an alert is left out); with `error` in `events`, failed checks are delivered too. Use it for agents, internal tools and automations that should react on their own. Every action works in one workspace (`workspace_id`, else the current one), and a webhook fires only for monitors in its own workspace.

1. `manage-webhooks(action="list")` first, to see what already exists and get each `hook_id`.
2. Read the destination URL back to the user and confirm it, because it will receive the content of the monitored pages. Then `manage-webhooks(action="create", target_url="https://example.com/hooks/pagecrawl", tag_ids=[...], events=["change_detected"])`.
   - `target_url` must be HTTPS. Slack, Discord and Microsoft Teams URLs are rejected: those apps are connected in the PageCrawl app (Settings > Integrations, in Notification Channels) and chosen with a monitor's `notifications`.
   - `match_type` (`all`, `monitors`, `tags`, `folders` or `domains`) decides which monitors fire it, with `monitor_ids`, `tag_ids`, `folder_ids` or `domains` from the webhook's workspace. Left out on create, it follows the one kind of IDs sent; several kinds are refused, and none means `all`. On update, IDs of the webhook's current kind replace its list; to switch kinds, pass `match_type` with the new ones.
   - `events` is `change_detected` (the default) and/or `error`. `payload_fields` trims the payload to the fields the receiver needs.
3. `manage-webhooks(action="test", hook_id=...)` sends one real delivery, built from the latest check of a monitor the webhook covers (`monitor_id` picks which). Tell the user before running it.
4. `manage-webhooks(action="logs", hook_id=...)` shows recent deliveries with status codes, for a webhook that is not arriving.

PageCrawl switches a webhook off after repeated failed deliveries (`list` shows `is_active` and the failures) and deletes one whose receiver answers 410. A switched-off webhook sends nothing, test included. Once the receiver is fixed, `manage-webhooks(action="update", hook_id=..., is_active=true)` turns it back on.

Every delivery is signed, so the receiver can tell it came from PageCrawl. These tools do not handle signature verification: the user sets it up in their receiver, following the `pagecrawl-api` skill and https://pagecrawl.io/help/tutorials/article/reference-implementations.

## Scheduled reports

A report sends a digest of what changed across a set of monitors on a schedule, and it takes over their alerts. While an enabled report covers a monitor, that monitor sends no change alerts of its own (email, chat or phone) and its changes go into the digest instead. The digest lists every change saved on the monitors it covers, including ones their Minimum importance held back (only first captures and stock-only checks are left out), filtered by the report's own Minimum change importance. Alerts about failed checks still go out through each monitor's own channels. Turning the report off or deleting it gives the monitors their change alerts back, unless another report covers them.

Before creating, widening or turning on a report, tell the user which monitors it covers and that their own change alerts stop. Prefer `match_type` `tags`, `folders` or `monitors` over `all` when other monitors should keep alerting. Coverage follows the monitors: a report that covers all monitors also takes over every new one, and one that covers tags, folders or sites takes over a monitor as soon as it gets that tag, moves into that folder or watches that site. `get-monitor-details` lists the `reports` that already cover a monitor.

- `manage-reports(action="create", name="Weekly competitor digest", match_type="tags", tag_ids=[...], schedule="weekly", schedule_day=1, schedule_hour=8, channels=["email"], filter_mode="important", important_threshold=50, ai_summary=true)`. The response's `monitors_covered` and `note` spell out what happens to their alerts; pass that on.
  - `schedule` is `daily`, `daily_except_weekends`, `daily_weekends`, `weekly`, `monthly` or `none` (on demand only). For `weekly`, `schedule_day` is 1 (Monday) to 7 (Sunday). For `monthly` it is the day of the month, and a month without that day is skipped, so use 1 to 28 to send every month. `schedule_hour` (default 9) is in the workspace's time zone.
  - `channels` are `email`, `slack`, `discord`, `teams`, `telegram`, `webpush` and `mobile` (the PageCrawl app, on devices with summary reports turned on). Chat apps must be connected in the PageCrawl app first. With no channels, digests are kept in the app only.
  - Email goes to `recipient_user_ids`, or to the signed-in user when none are given. Other members are added under Send to on the report in the app (Settings > Workspace > Notifications > Scheduled Summary Reports).
  - `filter_mode="important"` keeps only changes at or above `important_threshold`; a change below it is left out of the digest and never escalates.
  - Priority escalation, on a scheduled report, still sends each change at or above `escalation_threshold` through `escalation_channels` without waiting for the digest (`escalation_enabled=true`). Offer it when important changes should keep alerting.
- `manage-reports(action="list")` returns reports with their `slug`, which the other actions use; `action="get"` reads one.
- `manage-reports(action="update", slug="...", enabled=false)` turns a report off. An update changes only what you pass and keeps the rest, including who can open the digest links (Anyone with the link, or Team members only, set in the app). `channels`, `escalation_channels` and `recipient_user_ids` replace their whole lists when passed.
- `manage-reports(action="delete", slug="...")` removes a report, and its monitors send their own change alerts again. Confirm first.
- `manage-reports(action="generate", slug="...", period_start="...", period_end="...")` builds a digest now and delivers it to the report's channels when there is content. Confirm first.
- `manage-reports(action="digests", slug="...")` reads digests already produced.
- Creating, editing, deleting and generating need the team owner or an Administrator (`role` `manager` in `list-workspaces`). Scheduled reports also need a paid plan: on a plan without them, `create` and `generate` refuse and say so.
- Every action works in one workspace (`workspace_id`, else the current one), and a report covers only monitors in its own workspace.

Help: https://pagecrawl.io/help/notifications/article/scheduled-reports

## Polling for new changes

When an automation must pull rather than receive, use the cursor:

1. First call: `get-changes-since(since="24 hours ago")`, and store `next_cursor`, which is never null.
2. Every later run: `get-changes-since(after_check_id=<stored next_cursor>)`, storing the new `next_cursor` each time. Results come oldest first. When `truncated` is true, more are waiting: call again with the new cursor.
3. Skip check IDs you already have: the first poll after a call with `since` can return that call's newest changes again. After that, a poll does not repeat a change an earlier poll returned.

`next_cursor` is an opaque position: store it and pass it back unchanged, and never move `since` forward by hand instead. A change reaches polls about two minutes after its check, so a check that is still being saved is normally not passed over. Each change has a `status`, and only `notified` ones were sent as alerts. To send each change to another system without polling, use a webhook.

## Push data sources

When a value does not come from a web page, such as a stock count or a sales figure the user gives you, create a data source and record values to it. Pushed values get the same history, charts, alerts and AI summaries as any monitor.

1. `create-data-source(name="Warehouse stock", fields=[{"label": "units", "type": "number"}], ai_prompt="Units in stock at the main warehouse; drops matter more than rises")`. Field types are `number`, `price` and `text`; define number and price fields up front for values such as 12.99. A data source alerts like a new monitor, through the saved default or else email plus every connected chat channel; pass `notifications` to choose, or `[]` for none. At a plan limit it is not created: pass on the message.
2. `record-data(monitor_id="...", value="128")` fills the only field, or creates one named Value on a source without fields. On a source with several fields, pass `values` keyed by field label, for example `record-data(monitor_id="...", values={"units": "128", "status": "ok"})`, because a bare `value` there adds a new Value field. A label that does not exist yet becomes a new field, typed as a number only when the value is a plain number such as 1299.
3. Recording the same value again returns `changed=false` and adds nothing to the history, but every push counts toward the plan's check allowance, so push on a schedule that matches how often the value moves. `return_summary=true` also returns the AI summary and score of a recorded change.

- `ai_prompt` only shapes the AI summary and importance score. To alert only when a number crosses a value, the user adds a condition such as Less than under Conditions & Filters on the data source in the app.
- Set `ai_summaries_enabled=false` for frequent numeric metrics that only need charts and thresholds.
- The response includes a private `ingest_url` (accepts a plain POST without a header) and an `ingest_email` address. Anyone who has either one can push values, so share them only with the user and never publish them. By email, the message body is the value (the subject when the body is empty); for several fields, write one `label: value` line per existing field, otherwise the whole body becomes one value. A rejected email gets no reply.
- `trigger-check` does not apply to data sources. Every read tool works on them like any monitor.

Help: https://pagecrawl.io/help/integrations/article/push-api-data-sources

## No-code tools

PageCrawl also connects to n8n, Zapier and Make:

- https://pagecrawl.io/help/integrations/article/pagecrawl-n8n-integration
- https://pagecrawl.io/help/integrations/article/pagecrawl-zapier-integration
- https://pagecrawl.io/help/integrations/article/pagecrawl-make-integration
