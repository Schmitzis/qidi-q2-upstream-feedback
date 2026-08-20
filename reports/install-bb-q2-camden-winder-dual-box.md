# install-bb-q2.sh's shipped BOX 2 (dual-box) template — three real bugs, plus a hard architectural limit worth documenting

**Project**: [Camden-Winder/Qidi-Q2-superuser](https://github.com/Camden-Winder/Qidi-Q2-superuser), `Q2/install-bb-q2.sh`
**File**: generated `mmu/base/mmu.cfg` / `mmu/base/mmu_hardware.cfg` — the commented-out "BOX 2" sections shipped for anyone adding a second Box unit
**Related**: [Wazzup77/Happy-Hare](https://github.com/Wazzup77/Happy-Hare) `bunnybox` branch (`mmu_machine.py`, `mmu_sensors.py` — the actual enumeration logic these templates need to match)
**Severity**: three of these are real, reproducible config bugs with fixes below. The fourth isn't a bug at all — it's a hardware/architecture interaction that cost us multiple debugging sessions before we understood it, and is worth documenting directly in the shipped template's comments so the next person doesn't repeat that.
**Status**: all four verified on real hardware — two physically identical Qidi Box V2 units, chained through Qidi's official multi-box Hub (7-in-1 PTFE connector + shared hub sensor board, per [wiki.qidi3d.com/en/QIDIBOX/Multi-box-connection](https://wiki.qidi3d.com/en/QIDIBOX/Multi-box-connection)).

## Context

The installer/branch already ships full "BOX 2" sections in both files,
commented out, ready to uncomment for a second box — a nice touch. But
we hit real problems getting an actual second physical box working with
it, on top of Qidi's own official multi-box hardware (not a custom
rig). Filing this so the next person with a real 2-box setup doesn't
lose the time we did.

## Bug 1: shipped `[stepper_mmu1_gear]` naming doesn't match `mmu_machine.py`'s actual enumeration

The template (`mmu_hardware.cfg`) suggests, for box 2:

```ini
#[stepper_mmu1_gear]
#step_pin: mmu1:PC14
...
#[stepper_mmu1_gear_1]
...
```

But `mmu_machine.py` has:

```python
GEAR_STEPPER_CONFIG = "stepper_mmu_gear"
...
last_gear = 24
for i in range(1, last_gear):
    section = "%s_%d" % (GEAR_STEPPER_CONFIG, i)
    if not config.has_section(section):
        last_gear = i
        break
```

This enumerates `stepper_mmu_gear_1`, `_2`, `_3`, ... — a single global
prefix, continuously numbered across *all* units regardless of which
MCU each one's pins point to. It has no knowledge of (and doesn't look
for) a per-unit-prefixed name like `stepper_mmu1_gear`.

**Symptom**: following the template exactly (`num_gates: 4,4`, box 2's
steppers named `stepper_mmu1_gear*`) gives:

```
MMU is configured with 8 gates but 4 gear stepper configurations were found
```

— it silently only counts box 1's 4 gears (`_1`, `_2`, `_3` found,
`_4` missing since it doesn't exist under that name) and stops there.

**Fix**: name box 2's sections `stepper_mmu_gear_4`/`_5`/`_6`/`_7`
instead — keep the `mmu1:` pin prefix (that's what correctly routes
each pin to box 2's physical MCU; only the *section name* needs to stay
on the global counter):

```ini
[stepper_mmu_gear_4]
step_pin: mmu1:PC14
dir_pin: mmu1:PC13
enable_pin: !mmu1:PC15
...
```

Verified: with this rename (and nothing else changed), `num_gates: 4,4`
loads cleanly, `state: ready`, and `printer.objects.query?mmu` reports
`num_gates: 8`.

## Bug 2: shipped `controller_fan box2_board_fan` references box 1's steppers (copy-paste artifact)

```ini
#[controller_fan box2_board_fan]
#pin: mmu1:PA6
#heater: box2_heater
#stepper: stepper_mmu_gear, stepper_mmu_gear_1, stepper_mmu_gear_2, stepper_mmu_gear_3
```

