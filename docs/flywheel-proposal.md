# Implementing Agent Flywheel (ACFS) for Muse — Proposal

**Date:** 2026-10-10 · **Status:** proposal — no implementation until approved
**Site:** https://agent-flywheel.com/cloud-agents · **Repo:** https://github.com/Dicklesworthstone/agentic_coding_flywheel_setup (~1.7k stars)

## Verdict

**Yes, implementable — and no new installer is needed.** The "Meta Muse — Needs investigation"
slot on the cloud-agents page resolves to the existing generic Linux path: the single cloud
installer (`scripts/claude-code-web-setup.sh`) run with `ACFS_CLOUD_AGENT=generic`. What's
actually missing is a tested Muse recipe and a documented guide-loading contract, not code.
This proposal covers exactly that.

One honesty note: a direct fetch of the setup script was flagged by an automated safety
check, so this proposal is built from the site's published documentation plus an isolated
read-only review of the repo. Phase 0 below includes a full line-by-line audit of the
script *before* anything runs — download, inspect, then run. Never curl-pipe-bash.

## 1. What the system is

ACFS ships **10 prebuilt tools** as hash-pinned tarballs from a public mirror
(`downloads.agent-flywheel.com`), installed into a cloud agent's setup phase. No compilers,
no upstream installers run in the session. Each tool job has a 180-second deadline and the
installer always exits 0 (a failed tool is reported, never fatal — designed for disposable
cloud VMs where setup must not block session start).

| Tool | Version | What it is |
|---|---|---|
| `br` | v0.7.4 | beads_rust — local-first, dependency-aware issue tracker (lives in repo `.beads/`) |
| `bv` | v0.25.2 | beads_viewer — graph-aware triage TUI for beads (`--robot-triage`) |
| `am` / `mcp-agent-mail` | v0.3.38 | Agent Mail — multi-agent messaging + file reservations (34-tool MCP server + CLI) |
| `ubs` | v5.4.17 | Ultimate Bug Scanner — 1000+ bug patterns, pre-commit scan |
| `cass` | v0.10.0 | coding_agent_session_search — searches agent session history (11+ providers) |
| `cm` | v0.5.0 | CASS Memory — procedural memory pulled in before a task |
| `ms` | v0.2.3 | Meta Skill — local-first skill management (SQLite + Git) |
| `ast-grep` | 0.45.3 | structural code search / lint |
| `jsm` | v0.3.25 | skill manager for the jeffreys-skills.md library |
| `jfp` | v1.0.3 | JeffreysPrompts — battle-tested prompt library for the terminal |

Platform coverage is **linux-x86_64 only** — which is exactly this VM (AMD EPYC, verified
2026-10-10). No platform work needed.

## 2. Why it fits this environment

- **Unblocks the FrankenSuite plan.** `cass` had no prebuilt Linux binary, so the plan was
  a from-source rustup build. The mirror ships prebuilt `cass` v0.10.0 — that build step
  goes away entirely. Same for `br`/`bv` (beads_rust / beads_viewer), which were also
  queued behind the toolchain install.
