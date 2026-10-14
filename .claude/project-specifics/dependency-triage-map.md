# Dependency triage map — guest-book-examples

Read by `/dependabot-triage`. Keep it in lockstep with `.github/dependabot.yml`, the CI workflow, and any manual/e2e suite the repo has.

## Manifests & ecosystems

- npm (pnpm): /frontend  ·  cargo: /contract-rs  ·  github-actions: /  ·  devcontainers: /.devcontainer (not watched by `dependabot.yml`)
- **Audit cross-check commands for this repo.** The frontend is managed with **pnpm** — there is no `package-lock.json`, so audit it with `( cd frontend && pnpm audit --json )` instead of `npm audit --package-lock-only` (it reads `pnpm-lock.yaml` and needs no `node_modules`; `metadata.vulnerabilities` and `advisories` have the npm-audit shape). Fixes go through pnpm too: `pnpm update <pkg>` or a `pnpm.overrides` entry in `frontend/package.json`, then commit the regenerated `pnpm-lock.yaml`. **`Cargo.lock` is gitignored** — run `cargo audit --file contract-rs/Cargo.lock` only if a local lockfile exists from a prior build; otherwise report "cargo audit skipped — no committed lockfile".

## Verification surfaces

No manual/e2e suite — CI is the only automated gate; major bumps on untested surfaces need a manual run (below).

| Dependency / surface | How to verify | Why CI doesn't cover it |
|---|---|---|
| `near-api-js`, `@hot-labs/near-connect`, `function-call-key-plugin` (frontend) | manual — see below | CI only builds the static site; wallet connection, signing, and the access-key plugin only run in a browser |
| `next`, `react`, `react-dom`, `bootstrap`, `bootstrap-icons` (frontend) | CI `Frontend checks`; majors also need the manual run below | a green build doesn't prove the UI renders and behaves |
| `near-sdk` (contract) | CI `Contract checks` (unit + sandbox test); majors also need the manual run below | the sandbox doesn't exercise the live testnet deployment the frontend calls |
| `near-sandbox`, `near-api`, `cargo-near-build`, `tokio`, `serde_json`, `testresult` (contract dev-deps) | CI `Contract checks` | covered — test harness only |
| `eslint`, `eslint-config-next`, `@types/node`, `@types/react` (frontend dev-deps) | nothing exercises them — the lint script is broken and there's no TypeScript | low impact; merge on green |

## Manual-run recipes (covered by neither CI nor a suite)

| Package · location | Trigger dep | Run it — from `gh pr checkout <n> --repo {REPO}` |
|---|---|---|
| frontend · `frontend/` | `near-api-js`, `@hot-labs/near-connect`, `function-call-key-plugin`, or a major of `next` / `react` | `cd frontend && pnpm install --frozen-lockfile && pnpm run dev` → open http://localhost:3000; confirm messages load, sign in with a testnet wallet, post one message without and one with a donation, sign out |
| contract · `contract-rs/` | a major of `near-sdk` | `cd contract-rs && cargo test`, then deploy to a throwaway testnet account (`cargo near create-dev-account`, `cargo near deploy <account>`) and call `add_message` / `get_messages` — never redeploy `guestbook.near-examples.testnet` from a triage run |

## Published vs not-shipped

Nothing is published to a registry. **Shipped:** the frontend, as a static site on GitHub Pages built from `main` (`nextjs.yml`) — a runtime vulnerability in a frontend `dependencies` package reaches visitors' browsers, so that is the high-priority case (server-side Next.js advisories mostly don't apply to a static export; check before prioritising). The contract is deployed separately (`guestbook.near-examples.testnet`, not by this repo's CI), so contract dependency advisories only matter once someone redeploys. **Not shipped:** `contract-rs/tests/`, all dev-dependencies, `.devcontainer/`.

## Merge policy

Required check: `ci-passed` (the aggregate job in `ci.yml`), once the branch ruleset is set up. No merge queue. Dependabot cooldown is 10 days for npm and 7 for cargo / github-actions (`dependabot.yml`); there is no pnpm `minimumReleaseAge`. `tests-rs.yml` also runs `cargo test` on contract PRs (ubuntu + macOS) — informational, not required. Because `Cargo.lock` is gitignored, cargo resolves fresh on every CI run: in-range patch/minor releases reach CI without a Dependabot PR, and cargo PRs only appear when a release falls outside the `Cargo.toml` range.

## Repo-specific gates

none
