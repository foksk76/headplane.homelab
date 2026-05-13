# Handoff

## Purpose

This file captures the current working state of the `headplane.homelab`
repository so work can resume quickly in another chat, shell, or host session.

## Current snapshot

- Updated: 2026-05-13 UTC
- Repository path: `/root/headplane.homelab`
- Intended remote: `git@github.com:foksk76/headplane.homelab.git`
- Repository status: published and synchronized with `origin/main`
- Publication status: GitHub repository exists and receives pushes over SSH
- Latest tag: `v0.1.1`

## What is already prepared

- Bilingual `README.md` and `README.ru.md`
- Introductory README context describing the practical access problem and Headplane role
- Step-by-step docs for build, transfer, install, verify, SSO, backup/restore, and troubleshooting
- Live validation notes now include an OIDC enablement drill with a local test
  IdP and a rollback drill back to a pre-OIDC backup
- Russian step-by-step docs in `docs/ru/`
- README badges and `Docs` link-check workflow in `.github/workflows/docs.yml`
- Repo support files: `CHANGELOG.md` as the change history for the deployment path, plus `CONTRIBUTING.md`, `SECURITY.md`, `SUPPORT.md`, and `LICENSE`

## Suggested resume commands

```bash
cd /root/headplane.homelab
git status
git log --oneline --decorate -n 5
find . -maxdepth 2 -type f | sort
```
