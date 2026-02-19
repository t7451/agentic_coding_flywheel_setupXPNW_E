# CLAUDE.md — Agentic Coding Flywheel Setup (ACFS)

> AI assistant guide for this repository. Read this before touching anything.
> Canonical agent rules live in **AGENTS.md** — this file extends them with
> codebase structure, workflow recipes, SEO/DevOps automation patterns, and
> copy-paste commands.

---

## 1. What This Repo Is

ACFS bootstraps a production VPS into a fully configured AI-agent development
environment with one command:

```bash
curl -fsSL "https://raw.githubusercontent.com/.../install.sh?$(date +%s)" \
  | bash -s -- --yes --mode vibe
```

It ships four interlocked components:

| Component | Location | Language | Purpose |
|---|---|---|---|
| Website Wizard | `apps/web/` | Next.js 16 / React 19 / TS | Step-by-step GUI guiding users from laptop → live VPS |
| Installer | `install.sh` + `scripts/` | Bash | Idempotent, checkpointed VPS bootstrap (30+ tools) |
| Onboarding TUI | `packages/onboard/` | TypeScript / Shell | Interactive Linux + agent workflow tutorial |
| Module Manifest | `acfs.manifest.yaml` | YAML + Zod | Single source of truth for every installed tool |

---

## 2. Repository Layout

```
/
├── acfs.manifest.yaml        # EDIT THIS to add/remove/change tools
├── checksums.yaml            # Auto-updated by CI; never edit manually
├── install.sh                # Main entry point (4 346 lines)
├── VERSION                   # Semver string (currently 0.2.0)
│
├── apps/
│   └── web/                  # Next.js wizard website
│       ├── app/              # App Router pages (wizard/, learn/, docs/, …)
│       ├── components/       # Shared React components
│       ├── e2e/              # Playwright tests
│       └── public/           # Static assets
│
├── packages/
│   ├── manifest/             # acfs.manifest.yaml parser + Zod validator
│   │   └── src/
│   │       ├── generate.ts   # ← MODIFY THIS to change generated scripts
│   │       ├── schema.ts     # Zod schema for manifest validation
│   │       └── types.ts      # TypeScript interfaces
│   ├── installer/            # Installer helper utilities
│   └── onboard/              # Onboarding TUI source
│
├── acfs/                     # Files copied verbatim to ~/.acfs/ on VPS
│   ├── zsh/acfs.zshrc
│   ├── tmux/tmux.conf
│   └── onboard/lessons/      # 9 markdown lessons (00_welcome → 08_keeping_updated)
│
├── scripts/
│   ├── generated/            # ⚠️ AUTO-GENERATED — never edit directly
│   ├── lib/                  # Shared shell functions (logging.sh, etc.)
│   ├── providers/            # VPS-provider-specific guides
│   ├── hooks/                # Pre/post-install hooks
│   ├── templates/            # Config templates
│   └── sync/                 # Synchronisation utilities
│
├── tests/
│   ├── e2e/                  # BATS framework + expect scripts
│   ├── unit/                 # Unit tests
│   ├── vm/                   # VM bootstrap + resume tests
│   └── web/                  # Playwright web tests
│
├── docs/                     # Human-readable architecture docs
├── .github/workflows/        # CI/CD (website, installer, checksums, smoke)
├── .beads/                   # Issue tracker state (commit with code changes)
└── AGENTS.md                 # Hard rules for agents (read first)
```

---

## 3. Hard Rules (summary — full text in AGENTS.md)

- **Never delete files** without explicit per-session written approval.
- **Never run** `git reset --hard`, `git clean -fd`, `rm -rf` on non-temp paths
  without explicit approval.
- **Never edit** anything under `scripts/generated/` — regenerate instead.
- **Never use** `npm`, `yarn`, or `pnpm` — **Bun only**.
- **Always push** before ending a session (work is not done until `git push`
  succeeds).
- **Always commit** `.beads/` together with the code change it tracks.

---

## 4. Development Workflows

### 4.1 Website (Next.js)

```bash
cd apps/web
bun install          # install deps
bun run dev          # dev server (Turbopack)
bun run build        # production build
bun run lint         # ESLint 9
bun run type-check   # tsc --noEmit
bunx playwright test # e2e tests (requires built app or dev server)
```

