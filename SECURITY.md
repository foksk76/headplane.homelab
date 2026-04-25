# Security

## Scope

This repository documents deployment procedures. It does not ship a running
service by itself, but the documented steps involve secrets and privileged
service configuration.

## Security notes

- Do not commit real `cookie_secret` values.
- Do not commit real Headscale API keys.
- Do not commit real OIDC client secrets.
- Review all hostnames, addresses, and tokens before publishing terminal output.

## Reporting

If you find a documentation issue that could cause unsafe deployment behavior,
open a private report through the repository owner before filing a public issue.