That `stepper:` list is box 1's gears (`stepper_mmu_gear*`, unprefixed —
gates 0-3), not box 2's. Harmless at parse time (those objects exist,
just on the wrong board), but it means box 2's own board fan is keyed
off box 1's motor activity instead of its own — box 2's fan could stay
off while its own gears are actively running, or run when they're idle.

**Fix**: once bug 1's rename is applied, this should read:

```ini
stepper: stepper_mmu_gear_4, stepper_mmu_gear_5, stepper_mmu_gear_6, stepper_mmu_gear_7
```

## Bug 3 (minor, already partially self-documented): `environment_sensor`/`filament_heater` singular fields don't support a second box

The template already has an inline comment flagging this
(`# TEMP - Happy Hare does not yet support multibox env sensor` /
`filament_heater`), so this isn't a surprise to report — just
confirming it's accurate and noting the concrete workaround for anyone
who finds the singular fields silently not doing what they expect for
box 2: switch to the plural per-gate list form instead of the singular
fields, one entry per gate (repeating the same object name for every
gate that shares one physical heater/sensor):

```ini
environment_sensor:
filament_heater:
environment_sensors: temperature_sensor box1_env, temperature_sensor box1_env, temperature_sensor box1_env, temperature_sensor box1_env, temperature_sensor box2_env, temperature_sensor box2_env, temperature_sensor box2_env, temperature_sensor box2_env
filament_heaters: heater_generic box1_heater, heater_generic box1_heater, heater_generic box1_heater, heater_generic box1_heater, heater_generic box2_heater, heater_generic box2_heater, heater_generic box2_heater, heater_generic box2_heater
```

This works correctly today (verified live) — the plural fields just
aren't mentioned anywhere near the singular ones' "not yet supported"
comment, so it's easy to assume multibox temperature control isn't
available at all rather than just spelled differently.

## Not a bug: Qidi's official Hub sensor is only reachable from whichever box is physically *last* in the daisy chain — and Klipper can't home across that

This is the one worth the most words, because it looks exactly like a
software bug (and we spent real time debugging it as one across
multiple sessions) but is actually a fundamental interaction between
Qidi's official multi-box hardware topology and Klipper's homing model.

**The hardware**: Qidi's official multi-box kit (see
[wiki.qidi3d.com/en/QIDIBOX/Multi-box-connection](https://wiki.qidi3d.com/en/QIDIBOX/Multi-box-connection))
chains boxes: `printer -> box1 -> box2 -> Hub`. The Hub is a physically
separate component — in our case, downstream of two Bambu Lab 4-to-1
PTFE adapters (one per box), sitting ~75cm past the gates — with its
own single filament-presence sensor. That sensor's 2-wire cable
physically terminates in a connector on whichever box ends up last in
the chain. With only box 1 connected, that's box 1. Add box 2 behind
it, and box 2 becomes last instead — box 1's own equivalent pin
(`mmu:PB1` in our config) goes electrically dead for this purpose,
**even though nothing in software or config changed about box 1**.

**Confirmed empirically**: with both boxes connected and referenced
(`[mcu mmu]` = box 1, `[mcu mmu1]` = box 2), we wired
`gate_switch_pin: ^!mmu:PB1, ^!mmu1:PB1` (per Happy Hare's own
multi-unit sensor-manager logic in `mmu_sensors.py`, which requires
exactly 1 or `num_units` entries to get per-unit `unit_0_mmu_gate`/
`unit_1_mmu_gate` sensor names). Live-polled both raw sensor objects
while manually triggering the physical Hub sensor by hand: `unit_1_mmu_gate_sensor`
(box 2) flipped `true` and stayed there; `unit_0_mmu_gate_sensor` (box 1)
never moved, confirmed over dozens of polls.

**Why this can't be fixed in config**: box 1's gear steppers live on
its own MCU (`mmu`). Homing them to an endstop physically wired to a
*different* MCU (`mmu1`, box 2) hits Klipper's own hard restriction:

```
Multi-mcu homing not supported on multi-mcu shared axis
```

We also tried the obvious alternative — `gate_homing_endstop: extruder`
(homing box 1's gate-load step to the toolhead's extruder-entry sensor
instead of the gate sensor). Same shape of problem: the extruder
sensor lives on a third MCU (`THR`), box 1's gear stepper is still on
`mmu`. This ran the full `gate_homing_max` distance (1500mm) without
ever registering a valid home — no hard crash this time, just silent
non-detection, which took a live source read of `mmu_sensor_manager.py`
and `mmu.py`'s `_load_gate()` to confirm was the same underlying
issue and not a separate bug.

**The only two real fixes**, neither of which is a config change:
1. Give the box that's *not* last in the chain its own local, per-gate
   sensor (`post_gear_switch_pin_N`, one per gate, wired to that box's
   own MCU) and use `gate_homing_endstop: mmu_gear` for it instead of
   `mmu_gate`. This is Happy Hare's own documented mechanism for
   exactly this ("individual per-gate endstop (type-B MMU's)") and
   sidesteps the cross-MCU issue entirely, since each gate's own sensor
   naturally shares its stepper's MCU. Requires real wiring (a
   microswitch per gate) — not investigated further on our end since
   we didn't want to modify hardware.
