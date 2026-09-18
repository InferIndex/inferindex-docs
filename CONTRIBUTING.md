# Contributing

This repository holds the public documentation, examples and (eventually) a client SDK for the InferIndex
API. The backend itself (data collection, parsers, storage) lives in a separate private repository and is not
open source — contributions here are limited to what ships in this repo.

## What's welcome

- Fixes to `README.md` and `docs/api.md`: wrong parameter, outdated example, unclear wording.
- New usage examples (curl, or a client language) that call the public API.
- Reports of a documented route or response shape that no longer matches the live API at
  `https://api.inferindex.dev`.

## What isn't in scope here

- Requests to add a pricing source, change how prices are collected or parsed, or fix a specific price —
  those live in the backend, not this repository. Open an issue describing the problem; it'll be forwarded if
  it needs backend work.
- Anything about the promotion-detection or model-matching logic.

## Making a change

1. Fork the repository and create a branch.
2. Keep examples real: if you change a curl example or a JSON response sample, run it against the live API
   and paste the actual output rather than a hand-written guess.
3. Open a pull request describing what changed and why.

## Reporting a bug in the docs vs. a security issue

A wrong or outdated line in the docs: open an issue or a PR as above.

A security vulnerability (in the API itself, or something that shouldn't be publicly reachable): see
[SECURITY.md](SECURITY.md) instead — don't open a public issue for that.