Key conventions:
- All pages live under `app/` (App Router).
- UI = shadcn/ui + Tailwind CSS 4.
- State = URL query params + `localStorage` — no backend.
- Wizard step content defined in `app/wizard/` sub-pages.

### 4.2 Manifest / Code Generation

```bash
# After editing acfs.manifest.yaml:
cd packages/manifest
bun run generate          # regenerates scripts/generated/*.sh
shellcheck scripts/generated/*.sh   # verify shell correctness
```

Modify `packages/manifest/src/generate.ts`, not the generated output.

### 4.3 Installer (Shell)

```bash
# Lint
shellcheck install.sh scripts/lib/*.sh

# Full integration test (Docker, mirrors CI)
./tests/vm/test_install_ubuntu.sh
```

Console output rules (use helpers in `scripts/lib/logging.sh`):

```bash
echo -e "\033[34m[1/8] Step description\033[0m"  # Blue — progress steps
echo -e "\033[90m    Details\033[0m"              # Gray — indented detail
echo -e "\033[33m⚠️  Warning\033[0m"              # Yellow — warnings
echo -e "\033[31m✖ Error\033[0m"                  # Red — errors
echo -e "\033[32m✔ Success\033[0m"                # Green — success
# Progress → stderr; data → stdout; --quiet suppresses progress not errors
```

### 4.4 Running Quality Gates (before committing)

```bash
# Staged files only (fast)
ubs $(git diff --name-only --cached)

# Full workspace (slower)
bun run lint && bun run type-check
shellcheck install.sh scripts/lib/*.sh
```

### 4.5 Session Completion Checklist

```bash
git pull --rebase
bd sync              # sync beads issue tracker
git push -u origin $(git branch --show-current)
git status           # must show "up to date with origin"
```

Work is **not complete** until `git push` exits 0.

---

## 5. CI/CD Pipelines

| Workflow | File | Triggers | Gates |
|---|---|---|---|
| Website | `.github/workflows/website.yml` | Push / PR | lint, type-check, build, Playwright |
| Installer | `.github/workflows/installer.yml` | Push / PR | ShellCheck, YAML lint, manifest drift |
| Checksum monitor | `.github/workflows/checksum-monitor.yml` | Schedule | Tool checksum freshness |
| Checksum system tests | `.github/workflows/checksum-system-tests.yml` | Push | End-to-end checksum validation |
| Production smoke | `.github/workflows/production-smoke.yml` | Deploy | Smoke tests against live site |
| Playwright | `.github/workflows/playwright.yml` | Push / PR | Cross-browser e2e |

**SEO CI gates** to add when modifying the website (see Section 8):
- Sitemap presence + lastmod freshness
- robots.txt syntax
- Canonical tag consistency
- Structured data (`@type` presence)
- Lighthouse performance budget

---

## 6. Issue Tracking (bd / beads)

All tracking lives in `.beads/`. No markdown TODO lists, no GitHub issues as
primary tracker.

```bash
bd ready --json                          # unblocked work
bd create "Title" -t feature -p 1 --json
bd update bd-42 --status in_progress --json
bd close bd-42 --reason "Done" --json
```

Use `bv --robot-triage` for dependency-aware triage and priority analysis.
Use `bv --robot-next` for the single highest-value next action.

Always commit `.beads/` in the same commit as the code change it describes.

---

## 7. Key Tools Available on Target VPS

| Tool | Command | Purpose |
|---|---|---|
| Named Tmux Manager | `ntm` | Agent cockpit — manages sessions |
| MCP Agent Mail | `am` / MCP server | Agent-to-agent coordination |
| Ultimate Bug Scanner | `ubs` | Pre-commit bug scan |
| Beads Viewer | `bv --robot-*` | Graph-aware issue triage |
| Coding Agent Session Search | `cass search` | Search prior agent sessions |
| Cass Memory System | `cm context` | Retrieve relevant rules before tasks |
| Simultaneous Launch Button | `slb` | Two-person rule for dangerous commands |
| Destructive Command Guard | `dcg` | Claude Code hook blocking dangerous ops |
| Repo Updater | `ru` | Multi-repo sync + AI commit automation |
| Get Image from Internet Link | `giil` | Download cloud images for visual debug |
| Chat Share Conversation to File | `csctf` | Archive AI chat links to Markdown/HTML |