2. Accept that whichever box is earlier in the chain can't do real
   gate-homing at all. Note this is **not** just "less safe" — Happy
   Hare's `_load_gate()` only supports encoder-based or real-endstop-based
   homing; there's no "trust a calibrated distance, skip verification"
   mode to fall back to. So this genuinely means single-box operation
   (or accepting box 1 stays gate-check-broken) unless option 1 is done.

This also retroactively explains something that looked like a strange,
asymmetric bug the first time we hit it: in an earlier session, box 2's
own gates worked perfectly fine in a full dual-box config while box 1's
did not, despite what looked like parallel, identically-structured
config for both units. It wasn't asymmetric software behavior — box 2
was last-in-chain (the one with the real, working Hub sensor), box 1
wasn't. Anyone else building a real multi-box Qidi setup through the
official Hub will hit this in exactly the box that isn't last in their
own physical chain, and it's worth a line in the shipped template's own
"BOX 2 CONNECTION" comments warning about it directly, since it's very
easy to sink hours into treating this as a config/homing bug (we did,
across multiple sessions) before realizing it's the hardware topology.

## Reproduction

- Two Qidi Box V2 units, Katapult + mainline Kalico on both
  (`n3oney/qidi-q2-klipper`), Happy Hare `bunnybox` branch via
  `install-bb-q2.sh`
- Chained through Qidi's official Hub (2x Bambu Lab 4-to-1 PTFE
  adapters, one per box, feeding into the Hub's single merge point +
  sensor), per Qidi's own wiki wiring guide
- Apply the shipped "BOX 2" template as-is (bugs 1 and 2 above present)
  → `num_gates: 4,4` fails to load with the gear-stepper-count error
- Fix bugs 1-3 above → both units connect, `num_gates: 8`, box 2's own
  gates pass real `MMU_CHECK_GATE`/preload load-detect cycles
- Box 1's gates fail the same check consistently — "marked EMPTY" or
  (once) a full `gate_homing_max` move with no detection — traced to
  the Hub-sensor-ownership issue above, not any of bugs 1-3

## Fix verification

With bugs 1, 2, and 3 applied: printer boots clean (`state: ready`),
`num_gates: 8`, both `mmu`/`mmu1` MCUs connected and reporting live
independent heater/env-sensor temps, box 2's gates (last-in-chain, real
working Hub sensor) pass real load/detect cycles. Box 1's gates
confirmed structurally unable to gate-home for the hardware-topology
reason above, not a config error — verified via live sensor polling
during manual physical triggering of the Hub sensor, and via reading
Klipper's own `mcu.py` for the cross-MCU restriction's source.
