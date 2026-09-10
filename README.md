# upterm

Self-hosted [`uptermd`](https://github.com/owenthereal/upterm) relay (open-source instant terminal sharing) via Docker Compose, with linting, CI, and release automation wired in from [Self-Hosting-Template](https://github.com/Self-Host-Server/Self-Hosting-Template).

This replaces [tmate.io](https://tmate.io) — its official relay servers are shutting down (disabling 2026-12-11; see [nixos/nixpkgs#457502](https://github.com/NixOS/nixpkgs/issues/457502)). `upterm` is the same drop-in SSH-session-sharing model: run `upterm host -- <command>` on a machine, get back an SSH connection string, and anyone with it can join the session — now pointed at a relay we run ourselves instead of a third party.

## Deploying

1. Copy `.env.example` to `.env`. Everything in it is optional — uptermd runs fine with none of it set; see the comments in `.env.example` for what each knob does (PROXY protocol for a TCP reverse proxy).
2. Copy `authorized_keys.example` to `authorized_keys` and add the public key(s) allowed to **host** sessions through this relay (standard OpenSSH `authorized_keys` format, one key per line). This does not gate who may _join_ a session someone else is already hosting — that's the per-session token, unchanged. To add another key later: append a line and `docker compose restart uptermd`.
3. Start the stack:

   ```bash
   docker compose up -d
   ```

   On first start, the `uptermd-keygen` service generates a persistent SSH host key into a named volume before `uptermd` starts (so joiners don't get a "host key changed" warning on every container restart — see the comments in `compose.yml`).

4. `uptermd` listens on:
   - `2222` — SSH, what `upterm`/`ssh` clients connect through
   - `8080` — WebSocket + HTTP (`/health` for a liveness probe, `/getting-started` for connection instructions)
   - `9091` — Prometheus metrics (container-internal only, not published to the host)

   Put a reverse proxy in front of `8080` for TLS/your real domain if exposing this beyond a LAN. `2222` is raw SSH and doesn't need one — a TCP-mode stream proxy in front of it works, but Zoraxy (or whatever's routing `*.gavva.dev`) has to actually support TCP/UDP stream proxying, not just HTTP virtual hosting, since SSH doesn't carry the hostname the way TLS SNI/HTTP `Host:` does.

5. From a client machine:

   ```bash
   upterm host --server ssh://<this-host>:2222 -- bash
   ```

   This prints a connection string other people can use to join. See [upterm's own README](https://github.com/owenthereal/upterm#quick-start) for the full client-side usage (host/proxy/session commands, `--server wss://` for the WebSocket transport if fronted by a TLS reverse proxy).

See the comments in [`compose.yml`](compose.yml) for the verified reasoning behind the routing/node-addr/healthcheck/host-key choices — none of it is copied from upstream's Fly.io example as-is, since that config is Fly-specific (Consul routing, Fly-internal DNS for node addressing) and doesn't apply to a single self-hosted Docker Compose node.

## What's included

- **`compose.yml`** — the uptermd stack (`uptermd-keygen` init container, `uptermd`), configured via env vars from `.env`.
- **`environment.yml` / `requirements.txt`** — conda environment (Python, pip, `gh`) with Python deps installed via pip.
- **`pyproject.toml`** — [tox](https://tox.wiki) environments for linting and formatting:
  - `lint` — `ruff check`
  - `format` — `ruff format` + `ruff check --fix` + `prettier --write` + `taplo fmt`
  - `txt-lint` — [textlint](https://textlint.github.io/) over `**/*.txt`
  - `prettier` — `prettier --check` over CSS/JS/HTML/JSON/YAML/Markdown
  - `toml-lint` — `taplo` format/lint check over TOML files
  - `duplicate-code` — [jscpd](https://github.com/kucherenko/jscpd) zero-tolerance duplicate-code scan (config in `.jscpd.json`)
  - `secret-detection` — [TruffleHog](https://github.com/trufflesecurity/trufflehog) scan of the full git history via Docker
  - `zizmor` — [zizmor](https://github.com/woodruffw/zizmor) static-analysis security scan of the GitHub Actions workflows themselves
  - `github` — the full read-only CI chain (`lint` + `txt-lint` + `prettier` + `toml-lint` + `duplicate-code`)
  - `all` — `format`, then `github`, then `secret-detection`
  - Also configures [git-cliff](https://git-cliff.org/) for generating changelogs/PR descriptions from Conventional Commits.
- **`package.json`** — `prettier` and `textlint` (+ plugins), installed on demand by the relevant tox envs.
- **`.github/workflows/`**
  - `tests.yml` — runs `tox -e github` on every push and PR.
  - `secrets.yml` — TruffleHog scan on every push and PR.
  - `zizmor.yml` — zizmor scan of `.github/workflows/` on every push to `main`.
  - `duplicate-code.yml` — jscpd scan.
  - `cascade-merge.yml` / `release.yml` — on push to `main`, bumps semver based on Conventional Commit prefixes (`feat` → minor, `fix`/other → patch, `!`/`BREAKING CHANGE` → major) and publishes a GitHub Release with an auto-generated changelog.
- **`CODEOWNERS`** — defaults review ownership to `@Self-Host-Server/code-owners`.
- **`.gitignore`** — editor/AI-assistant artifacts (`.vscode`, `.cursor`, `CLAUDE.md`, etc.), `.env`, `node_modules`.

## Development

1. Set up the environment (env name comes from `conda_name` in `.env`):

   ```bash
   make conda
   conda activate upterm
   ```

2. Install `tox` and run the full check locally before pushing (requires Docker for `secret-detection`):

   ```bash
   pip install tox
   tox -e all      # format, then github (lint + txt-lint + prettier + toml-lint + duplicate-code), then secret-detection
   ```

   `tox -e github` runs everything except `secret-detection` — it's what CI's `tests.yml` runs, but it will **not** catch a leaked credential the way `tox -e all` (or CI's separate `secrets.yml`) does. `zizmor` is also standalone (own CI workflow, network access needed to install) — run it explicitly with `tox -e zizmor`.

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md) for the commit convention (Conventional Commits — it drives changelog generation and release versioning) and the local checks to run before opening a PR.
