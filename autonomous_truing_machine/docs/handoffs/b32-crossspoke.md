# Handoff: b32-crossspoke
Model: GLM-5.3-Flash      Branch: wip/b32-crossspoke      Base: the commit titled `docs(handoff): b32-crossspoke`
Worktree: $USERPROFILE/wt/b32-crossspoke      Reasoning effort: low or medium if Cline lets you choose

## Why this exists
The B3.2 strike data will be judged per spoke, and a tone that repeats at the same frequency on every
spoke would pass that test. The registration therefore adds a *report* that flags it for a human to look
at. The report must exist, tested, before any strike data does, so it cannot be shaped by the data.

## Already decided: do not revisit
Governing document: `docs/SOLENOID_CAMPAIGN.md`, heading **"Amendment 2 to this registration
(2026-09-19, after the air blocks, before any strike)"**, commit `bbf3ba9`. Read that section; it is the
specification. It is a **report only**: nothing is gated, filtered or removed, and no threshold moves.
Two words in it needed an exact meaning, and the lead has fixed them here:

1. **"within +-2 Hz of one another"** means the same rule `b30_verdict.coherent_lines` already uses for
   "the same line": take each clear's f1 as a centre `c`; the members are the clears with
   `abs(f1 - c) <= 2.0` (inclusive); the cell is flagged if the members come from **at least 3 distinct
   spokes**. Try every clear as a centre; use the centre with the most distinct spokes (ties: the lowest
   centre). The common frequency is the **median of the member f1 values**. Several clears from one spoke
   count as one spoke.
2. **"a line found in that station's air shots at the same width"** means a *coherent line* as B3.0
   defines it: `b30_verdict.coherent_lines` over the air-shot captures of that station and width, using
   the strong in-band peaks whose SNR is at least the min SNR read from the lines CSV. With 6 air shots
   that means a line in at least 3 of 6. These lines come from the lines CSV, which is produced under the
   baseline chain profile only, not under each candidate. The report must say so.

## Objective
Add `tools/b32_crossspoke.py`: from the bundle directory, the sweep CSV and (optionally) the lines CSV,
list per station, width and candidate every cell where clears from at least 3 exploration spokes share
an f1, and say whether that frequency matches an air-shot line.

## Scope
IN: the one new tool, with a `--selftest`, in the shape of `tools/b30_verdict.py` (argparse, `main()`,
`selftest()` using synthetic temp data, stdlib only).
OUT: changing any existing file; running the sweep or the lines harness; any gating, filtering, exit
code that depends on a flag, or the words PASS/FAIL (this is a prompt to look, not a verdict).

## Paths
MAY CREATE: `autonomous_truing_machine/tools/b32_crossspoke.py`. MAY EDIT: only this handoff file, to
append the return report.
FORBIDDEN: everything else, and the never-list in `.clinerules/00-repo.md`. In particular do not edit
`tools/b30_verdict.py` (import from it), `test/`, `docs/SOLENOID_CAMPAIGN*.md`, `docs/campaign_plans/`.
FROZEN by the lead while this is open: the same set, plus `test/test_acoustic_sweep/`.

## Known consumers
Nothing reads this tool. It reads `tools/b30_verdict.py` (import only). The sweep CSV format it parses
will later gain two-field candidate rows; keep all parsing of it inside one function so a format change
touches one place.

## Discover yourself
- Reuse, do not reimplement: `b30_verdict.load_bundles` (bundles + the exclusion ledger),
  `b30_verdict.parse_lines_csv`, `b30_verdict.coherent_lines`, `b30_verdict.LINE_TOL_HZ`. Add
  `sys.path.insert(0, os.path.dirname(os.path.abspath(__file__)))` as `tools/campaign_sequence.py` does.
