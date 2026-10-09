# Webhooks

## Contents
- Create a webhook
- Verify deliveries
- Node.js receiver
- Python receiver
- PHP receiver
- Payload
- Testing and debugging

## Create a webhook

```bash
curl -X POST "https://pagecrawl.io/api/hooks" \
  -H "Authorization: Bearer <api-token>" \
  -H "Content-Type: application/json" \
  -d '{"target_url": "https://your-server.example.com/pagecrawl", "match_type": "all", "event_type": "change_detected"}'
```

The response includes `signing_secret`. Store it with your other secrets (for example `PAGECRAWL_SIGNING_SECRET`). Webhooks created in the PageCrawl app show the secret on the webhook's settings. `match_type` can also limit a webhook to chosen monitors, tags, folders or domains, and the payload fields can be trimmed in the webhook's settings.

## Verify deliveries

Every delivery is a JSON `POST` with two headers:

- `X-PageCrawl-Signature: sha256=<hex digest>`
- `X-PageCrawl-Timestamp: <Unix seconds>`

The digest is `HMAC-SHA256(signing_secret, "{timestamp}.{body}")`, where `{body}` is the exact raw request body. To verify:

1. Reject a missing header or a timestamp more than 300 seconds from now (replay protection).
2. Compute the HMAC over the raw bytes. Never re-serialize parsed JSON; whitespace and key order would change the digest.
3. Compare in constant time, then parse the body.
4. Answer with a 2xx quickly and do slow work afterwards.

## Node.js receiver

```js
const crypto = require("crypto");
const express = require("express");

const SIGNING_SECRET = process.env.PAGECRAWL_SIGNING_SECRET;
const MAX_AGE = 300; // seconds

function verifySignature(secret, timestamp, rawBody, header) {
  if (!secret || !timestamp || !header) return false;
  const ts = parseInt(timestamp, 10);
  if (Number.isNaN(ts) || Math.abs(Date.now() / 1000 - ts) > MAX_AGE) return false;

  const expected = crypto.createHmac("sha256", secret).update(`${timestamp}.${rawBody}`).digest("hex");
  const provided = header.startsWith("sha256=") ? header.slice(7) : header;
  const a = Buffer.from(expected);
  const b = Buffer.from(provided);
  return a.length === b.length && crypto.timingSafeEqual(a, b);
}

const app = express();
app.post("/pagecrawl", express.raw({ type: "*/*" }), (req, res) => {
  const raw = req.body.toString("utf8");
  if (!verifySignature(SIGNING_SECRET, req.get("X-PageCrawl-Timestamp"), raw, req.get("X-PageCrawl-Signature"))) {
    return res.sendStatus(401);
  }
  const payload = JSON.parse(raw);
  res.sendStatus(204);
  // Process after responding.
  console.log("change on", payload.page?.name ?? payload.title, payload.short_summary);
});

app.listen(8080);
```

## Python receiver

```python
import hashlib
import hmac
import os
import time

from flask import Flask, abort, request

SIGNING_SECRET = os.environ["PAGECRAWL_SIGNING_SECRET"]
MAX_AGE = 300  # seconds


def verify_signature(secret, timestamp, raw_body, header):
    if not secret or not timestamp or not header:
        return False
    try:
        ts = int(timestamp)
    except ValueError:
        return False
    if abs(time.time() - ts) > MAX_AGE:
        return False
    expected = hmac.new(secret.encode(), f"{timestamp}.".encode() + raw_body, hashlib.sha256).hexdigest()
    provided = header[7:] if header.startswith("sha256=") else header
    return hmac.compare_digest(expected, provided)


app = Flask(__name__)


@app.post("/pagecrawl")
def receive():
    if not verify_signature(
        SIGNING_SECRET,
        request.headers.get("X-PageCrawl-Timestamp"),
        request.get_data(),
        request.headers.get("X-PageCrawl-Signature"),
    ):
        abort(401)
    payload = request.get_json()
    print("change on", payload.get("title"), payload.get("short_summary"))
    return "", 204
```

## PHP receiver

```php
<?php

function verify_signature(string $secret, ?string $timestamp, string $rawBody, ?string $header): bool
{
    if ($secret === '' || $timestamp === null || $header === null || ! ctype_digit($timestamp)) {
        return false;
    }
    if (abs(time() - (int) $timestamp) > 300) {
        return false;
    }
    $expected = hash_hmac('sha256', "{$timestamp}.{$rawBody}", $secret);
    $provided = str_starts_with($header, 'sha256=') ? substr($header, 7) : $header;

    return hash_equals($expected, $provided);
}

$rawBody = file_get_contents('php://input');

if (! verify_signature(getenv('PAGECRAWL_SIGNING_SECRET') ?: '', $_SERVER['HTTP_X_PAGECRAWL_TIMESTAMP'] ?? null, $rawBody, $_SERVER['HTTP_X_PAGECRAWL_SIGNATURE'] ?? null)) {
    http_response_code(401);
    exit;
}

$payload = json_decode($rawBody, true);
http_response_code(204);
```

In Laravel, read the raw body with `$request->getContent()` and the headers with `$request->header(...)`, and exclude the route from CSRF protection.

## Payload

Every field below is sent unless the webhook's payload fields were trimmed. `event_type` (`change_detected` or `error`) is sent only when selected.

| Field | Meaning |
|---|---|
| `id` | The check ID |
| `title` | The monitor's name |
| `status` | `ok` for a change, otherwise the error status |
| `changed_at` | When the check ran |
| `contents` | The current value of the primary tracked element |
| `original` | For prices and numbers, the text the value was read from |
| `difference`, `human_difference` | How much changed, as a percentage and in words |
| `short_summary` | A one-line description of the change |
| `markdown_difference`, `html_difference` | The text diff |
| `page` | The monitor: `id`, `name`, `url`, `slug`, `link`, `folder` |
| `page_elements` | One entry per tracked element; key them by `element_id`, which is stable across checks |
| `ai_summary`, `ai_priority_score` | The AI summary and importance score (0 to 100), when AI analysis ran |
| `page_screenshot_image`, `text_difference_image` | Signed image URLs |
| `previous_check` | The previous check's values and diffs |
| `json`, `json_patch` | For JSON content: the document and a patch from the previous one |

Full field list and examples: https://pagecrawl.io/help/integrations/article/webhook-integration

## Testing and debugging

- Send a test delivery from the webhook in the PageCrawl app. It uses real data from the latest check.
- Failed deliveries are retried for a while. A webhook that keeps failing is turned off automatically and has to be reactivated in its settings once the endpoint is fixed. The delivery log in the app shows each attempt.
- For local development, expose the receiver through an HTTPS tunnel; PageCrawl only delivers to HTTPS URLs.
- A 401 from your own receiver usually means the HMAC was computed over parsed or re-encoded JSON instead of the raw body, or the wrong secret is configured.
