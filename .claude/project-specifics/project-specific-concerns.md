**Most important concerns specific to this project**

- **Teaching-quality example.** This is a canonical NEAR example linked from docs.near.org — code and READMEs are teaching material. Keep changes minimal, idiomatic, and readable; prefer the obvious near-sdk / near-api-js pattern over a clever one. The `contract-rs/README.md` quotes `add_message` and `get_messages` — keep those snippets in sync with `src/lib.rs`.
- **Contract ↔ frontend interface sync.** Method names, argument names and types (`from_index` / `limit` as U64 strings), the `PostedMessage` JSON shape (`premium`, `sender`, `text`), the access key's `methodNames: ['add_message']`, and the sandbox test all change together. The frontend calls an already-deployed testnet contract (`frontend/src/config.js`), so a contract change does not reach the frontend until that contract is redeployed — say so in the PR when it matters.
- **State and storage compatibility.** Changing the `#[near(contract_state)]` struct, the Borsh shape of `PostedMessage`, or the `Vector` storage prefix `b"m"` breaks already-deployed state and needs an explicit migration.
- **Funds and storage.** The attached deposit decides `premium` (≥ `POINT_ONE` = 0.1 NEAR) and stays with the contract. Message `text` is unbounded and posters don't pay for the storage they consume — an accepted simplification for an example, but never make it worse (no new unbounded caller-controlled writes, no new payable paths that lose track of deposits).
- **Frontend key scope.** The function-call access key is scoped to `add_message` with a 0.25 NEAR allowance (`near-provider.jsx`). Never broaden its method names or allowance, and never commit keys, seed phrases, or private RPC credentials.
- **Static export.** `next.config.js` builds a static export (`output: "export"`, `distDir: 'build'`, production `basePath` `/guest-book-examples`) for GitHub Pages — no SSR, API routes, or other server-only Next.js features.

---

## Universal rules 

These apply to all work in this repository, on top of the project-specific concerns above:

- **Good test coverage, always.** Every code path added or changed gets unit tests covering the happy path, edge cases, all flows through the change, and malicious or hostile inputs. Classify each input by **trust boundary** — it decides bug vs hardening:
  - **Untrusted** (caller-controlled: request bodies / RPC responses, user and CLI input, file and config fields, env-derived values, deserialized or network payloads) — attack it hard; a break is a **real bug** and needs a test that fails on the unfixed code. Probe boundary/cardinality (zero, one, many, max, empty, off-by-one), malformed input (wrong type, `null`/`undefined`, bad encoding, `NaN`/`Infinity`), numeric edges (overflow, precision, negative/zero), and concurrency/replay (same effect twice, out-of-order, retry mid-flight).
  - **Internal contract** (a typed value your own code produced and already validated upstream) — a hostile value is usually unreachable, so a break is **hardening, not a live bug**; cover it only when you can show a real delivery path (an unvalidated cast, an unchecked cross-boundary field).
- **Regression tests for every bug fix.** Reproduce the bug in a test that fails on the old code for the right reason (the fix — not a typo or bad import — is what turns it green); the test stays in the suite forever.
- **Never commit secrets.** No keys, tokens, credentials, or customer data in code, config, tests, fixtures, or logs — use environment variables or secret stores.
- **Justify new dependencies.** Prefer the standard library or dependencies already in the repo; adding a new one requires a stated reason (capability, size, maintenance, security surface) in the PR.
