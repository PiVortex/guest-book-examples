**Steps that must be completed before an agent can create a PR**

# PR-to-`main` quality gate — guest-book-examples

Run this before committing and opening a PR into `main`. Work through it top to bottom.

## Hard rules (read first)

1. **Only run the checks for the area(s) your diff actually touched.** The contract and the frontend are independent builds.
2. **Run each area's commands from its own directory** (`contract-rs/` or `frontend/`) — there is no root workspace or root `package.json`.
3. **Contract prerequisites.** The toolchain is pinned by `contract-rs/rust-toolchain.toml` (1.97.1 with `rustfmt`, `clippy`, and the `wasm32-unknown-unknown` target; rustup installs it on first use). The [`cargo-near`](https://github.com/near/cargo-near) CLI must be on `PATH` — the sandbox integration test builds the wasm through it. On Linux, `libudev-dev` must be installed (CI runs `sudo apt-get install --assume-yes libudev-dev`).
4. **The frontend uses pnpm 10.** Install with `pnpm install --frozen-lockfile` — never `npm` or `yarn` (their lockfiles are gitignored and would drift from `pnpm-lock.yaml`). Run a non-frozen install only when you are deliberately changing dependencies, and commit the updated `pnpm-lock.yaml`.
5. **Never touch a live network pre-PR.** `cargo test` runs against a local near-sandbox only. Do not run `cargo near deploy`, `cargo near create-dev-account`, or anything that signs a transaction on testnet or mainnet.
6. **Don't run `pnpm run lint`.** It calls `next lint`, which Next.js 16 removed, and the frontend has no ESLint config — it fails on every tree and is not part of the gate.
7. **Don't commit a `Cargo.lock`** — it is gitignored by design.

---

## Step 1: Determine which area(s) changed

From the repo root, list changed files against the base branch:

```bash
git diff --name-only origin/main...HEAD
```

Map each changed path to an area by its top-level directory:

| Path prefix | Area | Gate to run (Step 2) |
|---|---|---|
| `contract-rs/` | Contract | Contract gate |
| `frontend/` | Frontend | Frontend gate |
| `.github/workflows/` | CI | Both gates — `ci.yml` runs every job when a workflow changes |
| `README.md`, `.devcontainer/`, `.claude/`, other `.github/` files, other root files | Docs / infra | None — review only |

Dependency edge: a change to the contract's public interface (method names, arguments, the `PostedMessage` shape) must also update the frontend calls in `frontend/src/pages/index.js`, the sandbox test in `contract-rs/tests/test_basics.rs`, and the `contract-rs/README.md` snippets — run both gates. The frontend talks to the already-deployed `guestbook.near-examples.testnet` contract (`frontend/src/config.js`), so the frontend build cannot catch an interface mismatch; check it by reading.

---

## Step 2: Run the gate for each changed area

### Contract gate

From `contract-rs/`:

```bash
cargo test
```

Runs the unit tests in `src/lib.rs` and the near-sandbox integration test in `tests/test_basics.rs`: it builds the wasm via cargo-near, starts a local sandbox (the first run downloads the sandbox binary), deploys the contract, and exercises `add_message` / `get_messages` with and without a deposit. It covers the contract end to end; it does not exercise the frontend.

### Frontend gate

From `frontend/`:

```bash
pnpm install --frozen-lockfile
pnpm run build
```

`pnpm run build` runs `next build` and writes the static export to `frontend/build/` — the same build the GitHub Pages deploy (`nextjs.yml`) ships. There are no frontend tests, so the build is the only automated check: manually exercise any UI change (`pnpm run dev`, sign in with a testnet wallet) and say so in the PR description.

---

## Step 3: Run formatting and checks (always)

Run the repository's formatters and static checks in the areas which changed, and fix anything they flag:

From `contract-rs/` (when the contract changed):

```bash
cargo fmt --all
cargo clippy --all-targets --all-features -- -D warnings   # fix everything it reports; `cargo clippy --fix` applies the mechanical ones
```

The frontend has no formatter or linter configured — match the surrounding file's style.

These are the **fixing** flavor — they rewrite files and surface problems so you fix them here, before pushing. CI runs the **check** counterpart of the same tools (e.g. `cargo fmt --check`, `npx prettier --check .`, `cargo clippy -- -D warnings`) and fails the PR on anything left unaddressed, so a clean run here is what makes CI green. This gate may also include extra checks CI does not run (see the rest of this file); CI is the enforcement subset, this gate is the superset.

---

## Step 4: Definition-of-done checklist

A change is **not** ready for `main` until all of the following are true:

- [ ] Changed areas identified (Step 1) and each one's gate run (Step 2) — every gate green
- [ ] Contract interface changes mirrored in the frontend calls, the sandbox test, and the `contract-rs/README.md` snippets
- [ ] No change to the contract state layout or the `b"m"` storage prefix without a migration, called out in the PR description
- [ ] Frontend UI changes exercised manually in `pnpm run dev`, stated in the PR description
- [ ] No `Cargo.lock`, `package-lock.json`, or `yarn.lock` committed; `pnpm-lock.yaml` changed only for an intentional dependency change

Universal items (every repo):

- [ ] Formatting and checks (Step 3) run clean
- [ ] Every added or changed code path has unit tests: happy path, edge cases, malicious inputs
- [ ] Bug fixes include a regression test that fails on the unfixed code
- [ ] No secrets (keys, tokens, credentials, customer data) anywhere in the diff
- [ ] Any new dependency is justified in the PR description
