# Copilot code review instructions for guest-book-examples

This repository is a NEAR Protocol example: a Guest Book smart contract in Rust
(`contract-rs/` — near-sdk 5, built with cargo-near, unit tests plus a near-sandbox
integration test) and a Next.js 16 static-export frontend in JavaScript/React
(`frontend/` — pnpm, near-api-js + @hot-labs/near-connect) that calls an
already-deployed testnet contract. It is teaching material linked from docs.near.org.
Apply the checks appropriate to the language of each changed file.

## What to review

Review the changed lines and their direct impact. Check for:

- **Security & safety:** Rust — `#[payable]` only where a deposit is intended;
  identity from `env::predecessor_account_id()`; no `unwrap`/panic or unchecked
  arithmetic on caller input; unbounded caller data written to storage not made
  worse; no keys outside sandbox-only test fixtures. JavaScript — the function-call
  access key's `methodNames` / `allowance` never broadened; deposits via
  `nearToYocto` (no float math on yoctoNEAR); no `dangerouslySetInnerHTML` for user
  text; no secrets committed; contract id and RPC endpoints only from `src/config.js`.
- **Architecture & patterns:** the contract interface (method/argument names,
  U64-as-string types, `PostedMessage` JSON shape) stays in sync across
  `contract-rs/src/lib.rs`, `contract-rs/tests/`, the `contract-rs/README.md`
  snippets, and `frontend/src/pages/index.js`; state layout and the `b"m"` storage
  prefix unchanged without a migration; wallet access through `useNearWallet`;
  `next.config.js` stays a static export matching the Pages deploy; example code
  kept minimal and idiomatic.
- **Bugs:** logic errors, off-by-one mistakes, missing error handling,
  incorrect return values, division by zero, unhandled edge cases.
- **Performance & production:** Rust — view iteration bounded by `limit`, no
  full-collection loads or clones of large state, release-profile settings
  preserved. JavaScript — no redundant RPC calls or polling, effects with correct
  dependencies and cleanup, optimistic UI rolled back on failure.

## Rules

- Only flag issues you are confident are real and introduced by this PR. Skip
  likely false positives, nitpicks, and pre-existing issues the change doesn't touch.
- Skip anything a linter or compiler already catches — CI enforces formatting and lint.
- For each issue, say why it matters and suggest a concrete fix.
- Comment only on what the diff affects; do not review unchanged code.
- Prioritize **blocking defects** (security, correctness, data loss, broken contracts).
  Treat style, naming, and debatable preferences as minor at most — don't block on them.
- If the PR description has a **"Design decisions / accepted tradeoffs"** section, treat
  every item there as settled and out of scope — do not flag or re-raise it.
