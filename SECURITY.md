# Security

## Reporting a vulnerability

Email **sec@pagecrawl.io**. If you would rather not use email, any support channel reaches us and we will move it somewhere private.

Please include what you did, what happened, and roughly how bad you think it is. A rough report beats no report.

- We reply within **2 working days**.
- We will tell you when it is fixed, and credit you in the release notes unless you would rather we did not.
- We will not take legal action against anyone who reports a problem in good faith, stops at the point of proving it, and does not access or damage other people's data while doing so.

## What this repository contains

Markdown skills (with code examples an agent adapts when you ask it to write an integration), JSON and YAML manifests, and the PageCrawl logo. Nothing here runs on your machine: there are no hooks, scripts or executables, and the one workflow, in `.github/workflows`, only runs this repository's own checks on GitHub. Reports we most want to hear about:

1. Anything in the skills that could lead an agent to leak credentials, run a destructive action without the user's confirmation, or follow instructions embedded in monitored page content.
2. Problems in the PageCrawl MCP server or API that the skills reach (`mcp.pagecrawl.io`, `pagecrawl.io/api`). Test against your own account only.
