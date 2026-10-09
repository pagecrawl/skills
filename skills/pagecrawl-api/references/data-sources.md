# Push data sources

A data source is a monitor that receives values instead of crawling a page. Pushed values get the same comparison, history, charts, alerts and AI summaries as any monitor. Use one for internal metrics, values from a partner API, or anything your own code already has.

## Create one

```bash
curl -X POST "https://pagecrawl.io/api/data-sources" \
  -H "Authorization: Bearer <api-token>" \
  -H "Content-Type: application/json" \
  -d '{"name": "Warehouse stock", "fields": [{"label": "units", "type": "number"}]}'
```

- `fields` is optional. Leave it out to track a single value; each named field otherwise gets its own history and chart. Field types are `number`, `price` and `text`.
- The response includes `ingest_url` and `ingest_email`. Both are secrets: anyone with the URL can push values. Store them like passwords and never put them in client-side code or public repositories.

## Push values

Secret URL, no Authorization header (the URL is the credential):

```bash
curl -d value=128 "$PAGECRAWL_INGEST_URL"
curl -H "Content-Type: application/json" -d '{"values": {"units": "128", "status": "ok"}}' "$PAGECRAWL_INGEST_URL"
```

Authenticated alternative: `POST /api/ingest/{id}` with the usual `Authorization: Bearer` header and the same body. Email works too: send the value as the message body to the `ingest_email` address.

Behaviour to rely on:

- Pushing the same value again records nothing new.
- Field names that do not exist yet are created on first use.
- Every accepted push counts as a check against the account's allowance, even when the value did not change, so push on change or on a sensible interval rather than in a tight loop.
- Ingest has its own hourly limit per data source; a 429 response carries `Retry-After`.

Reading values back uses the normal endpoints (`GET /api/pages/{id}`, `GET /api/pages/{id}/history`). More: https://pagecrawl.io/help/integrations/article/push-api-data-sources
