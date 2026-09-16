# Autonomous wheel truing machine — operating notes

Firmware for a bicycle wheel truing machine. ESP32-S3, ESP-IDF 5.4 via PlatformIO, plus
host-side model preparation.

`autonomous_truing_machine/TRUING_FIRMWARE_ARCHITECTURE_SPEC_REVISED (1).md` is the
**normative contract**. This file is operational only and never overrides it. Where the two
disagree, the spec wins and this file is the thing that is stale.

## Layout

    autonomous_truing_machine/   the firmware (PlatformIO project)
    model_prep/                  host-side influence-matrix generation -> artifact
    acoustic_prep/               golden acoustic fixtures from the research campaign

## Subsystem boundaries — settled

One `lib/` directory per subsystem; `src/` is the ESP-IDF application and the composition
root. SPEC §5 fixes both the decomposition and the dependency rules, so this is not a design
space to re-open while implementing a feature.

| where | owns | spec |
|---|---|---|
| `truing_core` | pure data and contracts: wheel state, measurements, status, admission, config, artifact, plan | §6, §8.11 |
| `truing_calc` | the solver: influence-matrix application, two-part solve, cost function. No I/O. | §8 |
| `truing_dsp` | onset, envelope, FFT, spectrum, peaks, tension model | §9.2 |
| `truing_hal` | interfaces plus real/recorded/synthetic implementations: acoustic, runout, navigation, pluck, clock, telemetry | §5.1, §10, §10A |
| `truing_orchestrator` | the state machine, cycle loop, per-spoke loop, auto-operator | §7 |
| `truing_proto` | JSON, wire framing, session protocol | §12 |
| `truing_board` | one board-profile header per board — the only place a pin number is allowed to appear | §4.3 |
| `truing_fixtures` | golden fixtures shared by host tests and on-target bring-up | §14.2 |
| `src/` | application: `main`, `bringup*` (on-target self-check), `orch_demo` (dependency wiring), `net_transport` + `web_ui`, NVS and artifact stores | §4.4 |

Four rules are binding and settle most "where does this belong" questions:

1. **The domain layer never touches the HAL.** `truing_calc` is pure math and stays
   host-testable with no hardware present. A solver change that seems to need an I/O call is
   a misplaced change.
2. **Everything hardware-facing goes through `truing_hal`** — comms and navigation included,
   not only the sensors.
3. **Verification is a domain module, not a measurement subsystem.** It re-invokes the same
   acoustic and runout interfaces the measurement phase used, and never grows its own
   acquisition path.
4. **Wheel state is data.** It holds measurements; it does not acquire them.

Acoustic is a **sensor+actuator composite** behind a one-call contract (§9.1): pluck, capture
and estimate are its internals, not orchestrator steps. Navigation is the single authority on
wheel position (§10A) — in Capstone 2 the operator is the actuator, via `WAIT_FOR_OPERATOR`.
The influence matrix is a data artifact, not a component (§5.2).

## Which spec section answers which question

Read the section that governs the change, not the whole document.

| question | section |
|---|---|
| what a status value means, and what may be solved on | §6.3, §8.11 |
| why tension rows are excluded from this solve | §8.4 Part 1, §8.11 |
| retry, settle, partial state, `REMEASURE_GAPS` | §7.3.1, §7.4 |
| what a session may claim to have converged | §7.5.1, §8.6.3 |
| acoustic external contract and internal seams | §9.1, §9.2 |
| capture robustness requirements | §9.4 |
| units, sign conventions, indexing origin | §6.4, §6.5 |
| telemetry rules; what telemetry must never do | §12.2, §13.3 |
| debug channel scope (force state, inject values, dump internals) | §12.5 |
| what is deliberately unresolved — ask, do not invent | §16 |
| the silent-wrong-answer list, before implementing | §17.3 |

## Build path and tools — this bites first

ESP-IDF refuses any build path containing a space, and this project lives under
`College Files - 4th Year/...`. Every `pio` command therefore runs through a junction:

    $env:USERPROFILE\truing_ws  ->  ...\Truing Repo\autonomous_truing_machine