- Imitate the shape of `tools/b30_verdict.py`.
- The bundle JSON fields you need: `campaign_stage`, `campaign_kind`, `campaign_station`, `pulse_ms`
  (a float such as `72.0`), `spoke_id`, `capture_result`. Read `test/test_acoustic_sweep/test_main.c`
  only to see the sweep's output format; do not run it.

## Must not silently reinterpret
- **Clear** = the sweep row's `status` is `suspect` **and** `f1_hz` is non-empty. A `rejected` row with
  an f1 is not a clear.
- **Strike bundle** = `campaign_stage == "B3.2"`, `campaign_kind == "strike"`, `capture_result` is `OK`
  or absent, and not in the exclusion ledger. `air_shot` and `no_fire` bundles are never strikes.
- **Cell** = (station, width in ms as an integer, candidate). **Candidate label** = `"<axis>=<value>"`
  from the sweep row's second and third columns (the baseline row gives `baseline=0`).
- **Exploration spokes per station** = 4 (a constant `EXPECTED_SPOKES = 4`). The report states how many
  distinct spokes actually have strike data per station; fewer than 4 is reported, not an error.
- Sweep row format, exactly (11 fields; skip anything else, and any row whose axis is empty):
  `bundle,axis,value,status,reason,n_strong_peaks,n_peaks_in_band,f1_hz,f2_hz,snr_db,clears_min_snr`
  EXAMPLE: `B3.2_t100_strike_sp17_seq44,baseline,0,suspect,PROVISIONAL_MODE_ID,12,3,501.000,1010.200,15.300,1`
- Anything labelled EXAMPLE is illustrative, not a requirement.

## You may decide
Function names, internal structure, report wording and layout, how the selftest builds its fixtures.

## Escalate: stop, write it in the report, do not decide
- Any doubt about what counts as a clear, a cell, or a flag beyond what is written above.
- Any need to edit or run anything on the forbidden list.
- Any place the governing section and the code disagree.

## Acceptance criteria: the selftest must assert these literally
All on synthetic data built inside the selftest. Spokes and stations are those of the registration
(RIGHT: 17, 5, 3, 15). Each case is one cell unless said otherwise.
1. Clears at f1 500.0 (spoke 17), 501.0 (5), 502.5 (3), 470.0 (15): **flagged**, common frequency
   **501.0**, spokes `[3, 5, 17]`.
2. Clears 500.0 (17), 501.0 (5), 503.5 (3): **not flagged** (only 2 within 2 Hz of any centre).
3. Boundary: 500.0, 502.0, 504.0 from three spokes: **flagged**. 500.0, 502.0, 504.01: **not flagged**.
4. Three clears from spoke 17 at 500.0, 500.1, 500.2 plus one from spoke 5 at 500.0: **not flagged**
   (2 distinct spokes).
5. Three spokes at 500.0 but one row's `status` is `rejected`: **not flagged**.
6. Three spokes at 500.0 but one bundle is in the exclusion ledger: **not flagged**.
7. Three spokes at 500.0 but at widths 60, 72 and 85: **not flagged**; the same three spokes at the
   same width but different candidates: **not flagged**; one of them recorded at station LEFT:
   **not flagged**. A cell never mixes stations, widths or candidates.
8. An `air_shot` bundle, a `no_fire` bundle and a `campaign_stage: "B3.0"` strike, each with a matching
   clear row, are **never counted** (the cell in case 1 with one of these replacing a real spoke is not
   flagged).
9. Air match, on a flagged RIGHT / 72 ms cell whose common frequency is 501.0: a coherent air line at
   501.4 in 3 of 6 air shots at RIGHT / 72 ms: **matched**. The same line in 2 of 6: **no air line**.
   A line at 503.5 in 6 of 6: **not matched** (2.5 Hz away). A line at 501.4 in 6 of 6 at RIGHT / 60 ms
   only: **not matched** (another width). A 501.4 peak below the min SNR in 6 of 6: **not matched**.
   With no lines CSV given, the report says the air check was **not performed**.
