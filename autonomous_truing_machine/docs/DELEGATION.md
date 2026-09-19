# Delegation Mode: handing bounded work to an external coding agent

**Status: adopted 2026-09-19, not yet active.** The rule is frozen here. It becomes active when the
operator says so explicitly, and even then only for a session in which the operator has switched the
mode on. **The mode is OFF by default in every session and is never inferred.**

This is human-mediated. The lead (the Claude Code session) cannot prompt or control the delegate. The
operator carries the handoff to the delegate and carries its report back. Internal Claude Code
subagents (`firmware-scout`, `firmware-worker`, `spec-reviewer`, `bench-verifier`) are a different
mechanism with their own rules in `CLAUDE.md`; nothing here changes them.

## 1. The loop

    lead recognises a bounded task -> lead writes and commits the handoff, creates the worktree
      -> operator opens the delegate in that worktree and pastes the prompt
      -> delegate works in the repo, commits on a wip branch, appends its return report
      -> operator tells the lead "done" (and pastes the report) -> lead reviews -> lead integrates

**The delegate** is a coding agent with full repository access (Cline driving GLM through OpenRouter):
it reads files, runs commands and tests, edits and commits. It shares none of the lead's context.
Because it can read the repo, a handoff carries the **decision envelope**, not the codebase: what is
decided, what is forbidden, what counts as done, when to stop. Anything it can cheaply discover, it
discovers.

**Standing rules are not repeated per task.** They live once in `.clinerules/` at the repository root,
which Cline loads into its system prompt: `00-repo.md` (never-list and always-list) and
`01-delegate-protocol.md` (how a task runs, and the return-report shape). A handoff that restates them
will eventually contradict them.

## 2. The toggle

| | |
|---|---|
| **Default** | `Delegation Mode: OFF` at the start of every session |
| **Turn on / off** | Only when the operator says so ("Delegation Mode: ON" / "OFF" or clearly equivalent). Never inferred from context, never left on across sessions |
| **OFF** | Work exactly as usual. Internal agents per the repo's ordinary rules. The lead does **not** stop to write external handoffs, and does not remark that a task would have qualified |
| **ON** | The lead tests each next unit of work against sections 3-5 and stops at the boundary when one qualifies (section 6). Everything that does not qualify continues as normal |
| **Turned OFF mid-task** | Return to the normal flow at once. An un-dispatched proposal is dropped. A report the operator later brings back is still reviewed under section 8 |

## 3. What never leaves the lead

- Working out the blast radius, the architecture, the experiment's meaning, or the correct requirement.
  **If that reasoning is the hard part, it is not delegatable yet.** Once the problem is bounded, the
  execution may be.
- Spec interpretation and reconciliation; anything under `src/` or `lib/`; anything touching the board.
- Evidence interpretation, and every claim about what a result supports.
- Pre-registrations, amendments, the campaign record, plans, provenance, thresholds and DSP constants.
- The commit that lands the work. Delegates commit to a wip branch; the lead integrates.

**Never delegate merely because it is technically possible.**

## 4. The gate: all must hold, or the lead does it

1. **A committed specification states the answer** (a doc section, a registration, a spec clause), so
   the delegate builds to a fixed target and the lead reviews against the same text.
2. **The blast radius is known**: the lead can list every file it touches and every consumer of it.
3. **It touches no record, plan or pre-registration, no `src/`, no `lib/`, and not the board.**
4. **A machine-checkable verifier exists** and is written into the handoff: a selftest, a test count,
   an expected output. "Looks right" is not a verifier.
5. **It is worth the round trip.** Roughly: it would cost the lead more than ~30 minutes of tool calls,
   the handoff costs less than a third of doing it, and the operator's copy-and-return effort is
   justified. A delegation that fails review costs more than the lead doing it.

## 5. Which model

### What the evidence says

| | GLM-5.3 | GLM-5.3-Flash |
|---|---|---|
| Size | 753B mixture of experts | 320B mixture of experts, 18B active |
| Context | up to 1M | 1M (attention drift reported past ~700K; tasks here are far smaller) |
| DeepSWE pass@1 | **69.0% ±2.7** | 63.4% ±4.1 |
| DeepSWE pass@4 | 87.6% | 85.0% (the gap narrows to 2.6 points) |
| **Breaks already-passing tests** | **4.4%** of runs | **6.9%** of runs (over 50% more likely) |
| Cost per rollout | $3.99 | $0.24 (about 17 times cheaper) |
| Average task | ~35 min, 125 steps | ~26 min, 123 steps; verbose thinking |
| Reported strengths | first-shot reliability, JS, query work, reasoning-heavy tasks | Python, concurrency, data modelling, protocol work, tool use, persisting through errors |
| Reported weakness | slower, dearer | more regressions; behind on the hardest reasoning (HLE 55.3 vs 62.5) |

