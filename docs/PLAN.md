# Vault Chat: build plan 

"Vault Chat" is a working name. This file covers what we're building, the security promises it makes, the architecture, who on the Agent Office team does what, and the phases.

## 1. Decisions (made by the user, 2026-11-17)

| Decision | Choice | Why |
| --- | --- | --- |
| Platform | **Tauri 2 desktop** (Windows first). Tauri mobile later | A signed app can't be swapped for a backdoored version on each load the way a web page can |
| Protocol | **MLS (RFC 9420) via OpenMLS**, MIT licence | An IETF standard built for groups, with forward secrecy and post-compromise security |
| v1 scope | **Text in 1:1 and group chats**, contacts, safety numbers, multiple devices, read receipts | Encrypted files and attachments come in v2 |
| Ciphersuite | `MLS_128_DHKEMX25519_AES128GCM_SHA256_Ed25519` (0x0001) with `openmls_rust_crypto` | Pure Rust, so there's no OpenSSL to build on Windows |
| Server | Rust: axum + tokio + sqlx on **SQLite** (Postgres-ready queries) | One language from end to end; v1 needs no extra infrastructure |
| UI | React + TypeScript + Vite inside Tauri | Codex's strongest UI stack |

## 2. Security promises and threat model

**Promises** (enforced by `AGENTS.md` and tested in phase 5):
- Only the sender's and recipients' devices can read a message. The server, its operator, the database and the network can't.
- No key escrow, admin decrypt path, plaintext logging or telemetry.
- Forward secrecy: a key stolen today doesn't decrypt yesterday's messages. Post-compromise security: after a commit, a stolen key stops working.
- Identity key changes are always shown to the user.

**What the server still learns (v1, stated honestly):** account usernames, which devices exist, who is in which group (MLS routing needs it), message sizes and timing, and IP addresses. Sealed sender and metadata hiding are v2 items, listed in §7.

