---
title: "ELP — Empirical Lane Parallelism"
subtitle: "A method for human-orchestrated parallel multi-agent software engineering"
author: "Eduardo Díaz Cortés"
date: "2026-05-05"
version: "0.1"
---

# ELP — Empirical Lane Parallelism

> *A method for building software with multiple AI agents working in parallel, under human orchestration, with empirical capture of the process.*

---

> *"Welcome back, my friends, to the show that never ends."*
> — Emerson, Lake & Palmer, *Karn Evil 9: First Impression Part 2*, 1973.

---

## Abstract

**ELP** (*Empirical Lane Parallelism*) is a software engineering method that combines three ideas:

1. **Parallel lanes:** each unit of work runs as an isolated session (a *git worktree*, its own branch, its own `tmux` session). Multiple lanes run in parallel.
2. **Brief as contract:** each lane receives a structured document defining its scope, decisions already made, what to do and what not to do, and what to deliver.
3. **Empirical capture of the process:** pre-code audits, mandatory per-lane retrospectives, and per-build measurements are kept as a dataset that feeds the next lanes.

ELP does not replace the classical development methods, it complements them with an explicit pattern for coordinating multiple agents without the system degrading within weeks.

---

## 1. Motivation

Working with AI assistants in a single conversation works well for small tasks. As the project grows, the pattern breaks:

- **Context fills up:** the agent loses relevant details that were present at the start.
- **Changes step on each other** when several tasks interleave.
- **No record remains** of what was tried and why it was discarded.
- **Each conversation starts from zero**, with no access to prior learning.

The methods in the current spectrum (vibe coding, Pantser, SDD, SPDD, BMAD) do not offer an operational answer to the problem when combined with scale (tens of PRs per day) and duration (multi-month projects). ELP aims to fill that gap.

---

## 2. Definitions

**Lane.** A parallelizable unit of work, tied to a single backlog issue. Each lane is materialized as a *git worktree* with its branch, its brief, and its `tmux` session. The agent that executes a lane runs in **autonomous mode** (in Claude: *auto mode on*): it executes the brief from start to finish without pausing for confirmation, except at the explicitly declared STOP-and-report points.

**Brief.** Structured document that defines the lane's contract. It is the input of the agent that executes the lane.

**Integrator.** AI agent with an interactive session that holds the shared context with the human, reads repo state, writes briefs, launches lanes, and monitors their progress. Does not touch code directly.

**Architect.** The human. Decides scope, sequencing, merges PRs, authorizes irreversible actions, corrects when the integrator or the lanes go wrong.

**Audit (Phase 0).** A *doc-only* lane that validates the premise of a milestone with data before any code is written. May discard the entire proposal.

**Lane retro.** Document each lane writes **before** opening the PR, with metrics, errors found, spec ambiguities, and *friction points*.

**Build TSV.** A *tab-separated* file where every build/test invocation is logged with timestamp, command, outcome, and elapsed time.

**STOP-and-report.** Behavior of the lane agent when it encounters a situation outside its scope: it stops and reports, it does not improvise.

---

## 3. Roles

ELP defines three roles with clear, non-overlapping responsibilities.

### 3.1 Architect (human)

Responsibilities:

- Decide scope of each milestone and sequencing across lanes.
- Approve or reject PRs (or enable auto-merge under conditions).
- Explicitly authorize irreversible actions (force-push, destructive cleanup, push to *main*).
- Correct when the integrator or the lanes get it wrong.
- Make the final call when there is disagreement.

What the architect typically **does not** do:

- Write code line by line.
- Re-discuss decisions already pinned in briefs.
- Approve PRs without the lane retro being written.

### 3.2 Integrator (AI agent, interactive session)

Responsibilities:

- Hold the shared project context, reading repo state (issues, PRs, prior retros, audits).
- Write detailed briefs before launching each lane.
- Launch lane agents (in separate `tmux` sessions with their worktree).
- Monitor progress via `tmux capture-pane` and *wakeup scheduling*.
- Surface critical findings to the human.

What the integrator **does not** do:

- Touch lane code directly.
- Make irreversible decisions without human authorization.
- Clean up worktrees or sessions without explicit authorization.

### 3.3 Lane (AI agent, isolated session, autonomous mode)

The lane agent runs in **autonomous mode end to end**. In Claude's case, this corresponds to operating with *auto mode on*: the agent does not request interactive confirmation between brief steps, executes the whole chain (explore, diagnose, implement, validate, retro, commit, push, PR), and only stops at STOP-and-report points explicitly declared in the brief, or when an irreversible action requires authorization (see §8).

