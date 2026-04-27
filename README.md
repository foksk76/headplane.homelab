Language: [English](README.md) | [Русский](README.ru.md)

# Building Headplane in a Homelab

[![Docs](https://img.shields.io/badge/docs-Headplane%20homelab-blue)](README.md)
[![Language](https://img.shields.io/badge/language-EN%20%7C%20RU-informational)](README.ru.md)
[![License](https://img.shields.io/github/license/foksk76/headplane.homelab)](LICENSE)
[![Release](https://img.shields.io/github/v/release/foksk76/headplane.homelab?include_prereleases&label=release)](https://github.com/foksk76/headplane.homelab/releases)

Docs-first repository for building Headplane on an intermediate host and installing it natively on a VPS with Headscale and Caddy.

> **Project status:** Working deployment notes for Headplane `v0.6.2`, validated with an intermediate Debian 13 build host and a Debian 13 VPS target.

> **Language policy:** `README.md` is the main English README. `README.ru.md` is the main Russian translation for homelab work and fast onboarding. Keep the language switcher as the first line in both files.

## Why this repository exists

Headplane can be built directly on a target server, but that is not always the
best operational choice. In this deployment, the build happened on an
intermediate host with a complete toolchain, while the VPS received only the
runtime artifact, service configuration, and reverse proxy changes.

This README explains how to build Headplane `v0.6.2`, package the runtime
correctly, transfer it to a VPS that already runs Headscale and Caddy, and avoid
the specific pitfalls that showed up during the real installation.

## Documentation style

The documentation is organized as a short entry point for a new reader: diagram
and components first, then build, install, verify, and troubleshoot guides.

## Logical Component Installation Diagram

```text
build-host.internal
  -> build Headplane v0.6.2
  -> archive /root/headplane-v0.6.2-runtime-r2.tar.gz
  -> transfer to headscale.example.net
  -> /opt/headplane + /etc/headplane/config.yaml + systemd
  -> Caddy: /admin/* to Headplane, all other paths to Headscale
```

## Main usage flow

1. Prepare a Debian build host with `git`, `go`, `node`, and `pnpm`.
2. Clone the upstream Headplane repository and check out `v0.6.2`.
3. Run the full build via `./build.sh` so the result includes:
   - web build
   - `hp_agent`
   - `hp_healthcheck`
   - `hp_ssh.wasm`
   - production `node_modules`
4. Create a runtime archive that also includes `drizzle/` migrations.
5. Transfer the archive to the VPS.
6. Install Headplane under `/opt/headplane`, configure `/etc/headplane/config.yaml`, add a `systemd` unit, and update Caddy to route `/admin`.
7. Verify local and external HTTP paths, then log in with a Headscale API key.

## Who this is for

- you already run Headscale on a VPS but want the web management UI that Headscale itself does not provide
- you prefer building on a separate host and copying only the ready runtime archive to the VPS
- you want a clear install order with checks after the important steps

## Installation Components

Replace `headscale.example.net`, `build-host.internal`, `<BUILD_HOST_IP>`, and
`<VPS_PUBLIC_IP>` with your own values before deployment.

| Component | FQDN / IP | OS | Main software | Purpose |
|---|---|---|---|---|
| Build host | `build-host.internal`, `<BUILD_HOST_IP>` | Debian 13 or compatible Linux | `git`, Go 1.24+, Node.js 22.x, `pnpm` 10.x | Builds Headplane and creates the runtime archive without installing the full toolchain on the VPS. |
| Headscale VPS | `headscale.example.net`, `<VPS_PUBLIC_IP>` | Debian 13 | Headscale `v0.28.0`, Headplane `v0.6.2`, Caddy `2.6.2`, Node.js 22.x, `systemd` | Runs Headscale, Headplane, and the TLS entry point. |

The layout follows common Headplane, Headscale, and Caddy guidance: one public
HTTPS entry point, loopback listeners for backend services, and generated
secrets kept out of the repository.

Reference material:

- Headplane: `https://headplane.net/install`
- Headplane configuration: `https://headplane.net/configuration`
- Headscale reverse proxy: `https://headscale.net/0.26.1/ref/integration/reverse-proxy/`
- Caddy reverse proxy practice: `https://swetrix.com/blog/caddy-reverse-proxy`

## Requirements

### Build host

- Debian 13 or similar Linux host
- `git`
- `go` 1.24+ recommended
- Node.js 22.x
- `pnpm` 10.x
- network access to GitHub and npm registries

### Target host

- Debian 13 VPS or similar
- working native `headscale`
- working `caddy`
- `node` 22.x installed
- access to `/etc/headscale/config.yaml`
- permission to run Headplane as `root` if using proc integration directly

### Known validated versions

| Component | Version | Notes |
|---|---|---|
| Headplane | `v0.6.2` | pinned upstream version in this repository |
| Headscale | `v0.28.0` | validated on target VPS |
| Node.js | `v22.22.2` | validated on build and target hosts |
| pnpm | `10.4.0` | validated on build host |
| Go | `1.24.4` | validated on build host |
| Caddy | `2.6.2` | validated on target VPS |
| Debian | `13 (trixie)` | validated on both hosts |

## Repository structure

- `README.md` - main English project overview
- `README.ru.md` - main Russian translation
- `docs/` - step-by-step guides in English
- `docs/ru/` - step-by-step guides in Russian
- `docs/01-build-on-intermediate-host.md` - build on `build-host.internal`
- `docs/02-transfer-runtime-artifact.md` - transfer the runtime bundle to the VPS
- `docs/03-install-on-vps.md` - native install on the anonymized VPS
- `docs/04-verify-and-login.md` - health checks and login verification
- `docs/05-troubleshooting.md` - troubleshooting
- `HANDOFF.md` - current repository handoff state
- `NEXT_STEPS.md` - next improvements for the repository and deployment process
- `CHANGELOG.md` - documentation history
- `CONTRIBUTING.md`, `SECURITY.md`, `SUPPORT.md`, `LICENSE` - repository governance files

## Quick start

Short path: build Headplane on the intermediate host, transfer the runtime
bundle, install it natively on the VPS, update Caddy, then verify `/admin`.

### 1. Read the step guides in order

1. [Build on the intermediate host](docs/01-build-on-intermediate-host.md)
2. [Transfer the runtime artifact](docs/02-transfer-runtime-artifact.md)
3. [Install on the VPS](docs/03-install-on-vps.md)
4. [Verify and log in](docs/04-verify-and-login.md)
5. [Troubleshoot if needed](docs/05-troubleshooting.md)

### 2. Minimal build summary

```bash
git clone https://github.com/tale/headplane.git
cd headplane
git checkout v0.6.2
pnpm install
./build.sh
```

Then create a deployment archive that keeps the migration folder:

```bash
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

### 3. Minimal VPS install summary

```bash
mkdir -p /opt/headplane /etc/headplane /var/lib/headplane /usr/libexec/headplane
cd /opt
tar -xzf /root/headplane-v0.6.2-runtime-r2.tar.gz

install -m 0755 /opt/headplane/build/hp_agent /usr/libexec/headplane/agent
install -m 0755 /opt/headplane/build/hp_healthcheck /usr/libexec/headplane/healthcheck
```

Then create:

- `/etc/headplane/config.yaml`
- `/etc/systemd/system/headplane.service`
- `/etc/caddy/Caddyfile` route for `/admin/*`

### 4. Verify the result

```bash
systemctl status headplane --no-pager
systemctl status headscale --no-pager
systemctl status caddy --no-pager

curl -I http://127.0.0.1:3000/admin/
curl -I http://127.0.0.1:8080/health
curl -I https://headscale.example.net/admin/
curl -I https://headscale.example.net/health
```

If all is well:

- `headplane` is active
- `headscale` is active
- `caddy` is active
- local `/admin/` answers
- external `https://headscale.example.net/admin/` reaches Headplane

## Pitfalls

- In `v0.6.2`, do **not** leave `integration.agent` in the config when the agent is disabled. If the block exists, validation still expects `integration.agent.pre_authkey`.
- Do **not** omit `drizzle/` from the runtime bundle. Headplane needs `drizzle/meta/_journal.json` at startup.
- `server.base_url` must be `https://headscale.example.net`, without `/admin`.
- Caddy must proxy `/admin/*` to Headplane and everything else to Headscale.
