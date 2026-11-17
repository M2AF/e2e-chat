# Vault Chat: rules for every agent

An end-to-end encrypted desktop messenger: Telegram-style chats and groups, but encrypted end to end in every chat. Read `docs/PLAN.md` before you start any work.

## Non-negotiable: no backdoors

1. **The server never sees plaintext.** It stores and forwards opaque MLS ciphertext, public keys and the routing metadata it needs. It holds no message keys, decryption code or "lawful access" path.
2. **No key escrow.** Private keys are created on the device and never leave it unencrypted. Nothing is uploaded so that someone else can recover a key.
3. **No hidden code paths.** No admin decrypt endpoint, debug mode that logs plaintext or keys, environment flag that weakens crypto, or hard-coded extra group member or device.
4. **We don't write our own crypto.** All protocol crypto comes from OpenMLS (`openmls`, `openmls_rust_crypto`). Primitives outside MLS (local database key, safety numbers) come from audited crates named in `docs/PROTOCOL.md`. Adding any other crypto crate needs Opus and a DeepSeek Pro review.
5. **Crypto stays in Rust.** The React UI never touches keys or ciphertext. It calls Tauri commands and gets plaintext only to display it.
6. **No telemetry, analytics, crash reporters, ad SDKs or third-party CDNs.** The app talks only to the chat server the user configured.
7. **Never log** plaintext, keys, key packages, ciphertext bodies or auth tokens, on the client or the server. Log ids, sizes and error kinds instead.
8. **Key changes are visible.** When a contact's identity key changes, the user is warned, and verification states are never silently reset.
9. **Fail closed.** If verification, decryption or epoch checks fail, show an error and drop the message. Never fall back to sending unencrypted.

If a task seems to need any of the above broken, stop and say so in your report. Don't work around it.

## Layout and ownership

| Path | What | Owner |
| --- | --- | --- |
| `crates/proto` | Wire types shared by the client and server (serde, versioned) | Claude (Opus) |
| `crates/core` | Client crypto core: identity, OpenMLS groups, encrypted local store, safety numbers | Claude (Opus) |
| `crates/server` | Relay and delivery service (axum, sqlx with SQLite) | Claude (Opus) |
| `apps/desktop/src-tauri` | Tauri 2 shell and commands, the bridge to `crates/core` | Claude (Opus) |
| `apps/desktop/src/api/contract.ts` | The UI ↔ Tauri command contract. Only Opus changes it | Claude (Opus) |
| `apps/desktop/src` (everything else) | React + TypeScript UI | Codex (Sol) |
| `design/` | Style guide, tokens, wireframes | Codex (Sol) |
| `apps/desktop/e2e`, `apps/desktop/tests` | UI and end-to-end tests | Codex (Sol) |

## Commands

- Everything: `./scripts/check.ps1` (fmt, clippy `-D warnings`, cargo tests, UI typecheck, lint and tests). It must pass before a step is done.
- Rust: `cargo test --workspace`, `cargo clippy --workspace --all-targets -- -D warnings`
- UI: `npm --prefix apps/desktop run typecheck`, `npm --prefix apps/desktop run lint`, `npm --prefix apps/desktop test`
- Run the server locally: `cargo run -p vault-server` (SQLite file in `./data/`, which git ignores)
- Run the app: `npm --prefix apps/desktop run tauri dev`

## Working rules

- Make the smallest change that does the task. Don't add dependencies the brief doesn't mention without saying why in your report.
- Every new behaviour in `crates/` gets a test. Every crypto behaviour gets a negative test too: wrong key, tampered ciphertext, stale epoch, removed member.
- Pin versions and commit lockfiles (`Cargo.lock`, `package-lock.json`).
- Report what you changed, what you ran and what passed. Never claim more than you verified.
