# SmechOS RC4 — "Founder Name Day Edition" (FNDE)

**Status: in active development, target 2026-11-30. This is a living
document — unlike [RC3.md](RC3.md), it describes a moving target.** Where
this conflicts with `spk-compile.py` itself, the code is correct and this
doc is stale — check `SMECHOS_PLASMA_LIVE_PHASES` and the relevant
`phase_*()` functions directly if anything here looks suspicious.

## Scope

RC4 opens a new release family (Lesobinaska, following RC1–RC3 + the
Anniversary Edition's Peritos family) built on SmechOS's **own**
cross-toolchain and from-source glibc, instead of inheriting the build
container's. Three real pieces of scope, not a version bump:

1. A real cross-toolchain (crosstool-ng) targeting a new triplet,
   `x86_64-smechos-linux-gnu`, with SmechOS's own from-source glibc.
2. Formalized `SABI.md`/`SAPI.md` — a declared ABI/API contract (multiarch
   path convention, `$ORIGIN`-relative RPATH policy, SONAME-transition
   policy, the `spk` PackageKit backend contract) instead of tribal
   knowledge scattered across code comments.
3. Progress toward `spk`/APT feature parity (unscoped as of this writing —
   needs its own investigation pass before it can be estimated).

Deliberately **not** chasing the newest Plasma/KDE release — still pinned
to the Plasma 6.6/Frameworks 6.24 LTS line for now; see the "Plasma 6.6 LTS
pivot" reasoning if considering a version bump, which trades stability for
a real integration cost each time it's reopened.

## What's actually cross-compiling right now

As of this writing, `CROSS_TRIPLET = "x86_64-smechos-linux-gnu"` and the
following phases genuinely cross-compile (not just run in a
glibc-matched container — see the README's cross-toolchain note):

- **`cross-deps`** — the Mesa/Qt6 dependency chain (zlib..dbus), cross-built
  so later phases stop silently falling back to the container's own copies
  via `PKG_CONFIG_PATH`'s additive host fallback (a real, confirmed-silent
  failure mode, not hypothetical).
- **`qt-deps`** — all 10 Qt6 modules (qtbase, qtshadertools, qtdeclarative,
  qtsvg, qttools, qtwayland, qtmultimedia, qt5compat, qtspeech,
  qtpositioning) confirmed cross-compiling clean, host+cross two-pass per
  module (`QT_HOST_PATH` for build-time tools like `moc`/`uic`/`rcc`, a
  separate cross pass for the actual target libraries). Two real bugs
  found and fixed to get here:
  - `pcre2` needed `--enable-pcre2-16` — only the 8-bit codepoint variant
    builds by default, but `QString`/`QRegularExpression` need the 16-bit
    (UTF-16) API.
  - CMake's `FindOpenGL` defaulted to GLVND mode and linked the
    *container's* own split `libOpenGL.so`/`libGLX.so`, not the
    cross-built target's single-library Mesa `libGL.so` — fixed with
    `-DOpenGL_GL_PREFERENCE=LEGACY` on every cross-compiled module.
- **`kernel`** — cross-compiles via the same toolchain. Also picked up
  three real config fixes this cycle, found via an actual kexec test on
  real hardware, not guessed: `CONFIG_BLK_DEV_NVME` was entirely absent
  (not even a module) — meaning a from-this-kernel install would never
  boot on the NVMe-based storage nearly every modern x86_64 machine ships
  with. `CONFIG_DM_CRYPT` and `CONFIG_DM_THIN_PROVISIONING` were missing
  too — the same class of bug for Calamares' encrypted-install and
  LVM-thin-pool install paths specifically.

**Not yet cross-compiling / status unclear as of this writing**: `kde`
(KDE Frameworks + Plasma) — check the phase's own recent history before
assuming either way.

## Known open issues

- **Black-screen session handoff** — after the first-boot wizard (running
  as a separate `plasma-setup` system user) hands off to the real user's
  desktop session, the screen goes solid black and stays that way.
  Reproduced identically on QEMU (`virtio-vga`) and real AMD Vega hardware
  via a real kexec boot — two unrelated GPU backends hitting the same
  symptom points at a session-handoff logic bug, not a driver issue.
  Real evidence gathered, root cause not yet found: see
  [`BUG_BRIEF_black_screen_handoff.md`](BUG_BRIEF_black_screen_handoff.md).

## Contributor entry points

RC4 is deliberately meant to be the point where SmechOS opens to outside
contributors — see the [co-maintainer call](https://github.com/Smech-Labs/spk-compile/issues/3)
for current open areas (PLM build integration, `spk`/APT parity, real
hardware boot-testing, and others).

## Checkpoint

An honest status review against this scope is planned before 2026-11-30
gets treated as fixed — if that hasn't happened yet by the time you're
reading this, that's the thing to push on, not guessing at new scope.
