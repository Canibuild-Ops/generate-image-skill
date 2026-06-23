# Canibuild-Ops — Agent Guidelines

Read this before reviewing or modifying any code in this org. It mirrors the policy in `CLAUDE-canibuild.md`.

## Secrets — hard rule

Never write a literal secret into source code, `.env` files, or config files. All production credentials live in Azure Key Vault `kv-canibuild-ops`.

Local dev scripts must read from environment variables:
- Python: `os.environ["SECRET_NAME"]` with a `sys.exit(...)` if not set
- Node: `process.env.SECRET_NAME` with a clear error if undefined

When reviewing code, flag any string that looks like an API key, token, password, or connection string.

Key Vault naming: `<app-prefix>-<descriptive-name>` (e.g. `coach-slack-bot-token`, `coach-database-url`).

When a request says "hook this up to our vaulted X" without naming the secret, nothing auto-selects one — picking the name is a deliberate choice. Default to the `shared-` variant when one clearly exists, the use is genuinely cross-app, and the data isn't sensitive; reuse the shared secret (or its shared gateway) rather than minting a duplicate. Stop and confirm before wiring when more than one candidate fits (e.g. a `mason-` AND a `shared-`, read-vs-write, per-account), when the secret is sensitive/scoped (finance, HR, comp, customer-PII), when choosing shared has an access implication (single static token, no per-app revocation), or when it would re-couple the app to another system's identity. See `CLAUDE-canibuild.md` (Secrets section) for the full rule.

## Stack defaults

- New dashboards: Next.js 16+ / React 19 / Tailwind v4 / NextAuth v5 / Recharts
- Azure hosting (App Service or Static Web Apps)
- App Settings use KV references — never plain-text secrets

## Shared HubSpot identity resolution

Resolving a person or meeting to a HubSpot **company/contact** (by email, name, phone, or synced-meeting start-time) lives in the shared Python lib **`Canibuild-Ops/hubspot-identity-resolver`** (`from hubspot_identity_resolver import resolve_company_identities`). Prefer it for any new HubSpot person/meeting→company resolution instead of re-rolling search/match logic. `ai-call-coaching` consumes it (vendored as a git submodule). Existing per-repo resolvers were intentionally left in place because they serve different shapes/abstractions: `mason` (zoom/email/ticket ingest, own engagement bridge), `tender-automation` (`HubSpotTenderClient`, company-by-domain), `hubspot-key-contact-sync` (fuzzy contact-within-company for a label migration). Migrate the relevant piece onto the lib **opportunistically** when you're next working in that code — extending the lib with any missing primitive at that point. Don't force a big-bang migration (and note: pushing to `mason`/tender triggers their production deploys).

## Authentication — hard rule

Every internal Canibuild app with a UI requires SSO login restricted to `@canibuild.com` — not just dashboards. Dashboards get the wiring from `dashboard-template`; any other app with a UI gets it from the standalone `@canibuild-ops/auth` package (decoupled from the dashboard template). Never remove or weaken either.

Full policy: https://github.com/Canibuild-Ops/.github/blob/main/SECURITY.md (Authentication section)

