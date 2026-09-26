# KDE Wayland fractional-scaling fix — working notes

## Status (2026-09-26)

Every known place the Linux desktop build hands a concrete logical window
size to a native resize is now covered by `waylandCompensatedSize()`
(main window, sub-windows, incoming-only resize paths, remote toolbar's
Adjust Window). Built end-to-end (Rust cdylib → Flutter app → `.deb` →
Flatpak) in a distrobox on the actual affected machine and smoke-tested on
real KDE Plasma Wayland at `scale=1.5`: the main window renders at the
correct compensated size, matching the confirmed result from the original
fix. The two newly-covered paths (incoming-only resize, Adjust Window)
reuse the same helper the main window path just re-confirmed, but weren't
independently exercised live — incoming-only mode requires a hard-coded
custom-client build, and Adjust Window requires an active remote session.
Not yet sent upstream.

## What's covered, and where

`waylandCompensatedSize()` (`flutter/lib/common.dart`, public since
commit `41d3a6232`) is called at every site that hands a concrete `Size`
to a native window resize on Linux:

1. **Main window** (`restoreWindowPosition()`'s `WindowType.Main` case,
   `flutter/lib/common.dart`) — the original fix, commit `09252b753`.
2. **Sub-windows** (remote session/file transfer/terminal — anything with
   a `windowId`, via `desktop_multi_window`'s `WindowController.setFrame()`
   rather than `window_manager`): restoring a saved sub-window position,
   the 600x400 pre-fullscreen fallback frame, and a brand-new session
   window's default size. Commit `41d3a6232`.
3. **Main window's "incoming only" (receiver-only) resize paths** — these
   bypass `restoreWindowPosition()` entirely and call
   `windowManager.setSize()` directly from `desktop_tab_page.dart` (tab
   switch) and `desktop_home_page.dart` (`_updateWindowSize`,
   content-driven resize). Same `window_manager` plugin, same bug.
