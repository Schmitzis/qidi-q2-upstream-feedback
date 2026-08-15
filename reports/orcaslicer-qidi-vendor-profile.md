# OrcaSlicer Qidi vendor profile — `use_3mf` blocks generic Klipper/Moonraker connections

**Project**: OrcaSlicer (bundled Qidi vendor profile), and/or wherever
the Qidi Q2 profile itself is maintained.
**Files**: `resources/profiles/Qidi/machine/Qidi Q2 0.4 nozzle.json`
(and the other nozzle-size variants)

## The problem

Any printer profile inheriting from Qidi's bundled system profile
(directly or via a user override that doesn't explicitly clear the
field) inherits:

```json
"printer_agent": "qidi",
"use_3mf": "1"
```

`use_3mf` (defined in `PrintConfig.cpp`: *"Enable this if the printer
accepts a 3MF file as the print job... sends the sliced file as a
.gcode.3mf, instead of a plain .gcode file"*) makes Orca upload a
`.gcode.3mf` bundle on "Send to Printer," regardless of what connection
type is actually configured. This is correct and required for Qidi's
own (patched) Moonraker fork, which has custom server-side logic to
unpack the embedded gcode from that bundle — but it's silently wrong
for **any** printer running vanilla/mainline Moonraker, which has no
such unpacking logic and simply rejects the upload:

```
Processing Uploaded File: <name>.gcode.3mf
server.files.metascan: File ... is not a valid gcode file
```

This is exactly the situation for anyone migrating a Qidi printer to
mainline Klipper + vanilla Moonraker (a real, documented migration path
— see e.g. n3oney/qidi-q2-klipper) while still wanting to use their
existing Qidi-based OrcaSlicer printer profile rather than starting a
generic-Klipper profile from scratch. Configuring a proper generic
Klipper/Moonraker "Physical Printer" connection (`host_type:
OctoPrint/Klipper`, printer agent `Moonraker`) is not sufficient by
itself — the upload format is a separate, profile-level setting that
silently overrides it.

## Reproduction

1. Start from Qidi's bundled `Qidi Q2 0.4 nozzle` profile (or any
   user profile inheriting from it without overriding `use_3mf`)
2. Configure a Physical Printer connection with `host_type:
   OctoPrint/Klipper`, `printer_agent: moonraker`, pointed at a vanilla
   Moonraker instance
3. Slice and "Send to Printer"
4. Moonraker logs `Processing Uploaded File: <name>.gcode.3mf` followed
   by `is not a valid gcode file` — the print never starts, with no
   clear indication in Orca's own UI of why

## Suggested fix

Two options, not mutually exclusive:

1. **Have the Physical Printer dialog's Host Type selection
   (`OctoPrint/Klipper` specifically, as opposed to a Qidi/Bambu-style
   LAN-discovery connection) automatically clear/override `use_3mf` to
   `0`** on save, since a generic OctoPrint/Klipper host type is by
   definition not the vendor-agent path `use_3mf` exists for. This
   would make the two settings consistent with each other automatically
   instead of requiring users to know `use_3mf` exists as a separate,
   hidden-by-default (`comAdvanced` mode) field.
2. At minimum, **surface a warning** in the send/upload flow when
   `use_3mf` is true but the resolved host type isn't a Bambu/Qidi-style
   vendor connection — something like *"This printer profile is
   configured to send `.3mf` project files, but the connection type
   doesn't support that — check `use_3mf` in Printer Settings."*

## What we did as a workaround

Added an explicit `"use_3mf": "0"` override to our own user profile
(inheriting from the Qidi system profile), alongside keeping
`"printer_agent": "moonraker"`. This is documented and works fine as a
one-time fix — flagging upstream mainly because the failure mode (a
silent, unhelpful Moonraker-side rejection with no hint in Orca's own
UI about the actual cause) cost real debugging time to trace back to
this one hidden field, and will likely hit anyone else doing a similar
Qidi-to-mainline migration.
