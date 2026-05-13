Language: [English](07-upgrade-headplane.md) | [Русский](ru/07-upgrade-headplane.md)

# Upgrade Headplane

This guide captures the native upgrade path used by this repository when
checking a live VPS against the latest stable Headplane release.

> Status: As of May 13, 2026, the latest stable release is `v0.6.2`.
> The validated VPS in this repository was already running `v0.6.2`, so no live
> binary rollout was needed during this check. The steps below document how to
> upgrade an older native install to the same stable target.

## Goal

Move an older native Headplane install forward without drifting away from the
official `v0.6.2` release behavior.

## Check the current versions first

On the VPS:

```bash
node -v
headscale version
sed -n '1,40p' /opt/headplane/package.json
systemctl is-active headplane headscale caddy
```

Healthy shape:

- `headplane` is already `active`
- `headscale` is already `active`
- `caddy` is already `active`
- `/opt/headplane/package.json` shows the currently deployed Headplane version

## Release-impact notes for `0.6.x`

These are the configuration and operational changes that matter most during an
upgrade inside the `0.6.x` line.

### `0.6.0`

- Headplane requires Headscale `0.26.0` or newer.
- `/admin` handling became stricter, so keep the reverse proxy route clean and
  do not strip the `/admin` prefix before Headplane sees it.

### `0.6.1`

- Headplane started using `/var/lib/headplane/hp_persist.db`.
- Sensitive values gained `_path` support, including:
  - `server.cookie_secret_path`
  - `integration.agent.pre_authkey_path`
  - `oidc.client_secret_path`
  - `oidc.headscale_api_key_path`

### `0.6.2`

- Headscale `0.28.0` support was added.
- `server.cookie_max_age` and `server.cookie_domain` became configurable.
- `./build.sh` became the preferred way to produce the full native runtime.
- OIDC behavior changed in ways worth checking explicitly:
  - `server.base_url` is the canonical source for the callback URL
  - `oidc.redirect_uri` is deprecated
  - `oidc.use_pkce` is now an explicit setting
  - `oidc.enabled` is available to make the intent explicit
  - `oidc.token_endpoint_auth_method` became optional

## Version-specific config note

The public documentation site is not perfectly version-pinned in every example.
For the exact `v0.6.2` runtime used by this repository, prefer the tagged
release source when field names disagree.

In particular, for `v0.6.2` OIDC use:

- `oidc.headscale_api_key`, or
- `oidc.headscale_api_key_path`

Do not rewrite that to `headscale.api_key` on this stable release line.

## Take a backup before rollout

Use the backup guide first:

- [Backup and restore](06-backup-and-restore.md)

This is the boring part that keeps the pager quiet later.

## Rebuild the runtime bundle on the intermediate host

```bash
cd /root
git clone https://github.com/tale/headplane.git
cd headplane
git checkout v0.6.2
pnpm install
./build.sh

tar -czf /root/headplane-v0.6.2-runtime-r2.tar.gz \
  --exclude="*.map" \
  --exclude="node_modules/.vite-temp" \
  -C /root \
  headplane/package.json \
  headplane/pnpm-lock.yaml \
  headplane/config.example.yaml \
  headplane/build \
  headplane/public \
  headplane/node_modules \
  headplane/drizzle
```

## Review config changes before restart

At minimum, check these items on the VPS:

- `server.base_url` uses the public URL without `/admin`
- `server.cookie_secure: true` is kept if the public path is HTTPS behind Caddy
- `server.cookie_max_age` and optional `server.cookie_domain` are set
  deliberately, not by accident
- OIDC uses `oidc.enabled: true` plus `oidc.headscale_api_key` or
  `oidc.headscale_api_key_path`
- if the agent is not used, remove `integration.agent` entirely in `v0.6.2`

## Roll out the new runtime on the VPS

Copy the new archive to the VPS, then:

```bash
systemctl stop headplane

cd /opt
tar -xzf /root/headplane-v0.6.2-runtime-r2.tar.gz

install -m 0755 /opt/headplane/build/hp_agent /usr/libexec/headplane/agent
install -m 0755 /opt/headplane/build/hp_healthcheck /usr/libexec/headplane/healthcheck

systemctl daemon-reload
systemctl start headplane
```

If the `systemd` unit or Caddy configuration changed, reload those too. If they
did not, do not create extra churn just to feel productive.

## Verify after the upgrade

```bash
systemctl status headplane --no-pager
systemctl status headscale --no-pager
systemctl status caddy --no-pager

curl -I https://headscale.example.net/admin/login
curl -I https://headscale.example.net/health
journalctl -u headplane -n 100 --no-pager
```

If OIDC is enabled, also check:

```bash
journalctl -u headplane -n 100 --no-pager | grep -i oidc
```

## What was verified in this repository

During the May 13, 2026 check:

- upstream latest stable was confirmed as `v0.6.2`
- the validated VPS was already on `v0.6.2`
- no binary upgrade was needed on that host
- service health remained good after the verification pass

## Navigation

Previous: [Backup and restore](06-backup-and-restore.md) | Next: [Troubleshoot if needed](08-troubleshooting.md)
