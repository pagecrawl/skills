# Reading what changed

## Contents
- Fewest calls for common questions
- Summaries
- Changes across monitors
- One monitor
- Evidence and screenshots
- What to avoid
- Output formats

To collect new changes on a schedule from an automation, use the cursor described in [automation.md](automation.md).

## Fewest calls for common questions

| Question | Calls |
|---|---|
| "What changed today?" | `get-changes-since(since="today", limit=20)`, then `get-check-diff(monitor_id="...", check_id=...)` for the ones worth describing |
| "Give me this week's summary" | `get-summary(period="7d")` for each workspace, then `get-changes-since(since="7 days ago", limit=20)` |
| "What are the prices right now?" | `list-monitors(search="...")` to find the IDs, then one `get-latest-values(monitor_ids=[...])` |
| "What changed on X last time?" | `get-check-diff(monitor_id="X")`; without `check_id` it returns the newest check that found a change |
| "How has X moved over time?" | `get-monitor-history(monitor_id="X", changes_only=true, limit=20)` |
| "When does X usually change?" | `get-intelligence(monitor_id="X")`: how often, the usual gap, and the days and hours changes land |
| "Which monitors need my attention?" | `list-monitors(has_unseen=true, include_disabled=true, sort_by="latest_change")` |
| "Find what we monitor about a topic" | `search(query="...")`, then `fetch(id="...")` for the results that matter |
| "What did X say on 1 March?" | `get-archive(action="wayback-resolve", monitor_id="X", as_of="2026-03-01")`, then `get-check-diff` with the `check_id` it returns when `resolved.changed` is true (see Evidence and screenshots) |

## Summaries

`get-summary` covers one workspace per call (`workspace_id`, else the current one). It counts changes by the AI's score (`critical`, `high`, `medium`, `low`). Changes the AI did not score, such as price, stock and availability moves, count as `other` (or `no_ai` on monitors with AI summaries off) whatever their `importance`, so take those from `value_changes` (the numeric monitors that moved, and by how much) and `get-changes-since`. It also returns active and paused monitors and a timeline of recent changes. `unreviewed` counts the cards on the Review Board's To Review lane, not unseen changes, and `has_ai` false means AI analysis is not available in the workspace, so every change counts as `other`.

Periods are `today`, `yesterday`, `24h`, `7d`, `30d` and `90d`, or a `since` and `until`. They count whole days in UTC up to now, so `period="yesterday"` also counts today. For one day alone, pass the same date twice: `get-summary(since="2026-03-01", until="2026-03-01")`. When the user is far from UTC, say which days the numbers cover.

## Changes across monitors

`get-changes-since` covers every workspace the user can open (or one, with `workspace_id`). With `since` it returns the newest changes first, up to `limit` (default 50, at most 200; ask for 10 to 20 for a digest).

- Each change has `check_id`, `monitor_id`, `monitor_name`, `checked_at`, `status`, `ai_summary`, `importance`, `priority_score` and the changed `elements` with their new `value` and `difference`. It carries no link: take each monitor's `link` from `get-check-diff` or `list-monitors`.
- `status`: only `notified` sent an alert. While a report covers the monitor, its digest lists the saved changes whatever their status (not first captures or stock-only checks), filtered by the report's Minimum change importance. The other statuses were saved without an alert ([troubleshooting.md](troubleshooting.md) explains each).
- `importance` (0 to 100) is the AI's score when the AI scored the change. Otherwise it is a rating worked out from the change itself for a price, number, stock, availability, yes/no, visual or file change: small moves rate low, while large drops, restocks and sell-outs rate high. It is null when neither applies, for example a text change while Importance Scoring is off. `priority_score` is the AI's own score alone, so it is null when only a price, stock level or availability moved.
- `min_priority` keeps changes whose `importance` is at least that value, plus changes with none, and always leaves out `learned_noise` and `stock_recorded` changes: the same set a scheduled report's Minimum change importance keeps. Leave it off when the user asks about stock levels.
- Text values are cut to their first 1500 characters (`values_truncated` is then true); numbers are never cut. Read the change itself with `get-check-diff`.
- `truncated` true means more changes matched than came back. With `since`, the list reaches back only to `covers_from`: say which stretch it covers, and narrow the period, raise `min_priority` or pass `workspace_id` to reach further, or take the totals from `get-summary`.

## One monitor