**Out of scope:** a compromised endpoint (malware on the user's PC), and users who never compare safety numbers when the server lies about keys. Key transparency for that case is v2.

## 3. Architecture

```
apps/desktop (Tauri 2)
 ├─ src/            React UI ── invoke() ──┐   plaintext only for display
 │   └─ api/contract.ts  (command types)   │
 └─ src-tauri/      Tauri commands ◄───────┘
        │ uses
crates/core   identity keys · OpenMLS groups · KeyPackages · safety numbers
              encrypted local store (SQLCipher, key in the OS keychain via `keyring`)
        │ crates/proto (versioned wire types)
        ▼  HTTPS + WebSocket (TLS 1.3, rustls)
crates/server  accounts · device auth (signed challenge) · KeyPackage directory
               MLS delivery service (per-group epoch ordering) · per-device queues
               SQLite: public keys + ciphertext + routing only
```

**Key flows**
- **Sign-up:** the device makes an Ed25519 identity key and an MLS credential, registers a username with its public keys, and uploads a batch of single-use KeyPackages.
- **Auth:** the server sends a nonce, the device signs it, and the server issues a short-lived session token. There are no passwords and no phone numbers.
- **1:1 chat:** a 2-member MLS group. Groups are the same with more members.
- **Sending:** the client encrypts an MLS application message, the server queues it for every member device, devices fetch over WebSocket and ack, and the server deletes the message once all devices have acked (or after 30 days).
- **Ordering:** the server accepts a commit only for the group's current epoch and rejects stale ones (409), so the client fetches, merges and retries.
- **Multiple devices:** each device is its own MLS leaf. To link a new device, the user scans a QR code from an existing device and compares a code. The existing device signs the new one into the user's device list and adds it to every group.
- **Safety numbers:** a fingerprint over both users' identity keys, shown as digits and a QR code. Verified state is stored locally, and a key change raises a warning and clears "verified".

## 4. Team: who does what

| Member | Strong at | Gets on this project | Never gets |
| --- | --- | --- | --- |
| **Opus** (Claude) | Hard code, architecture, crypto integration, debugging, merging | `crates/*`, `src-tauri`, the UI contract, protocol and threat-model docs, security fixes, release | UI styling |
| **Sonnet** (Claude) | Routine code | Only when Opus is out of usage, with a reason | Crypto design |
| **Codex Sol** | UI/UX, visual design, browser tests, independent re-checks | `design/`, the React UI, Playwright tests on the mock adapter, tauri-driver E2E smoke tests, the "prove there's no plaintext" tests, screenshot reviews | Rust crypto, releases |
| **DeepSeek Flash** | Reading huge context: whole repos, diffs, logs | Spec review against RFC 9420, whole-repo consistency reviews, failing-test log digging, dependency and licence review | Writing crypto, secrets (it runs on DeepSeek's servers) |
| **DeepSeek Pro** | The hardest security reviews | The phase-5 crypto and backdoor audit of `crates/core`, `crates/server` and `src-tauri` (reason: security-critical code) | Anything Flash can do |
| **Gemma** (local) | Structured extraction, small repetitive jobs, a first look at screenshots | UI string catalog, mock fixture data, test-report summaries, first-pass screenshot triage, changelog drafts. **Always validated by another agent** | Crypto, server, auth, anything security-sensitive, briefs over 8,000 characters |

Every change set also gets DeepSeek's automatic read in the Review tab. Nothing lands until you accept it.

> Source code goes to DeepSeek's API for its reviews. No message content, keys or secrets ever exist in the repo, and the office redacts anything that looks like a secret. Untick DeepSeek steps if you'd rather keep the code local.

## 5. Phases

Each phase is one Orchestrator run: plan, you approve, it runs, you review. Every step is listed as `role · kind · resources · check`.

### Phase 0: foundations
| # | Step | Who | Kind | Resources | Check |
| --- | --- | --- | --- | --- | --- |
| 0.1 | Monorepo scaffold: Cargo workspace (packages `vault-proto`, `vault-core`, `vault-server`; Tauri crate `vault-desktop`. Never name a package `core`, which clashes with Rust's own), Tauri 2 + React/TS/Vite app, ESLint, Vitest, `scripts/check.ps1`, `.gitignore` (`data/`, `target/`, `node_modules/`) | opus | code | `Cargo.toml`, `crates`, `apps/desktop`, `scripts` | `./scripts/check.ps1` |
| 0.2 | `docs/PROTOCOL.md` (wire format, every flow in §3, error codes) + `docs/THREAT_MODEL.md` + `apps/desktop/src/api/contract.ts` (every Tauri command and event, typed) | opus | docs | `docs/PROTOCOL.md`, `docs/THREAT_MODEL.md`, `apps/desktop/src/api/contract.ts` | files exist; `npm --prefix apps/desktop run typecheck` |
| 0.3 | Style guide, tokens and wireframes for every v1 screen (‖ with 0.1–0.2, isolated) | sol | art | `design` | `design/STYLE.md`, `design/tokens.css` exist |
| 0.4 | Review 0.2 against RFC 9420 concepts and AGENTS.md promises: gaps, ambiguities, metadata leaks (read-only) | deepseek | review | — | report |
| 0.5 | Fold in 0.4's accepted findings | opus | docs | 0.2's files | typecheck |

### Phase 1: crypto core and UI shell (two parallel tracks)
| # | Step | Who | Kind | Resources | Check |
| --- | --- | --- | --- | --- | --- |
| 1.1 | `crates/proto` wire types + `crates/core`: identity, KeyPackages, create/add/remove/update groups, encrypt/decrypt, commit handling, safety numbers, SQLCipher store. Tests: 2- and 3-member groups, removed member can't decrypt, tampered ciphertext rejected, stale epoch rejected, store unreadable without key (isolated) | opus | code | `crates/proto`, `crates/core` | `cargo test -p vault-core -p vault-proto` + clippy |
| 1.2 | UI against a **mock adapter** implementing `contract.ts`: onboarding, chat list, chat view, new chat/group, group info, safety-number screen, key-change warning, device linking, settings. Light and dark, 1280 and 900 px wide (isolated, ‖ 1.1) | sol | code | `apps/desktop/src/ui`, `apps/desktop/src/api/mock.ts`, `apps/desktop/src/styles`, `apps/desktop/src/main.tsx` | typecheck + lint + `npm --prefix apps/desktop test` |
| 1.3 | Mock fixtures: 6 fake users, 4 chats, 40 messages, as JSON matching a schema in the brief | gemma | code | `apps/desktop/src/mocks/fixtures.json` | a Vitest schema test written in 1.2 |
| 1.4 | Validate 1.3: run the test, read the data, wire it into the mock adapter (Gemma's result isn't trusted yet) | sol | test | `apps/desktop/src/api/mock.ts` | `npm --prefix apps/desktop test` |
| 1.5 | Playwright tests on the mock UI for every screen + screenshots in `apps/desktop/e2e/screenshots` | sol | test | `apps/desktop/e2e` | `npm --prefix apps/desktop run e2e` |

### Phase 2: server
| # | Step | Who | Kind | Resources | Check |
| --- | --- | --- | --- | --- | --- |
| 2.1 | axum server: register, challenge auth, username lookup, KeyPackage upload and claim (single use), group message submit with epoch ordering, per-device queue, fetch and ack, WebSocket notify, rate limits, size limits, 30-day expiry. sqlx migrations. No plaintext anywhere | opus | code | `crates/server` | `cargo test -p vault-server` |
| 2.2 | Integration test: start a real server and 3 `vault-core` clients, then sign up, make a group, chat, remove a member, rotate keys. **Canary test:** send `CANARY-7f3a…`, then assert that string isn't in the SQLite file, the server logs or captured responses | opus | test | `crates/server/tests`, `crates/core/tests` | `cargo test --workspace` |
| 2.3 | Read 2.1–2.2 for missing authz checks (can user A fetch B's queue? claim unlimited KeyPackages? post to a group they're not in?) (read-only) | deepseek | review | — | report → Opus fixes |

### Phase 3: bridge and real app
| # | Step | Who | Kind | Resources | Check |
| --- | --- | --- | --- | --- | --- |
| 3.1 | `src-tauri`: implement every `contract.ts` command over `core`. Keychain-held DB key, background sync task, events to the UI. Lock down Tauri capabilities (no shell or fs plugins for the UI), strict CSP, no remote URLs | opus | code | `apps/desktop/src-tauri` | `cargo test -p vault-desktop`, clippy |
| 3.2 | `apps/desktop/src/api/tauri.ts`, the real adapter, plus a switch between mock and real (mock only in dev/test builds) | sol | code | `apps/desktop/src/api/tauri.ts`, `apps/desktop/src/api/index.ts` | typecheck + tests |
| 3.3 | End-to-end smoke with tauri-driver + WebdriverIO: server + two app instances (separate data dirs), sign up both, verify safety numbers, chat both ways, make a group with a third | sol | test | `apps/desktop/e2e-tauri` | `npm --prefix apps/desktop run e2e:tauri` |
| 3.4 | Summarise the 3.3 test report into a pass/fail checklist (read-only), then Sol checks it against the raw report | gemma → sol | review | — | validated |

### Phase 4: UX polish
| # | Step | Who | Kind | Resources | Check |
| --- | --- | --- | --- | --- | --- |
| 4.1 | First-pass triage of phase 1/3 screenshots against `design/STYLE.md`: list visual issues (read-only) | gemma | review | — | Sol validates in 4.2 |
| 4.2 | Fix the real issues: empty states, loading and error states, offline banner, key-change and decryption-failure UX, keyboard shortcuts, accessibility (focus, contrast, ARIA) | sol | code | `apps/desktop/src/ui`, `apps/desktop/src/styles` | e2e + screenshot checks |
| 4.3 | String catalog `apps/desktop/src/i18n/en.json` from a key list given in the brief; Sol then swaps literals for keys | gemma → sol | code | `apps/desktop/src/i18n/en.json` | Vitest catalog-keys test |

### Phase 5: security audit ("no backdoor" proof)
| # | Step | Who | Kind | Resources | Check |
| --- | --- | --- | --- | --- | --- |
| 5.1 | Full audit of `crates/`, `src-tauri`, `contract.ts`, Tauri config: misuse of OpenMLS, missing epoch or credential checks, key or plaintext reaching logs or disk unencrypted, any hidden path or flag that weakens crypto, authz holes. File, line, severity (read-only) | **deepseekpro** (reason: security-critical crypto) | review | — | report |
| 5.2 | Dependency and licence review: `Cargo.lock`, `package-lock.json`, `cargo deny` / `cargo audit` / `npm audit` output. Flag telemetry or network-calling packages (read-only) | deepseek | review | — | report |
| 5.3 | Fix every confirmed finding, each with a regression test. Add `cargo-deny` config and `scripts/audit.ps1` | opus | bugfix | `project` | `./scripts/check.ps1` + `./scripts/audit.ps1` |
| 5.4 | Independent backdoor hunt: run the server + 2 apps, send canaries, then grep the DB, logs, app data dir and a network capture for plaintext. Confirm the app only connects to the configured server. Search the source for decrypt or log paths (read-only) | sol | test | — | report with evidence |

### Phase 6: release (optional, needs a release recipe)
Windows installer build, code-signing guide, `docs/REPRODUCIBLE_BUILD.md`, signed Tauri updater config, server Dockerfile and deploy notes. Opus only, after you accept.

## 6. Before phase 0 (user)

1. Install **Rust** (rustup, stable MSVC), **Node 22+**, and the **Tauri prerequisites** for Windows (MSVC Build Tools; WebView2 ships with Windows 11).
2. Make the folder a git repo with one commit. The office's writers need it:
   `git init`, then `git add -A`, then `git commit -m "Plan and agent rules"`.
3. In Agent Office, open the **Chat** tab, pick this folder and **Set up bank**. It keeps this `AGENTS.md` and adds `CLAUDE.md`. Optionally add the `playwright-testing`, `visual-Iteration` and `architecture-review` skills under **Project bank** and commit them.
4. Paste `docs/ORCHESTRATOR_PROMPT.md` into a new Orchestrator chat with **Plan first** on.

## 7. v2 backlog

Encrypted files and images (client-side AES-GCM blobs, keys inside the MLS message), sealed sender, key transparency log, Tauri Android/iOS, voice notes, disappearing messages, encrypted backups with a user passphrase (Argon2id; never server-recoverable), Postgres deployment.
