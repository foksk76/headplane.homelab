Language: [English](06-backup-and-restore.md) | [Русский](ru/06-backup-and-restore.md)

# Backup and Restore

This guide captures the backup and restore pattern used before changing
Headplane, Headscale, or the reverse proxy on a live VPS.

> Status: Rechecked against a live rollback from a pre-OIDC backup on Debian 13.
> The validated path included cleanup of newer local IdP files before restore
> and confirmed that Headscale API keys created after the backup point stopped
> working after `db.sqlite` was restored.

## Goal

Keep a rollback point for configuration and local state before touching:

- `/etc/headplane`
- `/etc/headscale`
- `/etc/caddy/Caddyfile`
- `headplane.service`
- Headplane and Headscale SQLite databases
- optional local IdP files if you run OIDC through a local helper such as Dex

## When to take a backup

Take a fresh backup before:

- enabling SSO or changing OIDC settings
- changing Caddy routes
- changing `systemd` service definitions
- editing Headscale or Headplane configuration on a host that is already in use

## Recommended archive name

Use a name that carries the repository, host, and UTC timestamp:

```text
headplane.homelab__headscale.example.net__YYYYMMDDTHHMMSSZ-backup.tar.gz
```

Example:

```text
headplane.homelab__headscale.example.net__20260513T115434Z-backup.tar.gz
```

## Create the backup on the target VPS

```bash
backup_name='headplane.homelab__headscale.example.net__20260513T115434Z-backup.tar.gz'

tar -czf "/root/${backup_name}" \
  --ignore-failed-read \
  --warning=no-file-changed \
  --absolute-names \
  /etc/headplane \
  /etc/headscale \
  /etc/caddy/Caddyfile \
  /etc/dex \
  /etc/systemd/system/headplane.service \
  /usr/lib/systemd/system/headscale.service \
  /etc/systemd/system/dex.service \
  /usr/local/bin/dex \
  /var/lib/headplane/hp_persist.db \
  /var/lib/headscale/db.sqlite \
  /var/lib/headscale/noise_private.key \
  /var/lib/headscale/derp_server_private.key
```

Notes:

- `--ignore-failed-read` is useful because not every host has every optional
  file
- it is fine if an optional DERP key is absent
- it is also fine if `/etc/dex`, `dex.service`, or `/usr/local/bin/dex` do not
  exist on a host that never used a local IdP
- the archive is meant for rollback, not for public sharing

## Copy the backup to the build host

Keep a second copy away from the target machine:

```bash
mkdir -p /root/backups
scp "root@headscale.example.net:/root/${backup_name}" /root/backups/
sha256sum "/root/backups/${backup_name}"
```

## Quick archive check

```bash
tar -tzf "/root/backups/${backup_name}" | sed -n '1,80p'
```

Make sure the listing includes the expected config files and both SQLite
databases.

## Restore configuration and local state

Stop the services before restoring files:

```bash
systemctl stop dex || true
systemctl stop headplane
systemctl stop caddy
systemctl stop headscale
```

If you are rolling back from a newer OIDC-enabled state to an older pre-OIDC
archive, remove the newer OIDC-only files first. Extracting an older tarball
does not delete newer leftovers for you.

Validated cleanup example:

```bash
rm -rf /etc/dex
rm -f /etc/systemd/system/dex.service
rm -f /usr/local/bin/dex
rm -f /etc/headplane/secrets/oidc_client_secret
rm -f /etc/headplane/oidc_client_secret
```

Restore from the archive:

```bash
tar -xzf "/root/${backup_name}" -P
```

Then reload `systemd` and start services again:

```bash
systemctl daemon-reload
systemctl start headscale
systemctl start headplane
systemctl start caddy
```

## Post-restore checks

```bash
systemctl status headscale --no-pager
systemctl status headplane --no-pager
systemctl status caddy --no-pager

curl -I http://127.0.0.1:8080/health
curl -I https://headscale.example.net/admin/
```

Expected shape:

- Headscale is `active`
- Headplane is `active`
- Caddy is `active`
- `/health` answers through Headscale
- `/admin/` reaches Headplane again
- if the archive predates OIDC, the login page no longer shows SSO controls

If you restored an older Headscale database, also verify that API keys minted
after the backup timestamp no longer work. That behavior is expected and is a
good sign that the rollback really rewound the local state.

## What this guide does not restore

- external IdP state
- DNS records
- certificates managed outside the archived files
- any intentional Headscale data changes made after the backup

That last point matters: restoring `db.sqlite` rolls Headscale state back to the
moment of the archive. Do that only when you mean it.

## Navigation

Previous: [Enable SSO with OIDC](05-enable-sso-oidc.md) | Next: [Upgrade Headplane](07-upgrade-headplane.md)
