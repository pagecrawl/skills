# Security

## Reporting a vulnerability

Email **security@pagecrawl.io**. If you would rather not use email, any support channel reaches us and we will move it somewhere private.

Please include what you did, what happened, and roughly how bad you think it is. A rough report beats no report.

- We reply within **2 working days**.
- We will tell you when it is fixed, and credit you in the release notes unless you would rather we did not.
- We will not take legal action against anyone who reports a problem in good faith, stops at the point of proving it, and does not access or damage other people's data while doing so.

## What this repository contains

Instructions only: Markdown skills and JSON manifests, with no scripts, hooks or binaries. Reports we most want to hear about:

1. Anything in the skills that could lead an agent to leak credentials, run a destructive action without the user's confirmation, or follow instructions embedded in monitored page content.
2. Problems in the PageCrawl MCP server or API that the skills reach (`mcp.pagecrawl.io`, `pagecrawl.io/api`). Test against your own account only.
