Language: [English](02-transfer-runtime-artifact.md) | [Русский](ru/02-transfer-runtime-artifact.md)

# Transfer the Runtime Artifact

This guide copies the runtime bundle from the intermediate host to the VPS.
The hostnames are anonymized; replace them with your build host and VPS names.

## Recommended transfer path

From your workstation or any host that can reach both machines:

```bash
scp root@build-host.internal:/root/headplane-v0.6.2-runtime-r2.tar.gz root@headscale.example.net:/root/
```

## If you transfer from the build host itself

Make sure the host has working SSH access first:

```bash
ssh-add -l
ssh root@headscale.example.net hostnamectl
```

## Verify the artifact on the VPS

```bash
ssh root@headscale.example.net 'ls -lh /root/headplane-v0.6.2-runtime-r2.tar.gz && sha256sum /root/headplane-v0.6.2-runtime-r2.tar.gz'
```

## Optional archive inspection

```bash
ssh root@headscale.example.net 'tar -tzf /root/headplane-v0.6.2-runtime-r2.tar.gz | sed -n "1,80p"'
```

Important files to see in the listing:

- `headplane/build/server/index.js`
- `headplane/build/hp_agent`
- `headplane/build/hp_healthcheck`
- `headplane/public/hp_ssh.wasm`
- `headplane/drizzle/meta/_journal.json`

## Navigation

Previous: [Build on the intermediate host](01-build-on-intermediate-host.md) | Next: [Install on the VPS](03-install-on-vps.md)
