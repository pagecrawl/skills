# Webhooks

## Contents
- Create and manage webhooks
- What is delivered
- Verify deliveries
- Node.js receiver
- Python receiver
- PHP receiver
- Payload fields
- Failures, retries and switch-off
- Testing and debugging

## Create and manage webhooks

```bash
curl -X POST "https://pagecrawl.io/api/hooks?workspace_id=<workspace-id>" \
  -H "Authorization: Bearer <api-token>" \
  -H "Accept: application/json" \
  -H "Content-Type: application/json" \
  -d '{"target_url": "https://your-server.example.com/pagecrawl", "match_type": "tags", "tag_ids": [12],
       "events": ["change_detected", "error"]}'
```

The answer is 201 with the webhook, including its `id` and `signing_secret`. Keep the secret with your other secrets and hand it to the receiver, as the examples below do; never print it. Webhooks made in the app show it on the webhook's form, under Signing Secret.

- `events`: `change_detected` (the only one when left out) and `error`, which also delivers failed checks. Do not send `event_type`: it only records which integration created the webhook.
- `match_type` decides which monitors fire it, all in the webhook's own workspace: `all` (the default), `monitors` with `change_ids` (IDs or slugs), `tags` with `tag_ids`, `folders` with `folder_ids` (subfolders included), or `domains` with `domains`, exact host names (`example.com` does not cover `www.example.com`). Always send `match_type` together with its list: a list sent without it is ignored and the webhook covers every monitor in the workspace. The list needs at least one entry: `monitors`, `tags`, `folders` or `domains` with an empty, `null` or missing list answers 422.
- `payload_fields` limits the body to the fields you list. Left out, every field is sent.
- `target_url` must start with `https://`. Slack, Discord and Microsoft Teams addresses are refused, because PageCrawl posts to those apps itself: connect them in the app (Settings > Integrations, Notification Channels) and choose them in a monitor's `notifications`.
- Creating again makes a second webhook, and every change then arrives twice. Call `GET /api/hooks` first and look for your `target_url`.
- Reading webhooks needs a role that can view the workspace; creating, changing, testing and removing them needs the owner, an Administrator or a Standard User.

| Method and path | Use |
|---|---|
| `GET /api/hooks` | The workspace's webhooks, with `is_active`, `failures`, `last_sent_at` and `last_failure_at`, and `fields`: an example delivery built from the latest check of the covered monitor checked most recently (null until a covered monitor has a check) |
| `GET /api/hooks/{id}` | One webhook, with the same `fields` example |
| `PUT /api/hooks/{id}` | Change it. `target_url` is required on every PUT, so send the current one to keep it; other fields left out keep their values. To change what it covers, send `match_type` together with the new list; the webhook's current `match_type` sent without a list keeps its links. `{"target_url": "<current>", "is_active": true}` turns a switched-off webhook back on, and `false` pauses it |
| `PUT /api/hooks/{id}/test` | Send one real delivery now (see Testing and debugging) |
| `GET /api/hooks/{id}/logs` | Recent attempts, 25 per page: `event` (`success`, `retrying`, `failed` or `skipped`), `status_code`, `error_message`, `attempt`, `check_id` |
| `DELETE /api/hooks/{id}` | Remove it. It works on a webhook the token's user created, in any workspace, and on any webhook in a workspace they were added to, given the owner, Administrator or Standard User role in the webhook's team (403 otherwise). Other IDs answer 404 |

## What is delivered

PageCrawl sends a `change_detected` delivery for every change it saves on a covered monitor, whether or not that change sent an alert:

- A monitor's first capture, a data source's first value, and the first check after a monitor's history was cleared. `previous_check` is null: store the values as the starting point.
- Changes PageCrawl kept quiet: scored below the monitor's Minimum importance, like changes that were marked as not useful, the first change after the settings were saved with Skip first notification (`skip_first_notification`), or values saved while the monitor's Always record change detections switch (in Conditions & Filters) is on. Without that switch, a change that does not meet the alert conditions set on the monitor is not saved, so it is never delivered. A value pushed to a data source is saved whenever it differs from the last one, whatever the conditions.
- Not delivered: a check where only a stock level moved and no Stock Alert was reached.

With `error` in `events`, a failed check (apart from brief, passing errors) sends a delivery that repeats the monitor's last saved check: the same `id`, `changed_at`, `contents` and diffs, with the error in `status` and `event_type` set to `error`. A monitor that never completed a check sends none.

