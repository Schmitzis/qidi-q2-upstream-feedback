# Camden-Winder/Qidi-Q2-superuser — install-bb-q2.sh feedback

**Project**: [Camden-Winder/Qidi-Q2-superuser](https://github.com/Camden-Winder/Qidi-Q2-superuser)
**Script**: `Q2/install-bb-q2.sh`

## What worked well

The installer's hardware-specific defaults were spot on for a real Box
V2 unit — `mcu_box1` serial pre-fill (picking the right one out of
several available serial devices by chip ID), `stepper_mmu_gear`/
`pre_gate_switch_pin_0..3` pin mappings in the generated
`mmu/base/mmu_hardware.cfg`, and automatically commenting out
`[include box.cfg]` (already aware of the conflict, no manual
intervention needed). Cross-checked the pin mappings independently
against another Box V2 owner's published wiring notes
([trainlights-creator/qidi-q2-mainline-conversion](https://github.com/trainlights-creator/qidi-q2-mainline-conversion))
and they matched exactly.

## One real gap: Kalico compatibility in the Happy Hare fork it installs

The installer clones `Wazzup77/Happy-Hare`'s `bunnybox` branch and runs
its own `install.sh` as a nested step. That branch's `mmu_machine.py`
has a real bug when running against Kalico specifically (as opposed to
stock Klipper or Danger-Klipper) — pressure advance breaks mid-print
with `Active extruder does not have a stepper`, then (once that's
fixed) a second error (`'MmuExtruderStepper' object has no attribute
'pressure_advance'`). Full details, root cause, and a verified fix in
[happy-hare-bunnybox-kalico-compat.md](happy-hare-bunnybox-kalico-compat.md)
in this same feedback set — filing that against the Happy-Hare fork
directly, but flagging it here too since this installer is what most
Box V2 users will actually run, and anyone using it against a Kalico
build (like n3oney/qidi-q2-klipper's, which this installer is clearly
designed to target) will hit this on their first real print with a
tool-change in it.

## Second real gap: shipped BOX 2 (dual-box) template has bugs, plus a hardware limit worth documenting

The installer's generated `mmu_hardware.cfg` already ships full "BOX 2"
sections, commented out, for anyone adding a second physical Box unit —
a nice touch, but we found two real bugs in that template (a
gear-stepper naming mismatch that undercounts gates entirely, and a
copy-paste bug in `box2_board_fan`'s stepper list) while actually wiring
up a second box. There's also a third thing that isn't a bug at all but
cost real debugging time before we understood it: if the two boxes are
chained through Qidi's official multi-box Hub, only whichever box is
physically *last* in the chain can use the shared Hub sensor for gate
detection — Klipper's own cross-MCU homing restriction makes this
unfixable in config for the other box(es). Full details, fixes, and
verification in
[install-bb-q2-camden-winder-dual-box.md](install-bb-q2-camden-winder-dual-box.md).

## Minor note: interactive installer automation

Not a bug in the script, just an observation from trying to drive it
non-interactively (for scripted/repeatable installs): some of its
prompts (and the nested Happy Hare `install.sh`'s own prompts) require
real TTY input — piping answers via stdin or writing to the pty slave
device doesn't reliably work (looks like it's "accepted" in captured
output due to terminal echo, but doesn't actually unblock a `read -p`).
`TIOCSTI` ioctl injection was the only reliable way found to fully
automate a run end-to-end. Not something to necessarily change (a
human running it interactively is probably the right default UX for a
one-time hardware setup step), but worth knowing if either of you is
looking at CI/scripted-testing options for this installer in the
future.
