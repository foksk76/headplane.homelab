Language: [English](04-verify-and-login.md) | [Русский](ru/04-verify-and-login.md)

# Verify and Log In

## Service checks

```bash
systemctl status headplane --no-pager
systemctl status headscale --no-pager
systemctl status caddy --no-pager
```

Healthy example:

- `headplane.service` is `active (running)`
- `headscale.service` is `active (running)`
- `caddy.service` is `active (running)`

## Port checks

```bash
ss -ltnp | egrep ':80 |:443 |:3000 |:8080 |:9090 '
```

Typical expected listeners:

- `127.0.0.1:3000` - Headplane
- `127.0.0.1:8080` - Headscale
- `127.0.0.1:9090` - Headscale metrics
- `*:80` and `*:443` - Caddy

## Local HTTP checks

```bash
curl -I http://127.0.0.1:3000/admin/
curl -I http://127.0.0.1:8080/health
```

Expected examples:

- Headplane local admin path: `302 Found`
- Headscale health path: `200 OK`

## External HTTP checks

Replace `headscale.example.net` with the public FQDN of your VPS.

```bash
curl -I https://headscale.example.net/admin/
curl -I https://headscale.example.net/admin/machines
curl -I https://headscale.example.net/health
```

Expected examples:

- `/admin/` may redirect to `/admin/machines`
- `/admin/machines` redirects to `/admin/login` before authentication
- `/health` returns `200 OK`

## Create the first login key

Headplane uses a Headscale API key for login if OIDC is not configured.

```bash
headscale apikeys create --expiration 90d
```

Then open:

```text
https://headscale.example.net/admin/login
```

Use the generated API key in the Headplane login form.

## Optional journal checks

```bash
journalctl -u headplane -n 100 --no-pager
journalctl -u caddy -n 100 --no-pager
```

During the validated deployment, the stable startup log included:

```text
[config] INFO: Found a valid Headscale configuration file at /etc/headscale/config.yaml
[config] INFO: Using Proc integration
[config] INFO: Found headscale serve (PID ...)
[server] INFO: Running on 127.0.0.1:3000
```

## Navigation

Previous: [Install on the VPS](03-install-on-vps.md) | Next: [Troubleshoot if needed](05-troubleshooting.md)
