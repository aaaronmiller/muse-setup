# Operating with the human outside the loop

Living notes on running autonomous agent loops effectively when the human
(Aaron) has stepped back and will only re-engage at checkpoints or on demand.
Started 2026-10-13 during the SmileDraft WP-00 iteration loop. Updated as
lessons accumulate.

## What works

- **Human-near-the-loop, not human-in-the-loop.** Run the full
  detect → investigate → fix → measure cycle autonomously. Bring the human
  ONE crisp decision at the irreversible boundary (e.g. "SAM 3 on your 4050,
  finetune, re-scope, or stop — your call"). Mid-task questions are the
  failure mode; end-of-loop approvals are the pattern.
- **Checkpoint during, not after.** Memory notes, todos, and commits land at
  every milestone, not at conversation end. A session that dies mid-flight
  resumes from state, not from zero. The durable surfaces (memory, ledger,
  tracking, git) are the handoff mechanism between sessions and agents.
- **Replay safety declared up front.** Every delegation brief states what is
  safe to retry and what must never repeat, plus halt conditions. This is
  what makes "run aggressively" safe: the blast radius is bounded before
  the loop starts.
- **Fork, don't ask.** Uncertain approaches go to branches/spikes. Cheaper to
  try, measure, and show than to ask. The branch-per-experiment-line
  convention (spike/wp-00c/d/e/f…) keeps failed lines disposable and the
  good state obvious.
- **Verify numbers at the source before shipping.** One unverified figure
  ("0.8262 teeth IoU") cost more trust than any delay would have. The rule
  now: no quantitative claim leaves the loop without a primary-source
  citation or a same-session measurement. Re-verify subagent-reported
  numbers at the source before relaying them upward.
- **Pivot, never idle.** A blocker on one axis (data unavailable, training
  infeasible, approach dead-ends) routes to another axis. Idleness while
  "waiting" is a choice, and usually the wrong one — there is almost always
  an independent work package (docs, edge cases, eval extensions, script
  hardening) that doesn't block on the gated item.

## What fails

- **Claiming a limitation instead of checking reality.** Twice in one week:
  declaring a model unavailable without per-route verification, and blaming a
  network block that was actually a dead local proxy. The correction that
  stuck: "You should never have Reddit blocked. That's never an excuse."
  Generalized: before reporting "can't", exhaust the local stack (proxy,
  processes, credentials, alternate routes).
- **Autonomous over-production on research questions.** A request for
  "options, pricing, and guidance" once got an unasked ten-option artifact
  page, an unprompted verification pass, and bundled housekeeping. The dial
  authorizes loops, not scope expansion: match the response shape to the
  ask, disclose side-work separately.
- **Treating remarks as mandates.** "The no paths directive was not a
  mandate" — build the generous version, let validation be the guardrail.
  Ask when the cost of guessing wrong is high; otherwise proceed and make
  the choice visible.
- **Interrupting the human with routine progress.** Training-step logs and
  "still running" notes are not checkpoints. A checkpoint carries a
  measurement and a decision: what was tried, the number, the delta, what's
  next. Everything else stays quiet.

## Process improvements (concrete)

- **Brief quality is the highest-leverage artifact.** A subagent with a
  precise brief (exact file paths, exact metric, halt conditions, what
  "done" looks like, what never to touch) needs no steering. A vague brief
  needs rescue. Spend the tokens on the brief.
- **Subagents don't inherit context — so the brief IS the context.**
  Every identifier, constraint, and prior finding the child needs must be
  in the message. "See the transcript" is not available.
- **One axis per agent, one branch per experiment line.** Parallel axes
  must not share mutable state (branches, report.json, eval sets) or they
  will clobber each other. The coordinator owns branch allocation and
  merging; workers own their branch.
- **Parallel axes get isolated git worktrees, not just branches.** Branches
  alone do not protect uncommitted work: two agents sharing one worktree
  will switch branches under each other and lose WIP (happened on
  SmileDraft 2026-10-13 — one axis's uncommitted work was stashed by
  another). The coordinator creates one worktree per axis up front
  (`git worktree add …/smiledraft-<axis> spike/wp-00<axis>`); each worker
  is told its worktree path in the brief and never `git checkout`s.
- **Halt conditions beat status polling.** "Halt and report if X" in the
  brief replaces a dozen check-ins. The runtime delivers completion; the
  coordinator never polls.
- **Measure on the same metric or don't compare.** Every segmentation
  candidate runs the identical 16-face IoU driver. A new number on a new
  metric is not progress, it's noise. Extending the eval set is allowed —
  but keep the old-16 subset scored so deltas stay honest.
- **Keep the human's decision surface small.** Options with a recommendation
  and a default action beat open-ended questions. "Your call" should appear
  at most once per checkpoint, attached to the irreversible step.
- **State hygiene across chats.** MEMORY.md, tracking items, and the repo
  ledger are written by every surface (main chat, side chats, subagents
  via their coordinator). A fact that lives only in one transcript is a
  fact that will be lost at compaction.
- **Cheap experiments first; negative results are results.** The
  SAMRefiner axis returned −0.010 IoU — a dead candidate identified in
  hours of CPU time, not days. Kill fast, record the kill in the ledger,
  pivot. A quick "no" is loop fuel.
- **When the loop plateaus far below the bar, research the bar.**
  Round-2 research found no published system at ≥0.85 teeth-IoU on smile
  photos — every 0.84+ result is on intraoral close-ups. The bar itself
  was uncalibrated. Options the loop generated: measure inter-annotator
  agreement to establish the human ceiling, gate as a fraction of that
  ceiling, or split the gate. Never optimize against label noise past the
  point the evidence identifies it.
- **Instrument the pipeline; inspect intermediates.** The wp-00k worker
  caught a silent data-corruption bug (masks scattered by an affine applied
  without upsampling) only because it looked at intermediate artifacts —
  then deleted the bad checkpoints and restarted clean. Briefs should
  require a spot-check of intermediates, not just final metrics.
- **Constrain the researcher, free the result.** The research brief that
  worked: "verify every number at the primary source, max 8 candidates,
  rank by gain-per-effort, explicitly list what you ruled out." It returned
  a source-cited shortlist with caveats and open questions — no rescue
  needed. Research quality is a function of brief strictness.

## Proactivity calibration

- Dial set by Aaron 2026-10-12: AGGRESSIVE. Run autonomous loops without
  asking; silent only when there is genuinely nothing new.
- The dial authorizes loops, not unverified numbers or unscoped
  deliverables. Aggression without the verification rule is just faster
  mistakes.
- Silence is neutral: an unanswered checkpoint is not approval and not
  rejection. Continue the loop; do not chase.

---
*End of coordinator-reviewed section. Below: durable lessons proposed for
the agent's operating manual (~/AGENTS.md). These are PROPOSALS — the human
or parent agent reviews before promotion.*

## Proposed for AGENTS.md (pending review)

1. No quantitative claim leaves the loop without a primary-source citation
   or a same-session measurement. Re-verify subagent numbers at the source.
2. Every delegation brief carries: exact paths, exact metric, halt
   conditions, never-touch list, and what "done" looks like.
3. One branch per experiment line; the coordinator allocates branches.
4. A checkpoint = measurement + delta + decision. Routine progress is not
   a checkpoint.
5. Blocker on one axis → pivot to another. Idleness is a choice; make it
   deliberately or not at all.

## Round 2 kickoff (2026-10-13) — integration branch pattern
- **Protected main + integration branch.** When the human says "don't touch main" but work packages depend on each other, create an integration branch (`phase-1`) as the working main line. Feature branches merge there; the protected branch stays pristine until the human approves. Record the strategy in the ledger the same turn it is created — future sessions must not have to infer it.
- **Branch-name reality check.** This repo's protected branch is `master`, not `main`. Specs say "main"; the git dir says otherwise. Verify `git branch` before writing any instruction that names the protected branch — a wrong name in a delegated brief can send an agent's merge at the wrong target.
- **First-wave selection.** tasks.md names the parallel lanes explicitly ({WP-01, WP-03} then {WP-02, WP-04} then {WP-05, WP-06}). Follow the spec's own parallelization guidance rather than inventing an order. Pure/no-dep packages (WP-03) pair with foundation packages (WP-01) in wave one.
- **Merge ownership stays with the coordinator.** Work-package agents report; they never merge. The coordinator merges sequentially (foundation first) and owns conflict resolution. Stated in every brief, no exceptions.

## Coordinator merge lessons (2026-10-13, WP-01/WP-03 merges)
- **Never trust the handoff alone — verify at source.** The WP-01 completion handoff was a one-line fragment about WebKit deps. The real report was reconstructed from the branch: commit message, ledger entries, test runs, and eyeballing a committed screenshot. A missing report is a verification task, not a merge blocker.
- **Post-merge verification must exceed the worker's branch state.** The merge itself introduces new interactions: stricter eslint (require-await fired on WP-03's harness under WP-01's config), tsconfig project-service coverage (vitest.config.ts), and — the nasty one — vitest's `include` silently dropping WP-01's 12 unit tests (48 passing instead of 60). Always re-run check + full test suite on the merged tree and compare test counts against the sum of the branches.
- **Reused worktrees accumulate untracked leftovers.** The coordinator's own worktree still held 5.9 GB of spike-era venvs/node_modules. Archive (don't delete) to a clearly-named dir outside any worktree, but first check git-ignored artifacts for value — the 0.6312 finetune checkpoints (.pt) were in there and are the S-16 upgrade path.
- **Ledger numbering needs a stated rule.** A worker invented 0009/0010, colliding with the established sequence (already at 0027). Renumber at merge time and put the numbering rule ("continue the Decision/Outcome sequence, dated entries") in every brief.
- **"Held-out" in an eval report is honest, not a violation.** WP-01's eval marks unmeasurable-in-CI clauses as held-out per the spec's own taxonomy. Verify what the word refers to before raising the flag — in this case it was the spec's evaluator-store concept, never the forbidden zip.
- **Conflict triage order:** ledger/standing (content, renumber) → package.json (union + keep codegen scripts) → tsconfig (app's wins, check the other side's strictness flags still compile) → .gitignore (union + weights) → bun.lock (delete and regenerate, never hand-merge).

