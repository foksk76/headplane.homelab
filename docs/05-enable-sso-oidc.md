Language: [English](05-enable-sso-oidc.md) | [Русский](ru/05-enable-sso-oidc.md)

# Enable SSO with OIDC

This guide adds browser SSO through Headplane's built-in OIDC mechanism.
It assumes that the base installation from this repository is already working.

> Status: This path was rechecked against a live Debian 13 VPS with a local test
> IdP. One version-specific detail matters: Headplane `v0.6.2` expects
> `oidc.headscale_api_key` in the `oidc` block.

## Goal

Keep the existing `/admin` deployment path, add IdP-backed browser login, and
preserve API key access as a fallback for administrators.

Before changing authentication on a live host, take a fresh archive with the
backup guide from this repository.

## Requirements

- a working Headplane installation
- `server.base_url` already set to the public URL without `/admin`
- an OpenID Connect provider that exposes a standard issuer URL
- a Headscale API key with a long enough expiration for server-side OIDC work

Good to know before you start:

- Headplane recommends using the same OIDC client for both Headscale and
  Headplane when possible
- if Headscale itself uses local users instead of OIDC, automatic matching will
  not work on the first login; the user will choose their Headscale identity
  once during onboarding

## Register the callback URL

In your identity provider, register this exact callback URL:

```text
https://headscale.example.net/admin/oidc/callback
```

If Headscale already uses the same identity provider and you want to reuse the
same OIDC client, also make sure the Headscale callback URL is present on that
client.

## Prepare secrets

Store the OIDC client secret on disk:

```bash
install -d -m 0700 /etc/headplane/secrets
printf '%s\n' 'REPLACE_WITH_OIDC_CLIENT_SECRET' > /etc/headplane/secrets/oidc_client_secret
chmod 0600 /etc/headplane/secrets/oidc_client_secret
```

Use a real OIDC client secret from your identity provider.

## Extend `/etc/headplane/config.yaml`

Merge this into your existing configuration for `v0.6.2`:

```yaml
oidc:
  issuer: "https://idp.example.com"
  headscale_api_key: "REPLACE_WITH_LONG_LIVED_HEADSCALE_API_KEY"
  client_id: "REPLACE_WITH_OIDC_CLIENT_ID"
  client_secret_path: "/etc/headplane/secrets/oidc_client_secret"
  scope: "openid email profile"
  use_pkce: true
  disable_api_key_login: false
```

Notes:

- in `v0.6.2`, `oidc.headscale_api_key` is required for OIDC in Headplane
- `client_secret_path` keeps the OIDC secret out of the main config file
- Headplane can auto-discover the authorization, token, and userinfo endpoints
  from the issuer metadata
- `use_pkce: true` is a good default because some IdPs insist on it
- `disable_api_key_login: false` preserves API key access as a break-glass path

If your provider requires extra authorization parameters, add them under
`oidc.extra_params`.

## Optional: local test IdP

If you want to prove the OIDC flow before wiring in a real identity provider,
you can use a small local Dex service behind Caddy.

Validated example shape:

```text
https://headscale.example.net/idp
  -> Caddy
  -> Dex on 127.0.0.1:5556
```

This is useful for smoke tests, but a real external IdP is still the proper
destination for a long-lived setup.

## Restart Headplane

```bash
systemctl restart headplane
systemctl status headplane --no-pager
```

If the service does not come back cleanly, check:

```bash
journalctl -u headplane -n 100 --no-pager
```

## Test the login flow

Open:

```text
https://headscale.example.net/admin/login
```

What to expect:

- the first OIDC user becomes `Owner`
- later OIDC users become `Member` until promoted
- API key sessions still keep full access and bypass the role model
- if Headscale uses local users, Headplane asks the user to select the matching
  Headscale identity once

After the first successful OIDC login, review the Users page and assign roles
deliberately. Five boring minutes here beat fifteen noisy ones later.

## Quick checks

```bash
curl -I https://headscale.example.net/admin/login
journalctl -u headplane -n 100 --no-pager | grep -i oidc
```

Useful signs:

- `/admin/login` returns `200 OK`
- the service log shows the OIDC flow starting without callback or issuer errors

## Navigation

Previous: [Verify and log in](04-verify-and-login.md) | Next: [Backup and restore](06-backup-and-restore.md)
