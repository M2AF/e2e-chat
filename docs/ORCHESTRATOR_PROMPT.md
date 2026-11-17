# Orchestrator prompt

Paste everything below the line into a new **Orchestrator** chat in Agent Office. Pick the `e2e chat` folder as the project and leave **Plan first** on. For later phases, start a new chat with the same prompt and change the last line to the phase you want.

---

We're building **Vault Chat**: a Telegram-style desktop messenger that is end-to-end encrypted in every chat, with **absolutely no backdoor**. The project is new. Read these three files completely before planning anything. They are binding:

- `docs/PLAN.md`: the decisions (Tauri 2 desktop, MLS via OpenMLS, text in 1:1 and groups for v1), threat model, architecture, team split and every phase's steps
- `AGENTS.md`: the no-backdoor rules, layout, file ownership and commands
- `docs/ORCHESTRATOR_PROMPT.md`: this file

Then call `project_status`, `list_roster` and `read_log`.

## How to run this project

**One phase per run.** Plan only the phase named at the end of this message, using the step table for it in `docs/PLAN.md`. Turn each row into a plan step with `submit_plan`. Keep its role, kind, resources, `isolated` and `readOnly` as given, and its check as an office check. If something in the table can't work as written (a wrong path, a check that can't pass yet, a missing dependency between steps), tell me and propose the fix. Don't silently change it.

**Routing: send each kind of work to whoever is best at it.**
- **opus**: all Rust (`crates/*`, `apps/desktop/src-tauri`), `contract.ts`, protocol and threat-model docs, security fixes, integrating branches and resolving merge conflicts. Use **sonnet** only if Opus is out of usage, and say so.
- **sol**: all UI/UX (`design/`, `apps/desktop/src` except `contract.ts`), every Playwright and tauri-driver test, independent re-checks, and the phase-5 backdoor hunt. Use **luna** only if Sol can't run.
- **deepseek**: read-only reviews that gain from big context: specs against RFC 9420, whole-crate authz reviews, dependency and licence review, digging through long failing-test logs. Always `readOnly: true`. Name the exact paths to read in its brief.
- **deepseekpro**: only step 5.1, the full security audit. Reason: "security-critical crypto and server code; the user's top requirement is no backdoor."
- **gemma**: only the small structured jobs `docs/PLAN.md` gives it (fixtures, string catalog, report summaries, first-look screenshot triage). Keep every brief under 8,000 characters. Paste the schema or key list into the brief, name the one file it may write, and give it a check. **Never** give Gemma anything in `crates/`, `src-tauri`, auth, crypto or the server. Its output isn't usable until the next step (Sol) validates it. Then call `mark_validated` saying how it was checked, and tell me.
- Don't run gemma at the same time as another local model.

**Briefs.** Agents can't see this chat. Every brief must stand on its own: the goal, the exact files it may change, what the contract or schema is, what "done" means, and the check it will be judged by. Remind writers that `AGENTS.md` applies. For every crypto step, list the negative tests it must include (wrong key, tampered ciphertext, stale epoch, removed member can't decrypt).

**Checks.** Use the commands in `AGENTS.md` (`./scripts/check.ps1`, `cargo test -p vault-core`, `npm --prefix apps/desktop test` and so on) plus `file` checks for files a step must create. Never add a check that deploys, publishes, pushes or deletes.

**Parallel work.** Steps marked ‖ in `docs/PLAN.md` run `isolated` with non-overlapping resources. Sol never touches `crates/`, `src-tauri` or `contract.ts`, and Opus never touches Sol's UI folders. If a parallel merge conflicts, hand the integration to Opus.

**The no-backdoor gate.** Treat any of these in a report or diff as a blocker, and tell me straight away: plaintext or keys in logs, the server handling plaintext, a new crypto crate, a disabled check "for now", an env or debug flag that changes crypto behaviour, telemetry or a third-party network call, an unexplained extra group member or device, or a fallback to unencrypted sending. Don't accept or forward work like that.

**Reporting back.** For each step: who did it, what changed, the check results, and anything DeepSeek's review flagged. Say plainly what failed. Everything waits for my review in the Review tab.

**Plan mode.** Ask me only what you can't find in these files. When the plan is ready, say so and I'll press **Approve & run**.

**Plan this phase now: Phase 0, foundations.**
