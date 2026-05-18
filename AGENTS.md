# Canibuild-Ops — Agent Guidelines

Read this before reviewing or modifying any code in this org. It mirrors the policy in `CLAUDE-canibuild.md`.

## Secrets — hard rule

Never write a literal secret into source code, `.env` files, or config files. All production credentials live in Azure Key Vault `kv-canibuild-ops`.

Local dev scripts must read from environment variables:
- Python: `os.environ["SECRET_NAME"]` with a `sys.exit(...)` if not set
- Node: `process.env.SECRET_NAME` with a clear error if undefined

When reviewing code, flag any string that looks like an API key, token, password, or connection string.

Key Vault naming: `<app-prefix>-<descriptive-name>` (e.g. `coach-slack-bot-token`, `coach-database-url`).

## Stack defaults

- New dashboards: Next.js 16+ / React 19 / Tailwind v4 / NextAuth v5 / Recharts
- Azure hosting (App Service or Static Web Apps)
- App Settings use KV references — never plain-text secrets

## Authentication — hard rule

All internal Canibuild dashboards require SSO login restricted to `@canibuild.com`. The dashboard template ships with the wiring; never remove or weaken it.

Full policy: https://github.com/Canibuild-Ops/.github/blob/main/SECURITY.md (Authentication section)

When reviewing or creating dashboard code:
- Use the **shared** `canibuild-internal-sso` Entra app (one app reg for all dashboards). Never create per-dashboard Entra apps — admin consent requires Global Admin.
- Per-dashboard isolation comes from `<dashboard>-auth-secret` (JWT signing key in KV), not from per-dashboard client_ids. The Entra client_id/secret are shared via `shared-entra-client-id` / `shared-entra-client-secret` in KV.
- Reject changes that: weaken the `@canibuild.com` `signIn` callback, change the issuer to `common`/`organizations`, expand the middleware matcher to expose routes outside `/api/auth/*`, or add public API routes outside `/api/auth/*` without a documented exception.

## Building a new dashboard (applies in every repo)

If asked to "create a new dashboard", "build a dashboard", "scaffold a dashboard", or anything equivalent — **regardless of which repo you're in** — default to this:

1. **Create the repo via `/create-repo`, then scaffold from `Canibuild-Ops/dashboard-template`.** Don't build from scratch or use raw `gh repo create`:
   `/create-repo dashboard-<thing>` then use `Canibuild-Ops/dashboard-template` as the starting point for the app code.
2. **Use `@canibuild-ops/ui-kit`** for `DashboardLayout`, `DashboardHeader`, `Card`, `KPI` / `KPIGrid`, `HeroCard`, `Badge`, `Spinner`, `SegmentedControl`, `Select`, `Leaderboard`, `DataTable`. Never hand-roll these. Compose existing props (`label`, `value`, `icon`, `hint`, `headerExtra`, `breakdown` accept `ReactNode`) before forking.
3. **Use `@canibuild-ops/design-tokens`** for colour and `chartPalette`. No raw hex codes — use Tailwind tokens (`bg-brand-purple`, `text-brand-offwhite`).
4. **Domain badges**: map domain words (`achieved`, `churn`, `upgrade`) to the library's 5 semantic `Badge` variants (`success | warning | danger | neutral | info`) at the call site, not in the library.
5. **New shared patterns belong in `@canibuild-ops/ui-kit`** with a version bump — don't bury them in the dashboard repo.

If the leader explicitly says "I just want a one-off, skip the template", confirm before complying — the whole point is consistency. npm scope is `@canibuild-ops/*` (not `@canibuild/*`) because GitHub Packages ties npm scope to the GitHub org name.

## Source control — access model

Canibuild-Ops uses a **least-privilege access model**. Org default permission is `none`. Three tiers:

- **Org admin** (Mark today). Admin on every repo via the org role.
- **Founders** (`founders` team — Tony, Dilan). Read on every repo in the org via the team. Write only on per-repo grants and self-created repos. Also members of `leaders`.
- **Leaders** (`leaders` team — every onboarded leader, founders included). Read on guardrail repos, write on the editable paths in `mason`. Operational repos are per-repo grants only.

Guardrail repos (covered by the `leaders` team): `claude-config`, `.github`, `repo-template`, `dashboard-template`, `design-tokens`, `ui-kit` (read), and `mason` (write on editable paths). These hold org rules, design system, and templates.

Operational repos (dashboards, ad-factory, ai-call-coaching, hubspot-*, tender-*, leader-owned projects): founders get `pull` automatically via the `founders` team; leaders see them only after `/grant-access`.

Repos created via `/create-repo` or `/new-dashboard` invite the non-admin actor as `admin` automatically and grant the `founders` team `pull` on the new repo. Mark has admin on everything via the org-admin role. Any org member can create new repos.

Two write workflows depending on repo category:

- **Operational repos**: push directly to `main`. No PRs required.
- **Mark-only paths in guardrail repos** (config, `ui-kit`, `design-tokens`, templates, `mason` protected paths — config/instructions/Slack mechanics): branch protection on; PR + Code Owner review required. Direct push to `main` is rejected.

Other rules:

- Any commits you make: append `Co-Authored-By: Codex (gpt-5.4) <noreply@openai.com>` to the commit message.
- **Never run `git push` directly.** All pushes to Canibuild-Ops repos must go through the `/push`, `/push-all`, or `/push-config` skills, which run sensitive-file checks, CI watch, Codex review, and Slack escalation. This applies to proactive offers too — suggest `/push`, not raw `git push`.
- **Never run `gh repo create` directly.** New repos must go through `/create-repo` (or `/new-dashboard` for dashboards), which creates from the standard template, posts Slack, sends the creator-admin invite when needed, and applies branch protection only for Mark-only repos.
- **Never grant per-repo access via raw `gh api collaborators` calls.** Use `/grant-access` and `/revoke-access` — they validate inputs, surface pending invitations, and keep the audit trail clean.

Run `/audit-repos` to see every repo's category and current governance state. Use `/apply-governance <repo>` to remediate drift on an existing repo.

## Repo naming

When suggesting a name for a new repo:
- Drop the `canibuild-` prefix — the org name says it.
- If the repo belongs to an existing product cluster, use the cluster's prefix: `ad-factory-*`, `dashboard-*`, `hubspot-*`, `tender-*`. So a new Ad Factory worker is `ad-factory-<thing>`, not `canibuild-<thing>` or a standalone name.
- Singletons (`mason`, `ui-kit`, `design-tokens`, `ai-call-coaching`) stay prefix-free until a sibling repo is created.
- Open-source skills use the `<purpose>-skill` suffix.

## Review priorities

Flag these immediately:
1. Hardcoded credentials or connection strings
2. Changes to auth / authorization logic
3. Irreversible schema changes (DROP TABLE, destructive migrations)
4. New external dependencies without clear justification

Be surgical — only change what is genuinely broken.

## Regression on fix

When you (or another agent) fix a bug, add a focused test that captures the bug — failing before the fix, passing after. Don't mandate broad coverage; just leave a regression test behind every fix.

Applies to: CI auto-fix diagnosing a real bug, a reported behavior bug, or a Codex-review finding that flags a real defect. Does not apply to new features, refactors with no behavior change, or doc/config edits.

If reviewing a bug-fix commit that doesn't include a test, flag it.
