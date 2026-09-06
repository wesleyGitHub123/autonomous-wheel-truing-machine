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

## Build path — this bites first

ESP-IDF refuses any build path containing a space, and this project lives under
`College Files - 4th Year/...`. Every `pio` command therefore runs through a junction:

    C:\Users\shomb\truing_ws  ->  ...\Truing Repo\autonomous_truing_machine

Always pass `-d`. Running `pio` from the project directory fails with
`Detected a whitespace character in project paths`.

    pio run -d C:\Users\shomb\truing_ws -e <env>

If the junction is missing, recreate it from an ordinary `cmd` (no admin needed):

    mklink /J C:\Users\shomb\truing_ws "<abs path>\Truing Repo\autonomous_truing_machine"

Absolute tool paths (PATH is not reliable here):

    pio      C:\Users\shomb\.platformio\penv\Scripts\pio.exe
    python   C:\Users\shomb\.platformio\penv\Scripts\python.exe
    esptool  C:\Users\shomb\.platformio\packages\tool-esptoolpy\esptool.py

Host tests are unaffected by the space problem and run from either path.

## Verification tiers

Run the cheapest tier that covers what changed, then stop. Costs are measured on this machine.

| Tier | What | Cost | Run when |
|---|---|---|---|
| T0 | flag-matrix syntax check (below) | 3 s | `build_mode.h` or `platformio.ini` changed |
| T1 | `node tools/ui_check.js` | 3 s | `src/web_ui.h` changed |
| T2 | `pio test -d C:\Users\shomb\truing_ws -e native` | 45 s | **always** |
| T3 | build the affected image(s) only | 5–100 s | any `src/` or `lib/` change |
| T4 | representative matrix: the 5 flag variants on one board + 1 build on the other | ~2 min | pre-commit for a cross-cutting change |
| T5 | all 10 release images | 8–11 min | **release gate only** — board profile, `platformio.ini`, or pre-tag |
| T6 | on-target probe | 3–5 min | driver, transport or timing changes |

**T5 is not a pre-commit gate.** A one-file change does not earn a ten-image sweep. Escalate
to T4 when unsure which images are affected; escalate to T5 only for the reasons listed.

Do not sub-select native tests. The whole suite is 45 s; deciding which subset to run costs
more than running all of it.

T0 in full — five legal flag combinations must produce no errors, three illegal ones must
fail (`build_mode.h` enforces them with `#error`):

    cd autonomous_truing_machine/src
    printf '#include "build_mode.h"\nint main(void){return 0;}\n' > /tmp/bm.c
    gcc -fsyntax-only -I. $FLAGS /tmp/bm.c

    legal   (none) | SELF_PLAY | FAST_DEMO | REAL_FRONT_END | FAST_DEMO+REAL_FRONT_END
    illegal SELF_PLAY+FAST_DEMO | SELF_PLAY+REAL_FRONT_END | ACOUSTIC_DEMO_SPOKES alone

## Images

Eleven ESP environments. They differ along exactly two axes — the board profile header (pins)
and the `src/build_mode.h` flags — so a change affecting neither rarely needs more than one
build per board.

| env | what it is |
|---|---|
| `s3_devkit`, `nano_esp32` | interactive: a person answers every wait |
| `*_selfplay` | auto-operator answers its own waits; the unattended evidence run |
| `*_mic` | interactive + the physical INMP441 front end |
| `*_fastdemo` | synthetic acquisition, selectable per session in the UI |
| `*_fastdemo_mic` | the acoustic demonstration: real mic, synthetic runout, 3 bounded plucks |
| `s3_devkit_provision` | one-off NVS fixture provisioning; not part of the release sweep |
| `native` | host unit tests |

`docs/BUILD.md` explains what each image means and why. Do not duplicate that here.

## Boards

|  | DevKit | Nano |
|---|---|---|
| env prefix | `s3_devkit` | `nano_esp32` |
| port | `COM4` (CH343 bridge) | `COM5` (native USB-Serial/JTAG) |
| base MAC | `dc:b4:d9:1a:95:94` | `74:4d:bd:a0:9f:0c` |
| AP SSID / pass | `truing-1a9595` / `truing-d91a9595` | `truing-a09f0d` / `truing-bda09f0d` |
| the INMP441 | not wired | **wired here** — bclk 5, ws 6, din 7 |

Both serve the UI on `http://192.168.4.1/`. `GET /id` is the only reliable way to tell which
board and which build you reached — check `build` matches the rev you flashed.

### Flashing

DevKit — ordinary, the bridge handles reset:

    pio run -d C:\Users\shomb\truing_ws -e <env> -t upload --upload-port COM4

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
- Demo goal: a few real plucks to show the acquisition path works, then fast-forward to the
  solver. Panelists will not watch 64 measurements.
- The shipped artifact has `n_mt_identified == false`, so the active layout is
  `TENSION_ABSENT` and **every tension row is excluded from the solve by policy**. The
  geometric correction comes from the runout rows. Do not "fix" this; it is SPEC §8.4 Part 1.
- Capstone 2's honest ceiling is `CONVERGED_GEOMETRIC_ONLY` (SPEC §8.6.3). Acoustic estimates
  are `suspect` / `PROVISIONAL_MODE_ID` and cannot support `CONVERGED`.

## Halt and ask

Proceed without checkpoints inside an agreed task: reading, writing code and tests, T0–T5,
committing on a branch, adding observability, capturing fixtures, refactoring within a
subsystem. Stop and ask for:

- a spec contradiction, or anything SPEC §16 lists as open;
- a change to a subsystem boundary or to who owns a decision;
- **any physical action** — flashing, wiring, jumpers, replugging;
- a calibration, threshold or DSP constant change without evidence for it;
- anything altering what a session claims about itself (provenance);
- a bug that will not reproduce after one capture cycle — do not iterate blind on hardware;
- destructive git, a new dependency, or a safety-policy change.

## House rules that have already cost time

- `src/web_ui.h` is a C string literal full of `\n`. Edit it with Edit/Write, never through a
  shell heredoc — Bash strips the backslashes and silently corrupts the page.
- Telemetry is best-effort by contract (SPEC §12.2). A probe that only reacts to events will
  hang on a frame that was dropped; poll `GET_CURRENT_STATE` as well.
- Serial is the ground truth when the UI disagrees with the firmware. Two of the last three
  hard bugs looked like a hung orchestrator and were not.
- Commits are atomic and conventional, and the body says cause, evidence and limits — match
  the existing style, including what a change does *not* establish.
