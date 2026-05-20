# url_checker

Ansible playbook that verifies a list of external URLs is reachable from wherever
the play runs. Useful for confirming that a host (or a network segment) has the
egress it needs before installing software that pulls from those endpoints.

## Run it

```bash
ansible-playbook playbooks/main.yml
```

The play runs against `localhost` with `connection: local`, so no inventory is
required. Each entry in `playbooks/files/websites.yml` is probed once per scheme
it lists. The play exits non-zero if any probe does not match its expected
status code.

## Adding a URL

Edit `playbooks/files/websites.yml`. Each entry has four fields:

| Field     | Description                                                       |
|-----------|-------------------------------------------------------------------|
| `url`     | Hostname (or host + path) to probe.                               |
| `method`  | HTTP method: `HEAD`, `GET`, `OPTIONS`, etc.                       |
| `status`  | Expected HTTP status code (integer).                              |
| `schemes` | List of schemes to probe — e.g. `[https]` or `[http, https]`.     |

Example:

```yaml
- url: example.com
  method: HEAD
  status: 200
  schemes: [https]
```

Pick the lightest method the endpoint accepts (`HEAD` whenever possible) and set
`status` to whatever that endpoint actually returns — some endpoints answer
`404` or `403` on the root path, and that is still a valid signal that the host
is reachable.

## Output

On success the play prints a one-line summary:

```
N of N URL probes passed.
```

On failure each failed probe is listed with the URL, method, expected status,
and what was actually returned, then the play fails with a non-zero exit code.
