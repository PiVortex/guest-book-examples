# guest-book-examples

A NEAR Protocol example: a Guest Book smart contract in Rust, plus a Next.js frontend that talks to a deployed instance of it. Users sign the guest book with a message; attaching ≥ 0.1 NEAR marks the message premium. The repo is a fork of `near-examples/guest-book-examples`, and its code is teaching material linked from docs.near.org — keep it minimal and idiomatic.

## Layout

- `contract-rs/` — the contract (near-sdk 5). `src/lib.rs` holds the contract and its unit tests; `tests/test_basics.rs` is a near-sandbox integration test. The toolchain is pinned in `rust-toolchain.toml` (1.97.1 + `wasm32-unknown-unknown`).
- `frontend/` — Next.js 16 (pages router), JavaScript/React, pnpm. `src/config.js` holds the network and contract id (`guestbook.near-examples.testnet`); `src/components/near-provider.jsx` wraps the wallet connector (`@hot-labs/near-connect`), the function-call access-key plugin, and the RPC view/call helpers exposed through `useNearWallet`. Builds as a static export to `frontend/build/`.
- `.github/workflows/` — `ci.yml` (PR CI; the aggregate `ci-passed` job is the required check), `tests-rs.yml` (contract tests on ubuntu + macOS), `nextjs.yml` (deploys the frontend to GitHub Pages on push to `main`), `claude-review.yml` (on-demand `/claude-review`), `issue-transfer.yml` (upstream's DevRel-project hook).
- `.claude/` — the PR command suite and this repo's rules (`project-specifics/`).

## Commands

Contract, from `contract-rs/` (needs [`cargo-near`](https://github.com/near/cargo-near), and `libudev-dev` on Linux):

```bash
cargo fmt --all
cargo clippy --all-targets --all-features -- -D warnings
cargo test          # unit tests + sandbox integration test (local sandbox only)
cargo near build    # build the wasm
```

Frontend, from `frontend/` (pnpm 10; CI uses Node 24):

```bash
pnpm install --frozen-lockfile
pnpm run dev        # http://localhost:3000
pnpm run build      # static export to build/
```

`pnpm run lint` is broken — it calls `next lint`, which Next.js 16 removed, and there is no ESLint config. Don't rely on it.

Before opening any PR, run the quality gate in `.claude/project-specifics/pr-quality-gate.md`.

## Invariants

- The contract interface (method names, arguments, the `PostedMessage` JSON shape) is mirrored in `frontend/src/pages/index.js`, `contract-rs/tests/test_basics.rs`, and the `contract-rs/README.md` snippets — change them together.
- Don't change the contract state layout or the `b"m"` storage prefix without a migration.
- The frontend's function-call access key is scoped to `add_message` with a 0.25 NEAR allowance — never broaden it.
- The frontend stays a static export (no SSR or API routes); its production `basePath` is `/guest-book-examples` for GitHub Pages.
- `pnpm-lock.yaml` is the only committed lockfile; `Cargo.lock`, `package-lock.json`, and `yarn.lock` are gitignored.

## Pull requests

- Only open a PR when asked. Follow `.claude/project-specifics/commit-conventions.md` for commit messages.
- When a PR makes a deliberate tradeoff, record it under a `## Design decisions / accepted tradeoffs` heading in the PR description — the reviewers (Claude and Copilot) treat everything listed there as settled and won't re-raise it.

## Installing dependencies

- Frontend: `pnpm install --frozen-lockfile` by default. Change dependencies only deliberately (`pnpm add` / `pnpm update <pkg>`), commit the regenerated `pnpm-lock.yaml`, and justify any new dependency in the PR. Never use npm or yarn here.
- Contract: edit `Cargo.toml` deliberately, and keep `near-sdk` on the same version in `[dependencies]` and `[dev-dependencies]`.

## Comments & documentation style

- The existing comments are tutorial-style explanations for readers learning NEAR — keep that tone where it already exists, but don't add comments that merely restate the code. Match the surrounding file's style.
- Keep READMEs succinct; when code they quote changes, update the snippet.
