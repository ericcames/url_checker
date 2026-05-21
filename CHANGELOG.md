# Changelog

All notable changes to this project will be documented in this file.

The format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## [Unreleased]

### Added
- Every entry in `playbooks/files/websites.yml` now probes both `http` and
  `https`, doubling probe coverage from 16 to 32. Per Red Hat KB 6972355,
  the Azure firewall must allow http (tcp/80) and https (tcp/443) to every
  listed host — port 80 is mostly used for redirect-to-443 or by dependent
  services that may need it. The KB's only HTTPS-only exception
  (`*.blob.core.windows.net`) is not in this list.
- `aap_config/` configuration-as-code so customers can load `url_checker` into
  their AAP as a Project + Job Template. Uses `infra.aap_configuration` 4.5.0;
  authenticates via `AAP_HOSTNAME` + `AAP_TOKEN` env vars (BYO token model — no
  programmatic token creation, no leak path).
- `docs/install-manual.md` — laptop/desktop install path for the AAP CaC.
- `.claude/skills/url-checker-install/SKILL.md` — Claude Code skill that
  walks the operator through the AAP CaC install interactively. Token is
  held in shell env only, never written to disk.
- `docs/install-with-ai.md` — customer-facing doc for the AI-assisted path.
- README "Load into your AAP" section linking to the new docs.
- `community.letsencrypt.org`, `access.redhat.com`, and `acs-mirror.azureedge.net`
  to the URL list. (`acs-mirror.azureedge.net` expects status `400` on a root
  `HEAD` — the host answering at all is the reachability signal we need.)
- Per-entry `schemes` field on each website so each URL is probed only on the
  protocol it actually serves.
- Failure summary: the play now prints each failed probe (URL, method, expected
  vs. actual status) and exits non-zero when any probe fails.
- `README.md` documenting usage and the `websites.yml` entry schema.
- `CHANGELOG.md` (this file).
- Repo scaffolding ported from `aap.selfservice` template:
  `CLAUDE.md`, `CODE_OF_CONDUCT.md`, `CONTRIBUTING.md`, `.gitignore`,
  `.yamllint`, `.ansible-lint`.
- `.github/` folder: `ISSUE_TEMPLATE/{bug_report,feature_request}.md`,
  `pull_request_template.md`, `SECURITY.md`.
- CI workflows: `yamllint.yml` and `ansible-lint.yml` on every PR.
- Scheduled `url-check.yml` workflow that runs the playbook weekly on a cron
  (and on manual dispatch); opens (or updates) a tracking issue labeled
  `scheduled-check-failed` on any probe failure — the repo is its own monitor.
- `README.md` References section citing Red Hat KB article 6972355 as the
  source of the Azure / Google / Microsoft egress URLs in `websites.yml`.

### Changed
- Documentation caught up to code: README repo-layout tree + updated
  `schemes` example + Output section reflecting the firewall-rule punch
  list; `docs/install-manual.md` "(coming soon)" reference dropped now
  that `install-with-ai.md` ships; `CLAUDE.md` updated file table, CI
  lint scope, and `ansible.builtin`-only scoping (probe playbook only,
  not the loader); `CONTRIBUTING.md` branch-naming pattern aligned with
  practice and failure-path test pattern documented via `-e @<fixture>`.
- CI `yamllint` and `ansible-lint` workflows now also lint `aap_config/`.
- `playbooks/files/websites.yml` schema comment now says `schemes` is
  required, instead of the aspirational "defaults to [https] if omitted"
  (no default was ever implemented — `subelements('schemes')` hard-fails
  on entries missing the field).
- Failure output is now a firewall-rule punch list. Each failed probe
  prints the corresponding `allow tcp/<port> to <host>` rule beneath it,
  and the end of the play prints a deduplicated list of every rule the
  customer needs to add. Failure exit code is unchanged (non-zero on
  any miss).
- Replaced the dual port-80 / port-443 loop in `playbooks/main.yml` with a
  single per-scheme loop driven by each entry's `schemes` field.
- `playbooks/files/websites.yml` reorganized around the new schema.

### Removed
- Bare `azureedge.net` apex entry (CDN parent domain that does not resolve to a
  meaningful endpoint); replaced by the specific `acs-mirror.azureedge.net`
  subdomain.
- Bare `microsoftonline.com` apex entry; `login.microsoftonline.com` is the
  real probe target and is retained.

[Unreleased]: https://github.com/ericcames/url_checker/compare/HEAD...HEAD
