# Contributing

## Workflow

**Every change follows this sequence — no exceptions:**

```
Open issue → branch from main → implement → open PR (Closes #N) → merge → issue closes
```

1. **Open an issue first** — before writing a single line of code or making any change, open a GitHub issue describing what you're fixing or adding and why. No implementation without an issue. One issue per change.
2. **Branch from `main`** — use the naming pattern `<issue#>-<short-description>` (e.g. `12-firewall-rule-output`, `10-add-http-schemes`). Issue-prefixed branches make traceability one-click from any commit.
3. **One concern per PR** — implement, commit, and open a PR for each issue separately. Group changes by shared root cause, not by item count.
4. **Reference the issue** — include `Closes #<number>` in your PR description so the issue closes automatically on merge.
5. **PRs target `main`** — direct pushes to `main` are not allowed.
6. **Delete your branch after merge** — GitHub is configured to delete branches automatically on merge. After your PR merges, prune your local copy: `git fetch --prune && git branch -d <branch>`.

## Commit messages

Follow the pattern used in this repo:

```
<type>: <short description>

<optional body explaining why, not what>
```

Types: `feat`, `fix`, `docs`, `refactor`, `chore`

## Code conventions

See [CLAUDE.md](CLAUDE.md) for:
- URL entry schema (`url` / `method` / `status` / `schemes`)
- Why the playbook must keep failing loudly on any miss
- `ansible.builtin`-only rule

## Testing

Run the playbook locally before opening a PR:

```bash
ansible-playbook playbooks/main.yml
```

If you changed playbook behavior (not just the URL list), also verify the failure path. The cleanest pattern is to override the `websites` variable via `-e @<fixture>`, which beats `vars_files` in precedence — no file edits required:

```bash
cat > /tmp/bad-websites.yml <<'EOF'
websites:
  - url: nonexistent-host-abc-12345.invalid
    method: HEAD
    status: 200
    schemes: [http, https]
EOF

ansible-playbook playbooks/main.yml -e @/tmp/bad-websites.yml
# Confirm exit code is non-zero and the firewall-rule punch list renders correctly
```

If you touched `aap_config/`, additionally verify the loader against a live AAP — installing the collection, running `aap_config/load.yml` with `AAP_HOSTNAME`/`AAP_TOKEN` set, and confirming the Project + Job Template land. The `url-checker-install` skill at `.claude/skills/` walks through this end-to-end.

Document how you tested in the PR description.

## CHANGELOG

Every PR adds at least one bullet under `## [Unreleased]` in `CHANGELOG.md`, grouped under `Added`, `Changed`, `Removed`, or `Fixed`.

## What never gets committed

- Credentials, tokens, or passwords of any kind (this repo doesn't need any — keep it that way)
- `*.retry` files
