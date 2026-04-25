Language: [English](05-troubleshooting.md) | [Русский](ru/05-troubleshooting.md)

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

Previous: [Verify and log in](04-verify-and-login.md) | Next: [Quick start](../README.md#quick-start)