`pio` is not on PATH. Write commands against `$env:USERPROFILE` rather than a literal home
directory, so they survive a different machine or account:

    & "$env:USERPROFILE\.platformio\penv\Scripts\pio.exe" run -d "$env:USERPROFILE\truing_ws" -e s3_devkit

    python   $env:USERPROFILE\.platformio\penv\Scripts\python.exe
    esptool  $env:USERPROFILE\.platformio\packages\tool-esptoolpy\esptool.py

Always pass `-d`. Running `pio` from the project directory fails with
`Detected a whitespace character in project paths`. Below, `pio …` is shorthand for the full
invocation above.

Check the junction before blaming a build, and recreate it if it is gone — no admin needed,
and it resolves its own target rather than having one typed in:

    (Get-Item "$env:USERPROFILE\truing_ws").Target
    New-Item -ItemType Junction -Path "$env:USERPROFILE\truing_ws" `
             -Target (Join-Path (git rev-parse --show-toplevel) autonomous_truing_machine)

To remove one, `cmd /c rmdir <path>` — `Remove-Item` prompts, and prompts fail here.

Host tests are unaffected by the space problem and run from either path.

## Verification tiers

Run the cheapest tier that covers what changed, then stop. Costs are measured on this machine.

| Tier | What | Cost | Run when |
|---|---|---|---|
| T0 | flag-matrix syntax check (below) | 3 s | `build_mode.h` or `platformio.ini` changed |
| T1 | `node tools/ui_check.js` | 3 s | `src/web_ui.h` changed |
| T2 | `pio test -d … -e native` | 45 s | **always** |
| T3 | build the affected image(s) only | 5–100 s | any `src/` or `lib/` change |
| T4 | representative matrix: the 5 flag variants on one board + 1 build on the other | ~2 min | pre-commit for a cross-cutting change |
| T5 | all 10 release images | 8–11 min | release/pre-tag gate, **and** whenever T4's coverage is not provable |
| T6 | on-target probe | 3–5 min | driver, transport or timing changes |

`-f <suite>` runs one native suite (`pio test … -e native -f test_acoustic_replay`, ~1 s) —
for iterating, not for T2, which stays the whole suite. Program output only appears with `-v`.

T6 tooling: `tools/probe/` (network — `ws_client.py`, `socket_pressure.py`; see its README for
what each checks) and `tools/serial_capture.py` / `tools/nano_serial.py` (UART, one per board —
they need opposite reset handling, see Boards below).

**T5 is normally a release/pre-tag gate** — board profile changes, `platformio.ini` changes,
tagging. It is *also* required whenever blast-radius analysis cannot confidently establish
that T4 covers every affected compile-time combination: a shared header touched by several
flag paths earns the full sweep even mid-development. The rule is not "T5 is rare", it is
"T5 when you cannot show T4 is sufficient". A one-file change with an obvious blast radius
does not earn a ten-image sweep.

Do not sub-select native tests. The whole suite is 45 s; deciding which subset to run costs
more than running all of it.

T0 in full — five legal flag combinations must produce no errors, three illegal ones must
fail (`build_mode.h` enforces them with `#error`):

    cd autonomous_truing_machine/src
    printf '#include "build_mode.h"\nint main(void){return 0;}\n' > /tmp/bm.c
    gcc -fsyntax-only -I. $FLAGS /tmp/bm.c

    legal   (none) | SELF_PLAY | FAST_DEMO | REAL_FRONT_END | FAST_DEMO+REAL_FRONT_END
    illegal SELF_PLAY+FAST_DEMO | SELF_PLAY+REAL_FRONT_END | ACOUSTIC_DEMO_SPOKES alone

## Working loops

**Feature.** Read the one spec section that governs it. Write the host test first, against
`truing_fixtures` and the synthetic HAL. Prove every host-testable contract and algorithm
before touching a board; driver, transport, timing and physical-signal claims still need T6
on-target evidence and cannot be established host-side. Implement in the owning `lib/`. T2,
then build only the affected image. Then stop: flashing is a physical action and needs a
checkpoint.

