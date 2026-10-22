# Endpoints

The specification at https://pagecrawl.io/api/openapi.yaml lists every documented endpoint and field. This page covers the endpoints integrations use most, and the behaviour that is easy to get wrong.

## Contents
- Tokens, roles and workspaces
- Errors and limits
- Create monitors
- Update and delete monitors
- List and read monitors
- Polling for new changes
- Diffs, screenshots and review

## Tokens, roles and workspaces

- Send `Authorization: Bearer <api-token>` on every request. The user creates the token under Settings > API access (Create Token), with a name and an optional Expiration. It is shown once, has full access on every plan, and answers 401 once it expires or is revoked on the same page.
- A token acts as the user who created it, with that user's role in the team. The owner, an Administrator or a Standard User can change things; a Viewer can only read (403 `{"message": "This action is unauthorized."}` on every change); a Billing manager cannot use monitors or webhooks at all.
- OAuth tokens that other apps received (an AI assistant connection, the mobile app, an automation tool) can carry narrower scopes, and then answer 403 `{"message": "Invalid scope(s) provided."}` outside them. Create a token for your own code instead of reusing one.
- The `api_token` query parameter also authenticates, but the token then lands in server and proxy logs. Use the header.
- `workspace_id` goes in the query string; a `workspace_id` in a JSON body is ignored. Without it, a request uses the workspace the user last opened in the app. The IDs are listed under Workspace Selection on the Settings > API access page (shown when the account has more than one workspace), and `GET /api/pages/{id}` returns each monitor's `workspace_id`. An ID the user cannot open answers 422 `{"message": "Unknown or inaccessible workspace_id."}`.
- Developer mode, a switch on the same page, shows page and check IDs in the app, an open folder's Folder ID, and Template ID and Auth ID columns. With it on, the screen that creates a monitor also shows the `POST /api/pages` request that builds the configuration on screen: a quick way to get the exact fields for a setup.
- Reading one monitor (`GET /api/pages/{id}`, its history, diffs and screenshots) finds it in any workspace the user can open; with `workspace_id` it must be in that workspace. Lists, creates, `PUT` and `DELETE /api/pages/{id}`, webhooks, data-source creation and authenticated pushes work in the request's workspace, and a monitor in another one answers 404.

## Errors and limits

Every endpoint named in this skill answers errors as JSON, whatever `Accept` header the request sends. Sending `Accept: application/json` is still good practice.

| Status | Body | Meaning |
|---|---|---|
| 401 | `{"message": "Unauthenticated."}` | Token missing, wrong, revoked or expired |
| 403 | `{"message": "This action is unauthorized."}` | The token's user has no right to do this (a Viewer making a change) |
| 403 | `{"message": "Invalid scope(s) provided."}` | An OAuth token whose scope does not cover the endpoint |
| 403 | `{"error": "email_unverified", ...}` | The account's email address is not verified yet; the user verifies it in the app |
| 403 | `{"error": "two_factor_required", ...}` | The team requires two-factor authentication and the user has not set it up |
| 403 | A message naming the limit | A plan limit; final, so report it |
| 404 | `{"message": "The route api/... could not be found."}` | No such endpoint: check the path |
| 404 | `{"message": "Monitor not found."}` | No such monitor within reach. Reading a monitor (details, history, diffs, screenshots) or pausing, resuming or checking it looks in every workspace the user can open, or in the `workspace_id` sent; `PUT` and `DELETE /api/pages/{id}` look only in the request's workspace |
| 404 | `{"message": "More than one of your monitors uses this slug. ..."}` | The slug is not in the selected workspace and several of the user's other monitors use it: send the monitor ID, or `workspace_id` |
| 404 | `{"message": "Not Found"}` | A check or webhook that is not within reach (a missing check can also answer an empty message) |
| 405 | `{"message": "The ... method is not supported ..."}` | The path exists but takes another method; the `Allow` header lists them |
| 406 | A message | Resuming a paused monitor was refused: the team that owns it is at its plan's active-monitor limit |
| 422 | `{"message": "...", "errors": {"field": ["..."]}}` | Validation failed and nothing was saved |
| 429 | `{"message": "Too Many Attempts."}` | Too many requests to that endpoint: wait for `Retry-After` seconds |
| 429 | `{"message": "Too Many Requests."}` | Too many requests from one IP address, to any endpoint: wait for `Retry-After` seconds |

