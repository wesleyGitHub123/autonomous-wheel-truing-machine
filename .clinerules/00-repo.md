# Standing rules for any external coding agent working in this repository

You are a delegate. A person handed you a task file under `autonomous_truing_machine/docs/handoffs/`.
The lead engineer who wrote it is not in this session and cannot answer questions while you work. These
rules apply to every task; the handoff adds the task-specific envelope. If the two ever conflict, the
stricter one wins. Read `CLAUDE.md` at the repository root before anything else.

## Never, whatever the handoff says

- **Never touch hardware.** No flashing, no serial or COM ports, no joining a WiFi network, and never
  run `campaign_runner.py`, `capture_fetch.py`, `flash.py`, `nano_serial.py`, `serial_capture.py`, or
  anything under `tools/probe/` or `tools/bench/`. Running the runner fires solenoids and burns trial
  numbers that belong to a pre-registered campaign.
- **Never write the campaign record or plans:** `docs/SOLENOID_CAMPAIGN.md`,
  `docs/SOLENOID_CAMPAIGN_PLAN.md`, `docs/campaign_plans/`, `docs/IMPLEMENTATION_NOTES.md`. Only the
  lead writes evidence and claims.
- **Never edit `src/` or `lib/`** unless the handoff's allowlist names the exact file.
- **Never run anything against real campaign data:** `test/fixtures/acoustic/captures/_campaign/`
  stays unread. Develop against synthetic fixtures created inside your own selftest. The lead runs your
  tool on real data first, so it cannot have been tuned to it.
- **No destructive git:** no `push`, `reset --hard`, `clean`, `rebase`, `checkout` of another branch,
  `stash drop`, force anything. Commit only on the branch the handoff names.
- **No new dependencies:** no `pip install`, `npm install`, `pio lib install`, no network fetches.
- **No firmware builds.** The only `pio` invocation you may run is `-e native` (host tests).

## Always

- **No attribution.** No `Co-Authored-By` trailer, no "Generated with" line, in any commit. Your
  harness may add one by default; check `git log -1 --format=%B` after every commit and amend it out.
- **Preserve each file's line endings.** Before editing an existing file run
  `git ls-files --eol <path>` and keep it as it is. New files are LF. A three-line change that shows
  as a whole-file diff means you flipped the line endings; undo it.
- **Keep the diff minimal.** No reformatting, no import reordering, no drive-by fixes. If you notice a
  problem outside your scope, do not fix it: list it in the report.
- **Do not commit local identifiers.** No board MAC addresses, SSIDs, WiFi passwords, or absolute paths
  such as `C:\Users\...` in any committed file.
- **Do not claim what you cannot show.** Never write "verified", "proven" or "tested on the board" in
  code, comments or messages. You cannot touch the board.
- **Never edit or delete an existing test to make something pass.** If an existing test fails, that is
  a finding: stop and report it.

## Host tests: run them in your own worktree

You work in a git worktree, not in the main checkout. The `$USERPROFILE\truing_ws` junction that
`CLAUDE.md` tells you to pass to `pio -d` points at the **main** checkout, so it would test someone
else's code. Run host tests against your own tree instead:

    "$USERPROFILE/.platformio/penv/Scripts/pio.exe" test -d "<your worktree>/autonomous_truing_machine" -e native

Host tests are unaffected by spaces in the path. Confirm your tree with `git rev-parse --show-toplevel`.

## When you are unsure

Stop. Do not guess and do not pick the plausible option. Write the question into the report under
`Escalations` and finish with status `BLOCKED` (see `01-delegate-protocol.md`). A precise question is a
successful outcome; a silent assumption is not.
