# Endpoints

The specification at https://pagecrawl.io/api/openapi.yaml is the contract. This page covers the endpoints integrations use most, and the details that are easy to get wrong.

## Contents
- Authentication and workspaces
- Pages
- History and diffs
- Screenshots
- Limits and errors

## Authentication and workspaces

- Send `Authorization: Bearer <token>` on every request. Tokens come from Settings > API in PageCrawl. OAuth access tokens work too.
- With several workspaces, pass `workspace_id` as a query parameter; otherwise the token's current workspace is used.

## Pages

| Method and path | Use |
|---|---|
| `POST /api/track-simple` | Create a monitor from a URL plus optional `tracking_mode`, `selector`, `prompt`, `frequency`, `ai_page_focus`, `folder_id`, `template_id`, `auth_id`, `notifications` |
| `POST /api/pages` | Create a monitor with full configuration (several elements, actions, headers) |
| `GET /api/pages` | List monitors with their latest data |
| `GET /api/pages/{id}` | One monitor's configuration and latest data |
| `PUT /api/pages/{id}` | Update a monitor |
| `DELETE /api/pages/{id}` | Delete a monitor and its history (permanent) |

Listing details:

- `simple=1` returns a lighter response without configuration.
- `take` limits how many recent checks come back per page (default 10, maximum 100). Use `take=1` when you only need the latest values.
- By default the list returns pages from the main view. Pass `folder=*` to include pages in every folder; without it, pages inside folders can be missing from the result.
- Each page carries `status` (`ok`, an error status, or `disabled`), `failed` (consecutive failed checks), and `latest` with `contents`, `changed_at`, `difference` and `human_difference` for the primary tracked element.
- Per-element values sit in each check's `elements`, keyed by `element_id`. Map them to your own records by that ID, not by position.

## History and diffs

| Method and path | Returns |
|---|---|
| `GET /api/pages/{id}/history?simple=1` | Recent checks for one monitor, newest first (`take` limits the count) |
| `GET /api/pages/{id}/checks/{checkId}/diff.markdown` | The text diff of one check as Markdown |
| `GET /api/pages/{id}/checks/{checkId}/diff.html` | The same as HTML |
| `GET /api/pages/{id}/checks/{checkId}/diff.png` | The same as an image (`height` and `compact` adjust it) |

Polling pattern: keep the newest check ID you have processed for each page, poll `GET /api/pages?simple=1&folder=*&take=1` on a slow interval, and fetch history or a diff only for pages whose latest check ID is new.

## Screenshots

| Method and path | Returns |
|---|---|
| `GET /api/pages/{id}/checks/latest/screenshot` | The latest screenshot |
| `GET /api/pages/{id}/checks/{checkId}/screenshot` | The screenshot of one check |
| `GET /api/pages/{id}/checks/latest/diff` | The latest visual comparison |
| `GET /api/pages/{id}/checks/{checkId}/diff` | The visual comparison of one check |

## Limits and errors

- Limits apply per endpoint and per account. A 429 response carries `Retry-After` in seconds: wait, then retry. Keep poll intervals well under the limit, especially when paging through many monitors.
- 422 means validation failed; the body names the fields.
- Over the plan's monitor limit, new pages are created disabled. A 403 for a plan limit is final; report it rather than retrying.
- 401 means the token is missing, wrong or revoked.
