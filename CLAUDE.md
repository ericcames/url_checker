# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Repo Does

Ansible playbook that probes a list of upstream URLs and reports which (if any) are unreachable. Useful for verifying egress before installing software that pulls from those endpoints — Red Hat container registries, Azure CDNs, Google APIs, Let's Encrypt, GitHub, and so on.

The intended customer scenario is Red Hat Managed AAP on Azure: the playbook is loaded *into* the customer's AAP via the configuration-as-code in `aap_config/` and run as a Job Template to verify the Azure firewall rules permit AAP's required egress.

| Path | Purpose |
|------|---------|
| `playbooks/main.yml` | The probe playbook. Iterates each URL on its declared scheme(s), prints a summary and a deduplicated firewall-rule punch list, fails non-zero on any miss. Uses `ansible.builtin` only. |
| `playbooks/files/websites.yml` | The URL list — `url`, `method`, expected `status`, and `schemes` per entry. KB-derived rationale in the file header. |
| `aap_config/` | Config-as-code to load the probe into a customer's AAP as a Project + Job Template. Uses `infra.aap_configuration`. Pinned in `aap_config/requirements.yml`. |
| `.claude/skills/url-checker-install/` | Claude Code skill that walks an operator through the AAP install interactively. Token is held in shell env only — never persisted. |
| `docs/install-manual.md` | Customer-facing manual install path. |
| `docs/install-with-ai.md` | Customer-facing AI-assisted install path. |

## Running

```bash
ansible-playbook playbooks/main.yml
```

Runs against `localhost` with `connection: local` — no inventory needed. The probe playbook uses `ansible.builtin` only and runs from a fresh box with `ansible-core`. The AAP loader in `aap_config/load.yml` is the one path that requires an extra collection — see `aap_config/requirements.yml`.

## Adding or Updating a URL

Edit `playbooks/files/websites.yml`. Each entry has four fields:

| Field     | Description                                                       |
|-----------|-------------------------------------------------------------------|
| `url`     | Hostname (or host + path) to probe                                |
| `method`  | HTTP method: `HEAD`, `GET`, `OPTIONS`, etc.                       |
| `status`  | Expected HTTP status code (integer)                               |
| `schemes` | List of schemes to probe — default `[http, https]` per KB 6972355 |

Pick the lightest method an endpoint accepts (`HEAD` whenever possible) and set `status` to whatever that endpoint actually returns on a probe of its root — some hosts answer `400` or `404` on `/` and that is still a valid reachability signal. Document any non-obvious expected status with an inline comment so future editors don't "fix" it.

The play uses `follow_redirects: all`, so an http probe that 301s to https resolves to the final https status — the same `status` value works for both schemes on every entry in the current list. Drop to `[https]` only if the host genuinely does not answer on port 80.

## Key Conventions

- **One concern per PR** — URL list edits, playbook behavior changes, and docs changes each go in their own issue/PR unless they share a root cause.
- **Document before fixing** — open a GitHub issue describing the change before writing code.
- **CHANGELOG.md is mandatory** — every PR adds an entry under `## [Unreleased]` using Keep-a-Changelog sections (`Added`, `Changed`, `Removed`, `Fixed`).
- **Failure path stays loud** — the playbook must exit non-zero on any URL miss. Do not reintroduce `ignore_errors: yes` at the play level.
- **`ansible.builtin` only in `playbooks/`** — the probe playbook stays zero-dependency so it runs from a fresh box without Galaxy/Hub setup. The loader in `aap_config/` is the one place an extra collection (`infra.aap_configuration`) is pulled in.

## CI

GitHub Actions runs on every PR:

- `yamllint` against `playbooks/`, `aap_config/`, the repo root, and `.github/`
- `ansible-lint` against `playbooks/` and `aap_config/`

A scheduled workflow (`url-check.yml`) runs the playbook itself on a weekly cron and opens a GitHub issue when any probe fails — the repo is its own monitor.
