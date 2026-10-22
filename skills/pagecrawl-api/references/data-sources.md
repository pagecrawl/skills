# Push data sources

A data source is a monitor that receives values instead of crawling a page. Pushed values get the same comparison, history, charts, alerts and AI summaries as any monitor. Use one for internal metrics, values from a partner API, or anything your own code already has. The user can also create one in the app, under Settings > API access (Create Data Source).

## Create one

```bash
curl -X POST "https://pagecrawl.io/api/data-sources?workspace_id=<workspace-id>" \
  -H "Authorization: Bearer <api-token>" \
  -H "Accept: application/json" \
  -H "Content-Type: application/json" \
  -d '{"name": "Warehouse stock", "fields": [{"label": "units", "type": "number"}], "notifications": ["mail"]}'
```

- `fields` is optional. Leave it out to track a single value; otherwise each named field gets its own history and chart. Field types are `number`, `price` (with an optional `currency`) and `text`. Define number and price fields up front for values such as $12.99.
- `notifications` decides who is alerted: email when it is left out, nobody with `[]` or `null` (changes are then saved with the status `no_channels`). It takes `mail`, `slack`, `discord`, `telegram` and `teams`.
- `ai_prompt` tells the AI what matters for its summary and importance score; it is not a threshold. To alert only when a value crosses a limit, the user adds a condition such as Less than under Conditions & Filters on the data source in the app. `"ai_summaries_enabled": false` skips the AI for frequent numeric metrics that only need charts.
- The answer is 201 with the monitor, including `ingest_url` and `ingest_email`. Both are secrets: anyone with the URL can push values and read the current values and recent history back, and the address cannot be changed later, so a leaked one means creating a new data source. Store them like passwords, and keep them out of client-side code, logs and public repositories.
- Past the account's monitor limit, or the API's own limit on some plans, the answer is 403.

## Push values

Secret URL, no Authorization header (the URL is the credential):

```bash
curl -d value=128 "<ingest-url>"
curl -H "Content-Type: application/json" -d '{"values": {"units": "128", "status": "ok"}}' "<ingest-url>"
```

- Bodies can be JSON or form-encoded. Send `value` for a data source with one field (on one without fields it creates a field named Value), and `values`, a map of field label to value, for several fields. A label that does not exist yet becomes a new field, typed as a number only when the value is a plain number such as 1299.
- The answer is 200 JSON with `changed`, `check_id` and the stored `values`. Add `"return_summary": true` to also get the AI `summary` and `score` of a recorded change.
- Authenticated alternative: `POST /api/ingest/{id}?workspace_id=<workspace-id>`, with the data source's ID or slug, the usual `Authorization: Bearer` header and the same body.
- Email: send the value as the message body to the `ingest_email` address.
- Errors: 404 for an unknown URL, 403 while the data source is paused, 422 for a value that does not fit (too long, no number for a number field, more fields than the plan allows), and 429 with `Retry-After` past the limit. A paused data source is resumed in the app, or with `PUT /api/pages/{id}?workspace_id=<workspace-id>` and `{"disabled": false}`, which queues no check for it (`PUT /api/pages/{id}/status` switches between paused and active on every call).

Behaviour to rely on:

- Pushing the same value again records nothing new and answers `"changed": false`. A field left out of a push keeps its previous value.
- The first value is saved as the starting point, and webhooks receive it as a first capture (`previous_check` is null).
- Every accepted push counts as a check toward the plan's allowance, even when the value did not change, so push when the value changes or on a sensible interval, never in a tight loop.
- The secret URL has an hourly limit per data source; authenticated pushes share one hourly limit across all of the account's data sources.

## Read values back

- `GET <ingest-url>` needs no Authorization header and returns `name`, `last_received_at`, and each field's current value in `fields` and `values`. Add `?history=20` (up to 100) for recent points. Reading does not count as a check.
- With a token, use `GET /api/pages/{id}?simple=1&take=1` and `GET /api/pages/{id}/history?simple=1&take=20`. The full `GET /api/pages/{id}` includes the `ingest_url` and `ingest_email`, so keep it out of logs.

More: https://pagecrawl.io/help/integrations/article/push-api-data-sources