Autonomy is what makes parallel execution viable: a human cannot babysit N lanes at once. The brief replaces *babysitting*, which is why its quality is the ceiling on the lane's quality.

Responsibilities:

- Receive a brief at startup, read it in full before acting.
- Work in its isolated worktree (own branch).
- Execute the flow: *explore → diagnose → implement → validate → retro → commit → push → open PR*, without pausing for confirmation.
- Enable auto-merge when green CI is expected.
- Perform **STOP-and-report** on design ambiguities that exceed its brief.

What the lane **does not** do:

- Improvise scope decisions.
- Touch files outside the scope declared in the brief.
- Clean up its own worktree or session.

---

## 4. Anatomy of a lane

### 4.1 File layout

Each lane lives in three places:

```
/tmp/wt-claude-prompt-<issue>-<descr>.txt   # the brief
/tmp/launch-<issue>.sh                      # wrapper that invokes the agent
~/work/<project>.<issue>-<descr>/           # the worktree with its branch
```

A `tmux` session named `<issue>-<descr>` runs the agent.

### 4.2 Launch

```sh
chmod +x /tmp/launch-NNN.sh
tmux new-session -d -s issue-NNN-descr \
    "wt switch --create issue-NNN-descr -x /tmp/launch-NNN.sh"
```

`wt switch --create` creates worktree + branch, `cd`s into it, and runs the wrapper. The wrapper does `exec claude "$(cat brief)"` and starts the agent with the brief as its initial input.

### 4.3 Monitoring

```sh
tmux list-sessions
tmux capture-pane -t <session-name> -p | tail -30
```

If the trajectory needs correction, use `tmux send-keys` to preserve the context the agent already has. **Re-briefing from scratch loses the learning accumulated in the session.**

### 4.4 Post-merge cleanup

Only after the lane's PR has merged:

```sh
tmux kill-session -t issue-NNN-descr
git worktree remove ~/work/<project>.issue-NNN-descr
git branch -D issue-NNN-descr
git pull --ff-only origin main
```

**Pinned rule:** cleanup requires explicit authorization from the architect. Premature cleanup has destroyed useful context in practice.

---

## 5. The brief as contract

The brief is the difference between an agent that delivers a *review-ready* PR and one that wastes time. Canonical structure:

### 5.1 Goal

One to three sentences. What is being delivered and why it matters now. Ideally cite the issue number.

### 5.2 Context

- Repo + branch (always `issue-NNN-descr` from the current `main`).
- Expected verification (e.g., `git log --oneline -5 origin/main`).
- Project pinned rules (`CLAUDE.md`, files not to touch).
- In-flight parallel lanes, to anticipate conflicts.

### 5.3 Read first

List of files the agent must read before writing code:

- The full issue.
- The relevant audit or design doc, if it exists.
- Retros of prior lanes that share context.
- Specific files in the code that the lane will touch.

### 5.4 Decisions pinned

Decisions already made that the agent **must not re-discuss**. This section is critical: the AI tends to "improve" what was already decided. Typical examples:

- *"Do not fix issue #219 in this lane."*
- *"Do not touch `CHANGELOG.md` or `VERSION`, the release bump regenerates them."*
- *"Smallest-delta wins: change the minimum needed to make it pass."*
- *"Diagnose before implementing: the first step is to understand the failure."*

### 5.5 Lane scope (in / out)

Explicit list of what to DO and what NOT to do.

`DO NOT` is as important as `DO`. The model is trained to be helpful; if you do not tell it to stop, it keeps going. Typical cases:

- Do not touch other modules.
- Do not touch versioning files (`CHANGELOG`, `VERSION`).
- Do not open new issues for adjacent gaps (separate scope).
- STOP-and-report on conditions X.

### 5.6 Required outputs

- Expected modified files.
- Commit message in the project format (typically *Conventional Commits*).
- PR body with what to include.
- Closing commands: `gh pr create`, `gh pr merge --auto`.

### 5.7 Discipline reminders

- Required test tier before push.
- Clean working tree at every commit.
- Auto-merge enabled if applicable.

### 5.8 Instrumentation (mandatory)

- Build logging to TSV.
- Retro doc at `docs/lane-experience-<lane>.md` with fixed sections (see §6.2).

---

## 6. Empirical capture