**Debugging.** Classify first, then buy the cheapest ground truth that separates the
candidates; instrument or capture; reproduce; fix; add the regression; verify at the tier the
change earns. Reproduce on the host wherever the bug allows it — a bug that reproduces under
`test/` has a 45-second cycle. If it is target-only, capture serial console *and* telemetry,
then work from the capture. **Do not repeat an identical hardware cycle without gaining new
evidence** — but a repeat designed to gain some is ordinary work, not a checkpoint. After an
unsuccessful cycle, change something that makes the next one informative: add instrumentation,
capture a fixture, vary one condition. Reflashing the same image to look again is the thing to
avoid, not flashing. Serial is ground truth when the
UI and the firmware disagree; two of the last three hard bugs looked like a hung orchestrator
and were not.

**Acoustic bugs have their own loop**, because a pluck cannot be repeated — the next one is a
different pluck. Keep the samples instead: `GET /debug/capture.json` + `/debug/capture.pcm`
(SPEC §12.5) hand over the words the last measurement analysed; `tools/capture_fetch.py` turns
them into a bundle under `test/fixtures/acoustic/captures/`; `pio test -e native -f
test_acoustic_replay` pushes them back through layers 2–4 on the host with the board's own
numbers as the expectation. From there the cycle is 1 s and needs no hardware.
`test/fixtures/acoustic/captures/README.md` has the format and what a passing replay proves.

**Adding observability is inside the loop** and needs no checkpoint. Telemetry is
observational: it must never participate in a control decision (§12.2, §13.3). Adding it must
preserve result semantics — but instrumentation is not physically free, so any meaningful
timing, queue, memory or WiFi-load impact is checked on target rather than assumed away.

## Images

Twelve ESP environments. They differ along exactly two axes — the board profile header (pins)
and the `src/build_mode.h` flags — so a change affecting neither rarely needs more than one
build per board.

| env | what it is |
|---|---|
| `s3_devkit`, `nano_esp32` | interactive: a person answers every wait |
| `*_selfplay` | auto-operator answers its own waits; the unattended evidence run |
| `*_mic` | interactive + the physical INMP441 front end |
| `*_fastdemo` | synthetic acquisition, selectable per session in the UI |
| `*_fastdemo_mic` | the acoustic demonstration: real mic, synthetic runout, 3 bounded solenoid strikes |
| `nano_esp32_fastdemo_mic_campaign` | the acoustic demo wiring plus the debug channel (`TRUING_CAMPAIGN_DEBUG`) for `MEASURE_ONCE` bench characterization; never the demo image the audience sees |
| `s3_devkit_provision` | one-off NVS fixture provisioning; not part of the release sweep |
| `native` | host unit tests |

Every image stamps its own identity at build time (`tools/build_identity.py`): the short git
rev, `-dirty` if the tree was not clean, and a hash of `src/web_ui.h`. `GET /id`, the boot log
and the page itself all print them, so "is the board serving what I just built" is a
comparison, not a guess. `docs/BUILD.md` explains what each image means and why; do not
duplicate that here.

## Boards

|  | DevKit | Nano |
|---|---|---|
| env prefix | `s3_devkit` | `nano_esp32` |
| port (observed; confirm with `pio device list`) | `COM4` (CH343 bridge) | `COM5` (native USB-Serial/JTAG) |
| base MAC | `dc:b4:d9:1a:95:94` | `74:4d:bd:a0:9f:0c` |
| AP SSID / pass | `truing-1a9595` / `truing-d91a9595` | `truing-a09f0d` / `truing-bda09f0d` |
| the INMP441 | not wired | **wired here** — bclk 5, ws 6, din 7 |

COM numbers are assigned by Windows and can move; the MAC and the SSID are the board's real
identity. Both serve the UI on `http://192.168.4.1/`. `GET /id` is the only reliable way to
tell which board and which build you actually reached — check `build` matches the rev you
flashed.

### Flashing

