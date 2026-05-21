---
name: url-checker-install
description: "Install url_checker into an Ansible Automation Platform as a runnable Job Template. Walks the operator through installing the infra.aap_configuration collection, collecting the AAP host URL and a personal token (never persisted to disk), running aap_config/load.yml, verifying the resulting Project + Job Template, and optionally launching the JT to report which URLs (if any) the AAP's egress cannot reach. TRIGGER when: the user is in the url_checker repo and wants to load it into their AAP, deploy it, or asks how to install it. SKIP: if the user only wants to run the probe playbook locally from their laptop — direct them to playbooks/main.yml instead."
---

# url-checker-install

This skill loads `url_checker` into the user's Ansible Automation Platform as a Project + Job Template named **URL Egress Check**, using the configuration-as-code under `aap_config/`. After install, the user can launch the JT from the AAP UI on demand to verify their firewall rules permit AAP's required egress.

## Overview

This skill performs five steps:

1. Preflight — verify `ansible-core` is installed and the loader collection is reachable.
2. Collect AAP host URL and personal token (held in shell env only — never written to disk).
3. Run `ansible-playbook -i aap_config/inventory/aap.yml aap_config/load.yml`.
4. Verify the Project synced and the JT exists via the Controller API.
5. Optionally launch the JT once and summarize results.

The token model is **BYO**: the user creates a personal token in the AAP UI; this skill never creates or deletes tokens on their behalf.

## Step 1 — Preflight

Run from the repo root.

```bash
command -v ansible-playbook >/dev/null && echo "ansible-core: present" || echo "ansible-core: MISSING"
test -f aap_config/requirements.yml && echo "aap_config/: present" || echo "aap_config/: MISSING — wrong repo?"
```

If `ansible-core` is missing, tell the user to install it (`dnf install ansible-core` on Fedora/RHEL; `pip install ansible-core` elsewhere) and stop.

If `aap_config/` is missing, the working directory is not the `url_checker` repo root — ask the user to `cd` there and re-invoke the skill.

## Step 2 — Collect credentials

### Check existing shell env first

```bash
echo "AAP_HOSTNAME=${AAP_HOSTNAME:-<unset>}"
echo "AAP_TOKEN=${AAP_TOKEN:+<set>}${AAP_TOKEN:-<unset>}"
```

If both are already set, ask the user whether to reuse them. If yes, skip to Step 3.

### Otherwise, collect from the user

Ask the user for:
- **AAP base URL** — e.g. `https://aap-foo.example.com` (no trailing slash, no path).
- **AAP personal token** — created in the AAP UI.

If the user does not yet have a token, give them these steps and wait:

> In the AAP UI:
> 1. Click your username (top right) → **My User**.
> 2. Open the **Tokens** tab → **Create token**.
> 3. Scope: `Write`. Description: anything you'll recognize later.
> 4. Copy the token value — it is shown **once**.

If the AAP uses a self-signed certificate, also ask whether to set `AAP_VALIDATE_CERTS=false`.

**Never write the token to a file.** Pass it inline to each subprocess (see Step 3). Do not echo the token back to the user. Do not save it to memory.

## Step 3 — Install the loader collection and run

Install the collection (this is idempotent — safe if already installed):

```bash
ansible-galaxy collection install -r aap_config/requirements.yml
```

If install fails citing missing Hub authentication, the user's `~/.ansible/ansible.cfg` is missing a Hub token. Direct them to add `token=<value>` under `[galaxy_server.rh_certified]` (Red Hat Console → Automation Hub → Connect to Hub → API token).

Run the loader, passing credentials inline so they do not land in shell history:

```bash
AAP_HOSTNAME='<url>' \
AAP_TOKEN='<token>' \
AAP_VALIDATE_CERTS='<true|false>' \
ansible-playbook -i aap_config/inventory/aap.yml aap_config/load.yml
```

Watch the play recap. Expected: `failed=0` with `changed=2` on first run (Project + JT created), or `changed=0` on a re-run (idempotent).

