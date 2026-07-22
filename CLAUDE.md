# CLAUDE.md

Guidance for Claude Code (and other AI assistants) working in this repository.

## What this repo is

**Windows-Linux--Docker-Handbook** (formerly Windows-Admin-Cheat-Sheet) — a
curated, security-conscious collection of sysadmin one-liners and reference
material, plus self-contained single-page HTML web apps that render the
cheat sheets. Maintained by Chris Grady (@cgfixit).
Published via GitHub Pages: <https://cgfixit.github.io/Windows-Linux--Docker-Handbook/>

## Repository layout

| Path | Purpose |
|------|---------|
| `Windows-Admin-Cheat-Sheet.md` | Master Windows cheat sheet (CMD / PowerShell / CIM). Authored as a runnable `.bat`-style reference with `::` comment banners and a compatibility key. |
| `Windows-Admin-Cheat-Sheet.html` | Single-page, dependency-light web-app version of the Windows sheet (search + copy buttons). |
| `Linux-Mac-Admin-Cheat-Sheet.md` | Companion cheat sheet for Linux (RHEL, Ubuntu, Fedora) and macOS. Structured to mirror the Windows sheet. |
| `Linux-Mac-Admin-Cheat-Sheet.html` | Single-page, dependency-light web-app version of the Linux/macOS sheet (search + copy buttons). |
| `Docker-Containers.md` | Concise skeleton cheat sheet for Docker Engine/CLI, Compose, and Buildx. Intentionally lightweight; not yet built out into an HTML app. |
| `README.md` | Human-facing landing / readme. |
| `LICENSE` | Repository license. |
| `.github/workflows/ci.yml` | CI: enumerates `.html` files and runs HTMLHint as a quality gate on pushes/PRs to `main`/`master`. |

Note: individual file names (e.g. `Windows-Admin-Cheat-Sheet.md`) are unchanged
by the repository rename above — only the GitHub repository identifier/URL
changed, not the files within it.

## Cheat-sheet conventions

- **Never commit secrets.** No passwords, API keys, tokens, license keys, or
  private hostnames/IPs. Use placeholders (`XXXXX`, `hostname`, `192.0.2.x`).
  Commands should read secrets from interactive prompts, a vault, or env vars.
- **Compatibility tags** flag where each command applies:
  - Windows sheet: `[ALL]`, `[2019+]`, `[2022+]`, `[2025]`, `[DEP]` (deprecated).
  - Linux/Mac sheet: `[ALL]`, `[LINUX]`, `[RHEL/Fed]`, `[Ubuntu]`, `[macOS]`, `[DEP]`.
  - Docker sheet: `[ALL]`, `[LINUX]`, `[DESKTOP]` (Docker Desktop only), `[DEP]`.
- **Prefer modern over legacy**, and show the modern replacement next to any
  deprecated command (e.g. `wmic` → `Get-CimInstance`, `ifconfig` → `ip addr`,
  `netstat` → `ss`).
- Keep the **numbered section + Table-of-Contents** structure. It keeps the
  docs easy to diff and easy to parse into the HTML apps. When adding platform
  coverage, mirror the existing organization so the sheets stay parallel and
  can merge into a unified web app later.

## Building / previewing

- The `.html` files are fully self-contained (CDN Tailwind + Font Awesome) and
  open directly in a browser — there is no build step.
- CI runs on `main`/`master` pushes and PRs (`windows-latest`) and runs
  HTMLHint over `**/*.html`. Keep any HTML valid so the quality gate passes.

## Working style for changes

- Develop on a feature branch and open a PR into `main` (see the repo's PR
  history). Keep commits focused and descriptively messaged.
- Do not add heavyweight build tooling or external runtime dependencies; the
  value here is portability — plain Markdown and a single self-contained HTML
  file per platform family.