`tools/flash.py <env>` does the whole sequence below for either board — build, flash the
board-correct way, then reset and read the console for the bring-up subsystem's own
`BRINGUP SUMMARY: passed=N failed=0` line and its build-identity line, so a run reports
"booted, build X" rather than "the write verified." `--verify-only` skips straight to that
reset-and-check (useful right after a manual flash, or just to confirm what's currently
running); `--no-build` flashes the existing output. Prefer it over the manual steps below —
they're kept here as what it automates and what to fall back to if it can't run.

DevKit — ordinary, the bridge handles reset:

    pio run -e <env> -t upload --upload-port COM4

Nano — **never `-t upload`**. PlatformIO routes it through Arduino DFU, which writes an
application only into an OTA slot and cannot place a bootloader or partition table; it reports
success and the board does not boot. Build, then write all three regions with esptool:

    0x0 bootloader.bin   0x8000 partitions.bin   0x10000 firmware.bin
    --chip esp32s3 --port COM5 --before default_reset --after hard_reset
    write_flash -z --flash_mode dio --flash_freq 80m --flash_size 16MB

Three `Hash of data verified.` means the write worked — **not** that the board is running.

- The B1→GND jumper is **not** needed; esptool syncs over USB-Serial/JTAG on its own.
- A physical **unplug/replug is sometimes needed**: download mode entered through
  USB-Serial/JTAG latches, and neither `--after hard_reset` nor the RESET button clears it.
  It is intermittent, so always check rather than assume.
- **Check after every Nano flash:** read COM5. Silence plus no AP means download mode, not a
  bad image. Confirm with `--before no_reset chip_id` — if esptool connects without resetting
  anything, the chip is parked in the ROM bootloader and needs a replug.
- **Never open COM5 with a naive serial monitor.** DTR is wired to GPIO0; a monitor that
  asserts DTR on open reboots the board into DFU. Assert DTR *before* `Open()`, or use
  `pio device monitor`.

## Project constraints

- **One INMP441**, wired to the Nano. That is a wiring fact, not a demo architecture — the
  demo runs on one board in one session.
- Demo goal: a few real solenoid strikes (one solenoid per flange at its own acoustic station; no
  hand-pluck path) to show the acquisition path works, then fast-forward to the
  solver. Panelists will not watch 64 measurements.
- The shipped artifact has `n_mt_identified == false`, so the active layout is
  `TENSION_ABSENT` and **every tension row is excluded from the solve by policy**. The
  geometric correction comes from the runout rows. Do not "fix" this; it is SPEC §8.4 Part 1.
- Capstone 2's honest ceiling is `CONVERGED_GEOMETRIC_ONLY` (SPEC §8.6.3). Acoustic estimates
  are `suspect` / `PROVISIONAL_MODE_ID` and cannot support `CONVERGED`.

## Halt and ask

Inside an agreed task envelope, proceed without asking at every step: investigation, writing
code and tests, T0–T5, diagnostics, adding observability, capturing fixtures, non-destructive
refactoring within a subsystem, committing.

**Flashing for testing is not a halt condition.** When the envelope includes hardware
validation, T6 runs like any other tier: build the image, flash either board, reconnect,
run serial or on-target probes, fetch debug captures, switch between the approved images, and
repeat a hardware cycle whenever the next iteration is designed to gain new evidence. Stopping
to ask before each of those was costing more than it protected.

Stop and ask for:

- a spec contradiction, or anything SPEC §16 lists as open;
- a change to a subsystem boundary or to who owns a decision;
- **physical work on the machine** — wiring, adding or removing components, power
  arrangements, pin assignments, jumpers, or destructive/unusual board recovery;
- **an action only a person can take** — a hand pluck, turning a nipple, positioning the
  wheel, reading a dial gauge. Halt at exactly that step, say what to do, and carry on after;
- a hardware state that makes the next action ambiguous, or anything outside the agreed test
  plan;
