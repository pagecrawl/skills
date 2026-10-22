# Troubleshooting monitors

## Contents
- Status values
- Why no alert was sent
- Noisy monitors
- Silent monitors
- Pages that will not load
- Tool errors

## Status values

`list-monitors` and `get-monitor-details` report each monitor's `status` and `failed` (consecutive failed checks). A monitor is healthy when `status` is `ok`, `unchanged` or `pending` and `failed` is 0.

| Status | Meaning | What to do |
|---|---|---|
| `ok`, `unchanged` | Checking normally | Nothing |
| `pending` | The first check has not finished yet | Wait a few minutes, then read it with `get-latest-values` |
| `selector_not_found`, `invalid_selector`, `number_not_found` | The tracked element is missing or the selector is broken, usually after a redesign | The user chooses Edit on the monitor and picks the element again, which keeps the history; or diagnose with `test-configuration` and create a corrected monitor (or use `ai_extract`) |
| `blank_page` | The page loaded with no content | It may need a sign-in, a click or a wait; see [tracking-modes.md](tracking-modes.md) |
| `product_unavailable` | The page says the product is out of stock or no longer sold | Usually not a broken monitor: tell the user. Checks continue and record the product when it returns |
| `custom_error` | The monitor's own Mark Check as Failed When step matched; the status carries its label after a colon | Report the label as the user's own rule. The user changes the rule in the app |
| `timeout`, `unreachable`, `unavailable` | The site did not answer in time, could not be reached, or returned a server error | PageCrawl retries on later checks. If it persists, check the URL works in a browser; long `wait` steps also use up a check's time |
| `soft_error`, `unknown_error` | The site served an error page instead of its content, or the check failed for an unexpected reason | Usually temporary, and no error alert is sent for either. If it persists, the user can contact PageCrawl support |
| `404_page_not_found` | The page is gone | Ask whether it moved. For a new address of the same page, `manage-monitors(action="update", monitor_id="...", url="...")` |
| `401_unauthorized` | The page needs a signed-in account | The user saves the account under Website Logins in the app; a corrected monitor then uses its `auth_id` |
| `requires_auth` | The site asks for HTTP Basic Authentication, the browser's own sign-in box | The user enters the credentials under HTTP Basic Authentication in the monitor's settings |
| `login_failed` | The saved login ran, but the content that proves a sign-in was not found | The user runs Test the login under Website Logins, which names the step that failed |
| `403_forbidden`, `429_rate_limit`, `access_denied`, `connection_reset`, `444_no_response`, any status starting with `blocked` | The site refused the visit or limited how often it can be checked. This is the site's protection, not the monitor's settings | See "Pages that will not load" below |
| `robots_txt_disallowed`, `bot_protection_respected` | The team chose to respect robots.txt and bot protection, and the site disallowed or turned away this check | Nothing is wrong with the monitor, and checks resume once the site allows them. Otherwise the user asks the site owner, or the team owner turns the setting off |
| `unexpected_redirect` | The page sent the check to another address while it loaded | If that is expected, the user adds a Wait for Redirect step in the app. If the page moved for good, ask whether to monitor the new address |
| `too_many_redirects` | The site redirects in a loop | Tell the user; the site owner usually has to fix it |
| `ssl_error` | The site's certificate is invalid | Tell the user; the site owner has to fix it |
| `javascript_error` | Custom JavaScript (a step or a tracked element) threw an error | Fix or remove the script |
| `currency_mismatch` | The page showed the price in a different currency from the one the monitor expects, so the check was not recorded | The site served another region. In the app, the user can set the monitor's location to the country they want, or change the price element's currency |
| `pdf_blank_page`, `navigation_aborted` | The PDF has no readable text (often a scanned file), or the address starts a download that cannot be shown | Compare the whole file instead: the user chooses Switch to File Checksum on the monitor page (offered for `pdf_blank_page`) or picks File Checksum as the tracked element under Edit, either of which keeps the history; or you offer a corrected monitor, `add-page-monitor(url="...", elements=[{"type": "file_hash"}])`, and offer to delete the old one |
| `empty_file`, `file_too_large`, `pdf_error`, `file_error` | The file came back empty, was too large, or could not be read. `file_error:not_a_document` means the address now shows a web page instead of the document | An empty download is retried on later checks. For a file that is too large, the user can contact support. A document that moved needs its new address |
| `proxy_unreachable`, `proxy_unsupported` | The connection chosen as the monitor's location, or the user's own proxy, failed or is set up wrongly | The user picks another location for the monitor in the app, or corrects their proxy settings |
| `400_bad_request` | The site rejected the request | Check the URL; another location may help |
| `402_payment_required` | The page is behind a paywall | PageCrawl cannot read content that requires payment; tell the user |
| `relay_offline` | The PageCrawl Relay this monitor uses is offline | The user starts the Relay on that computer, or changes the monitor's location |

