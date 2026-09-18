# Security policy

## Reporting a vulnerability

Please do **not** open a public issue for a security vulnerability.

Report it privately by either:

- email to **security@inferindex.dev** (actively monitored), or
- **GitHub's private vulnerability reporting** for this repository: **Security** tab → **Report a
  vulnerability**, which opens a private advisory visible only to maintainers.

Include a description of the issue and any proof of concept. For non-security questions, use
contact@inferindex.dev.

## Scope

This repository contains documentation and examples only — no backend code. Still in scope for a report:

- A vulnerability in the live API at `https://api.inferindex.dev` (e.g. an injection, an auth bypass, data
  exposure through a documented or undocumented route).
- Sensitive information (secrets, internal identifiers, non-public data) that shouldn't be in this repository
  or its history.

Out of scope: vulnerabilities in third-party pricing sources InferIndex reads from, and general availability
issues (rate limiting, downtime) that aren't a security concern.

## What to expect

We aim to acknowledge a report within a few days and to keep you updated as it's investigated. Please give us
reasonable time to address an issue before any public disclosure.