- Limits apply per endpoint and per account: listing and history, diff images, pushes to a data source and webhook tests each have their own. Keep poll intervals long and send requests one at a time.
- Separately, every IP address has a flood limit, signed in or not. Past it, every request from that address gets `429` with `{"message": "Too Many Requests."}` and a long `Retry-After`, for several minutes. Spread calls out instead of bursting, and honour every `Retry-After`.
- A few 429s carry no `Retry-After` (deleting many monitors in a short time on some plans): wait several minutes before trying again.
- Plan limits: `POST /api/pages` and `POST /api/data-sources` answer 403 past the account's total monitor limit, and some plans also limit how many monitors the API can manage (403). Past the active-monitor limit, a create succeeds but the monitor is paused: `POST /api/pages` answers 201 with `disabled: true`, `paused_over_limit: true`, `active_pages` and `max_pages`, and `POST /api/track-simple` answers 201 with `disabled: true`. Resuming past the limit answers 406. Tell the user rather than retrying.

## Create monitors

`POST /api/track-simple` takes a `url` and optional `name`, `tracking_mode` (with `selector` or `prompt` where the mode needs one), `track_stock`, `frequency`, `ai_page_focus`, `folder_id`, `template_id`, `auth_id` (a saved login), `tags`, `notifications`, `actions`, `headers`, `screenshots` and `ignore_duplicates`.

- Defaults: `frequency` 1440 (daily), cookie-banner and overlay removal (`actions` replaces them; `[]` means none), screenshots on, an automatic location, and alerts by email only. Left out or `null`, `notifications` means email even when the workspace has chat apps connected; `[]` turns alerts off; chat channels the workspace has not connected are dropped, and with none left the monitor sends no alerts.
- `headers` takes the same object (or JSON string) as on `POST /api/pages` below, under the same rules; `{}` means no custom headers.
- A `template_id` links the monitor to a template, whose settings apply wherever the template sets them. A template that is being deleted, or is not in the request's workspace, answers 422.
- It answers 201 with the page object, or 200 with the existing monitor when `ignore_duplicates` is true and the address is already monitored.

`POST /api/pages` takes the full configuration. `url` and `frequency` are required (`frequency` comes from the template when `template_id` is sent).

- `elements` lists what to track; the first one is the main value. Left out, the monitor tracks the whole page. Give each element a `label`. What `selector` means depends on `type`:

  | `type` | `selector` |
  |---|---|
  | `fullpage` | `*` for all text, `content` for the main content, `article` for long-form text |
  | `text`, `number`, `html` | A CSS or XPath selector (required) |
  | `price`, `stock` | `*` to find the value, or a CSS or XPath selector. `price` reads only the price, so add an `availability` element for in-stock status |
  | `availability` | Empty, or extra sold-out phrases separated by commas (never `*`) |
  | `json_path` | A JMESPath expression such as `items[0].price` (a leading `$.` is accepted; JSONPath filters are not) |
  | `javascript` | JavaScript that returns the value |
  | `ai_extract` | Leave empty and describe the value in `prompt` |
  | `feed` | Optional: the repeating items, found automatically when empty |

  The specification lists the other types.
- Unlike `POST /api/track-simple`, it starts with no page actions and no screenshots: send `"actions": [{"type": "remove_cookies_v2"}, {"type": "remove_overlays"}]` to remove cookie banners and overlays, and `"screenshots": true` for screenshots and visual comparison. The location is automatic.
- `notifications` left out means email; `[]` or `null` means no alerts, and changes are then saved with the status `no_channels`. A linked template that sets its own channels decides them instead.
- `headers` is an object of header names to text values, such as `{"Accept-Language": "en-GB"}`, or that object as a JSON string. Responses return it as a JSON string. An invalid header name, a value that is not text, or a line break answers 422.
- `ignore_duplicates: true` returns an existing monitor of the same address (and the same element selectors, when `elements` is sent) with 200 instead of creating another.
- Many at once: send `"track_type": "multiple"` with `urls`, one address per line or an array, where `https://example.com/a || Name` also sets a name. The answer is 200 `{"success": true, "queued": 2, "message": "..."}`: the monitors are created in the background, and addresses that already have a monitor with the same template, or the same main selector, are skipped. List the monitors afterwards to find their IDs. A list with no usable address answers 422, and a plan that does not include bulk creation through the API answers 403.