The formal methods of the current spectrum (SDD, SPDD, BMAD) produce artifacts. ELP **also measures the process**. Without this, the flow degrades within weeks.

### 6.1 Phase 0 audits

When a milestone spans several zones or assumes an unvalidated hypothesis, a **doc-only** lane is run that validates the premise with data before any code is written.

Audit structure (`docs/<milestone>-phase0-audit.md`):

1. Empirical inventory (greps, counts, scans of the current code).
2. Surface quantification (refs per file, types of change).
3. Survey of options (A/B/C/D) with cost + risk + resulting state.
4. Gating strategy (how the build stays green at every PR).
5. Risk inventory.
6. Verdict + recommended phase order.

**Typical result documented in practice:** some audits discard the entire proposal with empirical data, saving weeks of work on a wrong milestone.

### 6.2 Lane retros

Each lane writes `docs/lane-experience-<lane>.md` **before** the PR. **No retro, no PR.** Fixed structure:

- **Objective metrics:** start/end timestamps, build/test invocation counts.
- **Migration / change inventory:** files modified, lines added/deleted, sites migrated.
- **Compiler errors I encountered:** by error class, where, the fix, failed attempts.
- **Friction points:** specific questions the brief asked to answer.
- **Spec ambiguities or interpretive choices:** where the agent had to decide without the brief telling it.
- **Subjective summary:** confidence, hardest, easiest, what helped/hindered the tooling.
- **Limitations of this report.**

Retros serve four purposes:

1. They capture information while context is fresh.
2. They are input for the next lane (lessons learned).
3. They are dataset for honestly evaluating *LLM authorability* (which portion of the work the agent can actually complete).
4. They document gaps that turn into new issues.

### 6.3 Measurements (TSV)

Before the first invocation of `make` or equivalent, the brief instructs:

```bash
export LANE="issue-NNN-descr"
date -Iseconds > /tmp/lane-${LANE}-start.txt
echo -e "timestamp\tcmd\toutcome\telapsed_s" \
    > /tmp/lane-${LANE}-builds.tsv
```

Each `make` is logged as a TSV line. At lane close, it is appended to the retro.

The TSV captures:

- The lane's real wall-clock.
- How many retries each tier needed.
- Time spent on debugging vs. pure implementation.

It is the dataset for honestly evaluating how much each lane costs and comparing similar lanes.

### 6.4 Honesty targets

Pinned documents (`docs/<feature>-honesty-targets.md`) that record where the system's behavior is **fictional vs. real**. For example:

> *"0 fires in self-compile, here is why — the typer pre-increfs before the check."*

This does not inflate the metric. It reports what was measured, not what was projected. The culture is verifiable against the TSVs and the retros.

### 6.5 Findings as issues

When an agent identifies a gap outside scope, the flow is:

1. Document it in *Friction points* of the retro.
2. **Do NOT fix it.** STOP-and-report.
3. The integrator decides whether to open the issue now.
4. If opened, the new issue's body cites the retro as source.

Each new gap becomes **its own lane**. This prevents a lane from turning into an infinite yak-shave.

---

## 7. Operational patterns

### 7.1 Parallel vs. sequential lanes

**Parallel** when:

- They touch different zones of the system.
- They do not share consumer files.
- Their outputs are orthogonal.

**Sequential** when:

- They touch the same consumer files.
- One establishes a pattern the next replicates.
- The next one needs the **retro** of the previous one.

Operational criterion: do they collide in the work area or in the lesson?

### 7.2 STOP-and-report

The brief always includes explicitly: *"if you find X, STOP and report"*. Typical cases:

- A bug adjacent to scope.
- A design decision the brief does not cover.
- The need to touch more zones than anticipated.

The agent **does not improvise scope decisions**. It reports and waits.

### 7.3 Wakeup scheduling

So as not to require constant human polling and not to leave orphaned lanes, the integrator is scheduled to wake up at defined intervals.

Operational rules:

- Typical delay between 60 and 3600 seconds.
- Skip the 60-270 bracket if there is no urgency: prompt cache lasts 5 minutes, shorter waits lose cache.
- The wakeup prompt is self-contained: when it fires, it must be interpretable without prior context.

### 7.4 Auto-merge on green CI

When *branch protection* requires specific green checks, the lane enables auto-merge on every PR. When checks pass, the merge fires automatically.

**Lessons learned:**

