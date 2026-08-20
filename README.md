# Qidi Q2 mainline migration — upstream feedback

Findings, bugs, and suggestions gathered while migrating a Qidi Q2
(with the 4-slot Box multi-material unit) from stock firmware to a
fully mainline stack:

- Mainline Kalico firmware on all three MCUs (mainboard, toolhead,
  box) via **n3oney/qidi-q2-klipper**
- Config migration guidance from **MisterSheikh/Qidi_Q2_Mainline_Klipper**
- Vanilla Klipper/Moonraker/Fluidd (no Qidi vendor forks)
- **HelixScreen** touchscreen
- **Happy Hare** (Wazzup77/Happy-Hare `bunnybox` branch, installed via
  Camden-Winder/Qidi-Q2-superuser's `install-bb-q2.sh`) for box/MMU
  control
- OrcaSlicer as the slicer, connected via a generic Klipper/Moonraker
  physical printer instead of Qidi's vendor network agent

This repo is feedback for the maintainers of those projects — real bugs
found (with fixes), gaps in documentation, and things that worked
better than expected. Each report is written to be actionable
standalone (can be turned into an issue/PR against the relevant repo).

## Reports

| Report | Project | Summary |
| --- | --- | --- |
| [happy-hare-bunnybox-kalico-compat.md](reports/happy-hare-bunnybox-kalico-compat.md) | Wazzup77/Happy-Hare (`bunnybox` branch) | **Real bug + fix**: `mmu_machine.py` breaks pressure advance on Kalico — two separate incompatibilities, both fixed and verified |
| [n3oney-qidi-q2-klipper.md](reports/n3oney-qidi-q2-klipper.md) | n3oney/qidi-q2-klipper | Firmware/build notes: RFID crash + stub firmware patch, box driver binary reuse, extruder refactor compat gap |
| [config-changes-mistersheikh.md](reports/config-changes-mistersheikh.md) | MisterSheikh/Qidi_Q2_Mainline_Klipper | Documentation gaps in `config_changes.md` found while following it end-to-end |
| [install-bb-q2-camden-winder.md](reports/install-bb-q2-camden-winder.md) | Camden-Winder/Qidi-Q2-superuser | Installer experience notes, two real gaps (Kalico compat + dual-box template bugs, links to dedicated reports) |
| [install-bb-q2-camden-winder-dual-box.md](reports/install-bb-q2-camden-winder-dual-box.md) | Camden-Winder/Qidi-Q2-superuser + Wazzup77/Happy-Hare | **Real bugs + fix**: shipped BOX 2 template has a gear-stepper naming bug and a copy-paste fan-wiring bug; plus a documented hardware limit (Qidi's official multi-box Hub sensor only reachable from the last box in the daisy chain — unfixable via Klipper's cross-MCU homing model) |
| [helixscreen.md](reports/helixscreen.md) | prestonbrown/helixscreen | Install experience, one open question (touch input dependency) |
| [orcaslicer-qidi-vendor-profile.md](reports/orcaslicer-qidi-vendor-profile.md) | OrcaSlicer / Qidi vendor profile | `use_3mf` inheritance breaks generic Klipper/Moonraker connections for Qidi-profile-derived printers |

## Context

All of this came out of a real, hands-on migration project — full
history (including a lot of dead ends) is tracked in a private repo.
These reports distill only the parts worth sending upstream; they don't
include printer-specific details (serial IDs, IPs, calibration values)
beyond what's needed to reproduce or understand each finding.