## Update and delete monitors

`PUT /api/pages/{id}?workspace_id=<workspace-id>` changes only the fields in the body and answers 200 with the full page object.

- Kept when left out: every field, including `actions`, `elements`, `tags`, `notifications`, `headers` and the paused state.
- Lists replace the stored list: `elements` (each kept element must carry its `id`; an element without one is added, and one left out is deleted), `tags`, `notifications`, `actions`. `elements` set to `[]` or `null` goes back to one whole-page element; `actions` set to `[]` or `null` removes the page actions.
- `disabled`: `true` pauses. `false`, `0` or `null` resumes, after the same checks as turning the monitor on in the app: 406 when the team that owns the monitor is at its plan's active-monitor limit. Send JSON booleans, because the string `"false"` counts as true.
- Side effects of every update: the failure count goes back to 0, an older monitor moves to the current way of reading pages, and an active monitor checked every 5 minutes or less often gets a check queued (a resumed one gets exactly one). Paused monitors and data sources get no check. Never send updates on a timer.
- The next check can record a difference the update itself caused, for example after a new `url` (the history is kept, and the new page is compared with the old one). Send `"skip_first_notification": true` with the update to save that first change without an alert; webhooks still receive it.
- A monitor that follows a template ignores the `elements` you send, because the template decides them. `template_id: null` answers 422 with "Choose Customize this page beside the template selector before editing these settings": the user detaches the monitor in the app first. Sending another `template_id` moves the monitor onto that template.
- The monitor must be in the request's workspace (404 otherwise).

`DELETE /api/pages/{id}?workspace_id=<workspace-id>` removes the monitor with all its checks, values and screenshots for good. Confirm with the user before deleting.

## List and read monitors

`GET /api/pages?simple=1&folder=*&take=1` returns one JSON array of monitors. It is not paged, and each row carries the full text of its returned checks, so keep `take` small.

