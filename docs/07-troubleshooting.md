Language: [English](07-troubleshooting.md) | [Русский](ru/07-troubleshooting.md)

# Troubleshooting

## `integration.agent.pre_authkey must be a string (was missing)`

Cause:

- `v0.6.2` still validates the `integration.agent` block even when `enabled: false`

Fix:

- remove `integration.agent` entirely if the agent is not used
- or provide a real `pre_authkey` if the agent is used

## `Error: Can't find meta/_journal.json file`

Cause:

- the runtime bundle was created without `drizzle/`

Fix:

- rebuild the archive and include:
  - `headplane/drizzle/`
  - `headplane/drizzle/meta/_journal.json`

## `/admin` opens Headscale or a blank response instead of Headplane

Cause:

- Caddy still proxies all paths to Headscale

Fix:

- add a dedicated `/admin/*` matcher that proxies to `127.0.0.1:3000`
- keep the default route pointed at `127.0.0.1:8080`

## Login does not stick

Cause:

- `server.cookie_secure` is wrong for the actual access path

Fix:

- if the public path is HTTPS behind Caddy, keep `cookie_secure: true`
- if testing plain HTTP directly, use `cookie_secure: false`

## `OIDC is not enabled or misconfigured`

Cause:

- the `oidc` block is missing from `/etc/headplane/config.yaml`
- `issuer` is wrong or unreachable from the VPS
- `oidc.headscale_api_key` is missing or invalid

Fix:

- add a complete `oidc` section
- verify that the issuer URL is reachable from the Headplane host
- for `v0.6.2`, provide a working long-lived key in `oidc.headscale_api_key`

## `Redirect URI Mismatch` during IdP login

Cause:

- the callback registered in the IdP does not exactly match Headplane's public
  callback URL

Fix:

- register `https://headscale.example.net/admin/oidc/callback`
- keep `server.base_url` as `https://headscale.example.net`
- do not append `/admin` to `server.base_url`

## User signs in through OIDC but cannot see their machines

Cause:

- Headplane could not match the OIDC identity to a Headscale user

Fix:

- if Headscale already uses OIDC, prefer the same OIDC client for both services
- make sure the IdP provides the `sub` claim
- if you use different clients, make sure the `email` claim is available
- if Headscale uses local users, complete the one-time manual user selection in
  Headplane onboarding

## Headplane starts, but cannot integrate with Headscale settings

Cause:

- `headscale.config_path` is wrong
- Headplane cannot read the file
- proc integration cannot see the Headscale process

Fix:

- verify the real path to the Headscale config
- run Headplane with enough privileges for `integration.proc.enabled`

## `/admin` path breaks after rebuild

Cause:

- `server.base_url` incorrectly includes `/admin`

Fix:

- use `https://headscale.example.net`, not `https://headscale.example.net/admin`

## Caddy validation warning about formatting

Cause:

- the Caddyfile is valid but not formatted

Fix:

```bash
caddy fmt --overwrite /etc/caddy/Caddyfile
```

This is cosmetic and does not block runtime if `caddy validate` already says
`Valid configuration`.

## Navigation

Previous: [Backup and restore](06-backup-and-restore.md) | Next: [Quick start](../README.md#quick-start)