---

## 8. SearchOps Autopilot — SEO + DevOps Automation

> Role: **SearchOps Autopilot** — technical SEO + DevOps assistant.
> Every SEO finding maps to the Search pipeline: **Crawl → Index → Serve**.
> Every manual step gets an automation proposal.

### 8.1 Mental Model

```
Crawl      Discovery (robots.txt, sitemap, links, CDN/WAF blocks)
           Fetching  (server reachability, render, timeouts)

Index      Processing   (canonical, hreflang, noindex, duplicate content)
           Coverage     (Page Indexing report status buckets)

Serve      Ranking      (signals, Core Web Vitals, E-E-A-T)
           Eligibility  (rich results, AMP, SafeSearch)
           Rich features (schema, structured data, breadcrumbs)
```

### 8.2 Diagnosis Response Template

When a search visibility issue is reported, structure the response as:

```
1) Situation summary (1-5 bullets)
   - What/when/scope/what changed

2) Diagnosis map (Crawl / Index / Serve)
   - Findings (facts from data)
   - Likely causes (hypotheses)
   - Evidence to confirm (what report/log would prove it)

3) Immediate next actions (manual, prioritised)
4) Automation plan (Now / Next / Later)
5) Implementation artifacts (copy-paste ready)
6) Validation + rollback plan
```

Never fabricate data. If you lack a report/log/screenshot, state what is
needed and exactly how to obtain it.

### 8.3 Core Search Console Workflows

```
Page Indexing report   → coverage buckets (Crawled / Discovered / Excluded)
Performance report     → clicks, impressions, CTR, position over time
URL Inspection tool    → single-URL ground truth (indexing/render/schema/AMP)
Rich result reports    → structured data errors and warnings
```

Treat URL Inspection as the source of truth for any single-URL issue.

### 8.4 Robots.txt Token Reference

Use these tokens when writing or debugging `robots.txt` rules:

| Token | Affects |
|---|---|
| `Googlebot` | Search + Discover + most features |
| `Googlebot-Image` | Image indexing, logos, favicons |
| `Googlebot-Video` | Video indexing and video features |
| `Googlebot-News` | Google News surfaces |
| `Storebot-Google` | Shopping surfaces |
| `Google-InspectionTool` | Testing tools (Rich Results Test, URL Inspection) |
| `GoogleOther` | Generic Google fetchers — does NOT affect ranking |
| `Google-CloudVertexBot` | Site-owner-requested Vertex AI crawls |
| `Google-Extended` | Gemini training/grounding — does NOT affect Search |

### 8.5 Log Analysis — Safe Googlebot Verification

Do not trust User-Agent strings alone. Verify via:

1. Reverse DNS: `host <IP>` → must end in `.googlebot.com` or
   `geo.googlebot.com`.
2. Forward DNS: `host <reverse-result>` → must resolve back to original IP.
3. IP allowlist: cross-reference against
   `https://developers.google.com/search/apis/ipranges/googlebot.json`.

Use wildcards on Chrome version strings in log filters
(`Chrome/\d+\.\d+\.\d+\.\d+`) — do not pin exact versions.

### 8.6 Automation Plan Tiers

```
NOW  (quick wins — alerts, scheduled checks, basic CI gates)
     → sitemap lastmod freshness cron
     → robots.txt lint in CI
     → Search Console API alert on coverage drop > 5%

NEXT (deeper pipelines — log analysis, anomaly detection, regression suites)
     → server log → BigQuery pipeline for Googlebot activity
     → weekly canonical drift check (extracted vs declared)
     → structured data validation in staging deploy gate

LATER (advanced observability + IaC)
     → CDN WAF rule audit tied to crawl budget monitoring
     → Lighthouse regression suite per deploy
     → Blue/green deploy with automated GSC performance delta check
```

### 8.7 Hosting Blueprint (AutoHost DevKit)

When scaffolding or auditing the hosting stack:

```
CDN/WAF → edge cache → origin/app → storage/db → observability
```