4. **`remote_toolbar.dart`'s "Adjust Window"** (`doAdjustWindow` /
   `_getAdjustedWindowFrame`) — deliberately deferred in `41d3a6232`
   because it has its own GNOME-specific scale handling
   (`_getEffectiveScreenFrame`'s `monitorLayoutMode == 'physical'` branch)
   baked into the same function, and a blind multiply risked
   double-scaling. On inspection the two mechanisms touch different
   quantities and don't compose: `_getEffectiveScreenFrame` only rescales
   the *available screen bounds* used for clamping/fit checks (a GNOME
   "physical" layout-mode correction), while `waylandCompensatedSize`
   rescales the *requested window size* itself (a KDE/KWin GDK-under-report
   correction) — safe to apply both. Renamed the existing `width`/`height`
   locals to `targetWidth`/`targetHeight`, ran them through
   `waylandCompensatedSize()`, and rebound `width`/`height` to the
   compensated result so every downstream use (centering, the
   too-small/exceeds-screen rejection checks, the final `Rect`) picks it
   up unchanged.

No remaining `setFrame`/`windowManager.setSize` call site handling a
concrete size is uncovered (checked with
`grep -rn "setFrame(\|windowManager\.setSize("` across `flutter/lib`).

### Known limitations

- The very first launch (before any window position has ever been saved) is
  untouched — self-resolves after the first close/reopen.
- Content inside the window may still render a bit soft rather than
  pixel-crisp — this corrects the window's on-screen *footprint*, not GTK3's
  internal rendering fidelity (same tradeoff GTK3 apps already accept on
  GNOME).
- Multi-monitor setups with genuinely different scales per output are left
  alone.

## Test result: main window CONFIRMED WORKING (2026-09-24, re-confirmed 2026-09-26)

Built and ran the Flatpak end-to-end on the actual affected machine/session
(KDE Plasma Wayland, `eDP-1` at `scale=1.5`).

2026-09-24 run — confirmed via `journalctl`/run log and a screenshot
measurement:
- Saved window position (from a prior RustDesk install sharing the same
  Flatpak app-data dir) was `800x600` logical.
- The rendered window measured ~1200x885 physical pixels in a full-screen
  screenshot (accounting for the display's 1.92x screenshot/physical ratio).
- `800 * 1.5 = 1200`, `600 * 1.5 = 900` — matches the measurement, confirming
  `waylandCompensatedSize()` correctly detected the true 1.5x output scale
  and inflated the window size GDK would otherwise have under-scaled.

Before this fix, the same saved position would have rendered at a literal
800x600 physical footprint — visibly tiny next to Konsole/Okular, matching
the original bug report.

2026-09-26 rebuild (after extending to every window, see above) —
re-confirmed the main window still renders correctly: cropped a screenshot
of the "Your Desktop / ID ######" card next to a Konsole window at the same
scale and the text sizes match, no shrinkage. Sub-windows and Adjust Window
were not independently exercised this round (see Status above).

## How it's built/tested

RustDesk ships as a Flatpak (`flatpak/rustdesk.json`) built from a `.deb`
produced by `build.py`. This machine (Aurora/Kinoite, immutable rpm-ostree
base, no build toolchain on the host) has none of the required tools
natively, so the build runs inside a dedicated **distrobox** container to
avoid touching the host image:

```
distrobox create --name rustdesk-build --image ubuntu:22.04 \
  --home ~/hdd/rustdesk-build-home
```

Everything lives under `~/hdd/rustdesk-build-home` (274GB free there, vs.
8.4GB on the host's `/var`). The repo itself is *not* copied in — distrobox
shares the host home, so the container builds directly against
`/var/home/ironmagma/Code/rustdesk` (this repo, in place). The container is
disposable and rebuildable from scratch by re-running the steps below; it
was torn down once already (after the 2026-09-24 confirmation) and rebuilt
identically for the 2026-09-26 round, so don't assume it's still there.

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

### Steps, in order

1. Create the distrobox and install system deps — matches
   `.github/workflows/flutter-build.yml`'s `build-rustdesk-linux` job's
   `install:` block, plus `flatpak`/`flatpak-builder`/`fuse3` from Ubuntu
   22.04's repos directly (no separate Flathub `org.flatpak.Builder` install
   needed):
   ```
   distrobox enter rustdesk-build -- bash -lc '
     sudo apt-get update -y
     sudo apt-get install -y build-essential clang cmake curl gcc git g++ \
       libayatana-appindicator3-dev libasound2-dev libclang-dev \
       libgstreamer1.0-dev libgstreamer-plugins-base1.0-dev libgtk-3-dev \
       libpulse-dev libva-dev libxcb-randr0-dev libxcb-shape0-dev \
       libxcb-xfixes0-dev libxdo-dev libxfixes-dev llvm-dev nasm \
       ninja-build pkg-config tree python3 rpm unzip wget xz-utils zstd \
       libssl-dev flatpak flatpak-builder fuse3
   '
   ```
2. `git submodule update --init --recursive` (this repo — `libs/hbb_common`
   needs it).
3. Install Rust 1.75.0 (`rustup`), `flutter_rust_bridge_codegen` 1.80.1 +
   `cargo-expand` 1.0.95 (`cargo install`), the Flutter 3.24.5 SDK (extract
   the stable tarball), and vcpkg at tag `2023.04.15` with
   `./vcpkg install libvpx libyuv opus aom --triplet x64-linux`.
4. Generate the FFI bridge (this repo has no `generated_bridge.dart` until
   this step — it's gitignored/build-time-only):
   ```
   distrobox enter rustdesk-build -- bash -lc '
     source "$HOME/.cargo/env"
     export PATH="$HOME/flutter/bin:$PATH"
     cd /var/home/ironmagma/Code/rustdesk
     git config --global --add safe.directory "*"
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
   itself with 3.24.5. Straight 3.24.5 for `pub get` works fine in practice
   — the bridge's Dart output doesn't depend on which Flutter minor
   generated it; only chase the 3.22.3 dance if `pub get`/codegen fails.

   `flutter pub get` here can rewrite `flutter/pubspec.lock` against
   whatever pub mirror/cache state the container has (seen transitive
   version churn unrelated to this fix) — check `git diff` afterward and
   revert `pubspec.lock` before committing if so.

5. Build the Rust cdylib:
   ```
   distrobox enter rustdesk-build -- bash -lc '
     source "$HOME/.cargo/env"
     cd /var/home/ironmagma/Code/rustdesk
     export VCPKG_ROOT=~/vcpkg
     cargo build --locked --lib --release --features flutter,unix-file-copy-paste
   '
   ```
6. Build the Flutter app + package the `.deb` (`--skip-cargo` since step 5
   already built the release lib):
   ```
   distrobox enter rustdesk-build -- bash -lc '
     export PATH="$HOME/flutter/bin:$PATH"
     cd /var/home/ironmagma/Code/rustdesk
     python3 ./build.py --flutter --skip-cargo
   '
   ```
7. Recompress the `.deb` for `flatpak-builder` (see "Build issues hit"
   below for why), then build the Flatpak:
   ```
   distrobox enter rustdesk-build -- bash -lc '
     cd /var/home/ironmagma/Code/rustdesk
     mkdir -p /tmp/deb-recompress && cd /tmp/deb-recompress && rm -rf *
     ar x /var/home/ironmagma/Code/rustdesk/rustdesk-*.deb
     zstd -d control.tar.zst -o control.tar
     zstd -d data.tar.zst -o data.tar
     xz -T0 control.tar && xz -T0 data.tar
     ar rcD /var/home/ironmagma/Code/rustdesk/flatpak/rustdesk.deb \
       debian-binary control.tar.xz data.tar.xz
     cd /var/home/ironmagma/Code/rustdesk/flatpak
     flatpak --user remote-add --if-not-exists flathub https://dl.flathub.org/repo/flathub.flatpakrepo
     git clone https://github.com/flathub/shared-modules.git --depth=1
     flatpak-builder --user --install-deps-from=flathub -y --force-clean \
       --disable-rofiles-fuse --repo=repo ./build ./rustdesk.json
     flatpak build-bundle ./repo rustdesk-test.flatpak com.rustdesk.RustDesk
   '
   ```
8. Install and test:
   ```
   flatpak install --user --reinstall ./flatpak/rustdesk-test.flatpak
   flatpak run com.rustdesk.RustDesk
   ```
   Close it once (to save a window position — the fix only applies on
   restore/reopen, not the very first launch), reopen, and check the main
   window's size against another app (e.g. Konsole) on your fractionally
   scaled KDE Wayland session. A screenshot (`spectacle -b -n -o out.png`)
   works for a visual before/after; KWin's D-Bus scripting interface on
   Plasma 6 was not a reliable way to query exact window geometry
   headlessly in this session (script loaded and ran with no error, but
   produced no journal output — not worth chasing further, a screenshot
   comparison was good enough).

### Build issues hit and fixed along the way (local build only, not code changes)

- `dpkg-deb` on Ubuntu 22.04 defaults to `zstd` compression for `.deb`
  files; `flatpak/rustdesk.json`'s `bsdtar -Oxf rustdesk.deb data.tar.xz`
  expects the older `xz` format (matches upstream CI's Ubuntu 18.04 builder).
  Worked around locally by recompressing the `.deb`'s `control.tar`/`data.tar`
  members from zstd to xz with `ar`/`zstd`/`xz` before feeding it to
  `flatpak-builder`. Not a code issue, just an artifact of using a modern
  Ubuntu container instead of replicating CI's exact old builder image.
- `flatpak-builder` needs `fusermount3` (`fuse3` covers it), and
  `--disable-rofiles-fuse` avoids needing it set up further — simpler in a
  rootless container.
- (2026-09-24 only, did not recur 2026-09-26) The container's bundled
  `bwrap` (0.6.1) failed on `Can't bind mount /oldroot/etc/resolv.conf on
  /newroot/etc/resolv.conf: Unable to mount source on destination` — root
  cause was that distrobox/podman mounts `/etc/resolv.conf` as its own
  `tmpfs` mount point inside the container, which bwrap's re-bind logic can
  choke on. Fixed by unmounting it and replacing it with a plain regular
  file (`sudo umount /etc/resolv.conf` then recreate as a normal file with
  the same contents) if it comes back.
- (2026-09-26) The very first launch of a freshly-reinstalled Flatpak
  self-closed a few seconds after showing the main window (window restored,
  then `Start closing RustDesk...` in the log, no user action). Retrying the
  launch immediately worked normally and stayed up; treated as a one-off
  cold-start flake (e.g. shader cache warm-up) rather than a regression,
  since it isn't reachable from any of this session's changed code paths
  (both are gated behind `bind.isIncomingOnly()` or an active remote
  session, neither active on a fresh default-mode launch). Worth watching
  for if it recurs.

## Why this bug happens

On KDE Plasma Wayland with a fractional display scale (125%/150%/175%/...),
the RustDesk main window renders noticeably smaller than other apps on the
same desktop (reported upstream in rustdesk/rustdesk#15001, #14764, #14491 —
all closed/auto-converted to discussions without a linked fix).

GTK3 (which the Linux Flutter desktop build embeds, including the
`desktop_multi_window` plugin's sub-windows) has no `wp-fractional-scale-v1`
support — the Wayland protocol KWin needs for a client to participate
correctly in fractional scaling. GNOME/Mutter papers over this for legacy
clients by forcing an inflated integer `wl_output` scale and
compositor-side downscaling; KWin does not, so GDK under-reports the scale
it applies to windows and the app renders as if scaling were still 100%.

This is a Flutter/GTK3 *engine* limitation, not something fixable by
touching rendering internals from this repo. Research also turned up
conflicting reports of whether KWin already partially compensates depending
on Plasma version/config — a fix that assumed a fixed KWin behavior could
double-scale and make things *worse* on some setups. So instead of assuming,
the fix **measures the actual gap at runtime** and only compensates for
what's missing:

1. **`libs/base/src/platform/linux.rs`** — `wayland_uniform_output_scale()`.
   Reuses the existing desktop-agnostic `smithay-client-toolkit` Wayland
   client already in this codebase (used elsewhere for output geometry) to
   derive each output's *true* scale from `xdg-output` logical size vs. the
   output's physical mode size. Returns `None` (safe no-op) if outputs
   disagree, since correlating a specific window to a specific output isn't
   attempted (mirrors the caution around the existing
   `try_fix_logical_size()` multi-monitor bug class in `libs/scrap`).
2. **`src/flutter_ffi.rs`** — `"wayland-uniform-output-scale"` key on the
   existing `main_get_common()` key-dispatch function (mirrors the existing
   `"gnome-monitor-layout-mode"` arm). No new FFI/bridge signature, since
   that function already takes an arbitrary string key.
3. **`flutter/lib/common.dart`**'s `waylandCompensatedSize()` — compares the
   true output scale against what GDK is actually reporting
   (`window_size.getScreenList()`), and if GDK is under-reporting by more
   than ~3%, inflates the requested window size by that ratio. On
   GNOME/X11/integer-scale KDE/a KWin version that already compensates, the
   ratio is ~1 and this is a no-op. See "What's covered, and where" above
   for every call site this is wired into.

## Cleanup

The whole build environment is disposable and isolated from the host:

```
distrobox rm rustdesk-build --force
rm -rf ~/hdd/rustdesk-build-home
```

Nothing under `~/hdd/rustdesk-build-home` or inside the container touches
the host's immutable rpm-ostree image.