10. The rendered report contains neither `PASS` nor `FAIL`; `main()` returns exit code 0 whether or not
    a cell is flagged; the input files are byte-identical before and after a run (compare sha256).
11. With no strike bundles at all, and with only 2 of 4 spokes present, the tool runs and says so.

## Verification required
Run all of these in your worktree `W` and paste the verbatim tails.
- Before you change anything, and again at the end:
  `"$USERPROFILE/.platformio/penv/Scripts/pio.exe" test -d "$W/autonomous_truing_machine" -e native`
  expect 228/228 both times. `python tools/b30_verdict.py --selftest` must still print its `selftest ok`
  line, unchanged.
- `python tools/b32_crossspoke.py --selftest` -> paste it.
- Mutation checks, three, each run for real with the FAILING output pasted, then restored and passing:
  (a) change the "at least 3 spokes" threshold to 2; (b) make the +-2 Hz comparison exclusive
  (`<` instead of `<=`); (c) count clears instead of distinct spokes.
- Builds: none. Host-side only.

## Data rules
Do NOT run anything against `test/fixtures/acoustic/captures/_campaign` (it is not in your worktree; do
not go looking for it). Do NOT run `pio test -e native_sweep`, which defaults to that directory. The
lead runs the tool on real data first.

## Deliverables
Commits on `wip/b32-crossspoke` only: the tool and its selftest in one commit, then the return report
appended to this file as the final commit. Subject in the repository style, for example
`feat(tools): b32 cross-spoke coincidence report`. Body at most 10 lines: cause, evidence, limits.
No attribution trailer; check `git log -1 --format=%B` after each commit.

## Repeat of the hard limits
No hardware. No record, plans, `src/` or `lib/`. Touch only the one file. No attribution. Stop and
report `BLOCKED` when unsure.

## Return report
Handoff: b32-crossspoke    Base: aa52349    Branch HEAD: 2e49ddd (the tool commit; this report is the final commit on top of it)
Status: DONE

1. Files changed
   autonomous_truing_machine/tools/b32_crossspoke.py | +522/-0 | the one new tool, its selftest, and nothing else
   git diff --stat aa52349..HEAD:
     autonomous_truing_machine/tools/b32_crossspoke.py | 522 ++++++++++++++++++++++
     1 file changed, 522 insertions(+)

2. Acceptance criteria
   1 met  - selftest case 1 (flagged, common 501.0, spokes [3, 5, 17])
   2 met  - selftest case 2
   3 met  - selftest case 3 (both the 500/502/504 flag and the 504.01 non-flag)
   4 met  - selftest case 4
   5 met  - selftest case 5
   6 met  - selftest case 6 (B3.2_exclusions.jsonl ledger)
   7 met  - selftest case 7 (three sub-cases: width, candidate, station; cell keys asserted distinct)
   8 met  - selftest case 8 (air_shot, no_fire, and B3.0 strike rows never counted)
   9 met  - selftest cases 9a-9f (3/6 matched, 2/6 no line, 503.5 listed not matched, RIGHT/60 only, below min SNR, no lines CSV -> "not performed")
   10 met - selftest case 10 (no PASS/FAIL in output, main() rc 0 flagged and unflagged, sha256 of every input file identical before/after)
   11 met - selftest case 11 ("no B3.2 strike bundles"; "strike data from 2 of 4 exploration spokes")
   All asserted in selftest() of the tool; the report itself is rendered only after the same analyse() the cases assert on.