Repo scaffold:
```
/app      Application source
/infra    IaC (Terraform / Pulumi)
/scripts  Automation scripts
/seo      robots.txt, sitemap config, structured data templates
/docs     Architecture + runbooks
/ci       Reusable CI workflow fragments
```

SEO gates in every deploy pipeline:
1. Build / lint / type-check
2. Sitemap presence + `lastmod` freshness
3. `robots.txt` syntax validation
4. Redirect + canonical rule consistency check
5. Structured data `@type` presence
6. Performance budget (Lighthouse CI or equivalent)
7. Deploy to staging → smoke tests → promote to production

### 8.8 Validation + Rollback Pattern

For every change, define:

```
Success metric   → (e.g.) GSC Page Indexing "Indexed" count ≥ baseline within 14 days
Time window      → 7–14 days for indexing; 24–48 h for crawl signals
Rollback trigger → Indexed count drops > 10% from pre-deploy baseline
Rollback method  → git revert + redeploy; or blue/green swap
```

---

## 9. Structured Data Quick Reference

For any page that should earn rich results:

```json
{
  "@context": "https://schema.org",
  "@type": "SoftwareApplication",
  "name": "ACFS Installer",
  "operatingSystem": "Ubuntu 25.10",
  "applicationCategory": "DeveloperApplication"
}
```

Validation steps:
1. Run through Google's Rich Results Test.
2. Check URL Inspection → "Enhancements" tab.
3. Monitor Rich result status report for errors.

---

## 10. Common Recipes

### Add a new tool to the installer

1. Edit `acfs.manifest.yaml` — add entry with `install`, `verify`, and
   `checksum` fields.
2. Run `cd packages/manifest && bun run generate`.
3. Run `shellcheck scripts/generated/*.sh`.
4. Run `ubs scripts/generated/install_<category>.sh`.
5. Test locally or via `./tests/vm/test_install_ubuntu.sh`.
6. Commit `acfs.manifest.yaml` + `scripts/generated/` + `.beads/` together.

### Add a new wizard page

1. Create `apps/web/app/wizard/<slug>/page.tsx`.
2. Register the step in the wizard step list (check `lib/wizardSteps.ts` or
   equivalent).
3. Run `bun run lint && bun run type-check`.
4. Add a Playwright test in `apps/web/e2e/`.
5. Commit.

### Investigate a Search visibility drop

1. Open Search Console → Performance report → filter by date range around
   the incident.
2. Open Page Indexing report → note status bucket changes.
3. Use URL Inspection on affected URLs.
4. Check server logs for Googlebot 4xx/5xx or blocked IPs.
5. File a `bd create` issue with findings, hypotheses, and action items.
6. Follow the Diagnosis Response Template (Section 8.2).

### Regenerate checksums locally

```bash
# checksums are auto-updated by CI; run manually only if needed
bun run --filter @acfs/manifest checksums:update
```

---

## 11. Environment Variables

No `.env` file is checked in. All secrets are managed via:
- **HashiCorp Vault** on the VPS (for runtime secrets)
- **Vercel environment variables** (for website deployments)
- **GitHub Actions secrets** (for CI/CD)

Never commit secrets. Never hardcode API keys.

---

## 12. Dependency Management

| Layer | Tool | Lockfile |
|---|---|---|
| JS/TS (all workspaces) | Bun | `bun.lock` (root) |
| Shell tools | `acfs.manifest.yaml` checksums | `checksums.yaml` |
| OS packages | `apt` inside installer | Pinned in `install.sh` |

`bun.lock` is the only JS lockfile. Do not introduce `package-lock.json`,
`yarn.lock`, or `pnpm-lock.yaml`.

---

## 13. Branch + Commit Conventions

Branch naming: `claude/<description>-<session-id>`

Commit style: Conventional Commits

```
feat(web): add reconnect-ubuntu wizard step
fix(installer): handle missing curl on minimal Ubuntu images
chore(security): auto-update checksums for uv
docs: update CLAUDE.md with SearchOps patterns
```

Scope examples: `web`, `installer`, `manifest`, `onboard`, `ci`, `security`,
`seo`.

---

*Last updated: 2026-02-19 by SearchOps Autopilot (claude/create-searchops-claude-md-2Pu1x)*
