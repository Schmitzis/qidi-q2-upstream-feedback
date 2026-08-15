# n3oney/qidi-q2-klipper — firmware/build notes

**Project**: [n3oney/qidi-q2-klipper](https://github.com/n3oney/qidi-q2-klipper)

Three findings from a full migration (stock → mainline Kalico on all
three MCUs: mainboard, toolhead, box) on a Qidi Q2 with the Box V2
multi-material unit.

## 1. Box RFID crash — root cause + a working stub firmware patch

`box_count >= 1` on the mainline build crashes (or fails to reach
`ready`, depending on whether `[box_rfid card_reader_1/2]` sections are
present) because `box_extras.so`'s `delayed_init_rfid` unconditionally
tries to read an RFID card a few seconds after every startup, and
mainline firmware has no `fm17550_read_card_cb` command handler at all
(the FM17550 RFID chip was apparently never wired up in this build).

Two failure modes depending on config:
- Sections present: MCU protocol error (`Unknown command:
  "fm17550_read_card_cb"`) — clean failure, blocks reaching `ready`.
- Sections absent: unhandled Python exception in `box_stepper.py`'s
  `cmd_SLOT_RFID_READ` (`configparser.Error: Unknown config object`)
  inside a timer callback — a full Klippy crash.

**Fix**: a stub MCU command handler (not a real FM17550/SPI driver) that
implements the exact wire protocol `box_rfid.so`/`box_stepper.so`
expect (recovered via `strings` on the compiled binaries):

```
config_fm17550 oid=%c spi_oid=%c
query_fm17550 oid=%c rest_ticks=%u
fm17550_read_card_cb oid=%c
fm17550_read_card_return oid=%c status=%c data=%*s
```

Always replies `status=0` (no card), zero-length data — genuinely no
RFID reading happens, this only stops the crash. `box_extras.so`
handles the reply gracefully (logs `"<slot> did not recognize the
filament"` instead of crashing). Modeled on the existing
`sensor_cs1237.c` pattern already in this project's tree (same
`oid_alloc`/`DECL_COMMAND`/`sendf` conventions).

Built against `configs/kalico/mmu.config` (this project's own shipping
Kconfig for the box MCU target, `stm32f401xc` / Katapult app-start
`0x8004000`) — no guessing needed there, which was appreciated; that
config being checked in and easy to find made this fix much faster to
build correctly.

Happy to submit this as a PR (adds one new `src/fm17550_stub.c`, one
line in `src/Makefile`'s `src-y`) if there's interest in shipping a
"no crash, no real RFID" default for Box V2 users who don't need RFID
spool ID. Full source available on request.

## 2. Stock Qidi box driver binaries (`box_extras.so`, `box_stepper.so`,
   etc.) work unmodified against mainline MCU firmware

Not a bug — a positive finding that might be worth a doc mention. These
compiled Cython `.so` files are Qidi's own proprietary additions
(closed source, never open-sourced), bundled into every stock firmware
image. They are **not** Qidi-specific at the MCU wire-protocol level —
stepper motion (`queue_step`), GPIO, and I2C are universal Klipper
commands. The only thing mainline was missing was the *config section
type* (`[box_stepper slotN]`, `[box_extras]`, etc.) — i.e., no `.py`/
`.so` registered that section name.

Copying these binaries straight from a stock `~/klipper/klippy/extras/`
tree into the mainline `~/klipper/klippy/extras/` tree, then restoring
the original (unmigrated) `box.cfg`/`box1.cfg` config sections, gives
**real, working hardware control**: feed motors, button/endstop/buffer
sensing, and the heater — confirmed via live GPIO reads, not just "the
config loads." This might be worth documenting as an option for Box V2
users who want basic hardware control (heater, feed motors, sensing)
without going as far as installing Happy Hare — it's a much smaller
step, though it doesn't get you multi-slot tool-changing (that
orchestration layer genuinely isn't in these binaries).

## 3. `PrinterExtruder.extruder_stepper` → `extruder_steppers` refactor breaks reused binaries mid-print

Kalico's kinematics refactor (`kinematics/extruder.py`) replaced the
old singular `self.extruder_stepper` attribute on `PrinterExtruder` with
a real `self.extruder_steppers` list, to support multiple linked
extruder steppers. `box_extras.so` (reused per point 2 above) still
reads the old singular attribute directly — it's compiled, so it can't
be patched.

Symptom: not caught by any "does it load" test — only fires when the
box's physical filament buffer switch triggers mid-print (a real
feeding event):

```
File "box_extras.py", line 339, in extras.box_extras.BoxExtras.buffer_button_callback
AttributeError: 'PrinterExtruder' object has no attribute 'extruder_stepper'
```

This is an **unhandled exception escaping the reactor loop** — it kills
Klippy outright, not just the current gcode command.

**Fix applied** (on the mainline Klipper/Kalico side, not this repo,
but noting it here since it's directly relevant to anyone reusing these
binaries): added a back-compat `self.extruder_stepper` alias to
`kinematics/extruder.py`, kept in sync with `extruder_steppers[0]` (or
`None`) in `__init__`, `link_extruder_stepper()`, and
`unlink_extruder_stepper()`. Not an invented workaround — Kalico's own
`extras/trad_rack.py` (an MMU extra) still reads
`extruder.extruder_stepper.stepper` too, so the singular attribute is
still a recognized access pattern elsewhere in the same Kalico tree,
just no longer *set* by `PrinterExtruder` itself after the refactor.

If this project ships or recommends a specific Kalico build/branch, it
might be worth carrying this alias in that build directly, since any
user reusing stock Qidi binaries (not just `box_extras.so` — likely any
closed-source extra written against pre-refactor Klipper) will hit the
same crash the first time a real hardware event fires mid-print rather
than at startup.
