# Happy Hare (bunnybox branch) breaks pressure advance on Kalico

**Project**: [Wazzup77/Happy-Hare](https://github.com/Wazzup77/Happy-Hare), `bunnybox` branch
**File**: `extras/mmu_machine.py`
**Severity**: real, reproducible mid-print failure on any `homing_extruder: True` (Type B, no-encoder) setup running Kalico instead of stock Klipper/Danger-Klipper.
**Status**: found, fixed, and verified working on real hardware (Qidi Q2 + Box V2).

## Summary

Two separate incompatibilities in the same file, both stemming from
Kalico's `PrinterExtruder`/`ExtruderStepper` refactor (the move from a
single `extruder.extruder_stepper` attribute to a real
`extruder.extruder_steppers` list, to support multiple linked extruder
steppers). Both are silent until something actually reads the new
list-based API — which is why a print can run for many minutes before
either bug fires.

## Bug 1: `handle_connect()` bypasses the real stepper-registration API

`extras/mmu_machine.py`, in `MmuMachine.handle_connect()`:

```python
# Now we can switch in homing MmuExtruderStepper
printer_extruder.extruder_stepper = self.mmu_extruder_stepper
self.mmu_extruder_stepper.stepper.set_trapq(printer_extruder.get_trapq())
```

This is a **direct attribute assignment**. It's correct for pre-refactor
Klipper, where `.extruder_stepper` (singular) was the only thing that
existed. On Kalico, `PrinterExtruder` has a real
`self.extruder_steppers` list (see `kinematics/extruder.py`), and
`get_extruder_steppers()` — which real motion code
(`cmd_default_SET_PRESSURE_ADVANCE`, at minimum) actually calls — just
returns that list. The direct assignment never touches it, so it stays
empty even after this "registration" runs.

Symptom: mid-print, any `SET_PRESSURE_ADVANCE` call with no explicit
`EXTRUDER=` (the normal case — e.g. from a filament profile's
`filament_start_gcode`, fired again after a redundant slicer-emitted
tool-select) raises:

```
Active extruder does not have a stepper
```

...aborting the print. This is **not** a config bug and not something a
user can work around from `printer.cfg` — it needs a code fix.

### Fix

```python
printer_extruder.link_extruder_stepper(self.mmu_extruder_stepper)
```

`link_extruder_stepper()` (already present in Kalico's
`kinematics/extruder.py`, called by the normal `[extruder_stepper]`
config path) does everything the original two lines did (`set_trapq`,
plus `set_position`) *and* appends to `extruder_steppers` *and* sets the
same `.extruder_stepper` singular attribute for any code that still
reads it directly — a strict superset, drop-in safe.

Whether Kalico is in scope for this project or not, `link_extruder_stepper()`/
`unlink_extruder_stepper()` are the only way stock Klipper (post-refactor,
if that refactor ever lands upstream) or any Klipper fork with a similar
multi-stepper-list model will see this stepper — direct attribute
assignment is fragile against that whole class of refactor, not just
this specific one.

## Bug 2: `MmuExtruderStepper.cmd_SET_PRESSURE_ADVANCE` targets attributes that don't exist on Kalico

Same file, further down:

```python
class MmuExtruderStepper(ExtruderStepper, object):
    ...
    # Override to add QUIET option to control console logging
    def cmd_SET_PRESSURE_ADVANCE(self, gcmd):
        pressure_advance = gcmd.get_float('ADVANCE', self.pressure_advance, minval=0.)
        smooth_time = gcmd.get_float('SMOOTH_TIME', self.pressure_advance_smooth_time, minval=0., maxval=.200)
        self._set_pressure_advance(pressure_advance, smooth_time)
        ...
```

`MmuExtruderStepper` inherits directly from Kalico's real
`ExtruderStepper` (`from kinematics.extruder import ... ExtruderStepper`
at the top of this file) — so `super().__init__()` already sets up
Kalico's actual pressure-advance-model system (`self.pa_model`,
`self.smoother`, `self.pressure_advance_time_offset`). This override
reads `self.pressure_advance`/calls `self._set_pressure_advance(...)` —
neither exists on Kalico's `ExtruderStepper`. It's a leftover from
pre-refactor Klipper's much simpler PA implementation (a single float
attribute, no model/smoother system).

Symptom: once bug 1 above is fixed and `SET_PRESSURE_ADVANCE` actually
reaches this stepper, it immediately fails with:

```
'MmuExtruderStepper' object has no attribute 'pressure_advance'
```

### Fix

Remove the override entirely. `MmuExtruderStepper` then falls back to
the parent class's real, working `cmd_SET_PRESSURE_ADVANCE`. The only
functional loss is the custom `QUIET` flag on a *direct*
`SET_PRESSURE_ADVANCE EXTRUDER=<gear stepper name>` invocation (a
narrow, rarely-used path — not the default no-`EXTRUDER=` call a
filament profile would normally make). If the `QUIET` behavior matters,
it would need reimplementing against Kalico's real `pa_model`/`smoother`
attributes instead of the old single-float ones — happy to help with
that if useful.

## Reproduction

- Kalico firmware (n3oney/qidi-q2-klipper build, but any Kalico build
  with `homing_extruder: True` in `[mmu_machine]` should reproduce this)
- Happy Hare installed via the `bunnybox` branch (Camden-Winder's
  `install-bb-q2.sh`, though the bug is in `mmu_machine.py` itself, not
  the installer)
- Start any print that does at least one tool-select after the initial
  load (even a redundant "already loaded" one — very common, since
  sliced multi-material gcode re-emits the current tool at the start of
  layers/objects)
- Print fails with "Active extruder does not have a stepper" partway
  through, once a `SET_PRESSURE_ADVANCE` call fires

## Fix verification

Both fixes applied, `systemctl restart klipper`, then:

```
SET_PRESSURE_ADVANCE ADVANCE=0.04
```

Confirmed via Moonraker's `gcode_store`:

```
pressure_advance_model: linear
pressure_advance: 0.040000
pressure_advance_smooth_time: 0.030000
pressure_advance_time_offset: 0.000000
```

A subsequent real print (Box active, multi-material) completed
end-to-end (`print_stats.state: complete`) with no further errors of
this kind.

## Patched file

`../patches/mmu_machine.py` in this repo is the full patched file (both
fixes applied, with inline comments explaining each) — diffable against
`../patches/mmu_machine.py.original-bunnybox` (the unmodified file as
installed by `install-bb-q2.sh`, kept alongside for reference). Happy to
open a PR against the `bunnybox` branch directly if that's preferred
over a description-only report.
