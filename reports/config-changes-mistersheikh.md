# MisterSheikh/Qidi_Q2_Mainline_Klipper — config_changes.md feedback

**Project**: [MisterSheikh/Qidi_Q2_Mainline_Klipper](https://github.com/MisterSheikh/Qidi_Q2_Mainline_Klipper)
**File**: `docs/config_changes.md`

Overall this doc was accurate and made the mainline migration much
smoother than expected — every documented section change (MCU serial
paths, `[stepper_z]`/`[stepper_z1]` reverse-homing removal, MAX6675
`spi_bus` mapping, `[load_cell_probe]` replacing `probe_air`, chamber
`z_max_limit` removal, `sense_resistor`/`rref` additions) matched the
live printer exactly, with no surprises. A few gaps found while
following it end-to-end on a printer with the Box V2 multi-material
unit and OrcaSlicer (not just the touchscreen) as the client:

## 1. No mention of the `.3mf`/vendor-network-agent print pipeline

The doc's section 2.3 ("Stock macro audit") correctly flags
`BUFFER_MONITORING`/`box_extras` objects, `DISABLE_BOX_HEATER`, and
`CLEAR_LAST_FILE`/`save_last_file` as things to audit — but doesn't
mention that **OrcaSlicer's Qidi vendor printer profile always uploads
a `.3mf` project file**, not plain `.gcode`, and calls a
Moonraker-fork-specific `start_print` API that extracts the embedded
gcode server-side. If a user keeps Qidi's own (patched) Moonraker fork
after migrating only Klipper itself (a very plausible partial-migration
state, and the one this migration initially landed in), the first print
attempt via OrcaSlicer fails with `Unable to open file` — mainline's
stock `virtual_sdcard.py` has no `.3mf`/`.temp/` redirect logic and
filters its file listing to `gcode`/`g`/`gco` only.

This is a real, guaranteed-to-be-hit issue for anyone using OrcaSlicer
against a still-Qidi-Moonraker-fork setup, and the fix (porting stock's
`virtual_sdcard.py` behavior, or — simpler — switching OrcaSlicer to a
generic Klipper/Moonraker physical printer connection instead, see the
`orcaslicer-qidi-vendor-profile.md` report in this same feedback set)
is nonobvious enough that it's worth a paragraph. Happy to contribute
the writeup.

## 2. `UNLOAD_FILAMENT`/timelapse-macro gaps not covered

Section 2.3's audit list is a good start but isn't exhaustive — in
practice, the two "Unknown command" errors actually encountered after a
full migration were `UNLOAD_FILAMENT` (called from OrcaSlicer's
`machine_end_gcode`, defined in stock `box.cfg`, silently missing once
box.cfg's include is dropped or replaced) and `TIMELAPSE_TAKE_FRAME`
(called from OrcaSlicer's per-layer gcode regardless of any Moonraker
timelapse config — easy to accidentally break if `timelapse.cfg`'s
*macro* file gets removed instead of just disabling it via its own
`variable_enable` flag). Neither is Qidi-box-specific, so both would
trip up a non-Box Q2 owner too. Might be worth adding to the "Calls
associated with omitted Qidi ... components" list in 2.3, or a short
"commonly-missed slicer-side macro calls" callout.

## 3. Box V2 (multi-material) migration not covered at all

The doc is explicitly scoped to the base printer config
(`printer.cfg`/`gcode_macro.cfg`), which is reasonable, but Box V2
owners following this guide hit a much bigger wall immediately after:
`box.cfg`/`box1.cfg` reference several Qidi-only section types
(`[box_stepper slotN]`, `[box_extras]`, `[box_rfid]`,
`sensor_type: AHT20_F`) with **no mainline equivalent at all** — a
completely different category of gap than anything in the base
printer's config. A short note pointing Box owners toward either Happy
Hare (Camden-Winder's `install-bb-q2.sh`, if full multi-slot
tool-changing is wanted) or reusing the stock compiled binaries
directly (see the `n3oney-qidi-q2-klipper.md` report in this same
feedback set, for basic single-slot hardware control without Happy
Hare) would save a lot of independent discovery. Happy to draft this
section if useful — we went through the full discovery process on real
Box V2 hardware and have working notes for both paths.
