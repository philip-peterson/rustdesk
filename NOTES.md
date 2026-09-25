# KDE Wayland fractional-scaling fix — working notes

## What this is

On KDE Plasma Wayland with a fractional display scale (125%/150%/175%/...),
the RustDesk main window renders noticeably smaller than other apps on the
same desktop (reported upstream in rustdesk/rustdesk#15001, #14764, #14491 —
all closed/auto-converted to discussions without a linked fix).

## Root cause

GTK3 (which the Linux Flutter desktop build embeds) has no
`wp-fractional-scale-v1` support — the Wayland protocol KWin needs for a
client to participate correctly in fractional scaling. GNOME/Mutter papers
over this for legacy clients by forcing an inflated integer `wl_output`
scale and compositor-side downscaling; KWin does not, so GDK under-reports
the scale it applies to windows and the app renders as if scaling were still
100%.

This is a Flutter/GTK3 *engine* limitation, not something fixable by
touching rendering internals from this repo. Research also turned up
conflicting reports of whether KWin already partially compensates depending
on Plasma version/config — a fix that assumed a fixed KWin behavior could
double-scale and make things *worse* on some setups. So instead of assuming,
the fix **measures the actual gap at runtime** and only compensates for
what's missing.

## The fix (commit `09252b753` on branch `fix-scale`)

1. **`libs/base/src/platform/linux.rs`** — new `wayland_uniform_output_scale()`.
   Reuses the existing desktop-agnostic `smithay-client-toolkit` Wayland
   client already in this codebase (used elsewhere for output geometry) to
   derive each output's *true* scale from `xdg-output` logical size vs. the
   output's physical mode size. Returns `None` (safe no-op) if outputs
   disagree, since correlating a specific window to a specific output isn't
   attempted (mirrors the caution around the existing
   `try_fix_logical_size()` multi-monitor bug class in `libs/scrap`).

2. **`src/flutter_ffi.rs`** — new `"wayland-uniform-output-scale"` key on the
   existing `main_get_common()` key-dispatch function (mirrors the existing
   `"gnome-monitor-layout-mode"` arm). No new FFI/bridge signature, since
   that function already takes an arbitrary string key — this is why the
   bridge regeneration described below still produces byte-identical output
   for this function.

3. **`flutter/lib/common.dart`** — in `restoreWindowPosition()`, only the
   `WindowType.Main` case: compares the true output scale against what GDK
   is actually reporting (`window_size.getScreenList()`), and if GDK is
   under-reporting by more than ~3%, inflates the window size passed to
   `windowManager.setSize()` by that ratio. On GNOME/X11/integer-scale KDE/
   a KWin version that already compensates, the ratio is ~1 and this is a
   no-op.

### Known limitations

- Only the main window is covered, not remote-session or sub-windows.
- The very first launch (before any window position has ever been saved) is
  untouched — self-resolves after the first close/reopen.
- Content inside the window may still render a bit soft rather than
  pixel-crisp — this corrects the window's on-screen *footprint*, not GTK3's
  internal rendering fidelity (same tradeoff GTK3 apps already accept on
  GNOME).
- Multi-monitor setups with genuinely different scales per output are left
  alone.

## How it's being packaged/tested

RustDesk ships as a Flatpak (`flatpak/rustdesk.json`) built from a `.deb`
produced by `build.py`. Testing requires: build the `.deb`, drop it in
`flatpak/rustdesk.deb`, then run `flatpak-builder`.

This machine (Aurora/Kinoite, immutable rpm-ostree base, no build toolchain
installed on the host) has none of the required tools natively, so the build
runs inside a dedicated **distrobox** container to avoid touching the host
image:

```
distrobox create --name rustdesk-build --image ubuntu:22.04 \
  --home ~/hdd/rustdesk-build-home
```

Everything lives under `~/hdd/rustdesk-build-home` (274GB free there, vs.
8.4GB on the host's `/var`). The repo itself is *not* copied in — distrobox
shares the host home, so the container builds directly against
`/var/home/ironmagma/Code/rustdesk` (this repo, in place).

### Versions used (pinned to match CI — see `.github/workflows/flutter-build.yml`
and `.github/workflows/bridge.yml`)

- Rust: `1.75.0` (via rustup, installed inside the container)
- Flutter: `3.24.5` stable, extracted to `~/hdd/rustdesk-build-home/flutter`
- `flutter_rust_bridge_codegen`: `1.80.1` (`--features uuid`), from crates.io
  — **not** a custom fork; an older comment in `build.py` referencing a fork
  is stale, the real recipe is in `.github/workflows/bridge.yml`.
- `cargo-expand`: `1.0.95` (dependency of the bridge codegen step)
- vcpkg: pinned tag `2023.04.15`, packages `libvpx libyuv opus aom` only —
  **`ffmpeg`/`--hwcodec` deliberately skipped**: it's off by default in
  `build.py` (`get_features()` only adds it when `--hwcodec` is passed) and
  unrelated to this fix; skipping it avoids a much longer vcpkg build.

### Steps done so far (in order)

1. Created the distrobox, installed system deps (see the package list in the
   session — matches `.github/workflows/flutter-build.yml`'s
   `build-rustdesk-linux` job's `install:` block, plus `flatpak` and
   `flatpak-builder` from Ubuntu 22.04's repos directly, so no separate
   Flathub `org.flatpak.Builder` install was needed).
2. `git submodule update --init --recursive` (this repo — `libs/hbb_common`
   was uninitialized).
3. Installed Rust 1.75.0, `flutter_rust_bridge_codegen` 1.80.1 +
   `cargo-expand` 1.0.95, the Flutter 3.24.5 SDK, and built vcpkg's four
   libraries. All confirmed working (`flutter doctor`, `vcpkg list`).

