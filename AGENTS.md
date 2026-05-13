# AGENTS.md

## Mission

This repository documents a validated, anonymized path for building Headplane on
an intermediate host and installing it natively on a VPS with Headscale and
Caddy.

The repository is docs-first. Treat the Markdown files as the product and the
shell commands as the executable interface to that product.

## Project Snapshot

- Current documented Headplane version: `v0.6.2`
- Current validated OS family: Debian 13
- Canonical English entry point: `README.md`
- Canonical Russian entry point: `README.ru.md`
- English step guides: `docs/`
- Russian step guides: `docs/ru/`
- Repo state trackers: `HANDOFF.md`, `NEXT_STEPS.md`, `CHANGELOG.md`
- Local scratch file: `.codex` is intentionally gitignored

### Maintainer TODO

- TODO: record the next validated Headplane release when it is verified
- TODO: record the date of the next full reinstall check
- TODO: decide whether screenshots are in scope for this repo
- TODO: decide whether release tags should map to GitHub Releases every time

## Source of Truth

Use this order when the repository drifts or older notes disagree:

1. `README.md`
2. `README.ru.md`
3. `docs/` and `docs/ru/`
4. `HANDOFF.md`
5. `NEXT_STEPS.md`
6. `CHANGELOG.md`

## Boundaries

- Do not reframe the repository as part of a larger project family unless the
  user explicitly asks for that wording.
- Do not de-anonymize real hostnames, IP addresses, usernames, API keys, or
  cookie secrets in tracked files.
- Do not add speculative steps as if they were validated. Mark unverified
  guidance clearly or leave it out.
- Do not turn the docs into a stand-up routine. A little DevOps wit is fine;
  clown mode is not.

## Placeholder Policy

Use stable, boring placeholders:

- `build-host.internal`
- `headscale.example.net`
- `<BUILD_HOST_IP>`
- `<VPS_PUBLIC_IP>`
- `<HEADSCALE_API_KEY>`
- `REPLACE_WITH_32_CHARS`

Before finishing documentation work, search for leaked real values.

Suggested check:

```bash
rg -n 'ts\\.conf\\.bar|operator\\.homelab|([0-9]{1,3}\\.){3}[0-9]{1,3}' .
```

## Writing Rules

### English docs

- English is the canonical source.
- Prefer short, operator-friendly prose.
- Keep commands copy-pasteable.
- Explain why a step exists when it prevents a likely foot-gun.

### Russian docs

- Prefer Russian terms over unnecessary anglicisms.
- Good examples:
  - `build host` -> `хост сборки`
  - `runtime bundle` -> `исполняемый архив` or `архив для установки`
  - `reverse proxy` -> `обратный прокси`
  - `service` -> `служба`
  - `validation` -> `проверка`
- A little technical slang and dry DevOps humor are welcome, but the command
  still has to survive the night shift.

### Across both languages

- Keep the language switcher as the first line in both README files.
- If an English guide changes in meaning, update the Russian counterpart in the
  same session when practical.
- Keep `docs/` and `docs/ru/` structurally aligned.
- Prefer plain-text diagrams over image-only diagrams.

## Version-Sensitive Rules

These are easy to forget and painful to rediscover:

1. In Headplane `v0.6.2`, omit `integration.agent` entirely if the agent is not
   used. Leaving the block in place can still trigger validation of
   `integration.agent.pre_authkey`.
2. Keep `drizzle/` in the runtime archive. Without
   `drizzle/meta/_journal.json`, startup fails.
3. `server.base_url` must not include `/admin`.
4. Caddy must route `/admin/*` to Headplane and everything else to Headscale.

When version-specific traps change, update:

- `docs/05-troubleshooting.md`
- `docs/ru/05-troubleshooting.md`
- `CHANGELOG.md`

## Editing Checklist

Before you wrap a documentation change:

1. Update the English source.
2. Update the Russian counterpart if meaning changed.
3. Re-check placeholders and anonymization.
4. Confirm README language switchers still point to the right files.
5. Add a `CHANGELOG.md` entry for meaningful updates.
6. Update `HANDOFF.md` and `NEXT_STEPS.md` if repo state materially changed.

Optional checks:

```bash
git diff --check
find docs docs/ru -type f | sort
```

## Suggested Local Setup

These are not all mandatory, but they make Codex sessions less mushy:

1. Keep `.codex` as a local-only scratchpad for:
   - real hostnames
   - one-off commands
   - temporary notes from a live maintenance window
2. Use `.editorconfig` to reduce formatting churn in Markdown-heavy changes.
3. Consider adding one of these next:
   - `markdownlint` config for heading/list consistency
   - `Vale` or `cspell` word lists for bilingual terminology
   - `lychee.toml` if link-check exceptions become noisy
4. Keep automation light. This repo is documentation, not a Kubernetes operator
   that escaped containment.

## Suggested Local `.codex` Starter

This file is gitignored. Use it as a local memory cache, not as repo truth.

```text
Current real build host:
Current real target VPS:
Last manual validation date:
Open questions:
- 
- 
```

## Suggested Future Corrections

These are not urgent, but they would improve future Codex passes:

- Add a small `docs/context/` note for each fully revalidated installation run.
- Decide whether `HANDOFF.md` should always link the latest context file.
- Add a documented release process if tags are meant to signal validated states.

