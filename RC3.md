# SmechOS RC3 — architecture reference

**Status: stable, shipped.** This describes RC3 as it actually shipped —
it's not a moving target like [RC4.md](RC4.md). If something here looks
wrong, it's a documentation bug, not drift — RC3's own architecture isn't
changing.

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

## Build architecture

RC3's entire pipeline runs via plain `subprocess.run` calls against the
**build container's own toolchain** — there is no chroot, no `--sysroot`,
no cross-compiler. The container's glibc (Ubuntu 24.04, glibc 2.39) is
deliberately matched to the target rootfs's own glibc version, so the
container's native `gcc`/`cmake`/`moc`/etc. can be pointed at the target
tree via `-I`/`-L`/`LD_LIBRARY_PATH` and just work — see the main
[README](README.md)'s "Why builds happen in a container" section for the
full reasoning and the exact failure mode (`GLIBC_PRIVATE` symbol
versioning) that this works around.

This is a real, deliberate architectural choice, not a shortcut — see
RC4.md for why it's now being replaced for parts of the pipeline.

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

## Downloads & verification

ISOs and checksums: [smechos-site downloads page](https://os.smech.xyz/downloads.html),
release notes on [smechlabs-iso-files](https://github.com/Smech-Labs/smechlabs-iso-files/releases).