When reviewing or creating any UI app's code:
- Use the **shared** `canibuild-internal-sso` Entra app (one app reg for all apps). Never create per-app Entra apps — admin consent requires Global Admin.
- Per-app isolation comes from `<app>-auth-secret` (JWT signing key in KV), not from per-app client_ids. The Entra client_id/secret/tenant are shared via `shared-entra-client-id` / `shared-entra-client-secret` / `shared-entra-tenant-id` in KV.
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
- **Founders** (`founders` team — Tony, Dilan). Read on every repo in the org via the team. Write only on per-repo grants (founders can't create repos directly — same gate as leaders). Also members of `leaders`.
- **Leaders** (`leaders` team — every onboarded leader, founders included). Read on guardrail repos, technical write on `mason` + `mason-core`. Intended write scope is `mason-core/{agents,context,docs,ops,skills,templates}/`; every other path in either repo is wiring and requires talking to Mark first — there is no automated guardrail enforcing this, see the per-category breakdown below. Operational repos are per-repo grants only.

Guardrail repos (covered by the `leaders` team): `claude-config`, `.github`, `repo-template`, `dashboard-template`, `design-tokens`, `ui-kit` (read), and `mason` + `mason-core` (write). These hold org rules, design system, templates, and the Mason knowledge brain.

Operational repos (dashboards, ad-factory, ai-call-coaching, hubspot-*, tender-*, leader-owned projects): founders get `pull` automatically via the `founders` team; leaders see them only after `/grant-access`.

Only org admins (Mark) can create new repos — enforced at the GitHub org level (`Members can create repositories` is unchecked for Public/Private/Internal under Settings → Member privileges). Leaders running `/create-repo` or `/new-dashboard` hit an authorization gate as step 0 of the skill and are handed a copy-pasteable Slack request for Mark; the skill stops without calling the API. Mark runs the same skill on his side, which creates the repo from the template, grants the `founders` team `pull` automatically, then `/grant-access <leader-login> <repo>` gives the requesting leader write. (Centralised 2026-06-01 — previously self-serve, but leader-created repos kept landing without the standard governance.)

Write workflows by repo category:

- **Operational repos**: push directly to `main`. No PRs required.
- **Mason editable paths** (`mason-core/{agents,context,docs,ops,skills,templates}/`): push directly to `main` via `/push` from inside the submodule. No PRs required.
- **Protected guardrail repos** (`claude-config`, `.github`, `ui-kit`, `design-tokens`, `repo-template`, `dashboard-template`): read-only via team. PR required; branch protection + CODEOWNERS (`@mdeacon-cib`) enforce review. (Audit category: `protected`.)
- **Mason wiring paths** (`mason-core/CLAUDE.md`, `mason-core/connectors.md`, `mason-slack/`, `mason-wiki-editor/`, `scripts/`, `.github/`, `.claude/`): leaders have technical write but should not push without talking to Mark first. No automated guardrail — cultural rule.

Other rules:

- Any commits you make: append `Co-Authored-By: Codex (gpt-5.4) <noreply@openai.com>` to the commit message.
- **Never run a *destructive* push.** Never `--force` / `-f` / `--force-with-lease`, never a force refspec (`+`), never `--no-verify`, `--delete`/`-d`, `--mirror`, or any push that rewrites already-published history. Routine, non-destructive pushes to Canibuild-Ops repos are fine. **Prefer the `/push`, `/push-all`, or `/push-config` skills** — they add CI watch, Codex review, deploy reporting, and Slack escalation, so reach for them first (and suggest them on proactive offers). A direct `git push` is an acceptable fallback when a skill flow can't run cleanly; it's not a breach, since the server-side Secret-scan, Codex, and Slack-summary Actions fire on every push regardless.
- **Never run `gh repo create` directly.** New repos must go through `/create-repo` (or `/new-dashboard` for dashboards). Only org admins can actually create — leaders running these skills get a copy-pasteable Slack request for Mark via the skill's step-0 authorization gate. Mark then runs the same skill end-to-end (creates from template as an operational/direct-push repo, posts Slack), and follows up with `/grant-access <leader-login> <repo>`.
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

## Large-data requests — route locally when likely to stall

Mason's cloud runtime times out on large aggregate queries. Detect before dispatching and route locally when local capability exists.

**Signals (any one triggers a pre-flight check):**
- Aggregation across the whole corpus / pipeline / customer base ("top themes", "all deals across…", "VoC", "cross-corpus")
- Date windows ≥ 3 months combined with cross-account synthesis
- Quantifiers like "all", "every", "top N" applied to large object sets
- Per-record processing across thousands of records
- Explicit `approve large query` keyword

**Pre-flight probe** the dominant data source (cheap — runs in < 1s):
- Conversation corpus: `conversation_storage.count_summaries_matching()`
- HubSpot: `/crm/v3/objects/{type}/search` with `limit=0`, read `total`
- Chargebee: no cheap total-count probe (cursor-based pagination only); route any wide-window aggregation locally by default
- Shovels: `get_shovels_usage` first (agent already enforces)
- Atlassian (Jira): JQL search with `maxResults=0`, read `total`
- Pendo: aggregation spec preview where supported

**Routing thresholds:** 1,000 records OR > 30 seconds expected wall time.

**Routing decision (announce in one sentence before executing):**

| Probe result | Local capability | Route |
|---|---|---|
| ≤ thresholds | — | `ask_mason` |
| > thresholds | Yes (mason-slack venv + KV reachable, source module importable) | Local execution |
| > thresholds | No | `ask_mason` with stall warning + local-fallback plan |

If a leader's Claude Code shell lacks the local capability (missing KV access, no mason-slack venv), say so explicitly. Tell them to run `~/.canibuild-secrets-refresh.sh` to pull KV secrets locally (NOT `/pull-config` — that's config sync only), then retry; or escalate to a leader with the needed access. Never silently retry a doomed `ask_mason` polling loop.

**Generated artifacts go to `mason/exports/<type>/`.** Use `shared.local_exports.default_export_path(report_type, filename)` — do not dump PDFs / docx / decks in the repo root. Valid types: `voc`, `churn`, `sales`, `tenders`, `decks`, `canvases`, `misc`. Content is git-ignored (customer-identifiable). Structure (README + `.gitkeep`) is tracked.

## Regression on fix

When you (or another agent) fix a bug, add a focused test that captures the bug — failing before the fix, passing after. Don't mandate broad coverage; just leave a regression test behind every fix.

Applies to: CI auto-fix diagnosing a real bug, a reported behavior bug, or a Codex-review finding that flags a real defect. Does not apply to new features, refactors with no behavior change, or doc/config edits.

If reviewing a bug-fix commit that doesn't include a test, flag it.
