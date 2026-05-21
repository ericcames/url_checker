# Manual install — load url_checker into AAP from your laptop

This path is for customers (or operators) who want to install `url_checker`
into their Ansible Automation Platform from a laptop or desktop, using the
configuration-as-code under [`aap_config/`](../aap_config/).

After install, a **URL Egress Check** Job Template appears in the AAP UI,
runnable on demand to verify the firewall rules in front of AAP permit its
required egress.

If you'd rather have an AI agent walk you through this, see
[install-with-ai.md](install-with-ai.md).

## Prerequisites

- A shell with `ansible-playbook` (any `ansible-core` 2.15+). See
  [the next section](#get-a-shell-with-ansible-playbook) for the easiest
  way to get one on your platform — including Windows.
- An AAP instance you can reach over the network, and permission to create
  Projects and Job Templates in it.
- *(Optional — Red Hat employees only)* A working `~/.ansible/ansible.cfg`
  with an Automation Hub token in `[galaxy_server.rh_certified]`. The
  `infra.aap_configuration` collection is published on public Galaxy, so a
  Hub token is **not** required to install it — but if you already route
  Galaxy through Hub, your existing config will be honored.

## Get a shell with `ansible-playbook`

Pick whichever fits your workstation. The rest of this guide is identical
once you have a shell with `ansible-playbook` in your PATH.

### Easiest path: Azure Cloud Shell *(recommended, especially on Windows)*

Azure provides a free browser-based shell with `ansible-core` already
installed. It runs inside Azure adjacent to your AAP, so network reach is
typically a non-issue, and there is nothing to install on your workstation.

1. Open <https://shell.azure.com> (or click the **>_** icon in the Azure
   portal toolbar) and sign in with the same account you use for AAP.
2. Choose **Bash** (not PowerShell) when prompted.
3. Confirm Ansible is present:
   ```bash
   ansible --version
   ```
4. Clone this repo into your Cloud Shell home directory:
   ```bash
   git clone https://github.com/ericcames/url_checker.git
   cd url_checker
   ```
5. Continue with [§1 below](#1-install-the-loader-collection).

> Cloud Shell's home directory persists across sessions when you accept
> the offer to mount cloud storage on first use. If you skipped that, the
> install commands still work — you just re-clone next time.

### Linux

| Distro | Command |
|--------|---------|
| Fedora / RHEL 9+ / CentOS Stream | `sudo dnf install ansible-core` |
| Debian / Ubuntu 22.04+ | `sudo apt update && sudo apt install ansible-core` |
| Anything with Python 3.10+ | `python3 -m pip install --user ansible-core` |

Verify:
```bash
ansible --version
```

### macOS

```bash
brew install ansible      # Homebrew (recommended)
# or
python3 -m pip install --user ansible-core
```

### Windows (via WSL2)

`ansible-core` does not run natively on Windows. Either use Azure Cloud
Shell above (recommended) or install Windows Subsystem for Linux:

1. In PowerShell (Administrator):
   ```powershell
   wsl --install
   ```
   This installs Ubuntu by default. Reboot when prompted, then complete
   the Ubuntu setup wizard.
2. From the new Ubuntu shell:
   ```bash
   sudo apt update
   sudo apt install ansible-core git
   ansible --version
   ```
3. Clone the repo inside WSL (not in the Windows filesystem):
   ```bash
   git clone https://github.com/ericcames/url_checker.git
   cd url_checker
   ```
4. Continue with [§1 below](#1-install-the-loader-collection).

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
  install successfully. Re-run `ansible-galaxy collection install -r
  aap_config/requirements.yml` and watch for errors. If you have a
  `~/.ansible/ansible.cfg` that routes Galaxy through Automation Hub, make
  sure the Hub token is current (or temporarily move that file aside to
  fall back to public Galaxy).
- **Project sync fails inside AAP** — the AAP host can't reach
  `URL_CHECKER_SCM_URL`. Adjust the URL or open egress for it.