For a status not listed here, tell the user the status and give the monitor's `link`: the monitor page in PageCrawl explains it and offers the fixes that apply. More: https://pagecrawl.io/help/troubleshooting/article/common-issues-with-page-not-loading

A paused monitor keeps the status of its last check and shows `disabled` true. `get-monitor-details` also shows `paused_until` when checks are on hold until a date, for example after the account has used its checks. Turn a paused monitor back on with `set-monitor-status(monitor_id="...", enabled=true)` if the user wants it.

## Why no alert was sent

Find the change with `get-monitor-history(monitor_id="...", changes_only=true)` or `get-changes-since` and read its `status`:

- `notified`: passed the monitor's alert settings and went to its channels, or, while a report covers the monitor, to that report.
- `ok`: the conditions set on the monitor in the app did not match (the change was kept because Always record change detections is on), or the check was still being processed.
- `low_priority`: the AI scored it below the monitor's Minimum importance (`effective_min_importance` in `get-monitor-details`).
- `learned_noise`: similar changes were marked not useful, or kept scoring as unimportant.
- `skipped`: nothing was added or removed of the kind the monitor alerts on.
- `skipped_first`: the first change after the monitor's settings changed.
- `no_channels`: the monitor has no alert channel. If the user wants alerts, `manage-monitors(action="update", monitor_id="...", notifications=["mail"])`.
- `stock_recorded`: only a stock level moved and no Stock Alert was reached. These never add to `unseen`.
- No entry at all: no change was recorded. A change that does not meet the conditions set in the app is not recorded unless Always record change detections is on (in Conditions & Filters), and an element whose `threshold` is -1 (Do not trigger) never records a change on its own.

While a report covers the monitor, its digest lists the monitor's saved changes whatever their status, apart from first captures and stock-only checks, filtered by the report's Minimum change importance. So a `low_priority` or `no_channels` change can still be in the digest.

When the status is `notified` but nothing arrived:

- A report covers the monitor (`reports` in `get-monitor-details`): the change is in the report's digest ([automation.md](automation.md)).
- The workspace's change alerts are switched off under Settings > Notifications, which `get-monitor-details` also reports: a monitor no report covers then sends nothing to its channels.
- A chat channel delivers only once it is connected in the app. Phones need alerts turned on for the device in the PageCrawl app.
- Email reaches a member only when their address is verified and email notifications are on in their personal settings, and not when they muted the monitor or its folder. No tool shows these settings, so ask the person to check.
- A failing monitor sends one error alert per run of failures, once it reaches the number of failed checks set in Report Errors (on the monitor, or for the workspace under Settings > Notifications, Error Alerts). `soft_error` and `unknown_error` never send one, and an `unknown_error` check does not add to `failed`. So look for an error `status` as well as `failed` above 0, rather than waiting for an email.

## Noisy monitors

Signs: `get-intelligence` lists the monitor as changing on every check, or the user keeps clearing changes nobody cares about. Fixes, gentlest first:

