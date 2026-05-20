# Security Policy

## Scope

This is a small utility repo that probes a list of public URLs for reachability. It contains **no credentials, no tokens, no customer data, and no production endpoints**. The playbook runs locally against `localhost` and only makes outbound HTTPS requests to public hosts.

## Supported Versions

Only the latest commit on `main` is maintained.

## Reporting a Vulnerability

Because this repo has no production exposure, **open a public GitHub issue** to report any security concerns. There is no need to use private disclosure for this project.

When reporting, include:
- A description of the issue
- The file(s) affected
- Any suggested fix if you have one

## What Should Never Be Committed

This repo intentionally has no secrets and should stay that way. Never commit:

- Credentials, tokens, or passwords of any kind
- Internal hostnames or URLs that aren't already public

If you spot any of the above committed by mistake, open an issue immediately.
