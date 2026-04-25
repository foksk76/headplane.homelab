Language: [English](01-build-on-intermediate-host.md) | [Русский](ru/01-build-on-intermediate-host.md)

# Build on the Intermediate Host

This step guide describes how to build Headplane `v0.6.2` on an intermediate
host such as `build-host.internal`.

The hostname is anonymized. Use any reachable build host with enough disk, RAM,
and outbound access to upstream package sources.

## Goal

Produce a runtime bundle that can be copied to a VPS without rebuilding there.

## Host assumptions

- Debian 13
- root shell or equivalent sudo access
- outbound access to GitHub, npm registries, and upstream Go modules

## Install the toolchain

```bash
apt update
apt install -y curl git golang ca-certificates gnupg apt-transport-https
curl -fsSL https://deb.nodesource.com/setup_22.x | bash -
apt install -y nodejs
corepack enable
corepack prepare pnpm@10.4.0 --activate
```

Verify:

```bash
git --version
go version
node -v
pnpm --version
```

Expected example:

```text
git version 2.47.3
go version go1.24.4 linux/amd64
v22.22.2
10.4.0
```

## Clone upstream Headplane

```bash
cd /root
git clone https://github.com/tale/headplane.git
cd headplane
git checkout v0.6.2
```

Optional sanity check:

```bash
git describe --tags --exact-match
```

Expected:

```text
v0.6.2
```

## Install dependencies and build

Use the full build entrypoint, not only `pnpm build`:

```bash
pnpm install
./build.sh
```

Why `./build.sh` matters:

- it builds the React server/client bundle
- it builds `build/hp_agent`
- it builds `build/hp_healthcheck`
- it builds `public/hp_ssh.wasm`
- it prunes `node_modules` to production dependencies

## Validate build outputs

```bash
ls -lh \
  build/server/index.js \
  build/hp_agent \
  build/hp_healthcheck \
  public/hp_ssh.wasm \
  public/wasm_exec.js
```

## Create the runtime archive

The validated deployment used a bundle that keeps the migration directory.

```bash
cd /root
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

Optional checksum:

```bash
sha256sum /root/headplane-v0.6.2-runtime-r2.tar.gz
```

Validated example from the recorded deployment:

```text
9582cacf2222a2a55a6ab174ebd0494a23a67d0cf9881a45e5ad765db98b2a1d  /root/headplane-v0.6.2-runtime-r2.tar.gz
```

## Why `drizzle/` must be included

Headplane runs database migrations on startup. If `drizzle/meta/_journal.json`
is missing, startup fails with:

```text
Error: Can't find meta/_journal.json file
```

That exact failure happened during the validated install and is now part of the
packaging checklist.

## Navigation

Previous: [Quick start](../README.md#quick-start) | Next: [Transfer the runtime artifact](02-transfer-runtime-artifact.md)