1. Raise the monitor's Minimum importance: `manage-monitors(action="update", monitor_id="...", min_importance=60)`. Changes the AI scores lower stay in the history without an alert. It has no effect while Importance Scoring is off. The AI does not score a check where only one price, number or availability value moved (stock levels moving beside it do not count), or only stock levels, so the minimum never holds such a change back. When anything else changed in the same check, such as the page text on a monitor that also tracks it, the AI scores the change and the minimum can hold it back: read the `importance` those changes got with `get-monitor-history(monitor_id="...", changes_only=true)` and keep the minimum below it. The workspace default is in the workspace's AI settings in the app. It also applies only while the monitor has no alert conditions other than Stock Alerts and the Ignore text, Ignore numbers and Ignore dates filters (`has_alert_conditions` false in `get-monitor-details`). Once one is set, the conditions alone decide whether a change alerts, and the thumbs down in step 3 no longer mutes alerts either, so adjust the conditions instead. The user sees and changes them under Edit on the monitor in the app.
2. Sharpen `ai_page_focus` (What matters, in the app) so the scores reflect what matters: `manage-monitors(action="update", monitor_id="...", ai_page_focus="Only plan prices and limits matter; ignore testimonials, dates and counters")`.
3. In the app: the Ignore numbers and Ignore dates filters in Conditions & Filters, the Frequently changing content detected panel on the monitor, and a thumbs down on a change, which mutes similar ones.
4. Narrow what is tracked: `content_only` or `reader` instead of the whole page, or one element with a selector. The user does this under Edit on the monitor, which keeps the history, or you create a corrected monitor.

`min_priority` and a report's `filter_mode="important"` change only what is read or put in a digest, not what alerts.

More: https://pagecrawl.io/help/reduce-false-positives/article/reduce-false-positives-monitoring-website-for-changes

## Silent monitors

A monitor that has never recorded a change may be watching the wrong thing rather than a quiet page. Read what it captures with `get-latest-values`. If the value is empty, a cookie notice, or not what the user meant, diagnose with `test-configuration(url="...", tracking_mode="...", selector="...")` and fix it (Edit in the app, or a corrected monitor). `test-configuration` fetches the page live without signing in or running page steps, and it is slow: use it once per question, never in a loop. Also look in `get-monitor-details` for elements whose `threshold` is -1, and at `has_alert_conditions`: when it is true, a change that misses the conditions set in the app is not recorded unless Always record change detections is on.

## Pages that will not load

Some sites refuse automated visits or limit how often they can be checked. PageCrawl retries on later checks, and a monitor that checks hourly or less often can wait longer between checks the longer it keeps failing, until one succeeds, so `last_checked_at` can look overdue. If a page keeps failing:

- Confirm the URL works in a normal browser and is not behind a login. A page that opens in the user's own browser can still refuse automated visits.
- Rate limits: when several monitors watch one site, the team's Crawl Politeness setting spaces their checks out, and the monitor's Rate limits (429) handling setting retries after the wait the site asks for. Both are set in the app, and a lower check frequency also helps.
- `blocked_by_captcha`: the user can connect a CAPTCHA solving service under Settings > Workspace > Integrations: https://pagecrawl.io/help/troubleshooting/article/bypass-captcha-tracked-pages
- If the team respects robots.txt and bot protection, PageCrawl makes no second attempt another way: CAPTCHA solving, other connections, saved logins and HTTP Basic credentials are not used. https://pagecrawl.io/help/features/article/respect-robots-txt
- A failed check keeps no screenshot. To see what a failing check received, the user turns on Debugging mode under Power User Settings in the monitor's settings: the next failed check appears under Last error, and the mode switches itself off after three days.
- The options in the user's account: https://pagecrawl.io/help/troubleshooting/article/how-to-unblock-a-blocked-page

Do not try to fetch the page another way to work around the block.

## Tool errors

| Error | What to do |
|---|---|
| A plan limit or quota (monitor count, check allowance, on-demand checks, a feature the plan lacks) | Tell the user what the message says. Do not retry or work around it |
| Turning a monitor on asks to verify the email address first | Pass the message on: the user opens PageCrawl, chooses Resend code on the Verify your email screen and enters the new code from that email (an older code has expired). Then turn the monitor on again |
| Too many requests (HTTP 429) | A per-minute limit on the connection: wait a minute, then continue with fewer calls, batching with `monitor_ids` |
| A validation message (a missing `selector`, `prompt` or `workspace_id`, or a frequency or channel that is not offered) | Fix the arguments and call again |
| Monitor not found | Search with `list-monitors(search="...", include_disabled=true)`. A monitor in a workspace the user was not added to cannot be reached: the team owner or an Administrator adds them on the Members page |
| Workspace not found, or workspace_id must be a workspace ID | Pass a numeric ID from `list-workspaces` |
| View-only access | The user is a Viewer in that team. The team owner or an Administrator can give them the Standard User or Administrator role |
| Several monitors match a URL | Ask the user which one, showing names and what each tracks |
| An authentication error, or MCP is unavailable | The connection needs attention; see [setup.md](setup.md) |