If a task fails, stop and surface the error. Common causes:
- 401 — wrong token, or token lacks `Write` scope.
- Cannot resolve hostname — wrong `AAP_HOSTNAME`.
- TLS error — the AAP cert is self-signed; re-run with `AAP_VALIDATE_CERTS=false`.

## Step 4 — Verify

Query the Controller API to confirm both objects exist and the project sync succeeded.

```bash
echo "=== Project ==="
curl -sk -H "Authorization: Bearer $AAP_TOKEN" \
  "$AAP_HOSTNAME/api/controller/v2/projects/?name=url_checker" \
  | python3 -c "
import json, sys
d = json.load(sys.stdin)
for r in d.get('results', []):
    print(f\"  name={r['name']!r} status={r.get('status')} branch={r.get('scm_branch')}\")
if not d.get('results'):
    print('  MISSING')
"

echo "=== Job Template ==="
curl -sk -H "Authorization: Bearer $AAP_TOKEN" \
  "$AAP_HOSTNAME/api/controller/v2/job_templates/?name=URL+Egress+Check" \
  | python3 -c "
import json, sys
d = json.load(sys.stdin)
for r in d.get('results', []):
    print(f\"  name={r['name']!r} project={r['summary_fields']['project']['name']} playbook={r['playbook']}\")
if not d.get('results'):
    print('  MISSING')
"
```

Both should print one row. The project status should be `successful`. If it is `pending` or `running`, wait 10 seconds and re-query — the SCM sync may still be in flight.

If the project status is `failed`, the AAP itself cannot reach the SCM URL (this is itself a useful egress signal — note it for the user but proceed).

## Step 5 — Optional launch

Ask the user: *"Want to launch URL Egress Check now to see which URLs (if any) your AAP cannot reach?"*

If yes:

```bash
JT_ID=$(curl -sk -H "Authorization: Bearer $AAP_TOKEN" \
  "$AAP_HOSTNAME/api/controller/v2/job_templates/?name=URL+Egress+Check" \
  | python3 -c "import json,sys; print(json.load(sys.stdin)['results'][0]['id'])")

JOB_ID=$(curl -sk -H "Authorization: Bearer $AAP_TOKEN" -X POST \
  "$AAP_HOSTNAME/api/controller/v2/job_templates/$JT_ID/launch/" \
  | python3 -c "import json,sys; print(json.load(sys.stdin)['job'])")

echo "Launched job $JOB_ID. Polling..."
for i in $(seq 1 60); do
  STATUS=$(curl -sk -H "Authorization: Bearer $AAP_TOKEN" \
    "$AAP_HOSTNAME/api/controller/v2/jobs/$JOB_ID/" \
    | python3 -c "import json,sys; print(json.load(sys.stdin)['status'])")
  case "$STATUS" in
    successful|failed|error|canceled) echo "Final: $STATUS"; break ;;
  esac
  sleep 4
done

curl -sk -H "Authorization: Bearer $AAP_TOKEN" \
  "$AAP_HOSTNAME/api/controller/v2/jobs/$JOB_ID/stdout/?format=txt" | tail -40
```

Summarize for the user:
- On `successful`: report `N of N URL probes passed.` — the AAP has full egress.
- On `failed`: list each failed probe URL from the stdout — these are the firewall rules they need to add. The JT exit-code reflects this so the AAP UI shows it as failed (intentional — the playbook is loud on misses).

## Step 6 — Final summary

Tell the user where to find the JT:

```
Loaded into AAP at <AAP_HOSTNAME>:
  Project:      url_checker
  Job Template: URL Egress Check

Launch from the AAP UI anytime, or via:
  curl -sk -H "Authorization: Bearer <token>" -X POST \
    <AAP_HOSTNAME>/api/controller/v2/job_templates/<jt-id>/launch/
```

Remind them that the loader is idempotent — they can re-run it whenever the repo updates (e.g. when URLs are added to `playbooks/files/websites.yml`).
