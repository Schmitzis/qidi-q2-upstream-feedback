# HelixScreen — install experience + one open question

**Project**: [prestonbrown/helixscreen](https://github.com/prestonbrown/helixscreen)

## What worked well

The installer (`curl -sSL https://helixscreen.org/install.sh | sh`) was
smooth end-to-end on a Qidi Q2 (auto-detected as a "QIDI-class SBC"):
correct architecture/package selection, clean systemd service +
update-watcher install, automatic Moonraker `update_manager` section
and `moonraker.asvc` allowlist entry, and config migration into
`printer_data/config/helixscreen`. Also correctly detected that no
competing screen UI was present at install time (the stock Qidi
touchscreen client uses a different detection signature than what it
checks for, so this was arguably a near-miss rather than a clean
detection — worth double-checking that logic still catches the stock
`QD_Q2`/`makerbase-client` service by name, not just by some other
side-effect it happens to have).

Happy Hare integration (once installed afterward) shows up correctly in
the UI with live MMU status — this was actually the deciding factor for
switching from a "reuse stock compiled binaries" approach to Happy Hare
for box control, since the raw `box_extras`/`box_stepper` object model
has no UI support in HelixScreen at all (understandably, since it's a
closed-source Qidi-only interface).

## Open question: touch input dependency

The stock Qidi touchscreen client (`QD_Q2/bin/client`) depends on
`triggerhappy` being enabled — it reads touch events via
`/run/thd.socket`, and disabling `triggerhappy` (a otherwise-reasonable
system-hardening step, since it's not used for anything else on this
printer) kills touch input entirely for that client.

After replacing the stock client with HelixScreen, is `triggerhappy`
still required for HelixScreen's own touch input path, or does
HelixScreen read the touch device directly (e.g. via libinput/evdev)
independent of that daemon? Wasn't able to find this documented, and
haven't yet done a controlled test (disable `triggerhappy`, confirm
touch still works on the physical screen) to answer it empirically. If
it's *not* required, a doc note saying so would let Box V2 / stock-client
migrators safely include it in their post-migration service-hardening
checklist without guessing.
