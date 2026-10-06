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

### Release status

**RC3** is the current shipped release (Plasma 6.6.6 LTS / KDE Frameworks
6.24.0, kernel 6.12.16), with the **Founder Anniversary Edition** — a
cosmetic patch on top of RC3 (new wallpaper, branding, no code changes) —
released the same day this README was last updated.

**RC4 ("Founder Name Day Edition")** is in active development, targeting
real cross-compilation via a dedicated toolchain (`x86_64-smechos-linux-gnu`,
built with crosstool-ng) instead of relying on the build container's own
glibc matching the target's — see the cross-toolchain note below. Some
phases (`cross-deps`, parts of `mesa`/`qt-deps`/`kernel`) already build this
way; most of the rest of the pipeline still uses the container-glibc-match
approach described next. If you're picking up an issue, check which phase
you're touching — the two approaches have different debugging patterns.

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

## Contributing

See [CONTRIBUTING.md](https://github.com/Smech-Labs/.github/blob/main/CONTRIBUTING.md).
Issues: [smechos-issues](https://github.com/Smech-Labs/smechos-issues) —
several are labelled `good first issue`.

## License

MIT — see [LICENSE](LICENSE).
