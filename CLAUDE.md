# rmweb (Libby fork) — reMarkable Paper Pro

## What this is
A fork of `snydersaurus/rmweb` (Libby reading mode for the Paper Pro Move), which is itself a fork of
`exp78/rmweb` v0.9.7 (a WPE WebKit browser for reMarkable e-ink tablets). This fork is for reading
Libby loans (libbyapp.com) on a **reMarkable Paper Pro** (1620x2160, "ferrari", i.MX8M Mini).
The Libby changes were written and tested on a Paper Pro **Move** (954x1696, OS 3.27.3). The Paper Pro
is the target of this fork.

Libby mode runs as an **AppLoad window** (XOVI + AppLoad, `"qtfb": true`): xochitl keeps running,
rmweb paints into AppLoad's shared framebuffer and reads finger and pen input from its socket.

## Layout
- `engine/wpeqt/main.cpp` — the whole app (WPE engine, view, input, Libby handling, qtfb window mode).
  Headers next to it: `gesture.h` (swipe thresholds, scaled to the panel), `profile.h` (settings and
  profile files), `startpage.h`, `keyboard.h`, `url.h`.
- `device/libby-window/libby-window.sh` — the AppLoad launcher. `device/libby-entry.sh` — full-screen launcher.
  `device/rmweb-env.sh` — shared env (also turns on the TLS option from a marker file).
- `device/appload/libby/external.manifest.json` — the AppLoad manifest.
- `packaging/vellum/` — Vellum recipe (`libby-reader`). `scripts/build-vellum-apk.sh` builds the `.apk`.
- `docs/libby-wrapper.md` (what the fork changes), `docs/tls.md` (the TLS option),
  `docs/upstream-status-log.md` (historical upstream notes; some claims in it are wrong, check the code).

## Build
Cross-compile only, with the official ferrari Yocto SDK (aarch64, glibc 2.39) in Docker:
```sh
./scripts/fetch-sdk.sh
docker build -f toolchain/Dockerfile -t rmweb-sdk .
./scripts/stage-from-release.sh build/src/rmweb-0.9.7.tar.gz build/src/wpewebkit-2.48.5.tar.xz
./scripts/build-wpeqt.sh        # about a minute; only rmweb-wpeqt, WebKit comes from the upstream release
./scripts/package-libby.sh      # dist/libby/ and dist/libby-<version>.tar.gz
```
Host tests: `./scripts/run-tests.sh`. The version is in `LIBBY_VERSION`.

## The device
- Paper Pro: `ssh root@10.11.99.1` over USB. BusyBox, so `head -n N`, `ps -ef`.
- Install target: `/home/root/xovi/exthome/appload/libby` (self-contained app folder). `/` is full and
  read-only; WebKit's helpers need an overlay on `/usr/libexec`, which `libby-window.sh` mounts.
- Log: `rmweb.log` in the app folder. Profile: `/home/root/.rmweb` (settings, cookies). WebKit data:
  `/home/root/.local/share/wpe`.
- AppLoad must support the device's OS. v0.5.3 crash-loops on 3.28; use a build from its `3.28` branch.
- **xochitl start limit:** 4 starts in 10 minutes triggers an emergency reboot. Leave ~10 s between
  launches and restarts; use `/home/root/xovi/start` (not a bare `systemctl restart xochitl`).

## Paper Pro findings (this fork)
- **Aspect ratio:** the manifest uses `"original"` (upstream's rmweb app does too). `"move"` gave a
  wrongly shaped window on the Paper Pro. The Move needs `"move"`.
- **Window format:** `qtfbConnect()` picks the Paper Pro RGBA format (1620x2160) unless
  `/proc/device-tree/model` says Chiappa/Move.
- **TLS:** Libby's API hosts need three extra TLS 1.2 suites that reMarkable's system policy leaves out
  (`docs/tls.md`). It is opt-in: the app asks on first run and creates `tls-compat.on` in the app
  folder. After enabling it, check the network process: `OPENSSL_CONF` should be set in
  `/proc/$(pgrep WPENetworkProc | sed -n 1p)/environ`.
- **Setup code screen** ("Recover your data" → display-code) can hang with nothing in the log. Content
  blocking was switched off (`block=0` in `/home/root/.rmweb/settings.txt`) when it was first fixed, but
  whether that or just waiting on the slow interpreter-only JS fixed it was not isolated.
- **DPR is fixed at 2.0** (`RMWEB_DPR` overrides): the page sees 810x1080 CSS px here and 477x848 on the Move.
  Libby's layout and edge-tap behavior at 810 px were not verified.
- **Debugging page requests:** put a `user.js` in `/home/root/.rmweb/` that wraps `fetch`/XHR and posts
  to `window.webkit.messageHandlers.rmweb`; lines show up as `[msg] ...` in `rmweb.log`. It runs in every
  frame, so remove it afterwards. A longer harness is `scripts/debug/user-debug.js` (evaluates posted code,
  debug use only).

## Performance and debugging on the Paper Pro (measured 2026-10-09)
- **Page turns are CPU-bound in the WebKit web process,** not the network or panel: its main thread sat at ~90% of
  one core, with ~1 core more in llvmpipe compositing. Magazine page turn, key to new frame: 4.5 s median with the
  interpreter, ~3 s with the baseline JIT + 3 Skia painting threads (`libby-window.sh` sets both). DPR 1.5 was not
  faster (3.5 s) and looks softer; kept at 2.0. Memory is not growing (web process 640-760 MB, WebKit trims itself).
- `RMWEB_SKIA_THREADS` / `RMWEB_JIT` are read when `rmweb-env.sh` is sourced: export them before it.
- **Magazine article view** scrolls only with a real wheel event, and the paged view ignores wheel events (and the
  article view ignores arrow keys), so a vertical swipe on `/open/magazine/` sends both (`sendPageWheel`).
- **Measuring without a debugger:** `QT_MESSAGE_PATTERN="%{time boot} %{message}"` and
  `QT_LOGGING_RULES=rmweb.engine.debug=true` in the launcher timestamp the log (`[page] key` to `frame N ... NEW`
  is the render latency); `RMWEB_DEBUG_JSFILE=/tmp/rmdbg.js` runs `#key R|L`, `#wheel x y dy`, `#click x y`
  (window mode has no `#grab`: copy `/dev/shm/qtfb_*`, 1620x2160 RGBA, and convert it). Per-thread CPU:
  diff `/proc/<WPEWebProcess>/task/*/stat` utime+stime. Put debug env AFTER `rmweb-env.sh` only for runtime
  variables; remove it again afterwards.
- BusyBox quirks: `head -N` fails (use `head -n N`), `pgrep -x` and `getconf` are unreliable or missing.

## Engine facts that matter
- Rendering is CPU-only: Skia CPU painting, Mesa llvmpipe software EGL. JSC runs interpreter-only
  (JIT is broken on this toolchain), so heavy JS pages are slow.
- The page cannot be driven by script alone in Libby's reader (cross-origin frame): swipes and edge taps
  are sent as real arrow keys and clicks through `wpe_view_event`.
- Never pass JS containing `%` through `g_strdup_printf` without doubling it (`%%`): it crashes.
- A process segfault on the device can trigger a watchdog reboot (~100 s). SIGTERM exits via `_Exit`
  after draining the panel; Qt/WebKit teardown is never used.