## Interrupted-worker recovery (2026-10-13, WP-04)
- **Interrupted agents leave uncommitted work, not empty worktrees.** Before re-dispatching, inspect the worktree (`git status`, `git diff`) — the WP-04 agent had substantial uncommitted work (components, fixture script, dep changes) with zero commits. Resume in the same worktree; do not start over.
- **Resume briefs need an inventory step.** Instruct the new agent to diff and read every uncommitted change first, verify each against the spec clauses, keep/fix/revert explicitly, and report what was kept vs rewritten. Also flag scratch files (dbg.mjs) for deletion and validate dependency changes in package.json as legitimate-vs-accidental.
- **Stale stashes from other loops are not yours.** `git stash list` showed round-1 stashes on spike branches — leave them; note their existence in the report so nobody "helpfully" applies them.

## False "interrupted" signals (2026-10-13, WP-04)
- **An "interrupted" monitor status is not proof of death.** The runtime flagged the WP-04 agent interrupted; it was in fact alive and writing (mtimes 08:13/08:20, after the resume agent's 08:11 dispatch). Two agents then shared one worktree for hours. Before re-dispatching into an "interrupted" agent's tree: check file mtimes and `subagent.list` (completed_at vs status) — a live agent keeps writing.
- **The resume protocol still worked.** The resume brief's rules — inventory uncommitted work first, write no commits and no ledger entries — prevented destructive collision. The second agent became an independent verifier instead: it confirmed 97/97 + 19/19×3, fixed one strict-TS error, and corrected two stale docs. Accidental redundancy converted into a verification pass.
- **Reconcile, then commit once.** With two authors and zero commits, the coordinator: verified the tree itself (check + tests green), fixed the stale bits, appended one incident note to the ledger (outcome 0037), and committed everything as a single WP commit. Never let two agents commit to the same branch.

## Parallel agents collide on shared sequences (2026-10-13, WP-02 merge)
- **Ledger numbers and open-issue IDs are a shared sequence — parallel agents will collide.** WP-02 wrote 0030/0031/0032 and OI-4 while WP-04/WP-06 had already claimed them. The brief said "continue the sequence from the ledger tail" but two agents read the same tail. Fix at merge time (renumber to 0047/0048/0049, OI-5), and record the rule: the coordinator owns all numbering; agents may use provisional numbers but the merge renumbers.
- **vitest.config.ts is a merge magnet.** Three WPs have now touched it ($lib alias, svelte plugin + browser conditions, include globs). Resolve by union, then re-run the FULL suite and compare file/test counts against the pre-merge sum — a silently dropped include glob is the failure mode (caught twice now).
- **A worker's "base" goes stale fast.** WP-02 based on da47d9f (pre-WP-04/WP-06) and its report referenced collisions (port 5173 held by WP-04's preview server) that no longer exist post-merge. Read merge-time reality, not the report's assumptions, when reconciling.