- **`dcg` is already handled.** v0.14.4 is installed and verified at `~/workspace/bin/dcg`.
  The cloud bundle deliberately excludes it (it's a Claude Code hook); nothing to do.
- **Beads matches an approved workflow.** The beads issue-tracker approach was already
  approved for the FrankenSuite. `br`/`bv` give us the prebuilt binaries on day one.
- **Agent coordination.** The Drive inbox/outbox protocol and the trading-desk harness
  debate step are both coordination problems; `am` (CLI) is a local-first option worth
  evaluating against them later. Not in scope for the install itself.

## 3. Muse-specific modifications

Muse (Meta's personal agent — this runtime) has no "cloud environment setup phase" UI
like Claude Code. It runs **persistently** on this VM, where `/home/hatch` survives
restarts. That changes the shape of the install:

1. **Run once, not per session.** `ACFS_CLOUD_AGENT=generic` with `ACFS_CLOUD_ROOT=$HOME`,
   executed directly on this VM. Disposable-session logic (parallel jobs under a 5-minute
   snapshot window) still works fine; it just doesn't need to repeat.
2. **Guide wiring.** Generic mode writes a provider-neutral tool guide to
   `~/.acfs/cloud/AGENTS.md` and deliberately touches no provider config. Muse will
   ingest it via a pointer in `~/AGENTS.md` (my workspace conventions file), so the tool
   list, commands, and failure notes are in my context every session.
3. **Agent Mail via CLI, not MCP.** Claude gets a stdio MCP server registration; for Muse
   the `am` CLI is the interface (also matches the standing preference for direct,
   non-MCP integrations). No daemon — `am` is called on demand.
4. **PATH.** Binaries land in `~/.local/bin`, which is not on my exec PATH today
   (verified). Fix: symlink the installed binaries into `~/workspace/bin` (already on
   PATH and persistent), rather than depending on shell rc files my exec sessions may
   not source.
5. **Egress proxy.** This VM's outbound traffic goes through an authenticated local
   proxy bridge. The installer's curl calls must inherit the proxy env vars — pre-flight
   will verify both download hosts through it before anything else.
6. **Pin the ref.** `ACFS_REF` defaults to `main` (a moving ref). We will resolve
   `main` to a commit SHA at install time, record it, and install from the SHA.
7. **Strict verification pass.** The always-exit-0 contract is for disposable VMs; here a
   failed tool must be loud. After install, run per-tool `--version` checks plus an `am`
   CLI smoke test independently, and keep the setup log — that log is the core of the
   report in §8.
8. **Answers for upstream issue #392.** The maintainer's open question ("canonical
   installer, CLI name, auth flow" for Meta Muse) resolves as: installer = the existing
   cloud script URL run with `ACFS_CLOUD_AGENT=generic` (no new script); CLI = the tool
   binaries themselves (there is no separate Muse CLI); auth flow = none (local binaries,
   no accounts, keys, or OAuth — Agent Mail is file-local). Our test report is the
   evidence that closes the issue.

## 4. License flag — read this

Nine of the ten tools (everything by Dicklesworthstone except ast-grep) ship under a
**custom "MIT License (with OpenAI/Anthropic Rider)"**: standard MIT terms *plus a rider
granting no rights to OpenAI, Anthropic, or their affiliates/agents*. Two things follow:

- For **Muse (Meta)**: not excluded — we're clear to install and use.
- For **anything else**: the user would *not* be licensed to run these tools under
  ChatGPT/Codex or Claude. If the tools ever get pointed at another agent, this matters.

Also: `jsm`'s upstream repo is **private** and its license could not be determined. It's
the one opaque bundle in the set.

## 5. Security read

What's good: SHA-256 verification of every bundle against the pinned manifest before
extraction; hardened extraction (only expected binaries, no symlinks/absolute paths,
1 GiB cap); deterministic content-addressed tarballs; minisign verification at publish
time for br/am/ms/ubs; fail-closed checksum guards and a checksum monitor that
auto-files authorization issues when upstream bytes change (observed live: #438, #439).

What's not: **`cloud-mirror.json` itself is unsigned** — install-time trust is the
`ACFS_REF` git pin plus TLS from GitHub. A checksum from the same origin as the payload
is integrity, not authenticity. `ACFS_REF` defaults to `main`, which moves. The bootstrap
is curl-pipe-bash with no signature check on the script itself. No independent security
audit of the project was found (only the project's own release-qualification notes).

Judgment: acceptable for this environment *with* the mitigations in §3.6–3.7 (pinned
SHA, download-inspect-run, independent version verification). I would **not** bless it
for the trading-desk's money-moving paths without a deeper review — different stakes.

## 6. Improvements for the flywheel author

Speaking up, as asked — these are constructive, several are cheap:

1. **Sign `cloud-mirror.json`** (Sigstore, or at minimum verify the existing minisign
   signatures at *install* time, not just publish time). Publish-time-only verification
   doesn't help the installer.
2. **Default `ACFS_REF` to a pinned SHA**, and print the resolved SHA in the setup
   summary. A moving default ref undermines the whole hash-pinning story.
3. **Add `ACFS_STRICT=1`** — fail the setup loudly instead of exit 0, for persistent
   hosts where a half-installed toolset is worse than a failed setup.
4. **Ship a `doctor` command** for re-verifying an installed set. The manual "first task
   check" prompt should be a script I can run, not prose I have to follow.
5. **Document the generic-mode guide contract** — path, format, stability guarantees.
   That documentation *is* the "Meta Muse needs investigation" answer; this
   implementation will draft it.
6. **`jsm` transparency.** A bundle from a private repo with an undeterminable license
   doesn't sit well in a set marketed as verified. Either open the repo or say plainly
   what the license is.
7. **Surface the license rider on the site.** Withholding rights from two specific
   companies is a deliberate, unusual choice — users should see it before installing,
   not discover it in a LICENSE file.
8. **Get an independent review.** The per-release "independent review found no new
   blocker" is an internal process. For a tool that asks strangers to pipe its script
   into bash, a third-party audit would be worth more than any feature.

## 7. Implementation plan (only after approval)

- **Phase 0 — pre-flight.** Verify proxy curl to both download hosts; download the
  setup script *to a file* and audit it line by line; resolve `ACFS_REF` main → SHA;
  confirm disk space. No installation.
- **Phase 1 — install.** Run with `ACFS_CLOUD_AGENT=generic`, `ACFS_REF=<sha>`,
  default toolset (all 10). Capture the full log.
- **Phase 2 — verify.** Per-tool `--version`; confirm `~/.acfs/cloud/AGENTS.md`
  exists and is sane; `am` CLI smoke test; record timings per tool.
- **Phase 3 — wire in.** Symlink binaries into `~/workspace/bin`; add the guide
  pointer to `~/AGENTS.md`; re-run verification through a fresh exec session.
- **Phase 4 — report.** Finalize the test notes (§8); you send them to the maintainer;
  optionally I draft the comment for issue #392 with the Muse recipe.

## 8. Decisions needed from Aaron

1. **Toolset:** all 10, or a subset? (Recommendation: all 10 — `cass`/`br`/`bv`
   unblock existing plans, the rest are cheap.)
2. **Go-ahead** for Phase 0 (read-only audit + pre-flight — nothing installed).