- `get-check-diff(monitor_id="...", check_id=...)` shows one check's diff: added lines start with `+`, removed with `-`, unchanged with `=`. Without `check_id` it shows the newest change, stepping over checks that only recorded a stock level. When the diffs pass 50000 bytes, long unchanged runs are folded and an element that still does not fit is cut, with `truncated` true: say that part was cut and give the link.
- `get-monitor-history` returns checks newest first (`limit` default 20, at most 100). `changes_only=true` keeps only checks that found a change, and `since` takes an ISO 8601 date. Entries carry `status` and `importance` as in `get-changes-since`, and long text values are cut the same way.
- `get-latest-values` returns each element's current value in full, with the previous value when it changed. On a whole-page monitor that is the entire page text, so batch it for prices, numbers and short values, and read whole pages one at a time.
- `get-monitor-details` returns the configuration: the tracked elements, frequency, `notifications`, the `reports` that cover the monitor, the Minimum importance that applies (`effective_min_importance` and where it comes from), `has_alert_conditions` (true when alert conditions set in the app decide which changes alert, so the Minimum importance holds nothing back), its `template`, and its `screenshots` setting next to the team's `team_screenshots_enabled`.

## Evidence and screenshots

- `get-screenshot(monitor_id="...", check_id=...)` returns a short text block and then the stored image; it never fetches the live page. Without `check_id` it uses the most recent check PageCrawl kept, and its `note` says when that is older than the latest check. A tall page comes in parts of at most 2000 px: `parts` says how many, and `part=2` and so on return the rest. Take wording and values from `get-check-diff`, not from the image.
- `element_id` (from `get-monitor-details`) returns the cropped capture of a visual element; other element types have none.
- A check that fails keeps no screenshot; [troubleshooting.md](troubleshooting.md) shows how the user can capture one.
- While the team's Screenshots switch is off, no screenshots are captured. A team owner or Administrator turns it on under Active Features in the PageCrawl settings.
- `get-archive(action="wayback-calendar", monitor_id="...")` lists the days with a capture, and `get-archive(action="wayback-resolve", monitor_id="...", as_of="2026-03-01")` finds the capture current on a date. Both work for every monitor; `has_archive` says whether that capture also has a replayable web archive.
- To answer "what did the page say on that date": resolve the date, then call `get-check-diff` with its `check_id`, and `get-screenshot` with the same `check_id` when the layout matters. `get-check-diff` shows only the elements that changed in that check, so its lines starting `=` or `+` are that version's text for those elements only, and a long diff folds unchanged runs. When `resolved.changed` is false (for example the monitor's first capture), there is no diff: show that capture with `get-screenshot` and give the user its `screenshot_url`, or `replay_url` when it has one.
- `get-archive` with `action="info"`, `action="verify"` or `action="captured-pages"` and a `check_id` describes a stored web archive, which only monitors with web archiving on keep: its timestamp proofs, whether it still matches what was sealed, and the pages inside it. Point the user to the archive's link rather than restating it as fact.
- How long checks and screenshots are kept depends on the plan. When `wayback-resolve` answers `found` false with an `earliest` date, or the oldest result is later than the date asked about, say the earlier history is no longer stored.

## What to avoid

- Calling `get-monitor-history` once per monitor to build a digest. `get-summary` and `get-changes-since` answer for every monitor in one call.
- Describing a change from `ai_summary` alone. Read `get-check-diff` first.
- Loading the live page to confirm a change, or calling `trigger-check` to refresh before answering. The stored result is the answer; say when it was checked.
- Reporting every change at equal weight. Lead with the highest `importance`.
- Calling a cut-short list the whole period. When `truncated` is true, say which stretch it covers.

## Output formats

Daily or weekly briefing:

```
This week: <n> notable changes across <m> pages (<k> minor ones not listed).

1. <Monitor name>: <what changed, with old and new values>. <Why it matters>. Importance <score>. <link>
2. ...
```

A numeric change reads best with both values, for example "Netflix US pricing: the Standard plan rose from $15.49 to $17.99 a month (+16%)."

Rules for any format:

- Say what changed in plain words, from the diff, with old and new values for numbers (and the percentage when it helps).
- Include each monitor's `link` so the user can open the change in PageCrawl.
- Group related changes (several pages of one competitor, one product across shops).
- When nothing significant changed, say so in one line, for example "Nothing notable changed this week; 6 minor edits on 4 pages."
- Use dates and times from the results (`checked_at`), in the user's timezone if known.
