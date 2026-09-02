# Hex fleet notes — FieldMesh for LilyGo T-Echo

Fork of [TogeriX-hub/FieldMesh](https://github.com/TogeriX-hub/FieldMesh) (MIT), which is itself a
MeshCore fork adding GPS auto-advert — the feature that makes a T-Echo usable as a standalone
tracker. Used by the airsoft field app (`erichexter/airsoft-app`).

## Why this fork exists

`trk 1` was running FieldMesh **e9020de** (upstream `main`, 2026-04-28) and **had no way to power
off at all**:

- `hibernate` does not exist in that tree — the string appears once, in a TODO comment on an
  unrelated variant.
- `poweroff` / `shutdown` **do** exist in `src/helpers/CommonCLI.cpp`, but `CommonCLI` is
  instantiated only by `simple_repeater`, `simple_room_server` and `simple_sensor` —
  **not by `companion_radio`**, which is what a T-Echo player/tracker radio runs.

So the buttons offer GPS / messages / settings and nothing else, and there is no command path
either. The only ways to stop the radio were to flatten the battery or open the case.

## What this fork does about it

**`main` here is upstream's `update/v5.4` line, not upstream `main`.** That branch is 274 commits
ahead, 0 behind — a clean fast-forward — and syncs to MeshCore v1.16.0. It already fixes this:

| | upstream `main` (e9020de) | here (`6cd16c1`) |
|---|---|---|
| companion power-off | **none** | reachable `SHUTDOWN` page → `UITask::shutdown()` (needed the port below) |
| T-Echo companion UI | ui-orig | **ui-new**, with `UI_GPS_PAGE` and `UI_SENSORS_PAGE` |
| low-battery cutoff | — | `AUTO_SHUTDOWN_MILLIVOLTS=3300` |
| MeshCore base | pre-1.16 | v1.16.0 |

That rebase was necessary but **not sufficient**, and it took two flashes to find out:

- `ui-new` (what the T-Echo companion envs use) declares `_shutdown_init` and checks it in
  `HomeScreen::poll()`, but **nothing ever sets it true**. The `SHUTDOWN` page and its `KEY_ENTER`
  handler exist only in `ui-tiny`. So `powerOff()` was in the binary and unreachable — the only live
  path was the `AUTO_SHUTDOWN_MILLIVOLTS` low-battery cutoff. Confirmed on the device: every screen
  cycles, none is a power screen.
- Switching the envs to `ui-tiny` gave a working power-off and **broke every screen**. ui-tiny's
  layout targets a small panel: on the T-Echo's 200×200 e-ink the status bar is drawn through by
  double-size text (`MSG` at y=10, `BATT` at y=19, glyphs ~16 px tall on a grid meant for size-1),
  and its 32×32 icons are too small to tell advert from power apart.
- ui-tiny also **would not compile**: `AbstractUITask::newMsg` gained `is_favorite` and
  `channel_name` and ui-tiny was never updated, so it no longer satisfied the interface and
  `main.cpp` could not instantiate `UITask`. Fixed here regardless — that is a real upstream bug.

**The fix that worked** (`9404ec3f`): keep `ui-new`, which is display-size aware — its SOS page
branches on `display.height() >= 128` for icon and text size — and port in only the missing page.
Three additions: `SHUTDOWN` in the enum, a render branch, and the `KEY_ENTER` case that sets
`_shutdown_init`.

The page is **text-led, not icon-led** — `POWER` / `OFF` at size 3 with a `hibernate: long press`
hint, and `HIBERNATING` once armed — because the icons were indistinguishable at this resolution.

**Verify firmware by scanning the built binary for its strings, not by a green build.** One build in
this sequence silently produced ui-tiny again after a `git checkout platformio.ini` restored the
committed env rather than the intended one; only the binary scan caught it.

## Build

`.claude/skills/meshcore-firmware-build` in `airsoft-app` has the full procedure. The env for a
player or tracker radio is:

    LilyGo_T-Echo_companion_radio_ble     # phone-paired
    LilyGo_T-Echo_companion_radio_usb     # serial/bench

## Two traps, both already paid for

1. **FieldMesh defaults to EU 868** (`LORA_FREQ=869.618`, `SF=8`). The fleet runs
   **910.525 MHz / BW 62.5 / SF7 / CR5, channel 0**. Every radio must match *exactly* or they
   silently never hear each other, with no error anywhere. Set the region in the same serial
   session that sets the advert name.
2. **A T-Echo on MeshCore enumerates as `239a:8029` / `NRF52_DK`** — identical to a bare Nordic dev
   board and identical across every build. Only the USB serial number tells two units apart. Probe
   before flashing; the fleet register is `docs/mesh-fleet.md` in `airsoft-app`.
