# NTR-HR-VC: changes on top of xzn/ntr-hr

NTR-HR-VC is a fork of [xzn/ntr-hr](https://github.com/xzn/ntr-hr) (branch `full-width`), used together with the NTR-VC viewer (RAM / audio / frame sharing). All 3DS-side changes are based on upstream commit `c0d2034` ("readme audio"), which is the code the original NTR-HR CIA was built from.

Two builds were released from this work. Each one includes everything in the one before it.

| Build | CIA | Source commit | Date |
|---|---|---|---|
| streamfix | `BootNTR-HR-streamfix.cia` | `d56355a` | 2026-10-08 (re-applied 2026-10-19) |
| SDlog | `BootNTR-HR-sdlog.cia` | `366de82` (on top of `1db8a6f`) | 2026-10-19 |

## 1. streamfix (`d56355a`)

Fixes for Remote Play streams that stalled and never came back.

* **NWM ns main loop spin**: `nsMainLoop` pauses 0.5 s after a failed poll/accept instead of spinning at top priority (`source/ns.c`).
* **Socket timeouts**: 5 s send/receive limits on the control sockets so a half-open connection cannot block forever (`source/rt.c`).
* **Stream-only restart**: a control-link error restarts the stream (`nsControlRecv` returns 0) instead of tearing down the whole NTR session; the receive check uses `&&` so a short read is not treated as success.
* **Init retry**: NWM init and thread creation retry every 1 s instead of failing for good (`thread_main.rs`, `thread_nwm/vars.rs`).
* **Audio lock leaks**: two early-return paths in the audio thread no longer leave the lock held (`thread_audio.rs`).

## 2. SDlog (`1db8a6f` + `366de82`)

Adds a log on the SD card so failures can be diagnosed after the Wi-Fi link (and the viewer log) is gone, plus the fixes from the 2026-10-19 review.

### SD card log (`1db8a6f`)

* Every NTR debug line is written to `sd:/ntr-hr-log.txt`, flushed every 2 s.
* A `@ date time (up N s)` stamp is written each minute, so the last stamp shows roughly when the 3DS system core stopped.
* At 1 MB the log rotates to `sd:/ntr-hr-log.old.txt`.
* Control link connects, disconnects and listen errors are logged.

### Hardening (`366de82`)

* **NWM poll spin**: `nsPoll2` checks `POLLERR/POLLHUP/POLLNVAL`, recreates the UDP socket after 100 ms, and uses a 1 s poll timeout.
* **Bounded waits**: VBlank waits use a 100 ms `gspWaitForEventTimeout` (new libctru function, see below); `nwmPause` gives up after 5 s; a 3 s join watchdog logs which thread did not exit on a stream restart.
* **Port 8000**: the accepted socket is non-blocking in every process, with a 10 s idle / 5 s mid-packet timeout, so a dead viewer no longer holds the port.
* **No message boxes on network errors**: bind, listen and NWM socket errors are logged instead of popping a blocking `showDbg`.
* **Log timestamps** no longer overflow after 68 s.
* Restarts (`Nwm main loop restarted`) and `kcp session_established` are always logged, not only in debug mode.
* Lines containing fail/timeout/error/restart are flushed to SD within 0.1 s.

## libctru patch

`366de82` moves the `libctru` submodule to a local commit (`030623e`, "Add gspWaitForEventTimeout") that is not published on GitHub. On this branch the submodule points back at the published `xzn/libctru` commit `9d0678a`, and the change is shipped as a patch. Apply it before building:

```sh
git submodule update --init
git -C libctru am ../patches/libctru-0001-Add-gspWaitForEventTimeout.patch
```

## Building

Same as upstream NTR-HR: devkitARM + Rust nightly (`nightly-2026-08-02` was used) + LLVM/libclang for bindgen.

```sh
make -C libctru/libctru lib/libctru.a
make -j8
```

Then copy `release/ntr.hr.*.bin` into BootNTR's `romfs/`, regenerate `config_ver.h` from the payload hashes, and build BootNTR to get the CIA. On Windows, touch `source/nwm_rs/build.rs` after changing headers, or bindgen will not rerun.

## Known open issues

From the two reviews of the SDlog build (`docs/reviews/`):

* A stream restart can still hang inside NWM's `nwmSendPacket` while the link is down. The watchdog now names the stuck thread in the SD log, but does not recover it.
* `nsPoll2` relies on libctru's guessed poll flag values; an unknown error bit could still cause a fast re-poll.
* A persistent network error can cause up to ten urgent SD flushes per second; the urgent flush should be rate-limited.
* Not done yet: audio thread priority/poll, a handle leak, a periodic NWM status line, viewer-side control link fixes.