3. Commands run
   Baseline, in worktree C:/Users/shomb/wt/b32-crossspoke, before any change:
     pio.exe test -d "C:\Users\shomb\wt\b32-crossspoke\autonomous_truing_machine" -e native
       ... native test_proto_session PASSED / test_proto_wire PASSED
       ================ 228 test cases: 228 succeeded in 00:00:44.540 ================
     python tools/b30_verdict.py --selftest -> selftest ok: pass/fail/inconclusive, ... exclusions
   After:
     python tools/b32_crossspoke.py --selftest ->
       selftest ok: flag needs 3 distinct spokes within +-2 Hz (inclusive, ties to the lowest centre), common = median, one spoke counted once, rejected/ledgered rows and non-strike bundles never count, cells never mix station/width/candidate, air lines at the same station and width only (from the baseline-profile lines CSV), no PASS/FAIL, exit 0, inputs untouched, empty and partial data reported
     python tools/b30_verdict.py --selftest -> unchanged selftest ok line
     pio.exe test ... -e native ->
       ================ 228 test cases: 228 succeeded in 00:00:40.848 ================
   Baseline before: 228/228    After: 228/228

4. Mutation checks (each run for real, failing output verbatim, then restored and passing)
   (a) threshold 3 -> 2 (FLAG_SPOKES = 2):
       AssertionError: case 2: only 2 within 2 Hz of any centre must not be flagged   (exit 1)
   (b) inclusive -> exclusive (abs(f - c) <= tol -> < tol):
       AssertionError: case 3: 500.0/502.0/504.0 spans exactly +-2 Hz: flagged   (exit 1)
   (c) count clears instead of distinct spokes (len(members) in place of len(spokes)):
       AssertionError: case 4: 4 clears from 2 spokes must not be flagged   (exit 1)
   All three restored; final selftest prints the selftest ok line above, exit 0.

5. Reuse search
   Searched for load_bundles / parse_lines_csv / coherent_lines / LINE_TOL_HZ implementations and for the
   sys.path.insert pattern. Reused b30_verdict.load_bundles (bundles + exclusion ledger), b30_verdict.parse_lines_csv,
   b30_verdict.coherent_lines (air-shot coherence, 50% rule), b30_verdict.LINE_TOL_HZ (both the +-2 Hz coincidence
   band and the air-line match band). Imitated the shape of tools/b30_verdict.py (argparse, main(), selftest()
   with synthetic temp fixtures, stdlib only). Sweep row format taken from test/test_acoustic_sweep/test_main.c
   (read only, never run). Nothing reimplemented.

6. Decisions I took (all within "You may decide": internal structure, report wording, fixture building)
   - Common frequency = statistics.median over the chosen centre's member f1 values (standard median; an even
     member count would average the two middle values - no test case has an even count).
   - The exclusion ledger also excludes ledgered air shots from the air-line captures (the ledger is a bundle
     exclusion; load_bundles returns it for exactly that use, as b30_verdict.evaluate skips ledgered bundles).
   - Air shots are selected by campaign_kind == "air_shot", station and width only (no stage filter); the
     governing text says "that station's air shots at the same width" and B3.0 air shots at 40-85 ms widths do
     not collide with the reported cells in practice. Flagging for the lead's review.
   - Width = int(round(pulse_ms)) as a float pulse_ms such as 72.0; a strike bundle missing station, pulse_ms or
     spoke_id cannot form a cell and is not counted.
   - The report lists every cell that has at least one clear (flagged and not flagged) plus per-station spoke
     coverage, since a bare flagged-only list would not show the partial-coverage statement criterion 11 needs.

7. Escalations
   none

8. Assumptions
   - A B3.2 strike bundle that lacks campaign_station, pulse_ms or spoke_id is not counted (it cannot form a
     cell); the registered data shape always has these fields.
   - Sweep rows for bundles absent from the bundle directory are reported in the report as not counted, not
     silently dropped.

9. Deviations from the handoff
   none

10. What I think needs the lead's review
   - The tool has never seen real data; per the handoff the lead runs it on the campaign data first.
   - The no-stage-filter choice for air-shot selection (decision above) is worth a glance against the real
     bundle directory, where B3.0 and B3.2 air shots share the directory.

11. Anything in the handoff that was wrong, ambiguous or missing
   none