## Conflict resolution mechanics (2026-10-13, WP-05 merge)
- **Resolve conflicts line-by-line, not by string match.** An exact-string python replace failed on tab-indented conflict markers (twice). Line-based parsing — split on the marker lines, take/union the blocks — is robust where string matching is brittle. For ledgers: keep HEAD's chronological entries, renumber the incoming side, append.
- **Verify test counts as checksums.** After every merge: (branch A tests) + (branch B tests) = (merged tests). WP-05: 146 + 58 = 204. Any other number means a silently dropped include glob or file — the failure mode that keeps recurring.
- **Carry known limitations explicitly across merges.** WP-05's arch accuracy (82.8% vs 95% bar) was honestly recorded by the worker; the merge commit and standing.md both restate it with the mitigation hypothesis (WP-06's refined outline). A limitation restated at every handoff cannot become a surprise later.

## "Completed" does not mean "committed" (2026-10-13, WP-07)
- **A worker can finish with uncommitted work in its tree.** The WP-07 agent's session ended with two files (copy exports + segment wiring) uncommitted; the branch failed check without them. The coordinator's merge verification (run `check`, not just `test`) caught it. Always `git status` the worktree on completion handoff, not just the branch tip.
- **Port the uncommitted work, don't re-derive it.** The edits were coherent and matched the ledger decisions — copying the two files into the merged tree and re-verifying was the right call. Then commit as a clearly-labelled fixup so the history tells the truth.
- **A missing final report is a verification task.** The WP-07 handoff had no report body; the branch's commit messages + ledger entries + my own check/test runs reconstructed everything needed. Briefs should demand the report, but the coordinator must be able to proceed without it.

