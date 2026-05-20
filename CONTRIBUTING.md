# Contributing

## Workflow

**Every change follows this sequence — no exceptions:**

```
Open issue → branch from main → implement → open PR (Closes #N) → merge → issue closes
```

1. **Open an issue first** — before writing a single line of code or making any change, open a GitHub issue describing what you're fixing or adding and why. No implementation without an issue. One issue per change.
2. **Branch from `main`** — use the naming pattern `<type>/<short-description>` (e.g. `fix/azureedge-status`, `feat/scheduled-check`).
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

If you changed playbook behavior (not just the URL list), also verify the failure path by temporarily setting an entry's expected `status` to a value the host won't return, and confirm the play exits non-zero with a readable failure line. Document how you tested in the PR description.

## CHANGELOG

Every PR adds at least one bullet under `## [Unreleased]` in `CHANGELOG.md`, grouped under `Added`, `Changed`, `Removed`, or `Fixed`.

## What never gets committed

- Credentials, tokens, or passwords of any kind (this repo doesn't need any — keep it that way)
- `*.retry` files
