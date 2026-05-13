Language: [English](optional-enable-oidc.md) | [Русский](ru/optional-enable-oidc.md)

# Optional: Enable OIDC in Headplane

This guide adds OIDC login through Headplane's built-in SSO mechanism.
It assumes that the base installation from this repository is already working.

> Status: This OIDC path is aligned with the current upstream Headplane SSO and
> configuration documentation, but it has not yet been revalidated end-to-end
> on the anonymized example VPS from this repository.

## Goal

Keep the existing `/admin` deployment path, add IdP-backed browser login, and
preserve API key access as a fallback for administrators.

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

## Prepare secret files

Create a small directory for secrets:

```bash
install -d -m 0700 /etc/headplane/secrets
printf '%s\n' 'REPLACE_WITH_OIDC_CLIENT_SECRET' > /etc/headplane/secrets/oidc_client_secret
printf '%s\n' 'REPLACE_WITH_LONG_LIVED_HEADSCALE_API_KEY' > /etc/headplane/secrets/headscale_api_key
chmod 0600 /etc/headplane/secrets/oidc_client_secret /etc/headplane/secrets/headscale_api_key
```

Use a real OIDC client secret from your identity provider and a real Headscale
API key created for Headplane's server-side actions.

## Extend `/etc/headplane/config.yaml`

Merge this into your existing configuration:

```yaml
headscale:
  url: "http://127.0.0.1:8080"
  public_url: "https://headscale.example.net"
  config_path: "/etc/headscale/config.yaml"
  config_strict: true
  api_key_path: "/etc/headplane/secrets/headscale_api_key"

oidc:
  issuer: "https://idp.example.com"
  client_id: "REPLACE_WITH_OIDC_CLIENT_ID"
  client_secret_path: "/etc/headplane/secrets/oidc_client_secret"
  scope: "openid email profile"
  use_pkce: true
```

Notes:

- `headscale.api_key_path` is required for OIDC in Headplane
- `client_secret_path` keeps the OIDC secret out of the main config file
- Headplane can auto-discover the authorization, token, and userinfo endpoints
  from the issuer metadata
- `use_pkce: true` is a good default because some IdPs insist on it

If your provider requires extra authorization parameters, add them under
`oidc.extra_params`.

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
deliberately. This is one of those five-minute tasks that saves an annoying
fifteen-minute cleanup later.

## Quick checks

```bash
curl -I https://headscale.example.net/admin/login
journalctl -u headplane -n 100 --no-pager | grep -i oidc
```

Useful signs:

- `/admin/login` returns `200 OK`
- the service log shows the OIDC flow starting without callback or issuer errors

## Navigation

Previous: [Verify and log in](04-verify-and-login.md) | Next: [Troubleshoot if needed](05-troubleshooting.md)
