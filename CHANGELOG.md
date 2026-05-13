# Changelog

This file tracks changes to the validated Headplane deployment path in this
repository.
It does not list routine edits to the prose unless they change the actual build,
packaging, proxying, installation, or verification flow.

## Unreleased

### Added

- A dedicated SSO guide for Headplane's built-in OIDC flow.
- A dedicated backup and restore guide for configuration and local state before
  risky changes.
- A validated rollback note for the case where a host is restored from a
  pre-OIDC backup after later OIDC-related changes.
- A dedicated upgrade guide for checking the latest stable release and
  upgrading older native installs to `v0.6.2`.

### Changed

- The login flow now documents both Headscale API key access and Headplane's
  native OIDC path.
- The guide naming now follows the numbered step-by-step style across OIDC,
  backup/restore, upgrade, and troubleshooting.
- The OIDC configuration notes now reflect the validated `v0.6.2` behavior
  using `oidc.headscale_api_key` or `oidc.headscale_api_key_path`.
- The backup and restore flow now explicitly covers optional local IdP files,
  cleanup of newer OIDC artifacts before rollback, and the expected loss of API
  keys created after the backup point.
- The build guide now reflects the upstream `v0.6.2` engine constraints and the
  practical `Go 1.23+` requirement behind `./build.sh`.
- The OIDC guide now documents `oidc.enabled: true` and the file-based
  `oidc.headscale_api_key_path` option for `v0.6.2`.
- Troubleshooting now explicitly calls out the version-specific `v0.6.2` OIDC
  field naming.

## 0.1.1 - 2026-04-27

### Added

- A validated build path for Headplane `v0.6.2` on an intermediate Debian 13
  host with Node.js `22.x`, `pnpm 10.x`, and Go `1.24.x`.
- A native VPS installation path for Debian 13 with Headscale and Caddy already
  present on the target host.
- A runtime packaging step that ships the built application, helper binaries,
  production dependencies, and required migration files without dragging the
  full build toolchain onto the VPS.
- A verification flow for local and public endpoints before first login.

### Fixed

- Startup failure caused by a missing `drizzle/meta/_journal.json` by keeping
  `drizzle/` in the runtime archive.
- Headplane `v0.6.2` config validation failure by removing `integration.agent`
  completely from the no-agent deployment path.
- Incorrect `server.base_url` handling by documenting it without the `/admin`
  suffix.
- Reverse proxy routing so `/admin/*` stays on Headplane and the remaining
  traffic continues to reach Headscale.

### Removed

- Production source maps from the shipped runtime archive.
- The unused `integration.agent` block from the default deployment path when
  web SSH and agent features are not enabled.