### Steps still to do (pick up here)

1. Generate the FFI bridge (this repo currently has no `generated_bridge.dart`
   — it's gitignored/build-time-only):
   ```
   distrobox enter rustdesk-build -- bash -lc '
     source "$HOME/.cargo/env"
     export PATH="$HOME/flutter/bin:$PATH"
     cd /var/home/ironmagma/Code/rustdesk
     pushd flutter && flutter pub get && popd
     ~/.cargo/bin/flutter_rust_bridge_codegen \
       --rust-input ./src/flutter_ffi.rs \
       --dart-output ./flutter/lib/generated_bridge.dart \
       --c-output ./flutter/macos/Runner/bridge_generated.h
     cp ./flutter/macos/Runner/bridge_generated.h ./flutter/ios/Runner/bridge_generated.h
   '
   ```
   Note: CI generates this bridge against Flutter **3.22.3** specifically
   (with an `extended_text` pubspec downgrade patch), then builds the app
   itself with 3.24.5. Try straight 3.24.5 for `pub get` first since the
   bridge's Dart output doesn't depend on which Flutter minor generated it
   in practice; only chase the 3.22.3 dance if `pub get`/codegen fails.

2. Build the Rust cdylib:
   ```
   distrobox enter rustdesk-build -- bash -lc '
     source "$HOME/.cargo/env"
     cd /var/home/ironmagma/Code/rustdesk
     export VCPKG_ROOT=~/vcpkg
     cargo build --locked --lib --release --features flutter,unix-file-copy-paste
   '
   ```

3. Build the Flutter app + package the `.deb`:
   ```
   distrobox enter rustdesk-build -- bash -lc '
     export PATH="$HOME/flutter/bin:$PATH"
     cd /var/home/ironmagma/Code/rustdesk
     python3 ./build.py --flutter --skip-cargo
   '
   ```
   (`--skip-cargo` because step 2 already built the release lib.)

4. Build the Flatpak:
   ```
   distrobox enter rustdesk-build -- bash -lc '
     cd /var/home/ironmagma/Code/rustdesk
     mv rustdesk-*.deb flatpak/rustdesk.deb
     flatpak --user remote-add --if-not-exists flathub https://dl.flathub.org/repo/flathub.flatpakrepo
     cd flatpak
     git clone https://github.com/flathub/shared-modules.git --depth=1
     flatpak-builder --user --install-deps-from=flathub -y --force-clean \
       --repo=repo ./build ./rustdesk.json
     flatpak build-bundle ./repo rustdesk-test.flatpak com.rustdesk.RustDesk
   '
   ```

5. Install and test:
   ```
   flatpak install --user ./flatpak/rustdesk-test.flatpak
   flatpak run com.rustdesk.RustDesk
   ```
   Close it once (to save a window position — the fix only applies on
   restore/reopen, not the very first launch), reopen, and check the main
   window's size against another app (e.g. Konsole) on your fractionally
   scaled KDE Wayland session.

## Test result: CONFIRMED WORKING (2026-09-24)

Built and ran the Flatpak end-to-end on the actual affected machine/session
(KDE Plasma Wayland, `eDP-1` at `scale=1.5`). Confirmed via `journalctl`/run
log and a screenshot measurement:

- Saved window position (from a prior RustDesk install sharing the same
  Flatpak app-data dir) was `800x600` logical.
- The rendered window measured ~1200x885 physical pixels in a full-screen
  screenshot (accounting for the display's 1.92x screenshot/physical ratio).
- `800 * 1.5 = 1200`, `600 * 1.5 = 900` — matches the measurement, confirming
  `_waylandCompensatedSize()` correctly detected the true 1.5x output scale
  and inflated the window size GDK would otherwise have under-scaled.

Before this fix, the same saved position would have rendered at a literal
800x600 physical footprint — visibly tiny next to Konsole/Okular, matching
the original bug report.

### Build issues hit and fixed along the way (local build only, not code changes)

- `dpkg-deb` on Ubuntu 22.04 defaults to `zstd` compression for `.deb`
  files; `flatpak/rustdesk.json`'s `bsdtar -Oxf rustdesk.deb data.tar.xz`
  expects the older `xz` format (matches upstream CI's Ubuntu 18.04 builder).
  Worked around locally by recompressing the `.deb`'s `control.tar`/`data.tar`
  members from zstd to xz with `ar`/`zstd`/`xz` before feeding it to
  `flatpak-builder`. Not a code issue, just an artifact of using a modern
  Ubuntu container instead of replicating CI's exact old builder image.
- `flatpak-builder` needs `fusermount3` (`fuse3` covers it, but wasn't
  needed once `--disable-rofiles-fuse` was passed instead - simpler in a
  rootless container).
- The container's bundled `bwrap` (0.6.1) failed on
  `Can't bind mount /oldroot/etc/resolv.conf on /newroot/etc/resolv.conf:
  Unable to mount source on destination` - root cause was that distrobox/podman
  mounts `/etc/resolv.conf` as its own `tmpfs` mount point inside the
  container, which bwrap's re-bind logic can choke on. Fixed by unmounting
  it and replacing it with a plain regular file (`sudo umount /etc/resolv.conf`
  then recreate as a normal file with the same contents). Building a newer
  bubblewrap (0.11.0) from source was tried first and didn't fix it alone -
  the resolv.conf fix was the actual cause.

## Cleanup

The whole build environment is disposable and isolated from the host:

```
distrobox rm rustdesk-build --force
rm -rf ~/hdd/rustdesk-build-home
```

Nothing under `~/hdd/rustdesk-build-home` or inside the container touches
the host's immutable rpm-ostree image.