Neither model allows thinking to be fully disabled; `reasoning_effort` (low, high, max; default max)
is the lever where the harness exposes it.

**Caveats, stated because the numbers are soft.** The head-to-head is from Together AI, an inference
provider with a commercial interest. Sources disagree on secondary figures (Artificial Analysis lists
different index values from the vendor-adjacent posts; throughput ranges from 49 to 97 tokens per
second). The benchmark is generic repository tasks, not this repository. Treat the *direction* as
reliable (Flash is close on pass@k, much cheaper, and regresses more) and the exact figures as
approximate. Re-check when either model changes.

Sources (accessed 2026-09-19): Together AI, "GLM-5.3 vs GLM-5.3 Flash on DeepSWE";
MindStudio, "GLM 5.3 Flash vs GLM 5.3"; Hugging Face model cards `zai-org/GLM-5.3` and
`zai-org/GLM-5.3-Flash`; MarkTechPost, "Z.ai Releases GLM-5.3-Flash"; DataCamp, "GLM-5.3-Flash".

### What decides the routing here

- **Breakage 6.9% against 4.4%.** In this repository a silently broken tool corrupts the evidence
  record, not just a build. Together's own guidance is to gate Flash with a regression run.
- **pass@1 gap 5.6 points, pass@4 gap 2.6.** Flash recovers most of the gap through retries, so it
  suits work where **failure is loud and a retry is cheap**, and is the wrong choice where failure is
  **silent**. Here, silent means anything whose output is a number the campaign then trusts.
- **Human round trips are the scarce resource, not tokens.** Every correction costs the operator a
  copy-and-return. First-shot reliability therefore counts for more than raw price.

### The rule

| Send to | When all of these hold |
|---|---|
| **GLM-5.3-Flash** | Purely additive (a new file, or new rows in a generator); a committed spec states the answer; an in-repo pattern to copy; **failure is loud** (a selftest goes red); modifies no already-passing code the campaign depends on; Python, shell or JS, at most two files, roughly 300 lines |
| **GLM-5.3** | Bounded and understood, but it **edits existing working code**, or spans several files, or needs judgement about structure, or **a wrong answer would look plausible**, or it is C in the test harness. Also the single escalation when a Flash return fails review |
| **The lead** | The blast radius is not yet known; spec, evidence or campaign-rule meaning; anything in section 3 |

Examples that show it discriminates. A new report tool in `tools/` with its own selftest, built on an
existing pattern and a committed spec, is **Flash**. Adding candidate rows to
`test/test_acoustic_sweep/test_main.c` looks additive but edits a harness that B3.0's verdict already
depends on, and a regression there silently corrupts analysis: it is **GLM-5.3**.

## 6. The proposal: where the lead stops

When the mode is ON and a unit of work passes sections 3-5, the lead **stops before doing any of it**
and posts this, and nothing else, then waits:

1. **The unit and why it qualifies**: the five gate items, each with a one-line justification.
2. **The model** and the reason, referring to section 5.
3. **Set-up already done**: the handoff file committed on `main`; the worktree and branch created
   (`git worktree add "$USERPROFILE/wt/<task>" -b wip/<task> <base>`, a path without spaces); the
   worktree path the operator should open the delegate in; the paths the lead will not edit while the
   task is open.
4. **The prompt**, complete, in one fenced block, ready to paste.
5. **What to bring back**: "done" and the delegate's return report. The report is also committed on
   the branch, so the branch is the authority and the paste is a convenience.
6. **How the lead will review it**: the review depth it expects from section 8, and what would cause
   an outright take-back. The operator should know the cost of a bad return before sending.

The lead does not carry on with the delegated paths. If the operator says "carry on meanwhile", the
lead continues only on paths outside the task's allowlist.

## 7. The handoff

Committed to `autonomous_truing_machine/docs/handoffs/<task>.md` on `main` before dispatch, so it
exists in the worktree. The prompt the operator pastes is short: it names the handoff file and states
the branch. Template (Appendix A) fields, and what each is for:

| Field | Prevents |
|---|---|
| Why this exists | Solving a neighbouring problem. Gives the project reason, not the code reason |
| **Already decided** | The delegate re-opening or quietly re-deciding lead-level choices |
| Objective, Scope IN / OUT | Scope creep; OUT names what is excluded and why |
| Paths: may edit / forbidden | Extra files "because they seemed necessary" |
| **Known consumers** | The hidden cross-file interaction (failure 1): the lead names what else reads what is touched |
| Discover yourself; reuse, do not reimplement | Re-deriving the repo; duplicating an existing helper |
| Must not reinterpret | Terms with a precise project meaning; examples taken for requirements |
| You may decide / Escalate | Silent architectural decisions; guessing instead of stopping |
| **Acceptance criteria, quoted from the spec** | A test written to match the code rather than the specification |
| Verification, with mutation check | "Tests pass" with no proof; tests that cannot fail |
| Data rules | Tuning a tool against real campaign data |
| Deliverables and report shape | A summary that hides decisions |

**Tailoring to the model.**

- **Flash**: at most about a page and a half. Numbered, imperative steps; every requirement testable;
  exact command lines; function signatures or a skeleton given; golden expected outputs for the tests;
  "do exactly this and nothing else". Additive work only; no edits to existing files beyond a stated
  hook. The baseline suite run before and after, both pasted. The forbidden list repeated at the end.
  Ask for low or medium `reasoning_effort` if Cline exposes it.
- **GLM-5.3**: the objective, the requirements and the envelope, with local design choices left to it
  and listed under *You may decide*. Multi-file edits within the allowlist. A five-line design note in
  the report before the implementation detail. `reasoning_effort` high rather than the slow default
  max. Allow it more time.

## 8. The review

**Principle: the report is a claim; the diff is the evidence; the re-run is the proof.** The delegate
may have misunderstood something while sounding sure. Review is risk-scored so that delegation stays
worth doing, and nothing is skipped silently.

### The floor: every return, never skipped

Run in the delegate's worktree (`$W`), against its branch. **Do not use the `truing_ws` junction: it
points at the main checkout and would test the wrong tree.**

    git diff --name-only <base>..wip/<task>      # any path off the allowlist -> reject
    git diff --stat <base>..wip/<task>           # shape; a big stat on a small change = line endings
    git log <base>..wip/<task> --format=%B | grep -i "co-authored\|claude"
    "$USERPROFILE/.platformio/penv/Scripts/pio.exe" test -d "$W/autonomous_truing_machine" -e native
    <the handoff's own selftest commands, re-run by the lead>

Then read report fields 7-11 (escalations, assumptions, deviations, review requests, handoff errors)
**before** reading any code. Compare the report's file list and counts with `git diff --stat`: a
mismatch is a hidden deviation. Check that no existing test lost an assertion:
`git diff <base>..wip/<task> -- test/ | grep '^-' `.

### The score

| Signal | Points |
|---|---|
| Flash was used | 1 |
| Modified existing working code (not purely additive) | 1 |
| Behaviour-changing rather than tooling-only | 1 |
| Each assumption or deviation in the report | 1 |
| The implementation's shape differs from what the lead expected | 1 |
| Touched `src/` or `lib/`, or **any file off the allowlist** | **reject** |
| An existing test was edited or weakened | **reject** |

| Score | Depth |
|---|---|
| **0-1 Light** | The floor, then skim the diff and confirm the new test asserts the acceptance criteria |
| **2-3 Standard** | The floor, then read the whole diff; **check each acceptance criterion against the governing document, not against the code**; re-run one mutation check personally; grep for the helper it should have reused; search for other consumers of anything it changed |
| **4+ Deep** | Standard, plus `spec-reviewer` on the diff, plus running every consumer's selftest, plus running the tool on real data personally before any number from it is believed |

**Override: anything that produces a number the campaign will trust gets at least Standard**, whatever
the score. A green selftest written by the same agent that wrote the code is not evidence the number
is right.

### What differs from reviewing the lead's own work

- Read the **report's disclosures before the code**, and treat "none" on every field of a non-trivial
  task as a warning, not reassurance.
- Compare the diff to the handoff **item by item**: each acceptance criterion to the assertion that
  covers it, each file to the allowlist, each decision to the *You may decide* list.
- Read the new tests' expected values and ask where they came from. Values derived from the
  implementation's own output are worthless.