- `folder`: `*` for every active monitor, in folders or not (paused monitors are left out), `/app/pages/folders/<slug>` for one folder. Without it, a workspace that uses folders (the default) returns only the monitors outside folders, paused ones included. `disabled=1` lists only the paused monitors, in every folder.
- `take`: with `simple=1`, how many recent checks come back per monitor (10 when left out). Without `simple=1` the rows have no checks and `take` limits how many monitors come back instead.
- Labels: `filters[tags][]=<label>` keeps monitors with that label (all of several, or any with `filters[tags_mode]=or`), and `filters[exclude_tags][]=<label>` leaves them out.
- Each row has `id`, `name`, `url`, `slug`, `status`, `last_checked_at`, `latest`, `elements`, `checks` and `history`. `elements` here is the monitor's list of tracked elements (`id`, `label`, `type`, `selector`). `latest` describes the newest value of the main element, with `contents` cut to 300 characters, `changed_at`, `difference` and `human_difference`. `history` repeats the main element's value from each returned check, newest first. It is always a JSON array, and a check with no reading of the main element is left out, so it can be shorter than `checks`.
- Simple rows have no `disabled` or `failed`. A paused monitor keeps the `status` of its last check; read `disabled` and `failed` (consecutive failed checks) from `GET /api/pages/{id}`.
- Each check in `checks` (newest first) has `id`, `status`, `created_at`, `ai_summary`, `priority_score` (the AI's importance score as text such as `"72.00"`, or null) and `elements`, an object keyed by element ID whose values hold `contents`, `original` (for numbers and prices, the text the value was read from), `difference` and `changed`. Map values to your own records by element ID, never by position.
- A check's `status`: `notified` was sent as an alert (or, while a report covers the monitor, handed to that report); `initial` is a first capture or a new starting point; `stock_recorded` saved only a stock level; `low_priority`, `learned_noise`, `skipped_first`, `skipped`, `no_channels` and `ok` are changes saved without an alert. A covering report's digest lists the saved changes whatever their status, apart from `initial` and `stock_recorded`, filtered by the report's Minimum change importance.

One row, shortened:

```json
{
  "id": 343,
  "name": "Example pricing",
  "slug": "example-pricing",
  "url": "https://example.com/pricing",
  "status": "ok",
  "last_checked_at": "2026-10-07T19:30:16.000000Z",
  "latest": {"contents": "Standard: $17.99 / month ...", "changed_at": "2026-10-07 19:30:16", "difference": null},
  "elements": [{"id": 1494, "label": "Pricing", "type": "text", "selector": "#plans"}],
  "checks": [
    {
      "id": 815,
      "status": "notified",
      "created_at": "2026-10-07T19:30:16.000000Z",
      "ai_summary": "...",
      "priority_score": "75.00",
      "elements": {"1494": {"element_id": 1494, "contents": "Standard: $17.99 / month ...", "original": null, "changed": 1}}
    }
  ],
  "history": [{"element_id": 1494, "contents": "Standard: $17.99 / month ...", "changed": 1}]
}
```

`GET /api/pages/{id}?simple=1&take=1` returns the same row for one monitor, plus its `workspace_id`. Send `take`: without it, every check the plan keeps comes back. Without `simple=1` it returns the whole configuration, which can include custom headers, the HTTP Basic credentials saved on the monitor (for editors using a full-access token) and, for a data source, its `ingest_url` and `ingest_email`. Keep that object out of logs and other systems.

`GET /api/pages/{id}/history?simple=1&take=20` returns the same object as a list row, with the newest `take` checks. Always send `take`: without it, up to 2,000 checks come back with their full contents (every check for number and price monitors).

## Polling for new changes

1. On the first run, store each monitor's newest check ID (`checks[0].id`) and report nothing.
2. Every 10 minutes or slower, call `GET /api/pages?simple=1&folder=*&take=1&workspace_id=<workspace-id>`.
3. For each monitor whose newest check ID is higher than the stored one, read `GET /api/pages/{id}/history?simple=1&take=20` (raise `take` until the list reaches the stored ID), handle every newer check oldest first, then store the newest ID. A monitor with no checks yet has an empty `checks`.
4. A new check is not always a change worth reporting. Skip `initial` (a first capture, which every new monitor has) and `stock_recorded`. To report only what PageCrawl alerted on, keep `notified`. Compare the values in `elements` with the ones you stored rather than trusting the ID alone.
5. A paused monitor drops out of `folder=*` results and comes back when it is resumed. Treat a monitor missing from one poll as paused, not deleted: a deleted one answers 404 on `GET /api/pages/{id}`.

Pair a webhook receiver with this poll on a long interval (hourly or slower) to catch anything a delivery missed.

## Diffs, screenshots and review

- `GET /api/pages/{id}/checks/{checkId}/diff.markdown`, `diff.html` and `diff.png` compare a check with the one before it. `latest` works as `{checkId}`, and `diff.png` takes `height` and `compact`.
- `GET /api/pages/{id}/checks/{checkId}/screenshot` and `GET /api/pages/{id}/checks/latest/screenshot` return the whole-page screenshot as a JPEG, at the size it was captured: often thousands of pixels tall. `GET /api/pages/{id}/checks/{checkId}/diff` and `GET /api/pages/{id}/checks/latest/diff` return the visual comparison image.
- When there is no image to give (screenshots off for the team or the monitor, none for that check, an image older than the plan keeps, or a file monitor), these answer 200 with a PNG placeholder. Check `Content-Type` before storing one: a real screenshot is `image/jpeg`.
- `GET /api/pages/{id}/checks/{checkId}/before-after.jpg` is one 1440 x 1080 JPEG of the page before and after a change, focused on it. It is the image to hand a vision model; scale or crop full screenshots first, because the tallest are refused or shrunk until the text is unreadable.
- `POST /api/pages/move-to-board` with `{"page_ids": [123], "target_board": "reviewed"}` moves each monitor's latest change card on the Review Board and marks all of its unseen changes as seen, for everyone in the workspace. Do it only for monitors the user named.
- `GET /api/pages/{id}/recordings` lists the captures of checks that found no change, for monitors that record every check (when the plan includes it).
