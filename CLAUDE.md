# CLAUDE.md — generate-image-skill

Org-wide policy (Key Vault, SSO, push-to-main, stack defaults, Mason context) lives in
`~/.claude/CLAUDE.md` and is not repeated here. This file is repo-specific.

## What this is

A **Claude Code skill** that generates AI images via Google's Gemini API
(Nano Banana 2 / `gemini-3.1-flash-image-preview`). It is the Canibuild-Ops **fork** of the
open-source `tomc98/generate-image-skill` (`upstream` remote), so keep changes minimal and
mergeable. The published skill name is `generate-image`.

This is a local end-user skill that runs inside a leader's Claude Code shell — it is NOT an
Azure-deployed app. There is no App Service, no managed identity, no CI deploy.

## Structure

- `SKILL.md` — skill manifest (frontmatter `name` / `description` / `allowed-tools: Bash Read Write`)
  + the instructions Claude follows when invoked.
- `scripts/generate.py` — the only code. Python 3.8+, **stdlib only** (`urllib`, `base64`, `argparse`);
  no `requirements.txt`, no third-party deps. Keep it that way.
- `README.md` — human-facing install/usage docs.
- `AGENTS.md` — synced from `canibuild-config` (Codex guidelines mirroring org policy); do not hand-edit.
- `.github/CODEOWNERS` — `* @mdeacon-cib` (Mark-only write; applied by `apply-governance.sh`).
- `.github/workflows/secret-scan.yml` — gitleaks via the reusable `Canibuild-Ops/.github` workflow.
- `LICENSE` — MIT.

## How it works

`generate.py` POSTs to
`https://generativelanguage.googleapis.com/v1beta/models/gemini-3.1-flash-image-preview:generateContent`
with `responseModalities: [TEXT, IMAGE]`, decodes the inline base64 image from the response, and
writes a PNG.

CLI:
```bash
python3 $SKILL_DIR/scripts/generate.py "<prompt>" --name "<slug>" [--aspect 16:9] [--reference a.png b.png] [--output /path.png]
```
- Aspect ratios: `1:1 2:3 3:2 3:4 4:3 4:5 5:4 9:16 16:9 21:9` (default `1:1`).
- `--reference` accepts up to 14 images (used for edits / style / character consistency).
- Output filename defaults to `<sanitized-name>-<YYYYMMDD-HHMMSS>.png`.

## Invocation / install / test

- Invoked automatically when a user asks Claude to create/generate/edit an image.
- Install: `git clone https://github.com/Canibuild-Ops/generate-image-skill.git ~/.claude/skills/generate-image`
  (or via skill-manager `skill.sh install <url>`).
- Test: run `generate.py` directly with a prompt and confirm a PNG is written; no test suite exists.

## Conventions / gotchas

- **API key is an env var, NOT Key Vault.** The script reads `GEMINI_API_KEY` from the environment
  (set in `~/.claude/settings.json` under `env`); this is a local per-leader skill, so the org
  "vault every secret" rule does not apply to the key itself. Never hardcode a key in `generate.py`.
- **Output-dir mismatch to be aware of:** `SKILL.md` instructs Claude to save to `Archive/Files/`
  (Obsidian vault) and only prints a `![[...]]` wikilink when the path contains `Archive/Files`.
  But `generate.py`'s default output dir is `IMAGE_OUTPUT_DIR` or **cwd** — it does not default to
  `Archive/Files/`. Set `IMAGE_OUTPUT_DIR` or pass `--output` to control where files land.
- Keep `scripts/generate.py` dependency-free and the fork close to upstream.
- Do not hand-edit `AGENTS.md` — it is synced from `canibuild-config`.
