# PiVortex/guest-book-examples — automation guide

This repository was set up with automated code review, CI, and PR tooling. This guide has three parts: **what was installed**, the **one-time setup** you must finish for it to work, and **how to use** each piece day to day. Everything below describes only what was actually installed.

---

## What was installed

**`.github/`**
- `.github/workflows/claude-review.yml` — on-demand Claude PR review (comment `/claude-review` as @PiVortex)
- `.github/workflows/ci.yml` — PR CI: contract checks (`cargo fmt --check`, `cargo clippy -D warnings`, `cargo test`) and frontend checks (frozen `pnpm install`, `pnpm run build`), path-filtered, ending in the single required `ci-passed` check
- `.github/copilot-instructions.md` — repo-aware guidance for Copilot code review
- `.github/CODEOWNERS` — `* @PiVortex`
- `.github/dependabot.yml` — weekly grouped updates for npm (`/frontend`, pnpm), cargo (`/contract-rs`), and GitHub Actions

**`.claude/`**
- `.claude/commands/` — the PR command suite: `fix-issue`, `full-fix`, `quick-fix`, `resolve-pr-reviews`, `auto-resolve-pr`, `cleanup-worktrees`, `dependabot-triage`, plus shared `utils/` (`worktree`, `untrusted-input`, `self-review`, `check-and-fix-ci`)
- `.claude/project-specifics/pr-quality-gate.md` — the pre-PR gate (contract + frontend), in lockstep with `ci.yml`
- `.claude/project-specifics/project-specific-concerns.md` — this repo's review priorities + universal rules
- `.claude/project-specifics/commit-conventions.md` — Conventional Commits with scopes `contract` / `frontend`
- `.claude/project-specifics/dependency-triage-map.md` — repo knowledge for `/dependabot-triage` (pnpm audit, no committed `Cargo.lock`, manual-run recipes)
- `.claude/settings.json` — denies force-push, merges, and pushes to `main` / `master`

**Repo root**
- `CLAUDE.md` — project guide for Claude Code (layout, commands, invariants, conventions)
- `.gitignore` — added `.claude/worktrees/`
- `instructions.md` — this guide
- `contract-rs/src/lib.rs` — two test assertions changed from `assert_eq!(x, false)` to `assert!(!x)` so the new CI's `cargo clippy -D warnings` passes (no behaviour change)

---

# Part 1 — Finish the setup

Do these once. They need your tokens, your GitHub permissions, or the GitHub settings UI — the setup can't do them for you. Work through them in order.

## 1. Commit and push the setup

> **This repo is a fork.** GitHub keeps Actions disabled on a fork until you opt in, so do **step 7.1 (enable Actions)** before pushing — otherwise this push won't run CI.

The setup only changed your local working tree — nothing is committed yet. Review **every** generated file using the review table from the setup run, including the `.claude/commands/` files: they are installed from the framework with only their branch/allowlist placeholders filled, but they drive your PR automation and run with elevated tool permissions, so read them and confirm they do what you expect before trusting them. Then commit and push to `main` (directly or via a PR):

```bash
git status                 # review everything the setup added first
git add contract-rs/src/lib.rs
git commit -m "test(contract): use assert! for boolean checks"
git add -A                 # picks up .github/, .claude/, CLAUDE.md, instructions.md, and the .gitignore change
git commit -m "ci: add repo automation (CI, Claude review, dependabot, .claude commands)"
git push
```

The workflows only start working once they are on the **default branch**.

## 2. Claude review token

The `/claude-review` workflow authenticates with a Claude subscription (no API key).

1. **Generate the token** (needs an active Claude subscription):
   ```bash
   claude setup-token
   ```
