# How a delegated task runs, start to finish

## 1. Orient (before writing any code)

1. Read the handoff file in full, and the governing document section it cites. That section is the
   specification. The handoff's *Already decided* list is not open for discussion.
2. Confirm where you are: `git rev-parse --show-toplevel`, `git branch --show-current`,
   `git log --oneline -1`. The branch must be the one the handoff names, and the base commit must match.
   If not, stop and report `BLOCKED`.
3. Do your own discovery: the handoff's *Discover yourself* list, and a search for existing helpers
   before you write a new one. Record what you searched for and what you reused. Duplicating something
   that already exists is a defect.

## 2. Implement

- Touch only the paths in the handoff's allowlist. If another file looks necessary, that is not your
  call: stop and report it.
- Take only the decisions the handoff lists under *You may decide*. Anything on the *Escalate* list, and
  anything the handoff does not cover that matters, goes to the report, not into the code.
- Treat every item in the handoff marked `EXAMPLE` as illustrative. Treat every numbered acceptance
  criterion as a requirement.

## 3. Verify

- Run the baseline **before** you change anything and again **after**; record both counts. A drop in
  passing tests is a stop condition.
- Your new tests must assert the acceptance criteria **as written in the handoff**, not what your
  implementation happens to produce. If a criterion is awkward to assert, escalate; do not weaken it.
- For each new test, prove it can fail: break the thing it guards, run it, keep the **failing output**,
  restore, run it passing. Paste both. A mutation you describe but did not run does not count.

## 4. Commit

Commit only on the named branch, in the commits the handoff asks for. Subject in the repository's
conventional style (`feat(tools): ...`). Body of at most 10 lines: cause, evidence, limits. No
attribution trailer. Then append your return report to the handoff file as the final commit.

## 5. Return report

Append this to the handoff file, filled in completely. A field with nothing to say says `none`.

    ## Return report
    Handoff: <task>    Base: <sha>    Branch HEAD: <sha>
    Status: DONE | DONE_WITH_DEVIATIONS | BLOCKED

    1. Files changed
       <path | +added/-removed | why>   and the pasted `git diff --stat <base>..HEAD`
    2. Acceptance criteria
       <criterion number | met / not met | where it is asserted: file:line or test name>
    3. Commands run
       <exact command, then the verbatim last ~20 lines of its output>
       Baseline before: <n>/<n>    After: <n>/<n>
    4. Mutation checks
       <what you broke, the FAILING output pasted, confirmation it was restored and passes again>
    5. Reuse search
       <what you searched for, and what existing code you reused>
    6. Decisions I took
       <each: the decision, the alternatives, why. Only from the "You may decide" list>
    7. Escalations
       <anything from the Escalate list, or any ambiguity you could not resolve, or none>
    8. Assumptions
       <anything the handoff did not cover that you assumed, or none>
    9. Deviations from the handoff
       <anything done differently from what was asked, and why, or none>
    10. What I think needs the lead's review
    11. Anything in the handoff that was wrong, ambiguous or missing

Report `BLOCKED` when you stopped on an escalation: commit nothing further, state the exact question
and the options you considered, and stop. Report `DONE_WITH_DEVIATIONS` whenever anything in field 9 is
not `none`. A confident report on a messy task is harder to trust than an honest one.