- Doc-only PRs can get blocked if `paths-ignore` skips the workflow and the check is never generated. Solution: a self-skip workflow that reports SUCCESS without running.
- Non-required sub-checks may be red and the PR still merges. Lesson: run all gates **locally** before push, do not trust CI alone.

---

## 8. Absolute rules

Non-negotiable rules that apply to every actor in the flow:

1. **Do not touch versioning files** (`CHANGELOG`, `VERSION`); the release bump regenerates them.
2. **No direct push to `main`**; always PR + auto-merge.
3. **Without explicit authorization: no `worktree remove`, no `branch -D`.**
4. **Do not compromise the user's work**; investigate before deleting.
5. **Mandatory retros** for every lane.
6. **Phase 0 audits** for large milestones (>1 PR, unvalidated hypothesis, several zones).
7. **Honesty over optimism:** report what was measured, not what was projected.
8. **Detailed briefs:** a thin brief produces a lost agent.

---

## 9. Anti-patterns

Patterns documented as problematic that ELP explicitly avoids:

- **Premature doc files** (TODOs, plan.md, decision.md). Issues + lane retros cover what should persist; the rest lives in the conversation.
- **Re-fixing a bug outside scope:** open an issue, do not extend the lane.
- **Migration with surviving undocumented aliases:** every alias still alive needs a pinned cause in the PR body.
- **Auto-merge without verifying local gates:** sub-checks can lie.
- **Thin briefs:** *"implement X"* without context produces a lost agent.

---

## 10. When to apply / when not

### 10.1 ELP fits when

- The project will last months, not days.
- There is scope for multiple PRs in parallel (orthogonal zones of the system).
- There is value in preserving learning across lanes.
- The architect can dedicate time to writing briefs.

### 10.2 ELP does NOT fit when

- It is a throwaway prototype. The overhead of briefs and retros does not amortize.
- It is an urgent hotfix. The cadence of parallel PRs does not apply.
- The team has no mature tests or CI. ELP **amplifies** what already exists; without a pipeline, it amplifies the chaos.
- The architect cannot dedicate 10-20 minutes per brief. The brief's quality is the ceiling on the lane's quality.

---

## 11. Comparison with neighboring methods

### 11.1 ELP vs. BMAD-METHOD

BMAD divides by **functional specialty** (Analyst → PM → Architect → Dev → QA). ELP divides by **parallelizable unit** (generalist lanes with bounded scope).

| Dimension | BMAD | ELP |
|---|---|---|
| Division of work | by specialty | by parallelizable unit |
| Topology | sequential (assembly line) | parallel (pool of generalists) |
| Central artifact | shared PRD | brief + retro per lane |
| Source of *what to build* | PRD generated by PM | backlog issues |
| Process capture | partial | persistent (audits + retros + TSV) |
| Change isolation | one branch | one worktree per lane |

### 11.2 ELP vs. SPDD

SPDD treats the prompt as a **living artifact** versioned in sync with the code. ELP treats the **brief** as a single-use contract, but accumulates learning in the **retros**, which are versioned. They are orthogonal: they can be combined.

### 11.3 ELP vs. SDD

SDD delivers the spec before the code and discards it. ELP delivers a brief before the code and discards the brief, but keeps the retro. The retro replaces the discardable spec as input for future iterations.

### 11.4 ELP vs. Pantser

The term *Pantser* applied to software development was articulated by Eduardo Díaz in *Mapas o Brújulas* (lnds.net, 2025), borrowed from literature: the *Pantser* writes **by the seat of his pants**, without a rigid map, adjusting direction with every step. The classical counterpart is the *Plotter*, who plans meticulously before executing.

Pantser iterates empirically with disciplined human review, but the learning **dies with the conversation**. ELP captures the learning in persistent formats that survive session close. The central difference is **persistence of learning**, not the presence of empiricism.

---

## 12. Metrics

Metrics ELP recommends capturing at the project level:

- **PRs merged** per day / week.
- **Peak parallel lanes** (simultaneous concurrency).
- **% of lanes ending in useful STOP-and-report** (indicator of brief quality).
- **Wall-clock per lane** vs. estimate (from the TSV).
- **Retries per lane** (real debugging vs. first-time implementation).
- **Phase 0 audits run** and **% that discarded the milestone**.
- **Issues opened as follow-up** from retros (a flow-health signal).

---

## 13. Case study: a compiler for a functional language with algebraic effects

ELP was formalized from accumulated practice on a concrete project: building from scratch a **self-hosted functional programming language with algebraic effects** and an LLVM backend, executed between **April 20 and May 4, 2026** (14 consecutive days).

