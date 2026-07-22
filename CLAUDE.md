# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

This is a `.well-known` static-file repository, served as-is (no build step) at the `/.well-known/` path of a domain — most likely via GitHub Pages, given the presence of `.nojekyll`. It has no application code, no package manager, no build/lint/test tooling. Every file here is a file that gets served verbatim over HTTPS to whatever client requests it.

## Structure

- `index.html` — a minimal placeholder page (single `<h1>`), served at `/.well-known/index.html`.
- `.nojekyll` — empty marker file that tells GitHub Pages to skip Jekyll processing, so files starting with `_` (and dotfiles like this directory's own name) are served unprocessed.
- `appspecific/com.tesla.3p.public-key.pem` — the public key Tesla's Fleet API uses to verify this domain's third-party application, fetched by Tesla at `https://<domain>/.well-known/appspecific/com.tesla.3p.public-key.pem` during app registration/verification.

## Working conventions

- There is no build, lint, or test process — changes are just committed files. Verify changes by reading the file content directly, not by running commands.
- Every file in this repo is served publicly by design (that's the entire purpose of a `.well-known` directory). Do not add anything here that isn't meant to be public — most critically, **never place a private key in `appspecific/com.tesla.3p.public-key.pem`**; only the PUBLIC half of the Tesla Fleet API key pair belongs in this file. Before editing this file, confirm the PEM header reads `-----BEGIN PUBLIC KEY-----` (or `-----BEGIN EC PUBLIC KEY-----`), not `-----BEGIN EC PRIVATE KEY-----` — this repo's history has contained a private key at this path (commits `d5d85e6`, `e8d1b78`), which is a live secret-exposure risk if the site is publicly reachable.
- When adding new `.well-known` well-known-URI files (e.g. `security.txt`, `apple-app-site-association`, other `appspecific/` entries), follow the same pattern: drop the file at the exact path the consuming service expects, with no wrapper directories or build step.