The payload does not say whether PageCrawl sent an alert. Decide in the receiver, as the examples' classify step does:

- `status` other than `ok`: a failed check. `status` and `event_type` describe the monitor when the delivery was built, so read the values too before treating it as news.
- `previous_check` null: a first capture.
- `ai_priority_score`: the AI's importance score from 0 to 100 as text, such as `"72.00"`. It is null when the AI did not score the change, for example a change where only a price, number, stock or availability value moved, a first capture, or any change while AI analysis is off. Treat null as "keep", or a threshold silently drops every price move.
- PageCrawl's own verdict is the check's `status` in `GET /api/pages/{id}/history?simple=1&take=20` (match the delivery's `id`): `notified` means it was sent as an alert, or handed to a report that covers the monitor. That report's digest also lists changes with the other statuses, apart from `initial` and `stock_recorded`.

The same check can arrive more than once (retries, a test delivery, Sync History in the app), so make processing idempotent on `id` together with `status`. Deliveries are not guaranteed to arrive in check order: order them by `changed_at`.

## Verify deliveries

Every delivery is a JSON `POST` with these headers:

- `X-PageCrawl-Signature: sha256=<hex digest>`
- `X-PageCrawl-Timestamp: <Unix seconds>`
- `User-Agent: PageCrawl.io Webhook Sender/1.0`

The digest is `HMAC-SHA256(signing_secret, "{timestamp}.{body}")` over the exact bytes of the body. To verify:

1. Reject a missing header, a timestamp that is not a plain number, or one more than 300 seconds away from your clock (replay protection). Keep the receiver's clock synchronised.
2. Compute the HMAC over the raw bytes. Never re-encode parsed JSON: PageCrawl escapes slashes and non-ASCII letters (`\/`, `\u00e9`), so a re-encoded body never matches, even when it looks the same.
3. Compare in constant time. Answer 401 on a mismatch, never 403, 404 or 410 (see Failures, retries and switch-off).
4. Answer 2xx within 60 seconds and do slow work afterwards.

The Node.js and Python receivers below were tested against deliveries signed exactly as PageCrawl signs them, including an 8 MB delivery, tampered and re-encoded bodies, stale timestamps and malformed headers: they answer 204 to genuine deliveries, 401 to bad signatures and 400 to bodies that are not a JSON object.

## Node.js receiver

```js
const crypto = require("crypto");
const express = require("express");

const MAX_AGE_SECONDS = 300;

// True when rawBody (a Buffer) was signed with secret less than five minutes ago.
function verifySignature(secret, timestamp, signature, rawBody) {
  if (!secret || typeof timestamp !== "string" || typeof signature !== "string") return false;
  if (!/^\d{1,12}$/.test(timestamp)) return false;
  if (Math.abs(Date.now() / 1000 - Number(timestamp)) > MAX_AGE_SECONDS) return false;

  const expected = crypto.createHmac("sha256", secret).update(`${timestamp}.`).update(rawBody).digest("hex");
  const provided = signature.startsWith("sha256=") ? signature.slice(7) : signature;
  const a = Buffer.from(expected);
  const b = Buffer.from(provided);
  return a.length === b.length && crypto.timingSafeEqual(a, b);
}

// PageCrawl delivers every change it saves, not only the ones that sent an alert.
function classify(payload, minImportance = 50) {
  if (payload.status !== undefined && payload.status !== "ok") return "failed-check";
  if (payload.previous_check === null) return "first-capture"; // nothing to compare yet
  const score = payload.ai_priority_score; // "72.00", or null when the AI did not score it
  if (score != null && Number(score) < minImportance) return "low-importance";
  return "change";
}

// signingSecret: the webhook's signing secret, from your configuration.
// onDelivery(kind, payload): your handler, called after PageCrawl has its answer.
function createApp(signingSecret, onDelivery) {
  const app = express();
  // Raw bytes, before any app-wide express.json(): the signature covers them exactly.
  // Deliveries that carry page text can be several megabytes.
  app.post("/pagecrawl", express.raw({ type: "application/json", limit: "25mb" }), (req, res) => {
    const rawBody = Buffer.isBuffer(req.body) ? req.body : Buffer.alloc(0);
    const timestamp = req.get("X-PageCrawl-Timestamp");
    const signature = req.get("X-PageCrawl-Signature");
    if (!verifySignature(signingSecret, timestamp, signature, rawBody)) return res.sendStatus(401);
    let payload;
    try {
      payload = JSON.parse(rawBody.toString("utf8"));
    } catch {
      return res.sendStatus(400);
    }
    if (payload === null || typeof payload !== "object") return res.sendStatus(400);
    res.sendStatus(204);
    setImmediate(() => onDelivery(classify(payload), payload));
  });
  return app;
}

module.exports = { verifySignature, classify, createApp };
```

- Register this route before any app-wide `express.json()`: a body that was already parsed fails every check.
- `onDelivery` runs after the answer is sent, so catch its errors yourself or hand the work to a queue.
- Deliveries that carry page text can be large: several megabytes for a long page, and up to about 25 MB for the longest pages in scripts such as Chinese or Japanese. Keep any proxy in front of the receiver above that size (nginx accepts 1 MB by default), or limit `payload_fields`.

## Python receiver

```python
import hashlib
import hmac
import json
import time

from flask import Flask, request

MAX_AGE_SECONDS = 300


def verify_signature(secret, timestamp, signature, raw_body):
    """True when raw_body (bytes) was signed with secret less than five minutes ago."""
    if not secret or not timestamp or not signature:
        return False
    if not (timestamp.isascii() and timestamp.isdigit() and len(timestamp) <= 12):
        return False
    if abs(time.time() - int(timestamp)) > MAX_AGE_SECONDS:
        return False
    expected = hmac.new(secret.encode(), timestamp.encode() + b"." + raw_body, hashlib.sha256).hexdigest()
    provided = signature[7:] if signature.startswith("sha256=") else signature
    # Compare bytes, so a header with non-ASCII characters fails instead of raising.
    return hmac.compare_digest(expected.encode(), provided.encode("utf-8", "replace"))


def classify(payload, min_importance=50):
    """PageCrawl delivers every change it saves, not only the ones that sent an alert."""
    if payload.get("status", "ok") != "ok":
        return "failed-check"
    if "previous_check" in payload and payload["previous_check"] is None:
        return "first-capture"  # nothing to compare yet
    score = payload.get("ai_priority_score")  # "72.00", or None when the AI did not score it
    if score is not None and float(score) < min_importance:
        return "low-importance"
    return "change"


def create_app(signing_secret, on_delivery):
    """signing_secret: the webhook's signing secret, from your configuration."""
    app = Flask(__name__)

    @app.post("/pagecrawl")
    def receive():
        raw_body = request.get_data()
        if not verify_signature(
            signing_secret,
            request.headers.get("X-PageCrawl-Timestamp"),
            request.headers.get("X-PageCrawl-Signature"),
            raw_body,
        ):
            return "", 401
        try:
            payload = json.loads(raw_body)
        except ValueError:
            return "", 400
        if not isinstance(payload, dict):
            return "", 400
        on_delivery(classify(payload), payload)  # keep it quick, or hand it to a queue
        return "", 204

    return app
```

Run it behind a production WSGI server, and hand slow work to a queue so the answer goes back within 60 seconds.

## PHP receiver

The same steps in PHP:

1. Read the raw body with `file_get_contents('php://input')`, and the `X-PageCrawl-Timestamp` and `X-PageCrawl-Signature` headers.
2. Answer 401 unless the timestamp is digits only and within 300 seconds of `time()`.
3. Compute `hash_hmac('sha256', "{timestamp}.{body}", <signing secret>)` over the raw body, remove the `sha256=` prefix from the header, and compare the two with `hash_equals()`. Answer 401 on a mismatch.
4. Decode the body with `json_decode(..., true)` and answer 400 unless the result passes `is_array()`.
5. Answer 204, then do slow work: under PHP-FPM, call `fastcgi_finish_request()` first, or hand the work to a queue.

Set `post_max_size` (8M by default) above the largest delivery you expect, for example 32M: PHP warns about larger bodies before your code runs. A complete PHP receiver is in https://pagecrawl.io/help/tutorials/article/reference-implementations.

In Laravel, read the raw body with `$request->getContent()` and the headers with `$request->header(...)`, and exclude the route from CSRF protection.

## Payload fields

A webhook created with `POST /api/hooks` without `payload_fields` sends every field below. A webhook created in the app starts with the Essential set: `id`, `title`, `changed_at`, `status`, `difference`, `human_difference`, `short_summary`, `markdown_difference`, `page`, `ai_summary` and `ai_priority_score`, without `contents`, `page_elements` or `previous_check`. Choose All fields or Custom in its settings, or send `payload_fields`, when the receiver needs more. Read every field defensively.

| Field | Meaning |
|---|---|
| `id` | The check ID (for an error delivery, the monitor's last saved check) |
| `title` | The monitor's name |
| `status` | The monitor's status when the delivery was built: `ok`, or the error of a failed check |
| `event_type` | `change_detected` when `status` is `ok`, otherwise `error` |
| `changed_at` | When the check ran (ISO 8601, UTC) |
| `content_type` | The page's HTTP Content-Type, such as `text/html; charset=utf-8` (not the tracking mode) |
| `contents` | The current value of the monitor's main tracked element; for a whole-page monitor, the page text |
| `original` | For prices and numbers, the text the value was read from |
| `elements` | How many page elements the main selector matched (a number) |
| `difference`, `human_difference` | How much the main value changed: a whole-number percentage (the share of text that changed, or the signed move of a number or price), or null, and the same in a sentence |
| `visual_diff` | How much of the screenshot changed, in percent, when visual comparison ran |
| `short_summary` | A one-line description of the change |
| `markdown_difference`, `html_difference` | The text diff (null for JSON content) |
| `page` | The monitor: `id`, `name`, `url`, `url_tld`, `slug`, `link`, `folder` |
| `page_elements` | One entry per tracked element: `element_id` (stable across checks, so key your records on it), `label`, `type`, `contents`, `original`, `difference`, `changed`, `short_summary` |
| `ai_summary` | The AI summary of the change, or null |
| `ai_priority_score` | The AI's importance score as text such as `"72.00"`, or null when the AI did not score the change |
| `page_screenshot_image`, `text_difference_image` | Signed image links; the screenshot link is null while screenshots are off. The screenshot is the whole page at the size it was captured, so scale or crop it before passing it to a vision model |
| `previous_check` | The previous check's values, diffs and `page_elements`; null on a first capture. Not sent to Zapier |
| `json`, `json_patch` | For JSON content only: the document, and a patch from the previous one |

## Failures, retries and switch-off

- A 2xx answer within 60 seconds is a success. A 4xx or 5xx answer, a timeout or a refused connection is a failed attempt.
- A failed delivery is tried again with growing gaps, up to 15 attempts over about 17 hours, and dropped once the monitor has a newer check.
- A `410 Gone` answer deletes the webhook. A `403` or `404` on the fifth attempt switches it off, and so do 20 deliveries in a row that used up all their attempts. The workspace's members get a notice in the app. This is why a receiver must answer 401 for a bad signature.
- A switched-off webhook sends nothing, tests included. Once the receiver is fixed, turn it back on with `PUT /api/hooks/{id}` and `{"target_url": "<current>", "is_active": true}`, or in the app.
- PageCrawl follows up to 5 HTTPS redirects, but a `301` or `302` turns the delivery into a `GET` without a body. Point the webhook at the receiver's final address.
- Let the `User-Agent` above through any firewall or bot filter in front of the receiver.
- Deliveries can be lost during a long outage or while a webhook is switched off, so also run a slow reconcile poll ([endpoints.md](endpoints.md), Polling for new changes).

## Testing and debugging

- `PUT /api/hooks/{id}/test` (Test Webhook in the menu of the app's Configured Webhooks list) sends one real delivery built from the latest check of a monitor the webhook covers: `monitor_id` (an ID or slug) picks which, otherwise the covered monitor checked most recently. It answers 200 with `success`, `message` and `monitor`, or 422 when that monitor is not covered or has no checks yet, or when nothing covered has a check. Tests are limited per hour. A receiver cannot tell a test from a real delivery.
- For local development, expose the receiver through an HTTPS tunnel: only HTTPS addresses are accepted.
- A 401 from your own receiver usually means the wrong secret, a body that was parsed or re-encoded before the check, or a clock more than five minutes off.
- A `413` in the webhook's log means the receiver, or a proxy in front of it, refuses large bodies.
- After the receiver was unavailable, reconcile with `GET /api/pages/{id}/history?simple=1&take=20` for the monitors it covers.