### 13.1 Raw figures

| Indicator | Value |
|---|---|
| Total wall-clock | **14 consecutive days** |
| Commits | **868** |
| Pull requests merged | **144** |
| Issues closed | **69** |
| Releases | v0.1.0 → **v0.31.0** |
| Peak commits in a day | **135** |
| Phase 0 audits run | 3 |
| Lane retros produced | 25+ |
| Peak parallel lanes | 3 simultaneous |

The figures are not perception: they come out of `git log`, `gh pr list`, and the TSV of each lane. Every PR cites the retro that precedes it.

### 13.2 Commit distribution

```
   feat        ████████████████████   38%   new features
   docs        ███████████████        29%   audits + retros + design docs
   refactor    ██████                 12%
   fix         █████                  11%
   test/ci/chore █████                10%
```

**29 % docs is high** for a compiler. It is the empirical dataset ELP recommends: Phase 0 audits, lane retros, design docs per feature, honesty targets. **11 % fix is low**: features arrived clean thanks to the gates (selfhost byte-identical from day 1, mandatory tier 1 before push, mandatory retro before the PR).

> Paying the 29 % docs upfront is what keeps the 11 % fix.

### 13.3 What was built

A self-hosted functional language with properties rarely found together in a single implementation:

- **3-stage compiler**: stage 0 in C (~10K lines) → stage 1 bridge (~6K) → stage 2 self-hosted (~41K lines).
- **Selfhost byte-identical** as a gate from day 1: every change must be able to recompile itself byte-for-byte.
- **Effect system** Effekt-style: capabilities, handlers, *row inference*, effects like `Console`, `Spawn`, `Cancel`, `Actor`, `Mutable`, `Ffi`.
- **Perceus RC** Koka-style: reference counting compiled at compile time, no borrow checker, with `reuse-in-place` and elimination of redundant increfs/decrefs.
- **Fibers BEAM-style**: per-fiber private heap, copied messages, cooperative scheduler with `swapcontext`.
- **Stdlib + complete tooling**: `kai build/run/test/fmt/repl/lsp/doc/bench/check`.
- **Mandatory TCO**: tail-call optimization as a language rule, *self-tail-calls* rewritten to goto-loops in C-emit.

### 13.4 Industry comparison

Comparable public projects, with their time to functional self-hosting:

| Project | Time | Team |
|---|---|---|
| Roc (effects + functional) | ~5 years | research |
| Koka (invented Perceus) | ~10+ years | academic |
| Effekt (capabilities) | ~5 years | PhD |
| Crystal (LLVM + inference) | ~3 years | core team |

Human equivalent to reach the same technical *scope*:

> **3-4 senior compiler-expert engineers × 12-18 months ≈ USD 1.0-1.5 M in payroll.**

ELP reached the same point in **12-14 days, with one human (architect) + one integrator agent + parallel lanes**.

### 13.5 What of the method contributed

Honest attribution of the result, not extrapolation:

1. **Selfhost byte-identical from day 1.** Any change that breaks self-compilation is found in the build, not in production. It shielded every feature against invisible regressions.
2. **Three-stage bootstrap.** Allows stage 2 to use the full language without being limited to the subset a C-only compiler could implement. Stage 1 acts as a bridge.
3. **Phase 0 audits as culture.** Three audits already saved weeks: one fully discarded a bootstrap-split proposal with empirical data. The premise was false; without the audit, the team would have implemented on top of a wrong hypothesis.
4. **Lane retros as dataset, not documentation.** Retros that document compiler errors, failed fix paths, and *friction points* feed the next lane and are the actual data for evaluating *LLM authorability*.
5. **Pinned honesty targets.** Each feature documents where its behavior is fictional vs. real (e.g., *"0 fires in self-compile, here is why"*). The culture scales because honesty is verifiable against the TSVs.
6. **Issues as roadmap.** The functional *what to build* lives in GitHub, separate from the *how* (which lives in the brief). This prevents each lane from becoming a yak-shave.

### 13.6 What was closed as negative

Decisions that did not land, pinned as *negatives*:

- *Drop specialisation*: measured −1.7 % at `-O2`, +5.4 % regression at `-O0`. Lane closed as a negative. Phase 2 unboxing had absorbed the overhead.
- *Bootstrap split*: discarded by empirical audit before being implemented. Phase 0 saved 2-3 weeks of wrong work.

