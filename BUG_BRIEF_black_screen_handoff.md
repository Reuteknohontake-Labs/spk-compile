# Bug: Black screen after first-boot wizard hands off to the real desktop session

**This is a live bug in the currently-shipped RC3/FAE release, not an
RC4-only concern.** It was found by testing the Founder Anniversary
Edition ISO — RC3's real, checksum-verified binaries with only a cosmetic
patch on top, no session/compositor code touched. RC4's KDE/Plasma isn't
cross-compiled yet, so it cannot be the origin; it will almost certainly
carry forward into RC4 unless fixed here first. Treat this as affecting
real users downloading SmechOS today, not a future-release item.

## Summary

SmechOS's live/installed boot flow runs a first-boot setup wizard as a
separate system user (`plasma-setup`, uid 983) with its own `kwin_wayland`
instance. When the user clicks "Finish," control is meant to hand off to
the real user's (`smech`, uid 1000) desktop session — but the screen goes
solid black and **stays black indefinitely**. Reproduced identically on:

- QEMU (`virtio-vga` device, software/tcg emulation)
- Real hardware: AMD Vega (Picasso/Raven2 APU graphics, `amdgpu` driver)

Two completely different GPU backends hitting the identical symptom points
away from a driver-specific rendering bug and toward a logic bug in the
session handoff itself.

## What's confirmed (real evidence, not speculation)

- The wizard's own `kwin_wayland` (uid 983) is confirmed running during
  setup — environment dump captured, shows `XDG_SESSION_TYPE=wayland`,
  `KDE_SESSION_UID=983`, `_=/usr/bin/kwin_wayland_wrapper.real`.
- After clicking Finish, in one clean run, `systemd` itself logged a clean
  `Reached target Main User Target` / `Startup finished in 20.168s` for the
  real session — i.e., the *session's own init sequence* completes without
  reported failure.
- In another clean run, kernel-level stack-depth logging (`used greatest
  stack depth`) showed a `kwin_wayland` process (PID 11817) genuinely
  running at kernel uptime t=179s, post-Finish — i.e., a second compositor
  process does appear to start.
- Despite both of the above, the screen stays black. QEMU's own vCPU state
  shows the guest genuinely idle (`HLT=1`, stable RIP across samples), not
  crashed/deadlocked at the hardware level.
- On real hardware (kexec test, AMD Vega), the screen was confirmed stuck
  black "forever" (user observation, not a transient delay).

## What's NOT yet confirmed (the actual open question)

- Whether the wizard's `kwin_wayland` (uid 983) is still holding
  `/dev/dri/card0` (DRM master) when the real session's `kwin_wayland`
  (uid 1000) tries to start — the leading hypothesis, untested with hard
  evidence. (`fuser -v /dev/dri/card0` at the right moment would answer
  this directly.)
- Whether the real session's `kwin_wayland` actually succeeds in becoming
  DRM master, or silently fails/crash-loops without producing a visible
  journal entry.
- Whether `plasmalogin`/`plasma-setup`'s systemd service is ever actually
  stopped/torn down before the real session starts, or whether both
  sessions coexist indefinitely.

## Where to look

- `spk-compile.py`'s `phase_live_initramfs` (busybox live-boot init
  script) and whatever systemd units `phase_plasma_configure` /
  `phase_kde` generate for `plasmalogin`/`plasma-setup` vs. the real
  user's session (likely under `phase_patch_metadata` or similar — search
  for `plasma-setup`, `plasmalogin.service`, `KDE_SESSION_UID=983`).
- The actual fix is very likely either: (a) a missing `Conflicts=`/`Before=`
  systemd ordering so the wizard's session is guaranteed to fully stop
  (including its `kwin_wayland`) before the real session's compositor
  starts, or (b) a missing explicit DRM-master release step in whatever
  tears down the `plasma-setup` session on "Finish."

## How to reproduce

QEMU (fast iteration, no physical hardware needed):
```
qemu-system-x86_64 -m 3072 -smp 2 -machine accel=kvm:tcg \
  -cdrom smechos-plasma-live-fae.iso -boot d \
  -device virtio-vga \
  -serial file:serial.log -display none
```
Select the "debug console" GRUB entry (routes kernel/systemd journal to
`ttyS0`/serial.log). Click through the wizard, click Finish, screen goes
black. The live ISO's `/init` has several baked-in diagnostic hooks already
(search for `KWIN_WAYLAND ENVIRON`, `SMECHOS USER SYSTEMD DIAG`) that dump
process/journal state to serial — useful starting points, though some are
hardcoded to uid 983 (the wizard) rather than uid 1000 (the real session)
and would need adjusting to see the real session's state directly.

## Deliverable

A real root cause (not a guess) and a fix in `spk-compile.py` (or
whichever generated systemd unit/script is actually responsible) that gets
a real, visible KDE Plasma desktop after Finish — verified via an actual
boot, screenshot or equivalent proof, not just "should work now."
