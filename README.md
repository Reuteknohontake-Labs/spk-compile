# spk-compile

The build system for **SmechOS** — an independent Linux distribution built
from scratch by Smech Labs.

SmechOS is **not** a fork or derivative of ChromiumOS, Ubuntu, Fedora, Debian
or any other distribution, and is not affiliated with, endorsed by or
sponsored by any of those projects or their vendors. `spk-compile.py`
downloads upstream sources and compiles them into a bootable image: a
systemd/glibc base, a custom kernel, KDE Plasma as the desktop, and `spk` as
the package manager.

Bug reports live in **[smechos-issues](https://github.com/Smech-Labs/smechos-issues)**.

This README is split into two halves: **[RC3](#rc3--architecture-reference-stable)**,
the current shipped release (stable — won't drift), and **[RC4](#rc4--founder-name-day-edition-living-document)**,
the in-progress next release (a living section, updated as the work
changes). General build-system usage (below) applies to both.

---

## Contents

| File | Purpose |
|---|---|
| `spk-compile.py` | The whole build system: phases, package lists, patches |
| `build-one.sh` | Build a **single** package against an existing rootfs |
| `Dockerfile`, `build.sh` | Containerised full build |

---

## Start here if you're picking up an issue

Most open issues are of the form *"package X was never added to the component
list"*. The code change is genuinely one line. The expensive part is
**verifying** it, and you do not need a full distribution build for that.

```bash
./build-one.sh spectacle 6.7.2 /path/to/rootfs
```

That compiles just the one package against a rootfs you already have and
installs it. Minutes, not hours.

Requirements: `podman`, `sudo`, ~6 GB free, and an extracted SmechOS rootfs
(see below).

### KDE Gear vs Plasma — the trap

KDE ships on two independent release tracks with **different version
numbers**, and guessing wrong gives a confusing 404:

| Track | Examples | Version style | Flag |
|---|---|---|---|
| Plasma | `kwin`, `spectacle`, `kinfocenter` | `6.7.2` | *(default)* |
| KDE Gear | `konsole`, `dolphin`, `ark` | `25.08.3` | `--gear` |

```bash
./build-one.sh konsole 25.08.3 /path/to/rootfs --gear
```

---

## Getting a rootfs

Either extract one from an existing ISO:

```bash
mkdir -p /tmp/iso && sudo mount -o loop,ro SmechOS-1.0-RC2.iso /tmp/iso
unsquashfs -d ./rootfs /tmp/iso/live/filesystem.squashfs
```

…or build one from scratch (long — several hours):

```bash
sudo python3 spk-compile.py smechos-plasma-live
```

---

## Why builds happen in a container

This is the single most important thing to understand before touching the
build, and it is not obvious.

**The SmechOS rootfs ships no libc headers.** There is no
`/usr/include/bits`, and the multiarch include directory contains only
`qt6`. It is a pure *runtime* image, so it **cannot be used as a build
sysroot** — `--sysroot` against it fails immediately.

Building on a modern host instead runs into a second wall. The rootfs is
**Ubuntu glibc 2.39**; a current host is much newer. The rootfs's own build
tools (`qtpaths`, `moc`, `msgfmt`) then cannot run at all:

```
qtpaths: symbol lookup error: .../libc.so.6:
         undefined symbol: __nptl_change_stack_perm, version GLIBC_PRIVATE
```

That is the target's `libc.so.6` being loaded by a newer host `ld.so`. You
can work around it with explicit `ld.so --library-path` invocations, but
build systems call these tools by absolute path, so you end up bind-mounting
a directory of wrapper scripts over the rootfs's `bin` — at which point mount
propagation bites too.

`ubuntu:24.04` **is** glibc 2.39. Inside it every one of those problems
disappears at once: the rootfs's tools run natively, headers and libraries
agree, and output is ABI-correct by construction. `build-one.sh` does exactly
this and nothing clever.

**This describes most of the pipeline, but not all of it anymore.** RC4 is
introducing a real cross-toolchain (`CROSS_TRIPLET = "x86_64-smechos-linux-gnu"`,
built with crosstool-ng) for specific phases — `cross-deps`, and parts of
`mesa`/`qt-deps`/`kernel` — instead of the container-glibc-match trick above.
Those phases cross-compile properly (`CROSS_COMPILE=x86_64-smechos-linux-gnu-`)
and don't depend on the container's glibc version matching the target at
all. Qt6 specifically now builds twice per module — once natively for the
host (`QT_HOST_PATH`, needed for `moc`/`uic`/`rcc` and other build-time
tools) and once cross-compiled for the target — rather than once, as the
old container-match approach did. If you're touching one of those phases,
the glibc-mismatch failure mode above doesn't apply; if you hit something
that looks like it, you're more likely looking at a genuine cross-toolchain
issue (wrong sysroot, missing `--host=` flag, etc.), not this.

### RUNPATH leaks

CMake bakes the staging path into `RUNPATH` even with
`CMAKE_INSTALL_RPATH_USE_LINK_PATH=FALSE`:

```
RUNPATH: [/usr/lib/x86_64-linux-gnu:/home/you/rootfs/usr/lib/...]
```

Binaries still work (the real path is first, dead entries are skipped) but it
ships your home directory inside the ISO. `build-one.sh` strips this with
`patchelf` automatically. If you install packages by hand, check it:

```bash
readelf -d <binary> | grep -i runpath
```

---

## Repacking and testing

```bash
sudo mksquashfs ./rootfs iso-build/live/filesystem.squashfs -comp xz -noappend
grub2-mkrescue -o smechos.iso iso-build -- -volid SMECHOS_LIVE
```

Boot it. **The virtio flags matter:**

```bash
qemu-system-x86_64 -enable-kvm -m 4096 -smp 4 -machine q35 \
    -cdrom smechos.iso \
    -device virtio-vga-gl -display egl-headless,gl=on -vnc :1
```

- `virtio-vga-gl` (not `virtio-vga`) is required for virgl. Without it the
  guest reports `[drm] features: -virgl` and falls back to llvmpipe, which
  currently segfaults in a loop and gives you a black screen that looks like
  a total failure but is not (see issue #2).
- `-vnc :1` then `vncviewer localhost:5901` to interact. QEMU's own
  `screendump` returns `Error: no surface` under virgl, because the scanout
  is a host GL dmabuf with no CPU-side copy — capture over VNC instead.
- Add `-serial file:boot.log -append "console=ttyS0,115200n8"` (with
  `-kernel`/`-initrd`) to capture boot output as text.

Check the result:

```bash
journalctl -b | grep -iE 'segfault|error|failed'
```

---

## Structure of `spk-compile.py`

Phases run in order and are stamped, so completed work is skipped on re-run.
Run one phase with `--phase <name>`, which bypasses the stamp check.

```bash
sudo python3 spk-compile.py smechos-plasma-live --phase mesa
sudo python3 spk-compile.py --list smechos-plasma-live
```

Component lists live near the bottom: `kf6` (Frameworks), `plasma` (Plasma),
and the phase list `SMECHOS_PLASMA_LIVE_PHASES`. **Most package-missing bugs
are fixed by adding one string to one of these lists.**

---

## Building the full ISO in a container

`build-one.sh` (above) is for verifying a single package. Building the
**whole** ISO — all 27 phases, several hours end to end — needs a much
larger set of exact host packages (a specific GCC version, matching Qt6/KDE
build deps, kernel build tools…), and the only way to get that set reliably
is inside the same `ubuntu:24.04` container `build-one.sh` already uses,
built once from the `Dockerfile` in this repo.

```bash
# Build the compile image (a few minutes; cached after the first run)
podman build -t ghcr.io/smech-labs/smechos-build:latest -f Dockerfile .
# (docker build works identically if you have Docker instead of podman)

# Run the full build. Requires --privileged (kernel build needs it) and
# --cgroupns=host if your container runtime nests cgroups for podman-in-
# docker use elsewhere in the pipeline.
docker run --rm --privileged --cgroupns=host \
    -v /path/to/spk-compile-sources:/mnt/spk-compile-sources \
    -v /path/to/smechos_build_root:/mnt/smechos_build_root \
    -v /tmp/smechos_build:/tmp/smechos_build \
    ghcr.io/smech-labs/smechos-build:latest smechos-plasma-live
```

The two persistent volumes matter: `spk-compile-sources` caches every
downloaded tarball (so a re-run after a failure doesn't re-download
anything already fetched), and `smechos_build_root` is the actual rootfs
being assembled — it's what eventually gets squashed into the ISO. Losing
either one means starting that phase over, not the whole build.

**This only builds and bundles the rootfs — it does not produce an ISO.**
Packing is a genuinely separate `spk-compile.py` invocation
(`cmd_iso()`, triggered by `--iso live`, runs *instead of* the phase list,
not after it). Phase completion is stamp-file-persisted, so this second
run doesn't redo any of the first one's work — it only assembles the ISO
from the already-built rootfs:

```bash
mkdir -p /path/to/smechos-iso-output
docker run --rm --privileged --cgroupns=host \
    -v /path/to/spk-compile-sources:/mnt/spk-compile-sources \
    -v /path/to/smechos_build_root:/mnt/smechos_build_root \
    -v /tmp/smechos_build:/tmp/smechos_build \
    -v /path/to/smechos-iso-output:/mnt/smechos-iso-output \
    ghcr.io/smech-labs/smechos-build:latest smechos-plasma-live --iso live
```

The finished ISO lands at `smechos-iso-output/smechos-plasma-live.iso`.

**The container runs as root, so the ISO output directory ends up
root-owned on the host.** If you hash/sign it as a regular user afterward,
`chown` the directory first — otherwise `sha256sum ... | tee` and `gpg`
fail to write their output files silently (no `set -e` catches it, and the
script prints its final success line regardless):

```bash
sudo chown -R $(whoami):$(whoami) /path/to/smechos-iso-output
cd /path/to/smechos-iso-output
sha256sum smechos-plasma-live.iso > smechos-plasma-live.iso.sha256
gpg --detach-sign --armor smechos-plasma-live.iso
```

### If a phase fails on a missing package

This is the normal way this build breaks, and it's a one-line fix, not a
real bug: `spk-compile.py` targets exact upstream CMake/Meson dependency
names, and mapping "CMake couldn't find `Foo`" to the actual Ubuntu
`-dev` package that provides it is manual. When it happens:

1. Read the actual error (`Could NOT find X` / `Dependency "y" not found` /
   `undefined reference to Z`) — not just "it failed."
2. Find the Ubuntu package: `apt-cache search <name>` or
   `apt-cache policy <guessed-package-name>` inside a `docker run` shell
   into the image (`--entrypoint bash`).
3. Add it to the relevant `apt-get install` block in `Dockerfile` — a new
   `RUN apt-get install ...` line near the bottom keeps the big base layer
   cached and only invalidates a small layer, so the rebuild is seconds,
   not minutes.
4. Rebuild the image and re-run. Already-completed phases and packages are
   skipped via stamp files under `spk-compile-sources/.stamps/` — you only
   pay for the phase that just failed, not the whole build again.

A small number of dependencies aren't in apt at all (KDE-adjacent projects
with their own release schedule, e.g. `polkit-qt-1`, or third-party
libraries like `QCoro6`, `libdisplay-info`). For those, `spk-compile.py`
builds them from source directly — see the small inline build blocks near
the top of `phase_kde()` for the existing pattern to copy if you hit a new
one.

### Current phase list (`smechos-plasma-live`, 27 phases)

Reordered and extended for RC4 — notably `cmake-bootstrap` moved much
earlier (RC4's cross-built packages need real `cmake` sooner than Plasma
alone did), `cross-deps` is new, `wayland`/`wayland-protocols`/`libinput`
now run *before* `mesa` (Mesa's cross-configure actually requires an
already-installed `wayland-client`, confirmed directly — it can only
self-provide `wayland-protocols` as a fallback, not `wayland-client`), and
`bundle-spkg` is new at the end. Don't trust an older copy of this table —
check `SMECHOS_PLASMA_LIVE_PHASES` in `spk-compile.py` itself if in doubt.

| # | Phase | What it does |
|---|---|---|
| 1 | `userland-glibc` | Bootstrap GNU userland against host glibc |
| 2 | `etc` | Write `/etc` skeleton |
| 3 | `systemd` | Install systemd from Debian packages |
| 4 | `systemd-config` | Configure baseline systemd state |
| 5 | `locale` | Generate `en_US.UTF-8` locale |
| 6 | `grub` | Compile GRUB 2.12 EFI + BIOS |
| 7 | `cmake-bootstrap` | Bootstrap CMake |
| 8 | `cross-deps` | Cross-build the Mesa/Qt6 dependency chain (zlib..dbus) |
| 9 | `wayland` | Build Wayland |
| 10 | `wayland-protocols` | Build wayland-protocols |
| 11 | `libinput` | Build libinput |
| 12 | `mesa` | Compile Mesa stack |
| 13 | `qt-deps` | Compile Qt6 modules (host + cross pass per module) |
| 14 | `kde` | Compile KDE Frameworks + Plasma |
| 15 | `plasma-configure` | Configure display manager (PLM, fallback SDDM) |
| 16 | `kwin-deps` | Copy KWin runtime dependencies |
| 17 | `xwayland-deps` | Fetch Xwayland + xkbcomp + xkb-data |
| 18 | `qt6uitools` | Ensure Qt6UITools is present |
| 19 | `kernel` | Compile Linux 6.12.16 |
| 20 | `firmware` | Bundle GPU firmware (amdgpu + i915 + radeon) |
| 21 | `patch-metadata` | Patch metadata for SmechOS branding |
| 22 | `discover` | Compile Plasma Discover + PackageKit |
| 23 | `calamares` | Build the Calamares graphical installer |
| 24 | `firefox` | Install Mozilla Firefox stable |
| 25 | `live-initramfs` | Build the busybox live initramfs |
| — | `bundle` | Bundle output into spk-installable `.tar.xz` packages |
| — | `bundle-spkg` | Emit per-component `.spkg` packages from recorded manifests |

Run one phase in isolation the same way as on bare metal, just via
`docker run` instead of `sudo python3`:

```bash
docker run --rm --privileged --cgroupns=host \
    -v /path/to/spk-compile-sources:/mnt/spk-compile-sources \
    -v /path/to/smechos_build_root:/mnt/smechos_build_root \
    ghcr.io/smech-labs/smechos-build:latest smechos-plasma-live --phase kde
```

---
---

# RC3 — architecture reference (stable)

**This half of the README is stable, shipped, and won't drift** — unlike
the RC4 half below. If something here looks wrong, it's a documentation
bug, not normal staleness.

## What shipped

- **Desktop**: KDE Plasma 6.6.6 LTS, KDE Frameworks 6.24.0 (the
  Bullet-Proof KDE Initiative LTS line — the only Plasma 6.x release with
  real, whole-stack long-term support, not just a shell-only backport).
- **Kernel**: Linux 6.12.16.
- **Package manager**: `spk` 2.1.0, with [pkg.smech.xyz](https://pkg.smech.xyz)
  support.
- **Released**: 2026-09-30. A cosmetic-only patch, the **Founder
  Anniversary Edition** (new wallpaper, GRUB theme, a couple of hidden
  easter eggs — zero code changes, same binaries as RC3), shipped
  2026-10-06.

## RC3 build architecture

RC3's entire pipeline runs via plain `subprocess.run` calls against the
**build container's own toolchain** — there is no chroot, no `--sysroot`,
no cross-compiler. The container's glibc (Ubuntu 24.04, glibc 2.39) is
deliberately matched to the target rootfs's own glibc version, so the
container's native `gcc`/`cmake`/`moc`/etc. can be pointed at the target
tree via `-I`/`-L`/`LD_LIBRARY_PATH` and just work — see "Why builds happen
in a container" above for the full reasoning and the exact failure mode
(`GLIBC_PRIVATE` symbol versioning) that this works around.

This was a real, deliberate architectural choice, not a shortcut — see the
RC4 half below for why it's now being replaced for parts of the pipeline.

## The RC3 boot-chain bug chain (why RC3 took as long as it did)

RC2's ISO built successfully but never reached a working desktop. RC3's
entire scope was root-causing and fixing every layer of that, confirmed via
a real, visually-verified boot each time — not guessed at:

1. **Mesa/GBM was silently broken system-wide** — missing the actual
   `libLLVM.so.1` payload behind a symlink Mesa's DRI loader depends on.
   `kwin_wayland` failed to create a GBM device and exited clean every
   boot, with no crash, no journal entry, no core dump.
2. **The dynamic linker cache was never regenerated** against the built
   rootfs — the same class of dlopen-by-soname failure, independently.
3. **The graphical session was invisible to systemd** —
   `/usr/lib/systemd/user/dbus.{service,socket}` didn't exist in the
   image, so `plasma-workspace-wayland.target` sat permanently inactive
   even though `kwin_wayland` itself ran fine.
4. **Independently-started session components crashed on launch**
   (`kactivitymanagerd`, the PolicyKit agent, Powerdevil, the first-boot
   wizard) with `Could not find the Qt platform plugin "wayland"` —
   `QT_PLUGIN_PATH`/`QML2_IMPORT_PATH` were never propagated into the
   systemd `--user` environment.
5. **`xrdb` was missing entirely**, breaking `kcminit`'s X-resource-merge
   step.
6. **Xwayland couldn't start at all** — `error while loading shared
   libraries: libdecor-0.so.0` — which is why `kcminit`/`ksmserver` each
   blocked for a full 90-second systemd timeout waiting on an X11
   handshake that could never arrive.
7. **Fontconfig didn't exist in the image at all** — every piece of
   desktop text rendered as a tofu box, even after everything above was
   fixed.

All seven are permanently fixed in `spk-compile.py` itself, not patched
onto the ISO after the fact.

## RC3 downloads & verification

ISOs and checksums: [smechos-site downloads page](https://os.smech.xyz/downloads.html),
release notes on [smechlabs-iso-files](https://github.com/Smech-Labs/smechlabs-iso-files/releases).

---
---

# RC4 — "Founder Name Day Edition" (living document)

**This half of the README describes a moving target, target date
2026-11-30 — unlike the RC3 half above.** Where this conflicts with
`spk-compile.py` itself, the code is correct and this is stale — check
`SMECHOS_PLASMA_LIVE_PHASES` and the relevant `phase_*()` functions
directly if anything here looks suspicious.

## RC4 scope

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
glibc-matched container — see the cross-toolchain note earlier in this
README):

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

## RC4 known open issues

- **Black-screen session handoff** — after the first-boot wizard (running
  as a separate `plasma-setup` system user) hands off to the real user's
  desktop session, the screen goes solid black and stays that way.
  Reproduced identically on QEMU (`virtio-vga`) and real AMD Vega hardware
  via a real kexec boot — two unrelated GPU backends hitting the same
  symptom points at a session-handoff logic bug, not a driver issue.
  Real evidence gathered, root cause not yet found: see
  [`BUG_BRIEF_black_screen_handoff.md`](BUG_BRIEF_black_screen_handoff.md).

## RC4 contributor entry points

RC4 is deliberately meant to be the point where SmechOS opens to outside
contributors — see the [co-maintainer call](https://github.com/Smech-Labs/spk-compile/issues/3)
for current open areas (PLM build integration, `spk`/APT parity, real
hardware boot-testing, and others).

## RC4 checkpoint

An honest status review against this scope is planned before 2026-11-30
gets treated as fixed — if that hasn't happened yet by the time you're
reading this, that's the thing to push on, not guessing at new scope.

---
---

## Contributing

See [CONTRIBUTING.md](https://github.com/Smech-Labs/.github/blob/main/CONTRIBUTING.md).
Issues: [smechos-issues](https://github.com/Smech-Labs/smechos-issues) —
several are labelled `good first issue`.

## License

MIT — see [LICENSE](LICENSE).