These negatives are not failures, they are **data**. Each one has a pinned retro that prevents repetition.

### 13.7 Final observation

The project has a rarely-found property: it is **technically ambitious and operationally honest** at the same time. Phase 0 audits + lane retros + honesty targets build a dataset that a human team usually postpones or loses. The human (architect) did not write code; they kept scope judgment, architecture decisions, and "this does not convince me" calls. The agents executed lanes with detailed briefs.

If the underlying hypothesis (LLM authorability validable against the accumulated dataset) holds, this case is the first documented empirical evidence of a technically complex project built under ELP.

---

## 14. Known limitations

- If the architect is asleep and an agent issues a STOP-and-report, the lane stays **blocked** until the human responds. *Workaround:* briefs with pre-approved authorization for common cases.
- If CI takes long and the wakeup fires earlier, prompt cache is spent without progress. A conscious *tradeoff*.
- Detailed briefs take 10-20 minutes from the architect. Not always worth it for trivial lanes (e.g., a single-file edit).
- The method assumes mature versioning and CI tools; in projects without them it does not apply directly.
- A machine change or a session restart of the integrator loses live context. There is no structural fix today.

---

## 15. Origin and references

ELP was formalized from accumulated practice in long-duration projects with multiple AI agents working in parallel. The first public articulation of the method occurs in this document.

Related methods and references:

- **Díaz, E.,** *"Mapas o Brújulas"*, lnds.net, 2025-10-31. URL: <https://lnds.net/blog/lnds/2025/10/31/mapas-o-brujulas/>. Spanish-language articulation of the *Plotter / Pantser* pair applied to software development.
- **BMAD-METHOD** (Breakthrough Method for Agile AI-Driven Development) — multi-agent pipeline with functional roles.
- **SDD** (Spec-Driven Development) — the specification is the model's input.
- **SPDD** (Structured Prompt-Driven Development) — Zhang & Xia, Thoughtworks, 2026. The prompt as a living artifact.
- **Saltzer & Schroeder, *The Protection of Information in Computer Systems***, 1975 — secure-design principles that apply equally to the agent.
- **Karpathy, A.,** *"Vibe Coding"*, 2024.
- **Krouse, S.,** *"Vibe Code is Legacy Code"*, 2025.
- **Emerson, Lake & Palmer**, *Karn Evil 9: First Impression Part 2*, 1973. Origin of the *backronym* ELP.

---

## Appendix A — Minimal brief template

```markdown
# Lane brief — issue-NNN-<descr>

## Goal
[1-3 sentences. What is delivered and why.]

## Context
- Repo: <repo>, branch: issue-NNN-<descr>
- Verification: git log --oneline -5 origin/main
- Project pinned rules: CLAUDE.md
- In-flight parallel lanes: [list]

## Read first
- Issue: gh issue view NNN
- Audit: docs/<milestone>-phase0-audit.md (if applicable)
- Related retros: docs/lane-experience-<previous>.md
- Files to inspect: <paths>

## Decisions pinned
- [decision 1]
- [decision 2]

## Lane scope
DO:
- [action 1]
- [action 2]

DO NOT:
- [prohibition 1]
- [prohibition 2]

STOP-and-report if:
- [condition 1]

## Required outputs
- Expected modified files: <paths>
- Commit message: <type>(<scope>): <subject>
- PR body: [what to include]
- Closing: gh pr create + gh pr merge --auto

## Discipline
- Tier 0 + Tier 1 locally before push.
- Clean working tree at every commit.

## Instrumentation
- Build TSV at /tmp/lane-${LANE}-builds.tsv.
- Retro at docs/lane-experience-issue-NNN-<descr>.md (mandatory sections).
```

---

## Appendix B — Minimal lane retro template

```markdown
# Lane retro — issue-NNN-<descr>

## Objective metrics
- Start: <ISO timestamp>
- End: <ISO timestamp>
- Wall-clock: <duration>
- Build invocations: <count per tier>

## Migration / change inventory
- Files modified: <list>
- Lines added / deleted: +X / -Y
- Sites touched: <count>

## Compiler errors I encountered
- [Error class]: where, fix, attempts.

## Friction points
- [Brief question]: answer based on what was observed.

## Spec ambiguities or interpretive choices
- [Decision without explicit spec]: justification.

## Subjective summary
- Confidence: high / medium / low.
- Hardest: ...
- Easiest: ...
- Compiler / tooling help / hinder: ...

## Limitations of this report
- [What this report does not cover.]
```
