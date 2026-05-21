# AI-assisted install — load url_checker into AAP via Claude Code

This path is for customers (or operators) who prefer to have an AI agent walk
them through installing `url_checker` into their Ansible Automation Platform,
rather than running the commands by hand.

If you'd rather drive the install yourself from a shell, see
[install-manual.md](install-manual.md). The two paths produce the **same
result** — a Project and a "URL Egress Check" Job Template in your AAP.

## Prerequisites

- [Claude Code](https://docs.claude.com/en/docs/claude-code) installed and signed in.
- `ansible-core` (any 2.15+) on the same machine.
- A working `~/.ansible.cfg` (or `~/.ansible/ansible.cfg`) with a Red Hat
  Automation Hub token in `[galaxy_server.rh_certified]`. Required to install
  the loader collection.
- An AAP instance you can reach and permission to create Projects and Job
  Templates in it.

## How it works

1. **Clone this repo** and `cd` into it.
   ```bash
   git clone https://github.com/ericcames/url_checker.git
   cd url_checker
   ```

2. **Open Claude Code** in the repo directory. The `url-checker-install`
   skill at [`.claude/skills/url-checker-install/`](../.claude/skills/url-checker-install/SKILL.md)
   auto-loads.

3. **Tell the agent what you want.** For example:
   ```
   install url_checker into my AAP
   ```
   The agent will recognize the request and run the skill.

4. **The agent will ask for two things:**
   - Your AAP base URL (e.g. `https://aap-foo.example.com`).
   - A personal AAP token. If you don't have one yet, the agent will give
     you the exact UI steps to create one.

5. **The agent will run** `aap_config/load.yml` against your AAP and verify
   the resulting Project and Job Template.

6. **Optionally**, the agent will offer to launch the Job Template once and
   summarize the result — including which URLs (if any) your AAP cannot
   reach, so you have a punch list for your firewall team.

## Security notes

- The token you paste lives only in the agent's shell session as an
  environment variable. It is **never** written to a file, committed to git,
  or saved to Claude's persistent memory.
- The skill never creates or deletes tokens on the AAP. If you want to
  rotate the token later, do it in the AAP UI — the loader has no
  persistent connection to the token you used.
- If your AAP uses a self-signed certificate, the agent will offer to set
  `AAP_VALIDATE_CERTS=false` for the install run. This affects only that
  one shell session.

## Troubleshooting

- **"`aap_config/` not found"** — the agent is not running from the repo
  root. `cd` into the cloned `url_checker` directory and re-invoke.
- **`ansible-galaxy` cannot install `infra.aap_configuration`** — your
  `~/.ansible/ansible.cfg` is missing a Hub token. See
  [install-manual.md prerequisites](install-manual.md#prerequisites).
- **Project sync `failed` after install** — your AAP cannot reach
  `https://github.com/ericcames/url_checker.git`. That's itself a useful
  egress signal — fix the firewall rule and re-trigger sync in the AAP UI.
