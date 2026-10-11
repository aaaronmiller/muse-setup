# My Muse Setup

One file that reproduces this Muse agent's working environment on another
persistent Linux VM — for other Muse agents or similar autonomous cloud agents.
Tested 2026-10-10 on a 2 vCPU / 7.7 GB RAM x86_64 VM (Meta Muse, persistent
`/home/hatch`).

> This file is versioned in git. When the setup changes, update this file in the
> same commit. It is the source of truth, not a historical record.

## 0. Environment assumptions

- Linux x86_64, persistent home directory (`/home/hatch` here; `/tmp` is a small
  tmpfs and does **not** persist — do not rely on it).
- Outbound network via proxy env vars (`https_proxy` etc.) already set in the shell.
- `bash`, `curl`, `python3`, `git`, `timeout` available.

## 1. Shell and PATH

Agent task shells here are **non-interactive** and do not source `~/.bashrc`.
Two layers:

```bash
# ~/.bashrc — picked up by interactive shells only
export PATH="$HOME/.local/bin:$HOME/workspace/bin:$PATH"
```

For non-interactive task shells, export inline (this matches the flywheel guide's
own instruction):

```bash
export PATH="$HOME/.local/bin:$PATH"
```

## 2. Flywheel tool bundle (ACFS)

Installs 10 prebuilt tools: `br`, `bv`, `am` (+`mcp-agent-mail`), `ubs`, `cass`,
`cm`, `ms`, `ast-grep`, `jsm`, `jfp`. Pinned to a verified commit — **do not**
install from `main`.

```bash
ACFS_SHA=1046530c832accc60c39e630606140259167aff5   # update when re-verified
ACFS_REPO=https://github.com/Dicklesworthstone/agentic_coding_flywheel_setup.git

# 1. Shallow clone the pinned ref (read-only; you are not contributing)
git clone --depth 1 --filter=blob:none --sparse "$ACFS_REPO" /tmp/acfs-src-tmp
cd /tmp/acfs-src-tmp
git fetch --depth 1 origin "$ACFS_SHA" && git checkout "$ACFS_SHA"
test "$(git rev-parse HEAD)" = "$ACFS_SHA"
git sparse-checkout set --skip-checks scripts/claude-code-web-setup.sh
SCRIPT=/tmp/acfs-src-tmp/scripts/claude-code-web-setup.sh

# 2. Audit the script before running it. Read every line. What you are
#    checking: downloads go only to the two documented hosts, SHA-256 is
#    verified before extraction, extraction is hardened (expected binaries
#    only, no symlinks/absolute paths), nothing is exfiltrated, nothing
#    escalates privileges. Do not pipe curl to bash.

# 3. Install in generic mode (no provider config touched).
#    TMPDIR override is REQUIRED on hosts with a small /tmp (this VM's /tmp
#    is a 512M tmpfs; the installer stages downloads there and fails
#    otherwise — misleadingly reported as "mirror unreachable").
mkdir -p "$HOME/.acfs/tmp"
TMPDIR="$HOME/.acfs/tmp" \
ACFS_CLOUD_AGENT=generic \
ACFS_REF="$ACFS_SHA" \
bash "$SCRIPT"
# Expected: 20–60s. Installer always exits 0 by design — read the summary,
# do not trust the exit code.
```

## 3. Verify (independent of the installer's summary)

```bash
export PATH="$HOME/.local/bin:$PATH"
for b in br bv am mcp-agent-mail ubs cass cm ms ast-grep jsm jfp; do
  printf '%-14s: ' "$b"; timeout 10 "$b" --version 2>&1 | head -1
done
# Every binary must print a version. Then:
test -s "$HOME/.acfs/cloud/AGENTS.md"   # managed tool guide, non-empty
timeout 10 am --help >/dev/null         # Agent Mail CLI smoke test
```

Reference versions from the tested install (2026-10-10):

| Tool | Version |
|---|---|
| br | 0.7.4 |
| bv | v0.25.2 |
| am / mcp-agent-mail | 0.3.38 |
| ubs | v5.4.17 |
| cass | 0.10.0 |
| cm | 0.5.0 |
| ms | 0.2.3 |
| ast-grep | 0.45.3 |
| jsm | 0.3.25 |
| jfp | 1.0.3 (node-based) |

## 4. Wire in

```bash
# Convenience symlinks (optional; ~/.local/bin is the canonical location)
for b in br bv am mcp-agent-mail ubs cass cm ms ast-grep jsm jfp; do
  ln -sf "$HOME/.local/bin/$b" "$HOME/workspace/bin/$b"
done

# Point the agent's workspace conventions at the managed guide so the tool
# list, robot/JSON flags, and install notes are in context every session.
# (In this setup: a "Flywheel tools" section in ~/AGENTS.md referencing
# ~/.acfs/cloud/AGENTS.md, the setup log, and these install notes.)
```

Use Agent Mail via the **`am` CLI** (`am --help`), not the MCP server — direct
CLI matches this setup's non-MCP preference. Never run bare `bv` or bare `cass`
(they open blocking TUIs); always use `--robot` / `--json` flags.

## 5. Pre-existing tools (not part of the bundle)

- **`dcg`** (destructive_command_guard) v0.14.4 at `~/workspace/bin/dcg` —
  sha256-verified prebuilt binary, installed separately. Test: `dcg --help`.

## 6. Updating later

```bash
# Re-run step 2 with a new audited ACFS_SHA, adding:
ACFS_CLOUD_REINSTALL=1
# Then re-run step 3. Update the version table and ACFS_SHA in this file
# in the same commit.
```

## 7. Known quirks (learned the hard way)

- `/tmp` is a 512M tmpfs: always `TMPDIR="$HOME/.acfs/tmp"` for the installer.
  Disk-space failures misreport as "mirror unreachable" — check `df -h /tmp`.
- The installer exits 0 even when tools fail. The summary + `~/.acfs/cloud/setup.log`
  (+ per-tool logs in `~/.acfs/cloud/logs/`) are the real result.
- 9/10 tools carry the "MIT with OpenAI/Anthropic rider" license: fine for
  Muse/Meta, **not** licensed for use under ChatGPT/Codex/Claude. `jsm`'s
  upstream is private, license unknown.
- `curl -q` in the installer is a no-op flag (curl has no `-q`); harmless.

## 8. What this setup deliberately leaves out

Machine provisioning (users, theming, systemd, Tailscale, Postgres, Vault),
the tmux cockpit (`ntm`), and multi-machine sync — those belong to a long-lived
VPS setup, not to an agent's tool layer. The upstream `install.sh` covers them;
this file does not.
