Language: [English](03-install-on-vps.md) | [Русский](ru/03-install-on-vps.md)

# Install on the VPS

This guide installs Headplane natively on a VPS that already runs Headscale and
Caddy.

## Create directories

```bash
mkdir -p /opt/headplane
mkdir -p /etc/headplane
mkdir -p /var/lib/headplane
mkdir -p /usr/libexec/headplane
```

## Unpack the runtime bundle

```bash
cd /opt
tar -xzf /root/headplane-v0.6.2-runtime-r2.tar.gz
```

Install helper binaries in fixed paths:

```bash
install -m 0755 /opt/headplane/build/hp_agent /usr/libexec/headplane/agent
install -m 0755 /opt/headplane/build/hp_healthcheck /usr/libexec/headplane/healthcheck
```

## Generate a cookie secret

Headplane expects a 32-character cookie secret when `server.cookie_secret` is
set directly.

Example:

```bash
openssl rand -hex 16
```

Use the output as `REPLACE_WITH_32_CHARS`. It is 16 random bytes encoded as 32
hex characters. Do not reuse it across unrelated installs and do not commit the
real value. If you prefer a secret file, Headplane also supports
`cookie_secret_path`; in that case keep only the path in the config and store the
secret with restrictive permissions.

## Create `/etc/headplane/config.yaml`

This is the validated native-mode shape:

```yaml
server:
  host: "127.0.0.1"
  port: 3000
  base_url: "https://headscale.example.net"
  cookie_secret: "REPLACE_WITH_32_CHARS"
  cookie_secure: true
  cookie_max_age: 86400
  data_path: "/var/lib/headplane"

headscale:
  url: "http://127.0.0.1:8080"
  public_url: "https://headscale.example.net"
  config_path: "/etc/headscale/config.yaml"
  config_strict: true

integration:
  proc:
    enabled: true
```

Important:

- replace `headscale.example.net` with the public FQDN of the VPS
- keep `server.host: "127.0.0.1"` so Headplane is reachable only through Caddy
- `server.port: 3000` is the local Headplane HTTP listener used by Caddy
- `server.base_url` must not include `/admin`
- `cookie_max_age: 86400` means one day in seconds
- `data_path` must be persistent and writable by the `headplane` process
- if the agent is not used, omit `integration.agent` entirely in `v0.6.2`
- `headscale.url` should point to the local Headscale listener, not the public HTTPS URL
- `headscale.public_url` should match the public Headscale URL used by clients
- `headscale.config_path` must point to the real Headscale config file
- `integration.proc.enabled: true` means Headplane talks to the local Headscale process

## Create `systemd` unit

Path:

```text
/etc/systemd/system/headplane.service
```

Contents:

```ini
[Unit]
Description=Headplane v0.6.2
After=network-online.target headscale.service
Wants=network-online.target
Requires=headscale.service
StartLimitIntervalSec=0

[Service]
Type=simple
User=root
WorkingDirectory=/opt/headplane
Environment=HEADPLANE_CONFIG_PATH=/etc/headplane/config.yaml
ExecStart=/usr/bin/node /opt/headplane/build/server/index.js
Restart=on-failure
RestartSec=5s

[Install]
WantedBy=multi-user.target
```

Reload and start:

```bash
systemctl daemon-reload
systemctl enable --now headplane
```

## Update Caddy

Validated routing:

```caddyfile
headscale.example.net {
  @admin_no_slash path /admin
  redir @admin_no_slash /admin/ 308

  @headplane path /admin/*
  reverse_proxy @headplane 127.0.0.1:3000

  reverse_proxy 127.0.0.1:8080
}
```

Then:

```bash
caddy validate --config /etc/caddy/Caddyfile
systemctl reload caddy
```

## Optional: plan for OIDC now

If you want to use Headplane's built-in OIDC login later, keep these points in
mind while finishing the base install:

- `server.base_url` must stay on the public URL without `/admin`
- the IdP redirect URI must be `{server.base_url}/admin/oidc/callback`
- Headplane needs `headscale.api_key` or `headscale.api_key_path` for OIDC
- API key login remains available even after OIDC is enabled

Full OIDC steps live here:

- [Optional: enable OIDC in Headplane](optional-enable-oidc.md)

## Optional: enable agent later

If you want web SSH later, use this shape:

```yaml
integration:
  proc:
    enabled: true
  agent:
    enabled: true
    pre_authkey: "REUSABLE_PREAUTHKEY"
    executable_path: "/usr/libexec/headplane/agent"
    work_dir: "/var/lib/headplane/agent"
```

Generate `REUSABLE_PREAUTHKEY` on the VPS with Headscale. The exact flags can
vary by Headscale version, so check the local help first:

```bash
headscale preauthkeys create --help
```

A typical reusable key command looks like this:

```bash
headscale users list
headscale preauthkeys create \
  --user <USER_ID> \
  --reusable \
  --expiration 90d
```

For tagged infrastructure nodes in newer Headscale versions, the command may use
`--tags tag:<TAG>` without `--user`. The boring but reliable move is to trust
`headscale preauthkeys create --help` on the installed version before pasting.

Use the generated key as `REUSABLE_PREAUTHKEY`. Treat it as a secret: anyone who
has it can enroll a node until the key expires or is revoked. For less config
blast radius, use `pre_authkey_path` and keep the actual key in a root-readable
file.

## Navigation

Previous: [Transfer the runtime artifact](02-transfer-runtime-artifact.md) | Next: [Verify and log in](04-verify-and-login.md)