2. **Store it as a repo secret** named `CLAUDE_CODE_OAUTH_TOKEN` (paste when prompted — don't commit it):
   ```bash
   gh secret set CLAUDE_CODE_OAUTH_TOKEN -R PiVortex/guest-book-examples
   ```

> The token is tied to one account; every run counts against that account's usage limits. Regenerate with `claude setup-token` if reviews start failing with auth errors.

## 3. Copilot code review access

There is no ruleset, workflow, or token to set up — Copilot reviews on request. The one prerequisite is **Copilot code-review access**: the repo (or its org) must have Copilot code review enabled, and whoever requests the review needs a **Copilot seat** with available premium-request quota. If a Copilot request errors or no review appears, that access is what to check.

## 4. Dependabot

On a normal repo, Dependabot's scheduled **version-update** PRs start automatically once `.github/dependabot.yml` is on `main`. **This repo is a fork, and GitHub never auto-enables version updates on forks** — turn them on yourself, together with the two related features below. All three live in **Settings → Advanced Security** (labelled *Code security* / *Code security and analysis* on older layouts); the `/dependabot-triage` command's vulnerability scan depends on the alerts toggle:

1. **Dependabot version updates** — click **Enable** (fork-only step). Without it, `dependabot.yml` is ignored.
2. **Dependabot alerts** — turn this on so GitHub flags known vulnerabilities in your dependencies. `/dependabot-triage` reads them via `GET /repos/PiVortex/guest-book-examples/dependabot/alerts`; if the command reports that call 403s, this toggle is off. It is currently **off** on this repo — and a local `pnpm audit` of `frontend/pnpm-lock.yaml` already reports critical/high advisories (e.g. in `next`), so this is worth doing promptly.
3. **Dependabot security updates** (recommended) — auto-opens PRs that bump a vulnerable dependency to a patched version, on top of the scheduled version updates.

## 5. Continuous integration

The setup generated `.github/workflows/ci.yml`. It runs the check-mode counterpart of your pre-push quality gate (`.claude/project-specifics/pr-quality-gate.md`) on every pull request to `main` (and on the merge queue, if you enable one). No token or secret to set up. It ends in a single aggregate **`ci-passed`** job that succeeds when every check passed or was skipped — make **that** job the required status check in the branch-protection ruleset below, not the individual jobs, so docs-only PRs still pass. Review the jobs before pushing: confirm they cover every ecosystem and match what the quality gate fixes.

The pre-existing `tests-rs.yml` still runs `cargo test` on contract PRs (ubuntu + macOS). It overlaps with `ci.yml`'s contract job; keep it for the macOS coverage or retire it — either way, don't make it a required check (see the next step).

## 6. Branch protection ruleset

On github.com: `PiVortex/guest-book-examples` → **Settings** → **Rules** → **Rulesets** → **New ruleset** → **New branch ruleset**.

> If a ruleset already targets `main`, edit that one instead of creating a duplicate.

1. **Ruleset name**: `main`
2. **Enforcement status** (at the top): set to **Active**
3. **Bypass list**: this is a personal (non-org) repo, so there are no teams — click **Add bypass** and grant it to the **Repository admin** role.
4. **Target branches**: **Add target** → **Include by pattern** → `main`
5. **Branch rules** — enable exactly these:
   - ✅ **Restrict deletions**
   - ✅ **Require a pull request before merging**
     - **Required approvals**: `1`
     - ✅ **Require review from Code Owners**
   - ✅ **Block force pushes**
   - ✅ **Require status checks to pass** → add **`ci-passed`** only (the aggregate job from `ci.yml`; it appears in the picker once CI has run at least once). Do **not** add `tests-rs.yml`'s `workflows (ubuntu-latest)` / `workflows (macos-latest)` checks: that workflow is path-filtered to `contract-rs/**`, so requiring them would leave every frontend- or docs-only PR stuck waiting.
6. Click **Save changes** (code owners only — the CODEOWNERS file is what makes the Code Owner review requirement bite).

> As the only code owner, you can't approve your own PRs — merge them through the Repository-admin bypass. Add a second maintainer to `CODEOWNERS` if you want a real second review.

## 7. Additional repo-specific setup

This repo is a fork of `near-examples/guest-book-examples`, which needs a few extra steps:

1. **Enable GitHub Actions on the fork.** `PiVortex/guest-book-examples` → **Actions** tab → **I understand my workflows, go ahead and enable them**. Until you do, no workflow runs here — not CI, not `/claude-review`.
2. **Disable the upstream DevRel hook.** `issue-transfer.yml` runs on every opened PR/issue (`pull_request_target`) and adds it to the `near` org's project #117 using a `PROJECT_GH_TOKEN` secret this fork doesn't have, so it fails on every PR. After step 7.1:
   ```bash
   gh workflow disable "Add to DevRel Project" -R PiVortex/guest-book-examples
   ```
   (Only if this fork really should feed that project: instead set `gh secret set PROJECT_GH_TOKEN -R PiVortex/guest-book-examples` to a PAT with access to the `near` org's project.)
3. **GitHub Pages for the frontend (or disable the deploy).** `nextjs.yml` deploys `frontend/` to Pages (environment `github-pages`) on every push to `main` that touches `frontend/**`. Pages isn't enabled on this fork, so that job fails until you either enable it — **Settings → Pages → Source: GitHub Actions** (the production `basePath` `/guest-book-examples` already matches `https://pivortex.github.io/guest-book-examples/`) — or turn it off:
   ```bash
   gh workflow disable "Deploy Next.js site to Pages" -R PiVortex/guest-book-examples
   ```
4. **Pin `gh` to the fork in each clone.** In a fork, `gh` can resolve to the upstream `near-examples/guest-book-examples` (it asks interactively, and `gh pr create` offers upstream as the base). Run this once per clone so manual `gh` commands — and the PR commands' repo lookup — target the fork:
   ```bash
   gh repo set-default PiVortex/guest-book-examples
   ```
5. **Issues (optional).** Issues are disabled on this repo, so `/fix-issue <number>` has nothing to fetch — use its free-text mode (`/fix-issue "describe the problem"`) or enable **Settings → General → Features → Issues**. `/claude-review` on PRs works either way.

## 8. Smoke test

Open a test PR:

- As `PiVortex`, comment `/claude-review` — the workflow should add a 👀 reaction, then post one consolidated review comment. If it doesn't fire: confirm `claude-review.yml` is on `main`, Actions are enabled on the fork (step 7.1), the comment was posted by `PiVortex`, and `CLAUDE_CODE_OAUTH_TOKEN` exists (the Claude review token step above).
- Copilot review: request **Copilot** from the test PR's **Reviewers** panel (or let `/fix-issue` request it) and confirm it posts inline comments. If it doesn't appear: recheck Copilot code review is enabled and you have a seat + quota.

---

# Part 2 — How to use it

What you now have, and when to reach for each. Only the pieces installed above are listed.

## The PR command suite

Run these from a Claude Code session **inside a clone of this repo** — they resolve the GitHub repo, branch, and PR numbers themselves, and run each fix in an isolated git worktree under `.claude/worktrees/` so parallel work doesn't collide. AI reviews are **off by default** on the core commands; add a reviewer flag to request them (`--claude-review` and/or `--copilot-review`, or `--all-review`).

- **`/fix-issue <issue-number | issue-url | "free-text problem">`** — branch → research → plan (you approve) → implement with tests → quality gate + self-review → open a PR → wait for CI and fix failures. The everyday "turn this into a reviewed PR" command.
- **`/full-fix <input> [--max-passes N]`** — `/fix-issue`, then loop review→fix to **consensus** (up to 5 passes, `--all-review` by default). Use for changes that matter.
- **`/quick-fix <input>`** — `/fix-issue`, then **one** review→fix round. Use for small, low-stakes changes.
- **`/resolve-pr-reviews <pr> [--fix] [reviewer flags]`** — on an existing PR: classify the review comments, fix the blocking ones, keep CI green, and re-request the named reviewer(s). Use once reviews have landed.
- **`/auto-resolve-pr <pr> --all-review [--max-passes N]`** — drive an open PR to consensus hands-off (wait for reviews + CI → resolve → repeat). Requires a reviewer flag.
- **`/cleanup-worktrees`** — review and prune the worktrees the fix/resolve commands leave under `.claude/worktrees/`. Run it periodically; it never discards uncommitted/unpushed work without confirming.

Rule of thumb: `/quick-fix` for trivial, `/full-fix` for anything that matters, `/resolve-pr-reviews` or `/auto-resolve-pr` for PRs already open. None of them ever merge — that stays a human decision.

## Code review

- **Claude** — comment **`/claude-review`** on any PR (as `PiVortex`). A multi-agent review posts one consolidated comment splitting **blocking** defects from **design notes**. The fix commands can trigger it for you with `--claude-review`.
- **Copilot** — requested on demand (by the commands with `--copilot-review`, or the PR's **Reviewers** panel). What it flags is steered by `.github/copilot-instructions.md`.

Both reviewers skip anything listed under a PR's `## Design decisions / accepted tradeoffs` heading — record deliberate tradeoffs there so they aren't re-raised each round.

## Dependency triage

- **`/dependabot-triage [ecosystem] [--no-prs] [--no-vulns] [--md]`** — a **read-only** triage of open Dependabot PRs and security alerts, with a suggested action for each (and a local `pnpm audit` / `cargo audit` cross-check). It never merges, closes, or edits anything. Run it to decide what to merge, close, override, or dismiss. Security fixes are flagged 🔴 for priority, but a security fix that is also a major bump still gets a migration review and any manual run the triage map calls for before you merge it. Its vulnerability scan needs **Dependabot alerts** enabled (Part 1). Repo-specific verification notes live in `.claude/project-specifics/dependency-triage-map.md`.
