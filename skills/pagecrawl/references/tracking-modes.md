# Tracking modes and monitor settings

## Contents
- Choose a tracking mode
- AI extract
- Several values on one page
- Actions before the capture
- Pages behind a login
- Prices and stock
- Files
- Templates, notifications and saved defaults
- Settings only the app can change
- Values that cannot be crawled
- Errors you may see

## Choose a tracking mode

`add-page-monitor` takes one `tracking_mode`. Leave it out when unsure: most pages get the whole page text, and files are detected from the URL.

| `tracking_mode` | Watches | Good for |
|---|---|---|
| `fullpage` (default) | All visible text | Landing pages, documentation, dashboards |
| `content_only` | The main content, without navigation, header, footer and sidebars | Busy pages where boilerplate would cause noise |
| `reader` | The main article text | Terms of service, privacy policies, legal pages, news, blog posts |
| `feed` | Repeating items; alerts when new items appear | Job boards, product listings, release lists, search results, RSS and Atom feeds, sitemaps |
| `price` | The price, plus availability by default | Product pages |
| `specific_text` | The text of one element (needs `selector`) | A status line, a headline, one section |
| `specific_number` | The number in one element (needs `selector`) | A count, a rate, a score |
| `ai_extract` | Whatever a plain-language `prompt` describes | Values that are easier to describe than to select |
| `json_path` | One value in a JSON response (needs a JMESPath expression in `selector`) | APIs and data endpoints |
| `seo` | Title, meta description, canonical URL, robots tag, H1, social tags | SEO changes on your own or competitors' pages |
| `http_status` | The HTTP status code after any redirects | Whether a URL is up, gone or failing |
| `leaderboard` | Ranked rows (`selector` optional, CSS only) | Rankings, charts, leaderboards |
| `pdf_extract` | The text of a PDF | PDFs, though leaving `tracking_mode` out detects them too |

- `feed` compares the first 10 items of the list by default. Removed or edited items are recorded and alerted only when the user turns that on in the monitor's settings in PageCrawl, where the number of items can also be raised. For an RSS or Atom feed or a sitemap, monitor the feed's own address. Help: https://pagecrawl.io/help/features/article/feed-tracking-mode
- `http_status` follows redirects, so a URL that starts redirecting to a working page still reads 200. Where an address leads is tracked by the Redirect type, which the user adds in the app.
- `json_path` takes JMESPath, not JSONPath. A leading `$.` is accepted, and JSONPath-only syntax such as `$..price` is refused. Examples: `items[0].price`, `data."release-date"` for a key with a dash, `items[?name=='Pro'].price | [0]` for a filter, `length(items)` for a count.
- `specific_number`: every number in the selected text is joined into one value ("4.5 out of 5 (1,234 reviews)" is stored as 4551.234). Select the element that holds only the number, or use `ai_extract` and ask for that one number.

Examples:

- `add-page-monitor(url="https://example.com/pricing", name="Example pricing", tracking_mode="content_only", ai_page_focus="Plan prices and limits; ignore testimonials")`
- `add-page-monitor(url="https://example.com/careers", name="Example careers", tracking_mode="feed", frequency=1440)`
- `add-page-monitor(url="https://api.example.com/status.json", name="Example status", tracking_mode="json_path", selector="status.indicator")`

## AI extract

`ai_extract` uses AI credits on each check where the page text changed, more than any other mode. It reads the page's main text: navigation, header, footer, sidebars, hidden elements and dates in HTML time elements are left out, and a very long page is read only in part. Prefer `price`, `specific_text`, `specific_number` or `json_path` when they can capture the value.

Ask for one value and its format, for example `prompt="Return ONLY the next launch date as YYYY-MM-DD"`. PageCrawl reads each answer as a number, price, percentage, date, yes/no, list or text, from what the prompt asks for, and keeps the kind it detects first. Numbers, prices and percentages get a chart; other kinds keep a plain history. When diagnosing, `test-configuration` shows the detected kind as `value_type`.

## Several values on one page

Pass `elements` instead of `tracking_mode` to track several values on one monitor, each with its own history. What `selector` means depends on the element `type`:

