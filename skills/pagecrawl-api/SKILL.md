---
name: pagecrawl-api
description: Builds integrations with the PageCrawl REST API, webhooks and push data sources. Use when writing, reviewing or debugging code that creates PageCrawl monitors, reads change history or diffs, receives PageCrawl webhooks and verifies the X-PageCrawl-Signature header, polls for new changes, or sends values to a PageCrawl data source, including n8n, Zapier and Make workflows that call PageCrawl.
license: MIT
compatibility: Needs network access to pagecrawl.io and a PageCrawl API token created in Settings > API.
metadata:
  author: PageCrawl.io
  version: "1.1.3"
---

# PageCrawl API

For code that talks to PageCrawl. To use PageCrawl from a conversation through its MCP tools, use the `pagecrawl` skill instead.

- Base URL: `https://pagecrawl.io/api`
- Auth header: `Authorization: Bearer <api-token>`, where `<api-token>` is a token the user creates in PageCrawl under Settings > API. It works on every plan.
- Full specification: https://pagecrawl.io/api/openapi.yaml. Fetch it when you need exact request or response fields; it is the contract.
- Several workspaces: add `workspace_id` as a query parameter.

## Rules

1. **Keep the token server-side.** Read it from an environment variable or a secret store. Never commit it, log it, or ship it in browser or mobile code.
2. **Use documented endpoints only.** Anything not in the specification is internal and can change without notice.
3. **Verify every webhook** before trusting its body (below).
4. **Prefer webhooks to polling.** When you must poll, poll slowly and use the documented pattern.
5. **Handle limits.** HTTP 429 carries a `Retry-After` header: wait that long, then retry. A 403 for a plan limit or an exhausted quota is final; report it instead of retrying. Over the monitor limit, new pages are created disabled.
6. **Changes arrive after a scheduled check.** PageCrawl checks each page on its own schedule; do not build a loop that forces checks.

## Create a monitor

The quick endpoint needs only a URL:

```bash
curl -X POST "https://pagecrawl.io/api/track-simple" \
  -H "Authorization: Bearer <api-token>" \
  -H "Content-Type: application/json" \
  -d '{"url": "https://example.com/pricing", "tracking_mode": "content_only", "frequency": 1440,
       "ai_page_focus": "Plan prices and limits; ignore testimonials"}'
```

`tracking_mode` accepts the same modes as the PageCrawl MCP tools (`fullpage`, `content_only`, `reader`, `feed`, `price`, `specific_text`, `specific_number`, `json_path`, `ai_extract`, and more), with `selector` or `prompt` where a mode needs one. It responds with 201 and the page object. For full control (several elements, templates, logins) use `POST /api/pages`.

## Read changes

| Need | Endpoint |
|---|---|
| All pages with their latest values | `GET /api/pages?simple=1` |
| One page and its configuration | `GET /api/pages/{id}` |
| A page's check history | `GET /api/pages/{id}/history?simple=1` (`take` limits the count) |
| What changed in a check | `GET /api/pages/{id}/checks/{checkId}/diff.markdown`, `diff.html` or `diff.png` |
| Screenshots | `GET /api/pages/{id}/checks/latest/screenshot` or `GET /api/pages/{id}/checks/{checkId}/screenshot` |

Details, pagination and response fields: [references/endpoints.md](references/endpoints.md).

## Receive webhooks

Webhooks deliver each detected change to your server as a signed JSON POST. Create one with `POST /api/hooks` (the response includes the `signing_secret`) or in the PageCrawl app, where the secret is shown on the webhook.

Verify every delivery:

1. Read the headers `X-PageCrawl-Signature` (`sha256=<hex>`) and `X-PageCrawl-Timestamp` (Unix seconds).
2. Reject the delivery if the timestamp is more than 300 seconds from now.
3. Compute `HMAC-SHA256(signing_secret, "{timestamp}.{raw body}")` over the exact raw request bytes, not a re-serialized object.
4. Compare it with the header value in constant time. Reject on mismatch.
5. Respond quickly with a 2xx and do slow work afterwards.

Receivers in Node.js, Python and PHP, the payload fields, and local testing: [references/webhooks.md](references/webhooks.md).

## Poll for changes

When a webhook is not possible, poll `GET /api/pages?simple=1` on a slow interval, or `GET /api/pages/{id}/history?simple=1` for each page you care about, and compare check IDs with the last ones you stored. Agents connected through MCP can use the `get-changes-since` cursor instead.

## Push values in

For values that are not on a web page (internal metrics, partner API responses), create a data source and push values to it:

```bash
curl -d value=42 "$PAGECRAWL_INGEST_URL"
```

The `ingest_url` needs no Authorization header because the URL itself is the secret: store it like a password. Details, named fields and the authenticated variant: [references/data-sources.md](references/data-sources.md).

## No-code tools

n8n, Zapier and Make have PageCrawl integrations, so a workflow often needs no code:

- https://pagecrawl.io/help/integrations/article/pagecrawl-n8n-integration
- https://pagecrawl.io/help/integrations/article/pagecrawl-zapier-integration
- https://pagecrawl.io/help/integrations/article/pagecrawl-make-integration
