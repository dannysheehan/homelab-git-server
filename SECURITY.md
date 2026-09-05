# Security Policy

## Supported versions

This repository tracks a single mainline deployment for Synology + Forgejo. Use the latest commit on `main`.

## Reporting a vulnerability

Please do **not** open a public issue for security-sensitive findings.

Email the maintainer via the address on your GitHub profile contact, or open a private [GitHub security advisory](https://docs.github.com/en/code-security/security-advisories/guidance-on-reporting-and-writing-information-about-vulnerabilities/privately-reporting-a-security-vulnerability) on this repository.

Include:
- Affected commit or tag
- Reproduction steps
- Impact assessment

## Secrets in forks and clones

Never commit real `inventory.ini`, `group_vars/all/vars.yml`, or `group_vars/all/vault.yml`. Those paths are gitignored; copy the `*.example` files locally instead.