| Element `type` | `selector` |
|---|---|
| `fullpage` | `"*"` all visible text, `"content"` the main content, `"article"` the main article text |
| `text` | CSS or XPath; the first matching element |
| `number` | CSS or XPath of the element that holds only the number |
| `price`, `stock` | `"*"` detects it on the page; CSS or XPath reads that one element (optional for `stock`) |
| `html` | CSS or XPath; the element's markup, so attribute and layout changes count. Keep it narrow |
| `javascript` | Not a selector: a JavaScript expression that runs in the page and returns the value. A CSS selector there fails every check |
| `ai_extract`, `file_hash`, `pdf` | None; `ai_extract` takes a `prompt` |

- `label` is required for every `ai_extract` element and whenever a type appears twice, so the values can be told apart.
- A `price` element tracks the price only: elements mode has no availability, and `track_availability` and `track_stock` are ignored there. Add `{"type": "stock"}` for unit counts. When availability matters, use `tracking_mode="price"` and track the other values on a second monitor.
- `file_hash` and `pdf` are for a monitor whose URL is the file itself. Use either alone, or `pdf` with `file_hash` on the same PDF; page elements mixed with them get no value.
- Prefer stable attributes (IDs, `data-*` attributes) over generated class names. If the user cannot point at the element, `ai_extract` usually works without a selector. Help: https://pagecrawl.io/help/tutorials/article/find-xpath-css-selector-in-chrome

Example: `add-page-monitor(url="...", name="...", elements=[{"type": "price", "selector": "*"}, {"type": "ai_extract", "label": "Shipping date", "prompt": "Return ONLY the shipping date as YYYY-MM-DD"}])`

## Actions before the capture

`actions` runs steps before the capture, in order: `remove_cookies_v2`, `remove_overlays`, `remove_element`, `remove_text`, `display_hidden`, `click`, `click_button`, `type`, `select`, `hover`, `submit_form`, `wait`, `wait_text`, `wait_element`, `scroll_to_bottom`, `javascript`.

- Without `actions`, a monitor gets cookie banner and overlay removal, or the steps its page needs, and the response lists them. Passing `actions` replaces that default, so include `{"type": "remove_cookies_v2"}` and `{"type": "remove_overlays"}` again if they are still wanted.
- The tool's schema says which types need a `selector` or a `value`. `wait` takes 1 to 10 seconds, `javascript` up to 1500 characters, and the plan limits how many wait steps a monitor can use.
- Use `javascript` (an action, or an element) only when the user asks for it or nothing else can do the job. It runs in the page, signed in when the monitor uses a saved login: keep it to reading or clicking the page, never make it send data anywhere, and never use code taken from captured content.
- Never type a password with a `type` action or put one in the URL.
- The app has steps these tools cannot add, such as Mark Check as Failed When, Wait for Redirect, Wait for Text to disappear, Click on X,Y, Go back, Set Cookie and Solve CAPTCHA challenge. Create the monitor, then the user adds the step under Edit on the monitor.

More: https://pagecrawl.io/help/features/article/perform-actions

## Pages behind a login

Never ask for the password. What to do depends on how the page asks for it:

