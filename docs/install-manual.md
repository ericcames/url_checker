# Manual install — load url_checker into AAP from your laptop

This path is for customers (or operators) who want to install `url_checker`
into their Ansible Automation Platform from a laptop or desktop, using the
configuration-as-code under [`aap_config/`](../aap_config/).

After install, a **URL Egress Check** Job Template appears in the AAP UI,
runnable on demand to verify the firewall rules in front of AAP permit its
required egress.

If you'd rather have an AI agent walk you through this, see
[install-with-ai.md](install-with-ai.md) *(coming soon)*.

## Prerequisites

- `ansible-core` (any 2.15+). The probe playbook itself uses only
  `ansible.builtin`; the loader uses one extra collection (next step).
- A working `~/.ansible.cfg` (or `~/.ansible/ansible.cfg`) with a Red Hat
  Automation Hub token in the `[galaxy_server.rh_certified]` section. This is
  the standard developer setup at Red Hat — if you can already
  `ansible-galaxy collection install` certified content, you're set.
- An AAP instance you can reach over the network, and permission to create
  Projects and Job Templates in it.

## 1. Install the loader collection

From the repo root:

```bash
ansible-galaxy collection install -r aap_config/requirements.yml
```

This installs `infra.aap_configuration` from your configured Galaxy source.

## 2. Create a personal AAP token

In the AAP UI:

1. Click your username (top right) → **My User**.
2. Open the **Tokens** tab → **Create token**.
3. **Scope:** `Write`. Description: anything you'll recognize later.
4. Copy the token value — it is shown **once**.

## 3. Export credentials

```bash
export AAP_HOSTNAME=https://your-aap.example.com
export AAP_TOKEN=<the token you just copied>
```

If your AAP uses a self-signed certificate:

```bash
export AAP_VALIDATE_CERTS=false
```

## 4. Run the loader

```bash
ansible-playbook -i aap_config/inventory/aap.yml aap_config/load.yml
```

Expected: a clean play recap with no failures. The loader is idempotent — safe
to re-run.

## 5. Verify in the AAP UI

- **Resources → Projects** — a project named `url_checker` should be present
  and finish syncing (green dot).
- **Resources → Templates** — a job template named `URL Egress Check` should
  be present, pointing at `playbooks/main.yml` on the `url_checker` project.

## 6. Launch the egress check

In the AAP UI, launch **URL Egress Check**. On success the job output ends
with a line like:

```
N of N URL probes passed.
```

On failure, the job fails non-zero and prints each missed URL — that's the
list of firewall rules that need attention.

## Optional environment variables

| Variable | Default | Purpose |
|----------|---------|---------|
| `AAP_VALIDATE_CERTS` | `true` | Set `false` for self-signed AAP. |
| `URL_CHECKER_ORG` | `Default` | AAP organization to create the Project/JT under. |
| `URL_CHECKER_EE` | `Default execution environment` | Execution environment for the JT. |
| `URL_CHECKER_INVENTORY` | `Demo Inventory` | Inventory the JT runs against (the probes use `localhost` regardless, so any inventory will do). |
| `URL_CHECKER_SCM_URL` | `https://github.com/ericcames/url_checker.git` | Override if you've forked the repo. |
| `URL_CHECKER_SCM_BRANCH` | `main` | Override to test from a branch. |

## Troubleshooting

- **`AAP_HOSTNAME and AAP_TOKEN must be exported`** — the loader's pre-task
  caught a missing env var. Re-export and re-run.
- **`Could not find role 'infra.aap_configuration.dispatch'`** — step 1 didn't
  install successfully. Check your `~/.ansible.cfg` Hub token, then re-run.
- **Project sync fails inside AAP** — the AAP host can't reach
  `URL_CHECKER_SCM_URL`. Adjust the URL or open egress for it.