## Verify conflict resolutions with the type checker, not your eyes (2026-10-13, WP-12 merge)
- **A "clean" conflict resolution can still be broken.** My strings.ts union dropped manualMark's closing `} as const;` — invisible in the diff, caught instantly by svelte-check. Rule: after every manual conflict resolution, run the type checker before anything else. Eyes lie; compilers don't.
- **Use checksums, not vibes, for test counts.** WP-12 merge: 259 + 50 = 309 expected, 309 observed. Any other number means a dropped include glob or a lost file. This is now the third merge where the count check paid for itself.

## Merging stale branches: ports, renames, and hidden conflicts (2026-10-13, WP-10 merge)
- **Dev ports in specs/configs are a merge magnet too.** Each parallel agent claimed its own port (5177, 5181, 5182, 5183) in package.json, vite.config.ts, playwright.config.ts AND hardcoded in e2e specs. Resolution: pick the port with the most hardcoded references (WP-10's 5181 — three spec files), take that side for the configs, then sed the losing side's spec files. Always re-run the touched specs post-merge.
- **Cross-WP renames break merged consumers silently.** WP-10 renamed `toPreview` to `toSource` in prepare.ts; WP-08's FixToolsScreen (already merged) used the old name. After every merge, grep for the old identifier across the tree before running check — the type checker catches it, but only if you run it on the merged tree.
- **One file can hide multiple conflict regions.** My strings.ts resolution fixed the first conflict and I stopped looking; a second conflict (export/dentist-card copy blocks) survived until svelte-check caught it. After resolving, `grep -r '^<<<<<<<'` the whole tree — every time, no exceptions.
- **Standalone exports vs object properties: read the structure before concatenating.** My strings.ts union dropped exportScreen/dentistCard *inside* advancedPanel's object literal — syntactically absurd, caught by the checker. The fix was mechanical (close the object, then the standalone exports), but the lesson is: reconstruct both sides' structure from their branch tips before merging text.
- **A worker's "all green" is pre-merge green.** WP-10 was 258/258 on its branch; the merge still needed coordinator verification (343/343 unit, 11/11 export e2e, privacy + segment e2e) because cross-WP interactions (renames, ports, copy blocks) only exist in the merged tree.

## E2E on dev benches does not cover the real screen (2026-10-13, WP-09 merge)
- **A latent bug survived because e2e tested `/dev/*` routes, not the real screen.** The merged AdjustScreen created the WebGL Renderer on the stage canvas, so `getContext('2d')` returned null and compositing silently no-op'd — the user would have seen a broken preview. The 8/8 render e2e passed because it exercises `/dev/render`, never the AdjustScreen. WP-09's branch fixed it (offscreen GL canvas + 2D stage) as a side effect of its rework. Lesson: at least one e2e assertion must run against the real screen per work package, or record the gap explicitly (added to standing.md watch items for WP-15).
- **When two WPs diverge the same store, hand-merge semantically.** WP-12's adjust store (snapshot undo with advanced settings) and WP-09's (labelled DialValues history) could not be text-merged. The coordinator wrote the union by hand: snapshot-based UndoEntry + label field, `commitAdjustGesture(before, label)`, WP-09's history/compare/variant state, AdvancedPanel threading control labels through its gesture lifecycle. The type checker + the unioned test suite (21/21) verified it.
- **Verify file state after every programmatic edit, not just at the end.** A batch of python edits to the test file appeared to apply (in-memory check passed) but the file on disk lacked them; only a direct re-read caught it. The `muse.edit` tool + immediate `grep` is the reliable loop for surgical changes.

## The strings.ts nesting trap is structural, not careless (2026-10-13, WP-13 merge)
- **The same bug happened three times (WP-10, WP-09, WP-13 merges) — so it's a pattern, not a slip.** When git conflicts a docstring-continuation region in strings.ts, the shared `} as const;` after the marker belongs to ONE side's block, and naively concatenating "both blocks" nests the second export inside the first object literal. The reliable resolution: reconstruct from the branch tips — check whether each side's final block ends with its own `} as const;` INSIDE the conflict. If the closer is only in the shared `after` region, the first block needs its closer re-added before the second block starts. svelte-check catches it every time, but the fix is mechanical once you see the structure.
- **Port unification needs a consistency check across all four files.** WP-14 changed package.json + vite.config to 5185 but left playwright.config.ts at 5181 (it added its own wp14 config); WP-13's merge then moved the shared config to 5184. The merged tree had three different ports. After every merge: grep the port across package.json, vite.config.ts, playwright.config.ts and confirm they agree.