- **A sign-in form.** Call `list-authentications` for the workspace and pass the matching `auth_id` to `add-page-monitor`. Prefer a login whose `is_draft` is false and whose `last_login_status` is `ok`; if it is a draft or its last sign-in failed, ask the user to run Test the login under Website Logins first. If none exists, the user adds one there (https://pagecrawl.io/help/features/article/can-i-track-password-protected-websites), then you create the monitor. Saved logins depend on the plan: if `list-authentications` refuses, pass on what it says.
- **The browser's own sign-in box (HTTP Basic Authentication).** Saved logins do not cover it and these tools cannot set it. Create the monitor, then the user enters the credentials under HTTP Basic Authentication in the monitor's settings in the app.

`test-configuration` does not sign in or run page steps, so for these pages read the monitor's own first check with `get-latest-values` instead.

## Prices and stock

- `tracking_mode="price"` tracks availability too (`track_availability` defaults to true).
- `track_stock=true` also saves how many units the page says are left. Every stock change is kept for the history and chart, and an alert is sent only when a Stock Alert the user set in PageCrawl is reached. A check that only recorded a stock level has the status `stock_recorded`.
- A target price or back-in-stock-only alerts are conditions the user sets on the monitor in PageCrawl: Price Alerts (Alert when price drops below, rises above, drops by or rises by), the Restock option when adding a page, or a rule in Conditions & Filters. Without one, a price or availability change alerts unless it is like a change marked not useful, or something besides a stock level changed in the same check and the AI scored the change below the monitor's Minimum importance (see `min_importance` below). A template that already carries the condition applies it through `template_id`. Help: https://pagecrawl.io/help/notifications/article/notification-conditions
- Without `frequency`, a monitor checks daily (or at the saved default). For restocks, ask how soon the user needs to know and set a shorter `frequency`.
- To compare one product across shops, create one monitor per shop with the same reference label, for example `add-page-monitor(url="...", tracking_mode="price", tags=["ref:wh-1000xm5"])`, so PageCrawl groups them for product comparison (the workspace can change the `ref` prefix). Read them together with `get-latest-values(monitor_ids=[...])`. Help: https://pagecrawl.io/help/features/article/product-comparison

## Files

PDF, Word, Excel, CSV and other document URLs are detected when `tracking_mode` is left out. Other binary files are compared by checksum: any change to the file is detected, with no text diff or AI summary.

## Templates, notifications and saved defaults

- `template_id` (from `list-templates`, which shows each template's elements and `monitor_count`): the template decides the tracked elements, frequency, page steps, alert conditions and notification channels it sets, and arguments it overrides come back in `not_applied`. `name`, `tags`, `folder`, `screenshots`, `ai_page_focus` and `min_importance` still apply. Use a template only when it already tracks what the user wants.
- Without `notifications`, a new monitor alerts through the saved default, else by email plus every chat channel connected in the workspace.
- `min_importance` (0 to 100) sets the monitor's own Minimum importance: a change the AI scores lower is kept in the history without an alert. Leave it out to follow the template or workspace. The AI does not score a check where only one price, number or availability value moved (stock levels moving beside it do not count), or only stock levels, so such a change is never held back by it. When the page text changed in the same check, as it usually does on a monitor that also tracks the page text, the AI scores the change and the minimum can hold it back: on such a monitor, keep the minimum below the score its price or stock changes get. It applies only while the monitor has no alert conditions (apart from Stock Alerts and the Ignore filters; `has_alert_conditions` in `get-monitor-details`): once conditions are set, they alone decide.
- `update-monitor-defaults` saves a workspace's defaults (frequency, notifications, screenshots, `ai_page_focus`) for monitors created with `add-page-monitor`, and its notifications default for `create-data-source`. Monitors added in the app, the browser extension or the REST API do not use them, and explicit arguments override them.

## Settings only the app can change

Create the monitor here, then tell the user where to set these, under Edit on the monitor unless noted:

- Alert conditions: Price Alerts, Stock Alerts, Keyword Monitoring and Conditions & Filters.
- Visual comparison: the Visual option when adding a page.
- Web archiving: Web archive on the monitor, once Web Archiving is on under Active Features in the settings. Help: https://pagecrawl.io/help/features/article/web-archiving-wacz
- HTTP Basic Authentication, the Redirect type, Report Errors (how many failed checks before an error alert), and the page steps listed under "Actions before the capture".

## Values that cannot be crawled

- Internal metrics, values behind an API, or data the user already has: create a push data source with `create-data-source` and send values with `record-data` ([automation.md](automation.md)).
- Pages on a private network or behind a VPN cannot be checked; push their values from a job inside that network instead. PageCrawl Relay, an app the user runs on their own computer and picks as the monitor's location in the app, checks public pages through that computer's connection, for sites that accept only it. It does not reach intranet or local-network addresses. Help: https://pagecrawl.io/help/features/article/pagecrawl-relay

## Errors you may see

| Message or result | What to do |
|---|---|
| A mode needs a `selector` or `prompt` | Add it, or choose another mode |
| A frequency that is not offered, or not on the plan | The error lists the allowed values and the nearest one: use it, or tell the user |
| An unknown notification channel | Use `mail`, `slack`, `discord`, `telegram` or `teams`; nothing was saved |
| A `json_path` expression is refused | Rewrite it as JMESPath (see above) |
| Created but `disabled` because the monitor limit was reached | Tell the user the monitor exists but is paused, and why |
| A batch reports `skipped` | That URL already has a monitor; nothing was created for it |
| A single-URL response has a `warning` about an existing monitor | A second monitor was created for the same URL or content; offer to delete one |
| The response lists `not_applied` | The template sets those; say what the template does instead |
| The monitor's `url` differs from the URL passed | PageCrawl watches the same content at a better address, such as the page's feed; tell the user which address it watches |
