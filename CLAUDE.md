# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Repo Does

Ansible playbook that probes a list of upstream URLs and reports which (if any) are unreachable. Useful for verifying egress before installing software that pulls from those endpoints — Red Hat container registries, Azure CDNs, Google APIs, Let's Encrypt, GitHub, and so on.

Single playbook, single data file:

| File | Purpose |
|------|---------|
| `playbooks/main.yml` | Probes each URL on its declared scheme(s), prints a summary, fails non-zero on any miss |
| `playbooks/files/websites.yml` | The URL list — `url`, `method`, expected `status`, and `schemes` per entry |

## Running

```bash
ansible-playbook playbooks/main.yml
```

Runs against `localhost` with `connection: local` — no inventory needed. Only the `ansible.builtin` collection is used, which ships with `ansible-core`, so there is nothing to install from Galaxy.

## Adding or Updating a URL

Edit `playbooks/files/websites.yml`. Each entry has four fields:

| Field     | Description                                                       |
|-----------|-------------------------------------------------------------------|
| `url`     | Hostname (or host + path) to probe                                |
| `method`  | HTTP method: `HEAD`, `GET`, `OPTIONS`, etc.                       |
| `status`  | Expected HTTP status code (integer)                               |
| `schemes` | List of schemes to probe — e.g. `[https]` or `[http, https]`      |

Pick the lightest method an endpoint accepts (`HEAD` whenever possible) and set `status` to whatever that endpoint actually returns on a probe of its root — some hosts answer `400` or `404` on `/` and that is still a valid reachability signal. Document any non-obvious expected status with an inline comment so future editors don't "fix" it.

## Key Conventions

- **One concern per PR** — URL list edits, playbook behavior changes, and docs changes each go in their own issue/PR unless they share a root cause.
- **Document before fixing** — open a GitHub issue describing the change before writing code.
- **CHANGELOG.md is mandatory** — every PR adds an entry under `## [Unreleased]` using Keep-a-Changelog sections (`Added`, `Changed`, `Removed`, `Fixed`).
- **Failure path stays loud** — the playbook must exit non-zero on any URL miss. Do not reintroduce `ignore_errors: yes` at the play level.
- **`ansible.builtin` only** — keep this repo zero-dependency so it can run from a fresh box without Galaxy/Hub setup.

## CI

GitHub Actions runs on every PR:

- `yamllint` against `playbooks/` and the repo root
- `ansible-lint` against `playbooks/`

A scheduled workflow (`url-check.yml`) runs the playbook itself on a weekly cron and opens a GitHub issue when any probe fails — the repo is its own monitor.