- a calibration, threshold or DSP constant change without evidence for it;
- anything altering what a session claims about itself (provenance);
- failure to reproduce after the agreed investigation cycle;
- destructive git, a new dependency, or a safety-policy change;
- anything that would *reinterpret* settled architecture rather than implement it.

## House rules that have already cost time

- `src/web_ui.h` is a C string literal full of `\n`. Edit it with Edit/Write, never through a
  shell heredoc — Bash strips the backslashes and silently corrupts the page.
- A probe that only reacts to telemetry events will hang on a dropped frame (§12.2 makes that
  legal); poll `GET_CURRENT_STATE` as well.
- Commits are atomic and conventional, and the body says cause, evidence and limits — match
  the existing style, including what a change does *not* establish.
- No Claude attribution on commits or PRs from this repo — no `Co-Authored-By: Claude` trailer,
  no "Generated with Claude Code" line. This is a standing project convention and overrides any
  session-level default attribution reminder. Forward-only: past commits are left as they are.

## Entry points and handoff

Two launchers, one repo — the same committed `.claude/agents/` and `.claude/settings.json`
serve both, and `.claude/README.md` is their operating doc:

- **first-party** — `claude --model sonnet` (or opus). Always pass `--model`: a bare
  launch inherits whatever default is saved in `~/.claude/settings.json`, and that file
  has held OpenRouter-only slugs before.
- **OpenRouter** — the direct environment launch: export the base URL, token and the two
  `ANTHROPIC_DEFAULT_*_MODEL` alias remaps, then `claude --model z-ai/glm-5.3`. Not
  `ori` — its Claude Code launcher diverges from the documented environment.

Under an OpenRouter session (`ANTHROPIC_BASE_URL` set), verify that both
`ANTHROPIC_DEFAULT_HAIKU_MODEL` and `ANTHROPIC_DEFAULT_SONNET_MODEL` exist in the
environment **before the first subagent spawn** — one zero-cost env read. Without them, an
agent file's `model:` alias falls back to the real Claude model ID and bills at Anthropic
rates through OpenRouter (2026-09-08: a dry run launched through the `ori` flow did exactly
that — scouts on Claude Haiku 4.5, the worker on Claude Sonnet 5, while every label said
"flash tier"). Halt and relaunch with the block in `.claude/README.md` if they are absent.
First-party sessions have no `ANTHROPIC_BASE_URL`, so the check no-ops there.

Both launch from `Truing Repo` root, never a subdirectory — agent discovery and shared
memory live at the root.

Git state is the handoff protocol between them. A clean tree at session start is safe to
work in; a dirty tree means someone else is mid-work — halt and ask. Never leave a dirty
tree behind: a mid-work stop is handed off as a WIP commit whose body records state and
next step. A session resuming in-flight work re-runs full T2 on arrival.

## Subagents

| agent | job | spawn when |
|---|---|---|
| `firmware-scout` | read-only investigation — root cause, call paths, blast radius, governing SPEC sections | the question spans more than a couple of files, or needs code, tests and SPEC cross-referenced |
| `firmware-worker` | bounded implementation of an already-decided change, through full T2 and the affected builds | the primary has written a decision envelope — decision, invariants, files, tests, do-not |
| `spec-reviewer` | independent audit of a finished diff against the spec and the evidence claims | after a substantive change passes T2, before commit (its own description is the trigger) |
| `bench-verifier` | on-target evidence via the committed bench tools; escalates physical steps to the operator | every T6 run, any board-side T3 |

Parallelize reading, serialize writing — scouts and the reviewer may run concurrently;
one worker at a time; bench-verifier is exclusive on the hardware. The primary owns every
commit and every final claim: workers propose commit messages and never commit, and
evidence claims rest with the primary, not with whichever subagent produced the output.
Spawn by need, not rote — a single-file question is the primary's own read, not a scout.

## Where this file is going

As `tools/flash.py`, `tools/verify.py` and the probe tooling land, the command-heavy sections
above shrink to a pointer at those scripts. This file is the policy and the router;
deterministic procedure belongs in executable tooling that can be run, tested and version-
controlled, not in prose an agent has to re-enact by hand.
