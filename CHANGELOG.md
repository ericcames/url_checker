# Changelog

All notable changes to this project will be documented in this file.

The format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## [Unreleased]

### Added
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

### Changed
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