- Search independently for call sites and consumers of anything changed; the delegate could not see
  what the lead knew about.
- **Authority exceeded is a reject even when the code is good.** Either the lead re-decides the point
  and accepts it explicitly, recording that it decided, or it takes the task back. It never accepts
  silently.

## 9. Failure policy

| Situation | Action |
|---|---|
| Small misunderstanding; scope and envelope respected | **One** correction handoff: a short delta naming the failed criteria with evidence |
| Scope expanded, or a decision on the escalate list was taken | **Take the task back.** No second round, even if the code works: a correction brief does not fix judgement |
| A test written to match the code, an existing test weakened, or fabricated evidence | **Take it back**, and rewrite the acceptance section of the template |
| Report and diff disagree (a hidden deviation) | Take it back, and review that model's next three returns at Deep |
| The same class of failure in two different tasks | **Fix the template or the routing rule.** Stop paying the correction cost |
| Flash regressed existing code | Route that task class to GLM-5.3 from now on |
| A second failure of the same task | The lead takes it: the brief was under-specified or the task was not delegatable |

## 10. Integration

The lead lands the work: `git merge --squash wip/<task>` then a commit with the lead's own message,
so the evidence claims and the attribution rule stay the lead's. No delegate commit reaches `main`
directly. The return report is kept at `docs/handoffs/<task>.md` on `main`. The worktree and branch
are removed after acceptance. One row is added to the log below.

## 11. Delegation log and calibration

Append-only. The point is to replace intuition with a record: after three consecutive clean Light
returns from one model on one task class, that class may drop one review step; any hidden deviation
resets it. If the first few tasks show the template or the routing is wrong, this doc changes.

| Date | Task | Model | Review depth | Rounds | Outcome | Lead's review cost / notes |
|---|---|---|---|---|---|---|
| | | | | | | |

---

## Appendix A: handoff template

    # Handoff: <task>
    Model: GLM-5.3 | GLM-5.3-Flash      Branch: wip/<task>      Base: <commit>
    Worktree: <path without spaces>

    ## Why this exists
    <2-3 sentences: the project reason. What stays blocked without it.>

    ## Already decided: do not revisit
    <decisions from the lead's reasoning that the repo does not explain on its own>
    Governing document: <file, heading, commit hash>. Read that section; it is the specification.

    ## Objective
    <one sentence>

    ## Scope
    IN:  <exact behaviour>
    OUT: <adjacent things deliberately excluded, and a clause on why>

    ## Paths
    MAY CREATE/EDIT: <explicit list>
    FORBIDDEN: everything else (and the standing never-list in .clinerules/00-repo.md)
    FROZEN by the lead while this is open: <paths the lead will not touch>

    ## Known consumers of what you touch
    <what else reads or depends on it. If you find a consumer not listed and it is not read-only,
    stop and report.>

    ## Discover yourself (do not ask, do not assume)
    - <questions the repo answers>
    - Reuse, do not reimplement: <module and the functions to reuse>
    - Imitate the shape of: <one or two exemplar files>

    ## Must not silently reinterpret
    - <term with a precise project meaning, and its definition>
    - Anything labelled EXAMPLE is illustrative, not a requirement.

    ## You may decide
    <naming, internal structure, test arrangement, message wording>

    ## Escalate: stop, write it in the report, do not decide
    - A threshold, a DSP constant, a pre-registered rule, provenance, a spec contract, or what a
      session claims about itself.
    - Any path off the allowlist appearing necessary.
    - Any place the governing document and the code disagree.

    ## Acceptance criteria: the test must assert these literally
    1. <criterion quoted from the governing document>
    2. ...
    These come from the specification, not from your implementation. If one is awkward to assert,
    escalate; do not weaken it.

    ## Verification required
    - Baseline before and after: <exact command> -> expect <n>/<n>, paste both tails
    - <selftest command> -> paste verbatim
    - Mutation check: break <specific thing>, paste the FAILING output, restore, paste it passing
    - Builds: none. Host-side only.

    ## Data rules
    <Do NOT run against test/fixtures/acoustic/captures/_campaign. Use synthetic fixtures.>

    ## Deliverables
    Commits on wip/<task> only. Then the return report from .clinerules/01-delegate-protocol.md,
    appended to this file as the final commit.

    ## Repeat of the hard limits (recency)
    No hardware. No record, plans, src/ or lib/. No attribution. Stop and report when unsure.
