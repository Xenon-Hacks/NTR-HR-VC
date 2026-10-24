# NTR-HR 3DS build: second, independent review (2026-10-19)

Reviewed: `D:\Claude\ntr-hr-full-width` at `366de82` (working tree clean), as built into `D:\Claude\BootNTR-HR-sdlog.cia` and `BootNTR-HR-sdlog.cxi` (both 00:43 local on 2026-10-19). The brief named the CIA `BootNTR-HR-sdlog-20261019T044325-0eb0.cia`; no file with that name exists in `D:\Claude`, and the file reviewed is the only sdlog CIA there. The earlier review (`NTR-HR-3DS-review-2026-10-19.md`) was read; conclusions below are my own. Nothing in the source tree was changed and nothing was rebuilt.

## Does the CIA match the source?

Yes, confirmed.

* `ctrtool --contents` on the CIA gives one content (`content.0000.2f2f5e14`) whose SHA-256 equals `BootNTR-HR-sdlog.cxi`.
* The romfs inside that content holds the seven `ntr.hr.*.bin` payloads. Their SHA-256 hashes are identical to `ntr-hr-full-width\release\*.bin` and to `BootNTR\romfs\*.bin`.
* The release bins contain the strings that `366de82` introduced ("nwm_recv_sock poll error", "thread not exited 3 s", "Nwm pause requested", "Nwm main loop restarted", "log buffer full", "NTR-HR started", unconditional "kcp session_established"), and no longer contain the old `showDbg` "bind failed: " text in the menu payload. The old-3DS nwm payload contains the same new network strings.
* `ntr.hr.nwm.map` shows `gspWaitForEventTimeout` and `LightEvent_WaitTimeout` linked in, so the libctru submodule bump (`030623e`) is in the build.

Caveat: the release bins are stamped 00:42 and the commit was made at 00:43, so this is "built from the same files that were then committed", not "built after commit". The string evidence above covers every file the commit touched.

## Claims, checked against the code

| Claim | Status |
|---|---|
| Oct 8 fixes still present (0.5 s backoff in `nsMainLoop`, 5 s socket limits in `rt.c`, stream-only restart via `nsControlRecv` returning 0, `&&` recv check, 1 s init/thread-create retry, two audio lock leaks) | Confirmed, all in `d56355a` and in the current tree |
| Backoff and reconnect for the NWM ns thread on socket errors | Confirmed with a gap (finding 2): 1 s `poll2` timeout, `POLLERR/HUP/NVAL` on the UDP socket recreates it after 100 ms, TCP errors fall through to accept/recv |
| Time limits on stream restarts: VBlank wait | Confirmed: 100 ms `gspWaitForEventTimeout`, and `do_screen` checks `reset_threads()` each loop |
| Time limits on stream restarts: Wi-Fi send call | **Not implemented** (finding 1) |
| `nwmPause` bounded | Confirmed: 5 s, then logs and carries on |
| Port 8000 freed after a half-open connection | Confirmed: accepted socket forced `O_NONBLOCK` in every process, `rtRecvSocket` returns -1 after 10 s idle or 5 s mid-packet, loop closes the socket and goes back to `accept` |
| No blocking message box on network errors | Confirmed for bind, listen, nwm socket, nwm bind, poll failure. Remaining `showDbg` in NWM are boot-only (`srvInit`, payload sync), mempool corruption checks and the Rust panic handler |
| SD log `ntr-hr-log.txt`, lines flushed every 2 s, stamp every minute | Confirmed (`nsLogFlush` from the menu main loop, 20 ticks of 100 ms; stamp every 600 ticks) |
| Error lines flushed within 0.1 s | Confirmed: `nsLogWrite` scans for fail/timeout/error/restart; the literal parts of a format string reach it as one chunk, so the words are matched |
| Rotation to `ntr-hr-log.old.txt` at 1 MB | Confirmed (`size > 0x100000` at open, delete old, rename, reopen) |
| Timestamps correct past 68 s | Confirmed in both `ns.c` and `ui.c` (seconds and microseconds computed separately, no overflow); the minute stamp uses `tick / (SYSCLOCK_ARM11/1000)` in u64 |
| Restart and connection lines always logged | Confirmed ("Nwm main loop restarted" and "kcp session_established" are no longer gated on debug mode; "client connected/disconnected", "listen socket closed", watchdog lines added) |

## Findings, ranked by severity

### 1. The restart hang is not bounded; it is only named (high, confirmed)

`utils.rs:214-225`, `thread_nwm/vars.rs:967-977`, `thread_main.rs:321-445`

The brief says both waits that had no limit, VBlank and the Wi-Fi send call, now have one. Only VBlank does. `JoinThread::drop` waits 3 s, prints the thread name and "not exited 3 s after reset, still waiting", then calls `svcWaitSynchronization(handle, -1)` again. `nwm_output` still tail-calls `nwmSendPacket`, NWM's internal TX routine, with no timeout, and there is nothing in NTR that can interrupt it. So if the nwm thread blocks inside NWM's TX path while the link is down, the stream still never restarts; the difference from last night is that the SD log will now say `nwm` and the watchdog line.

What is real: `nwmPause` gives up after 5 s, so NWM's notification thread no longer wedges on a sleep or shutdown during a hung restart. That removes the "sleep lid during a hang kills Wi-Fi for good" case, but not the "stream stays dead" case.

This is the right behaviour for a diagnostic build (the next failure will name the thread), but it should not be described as a fix. If the log shows `nwm` as the stuck thread, the next step is to stop calling `nwmSendPacket` while NWM reports the link down, not to add another timeout.

### 2. The anti-spin check depends on guessed poll flag values (medium, inferred)

`main_nwm.c:270-290`, `libctru/include/poll.h:3-8`

`nsPoll2` now treats `POLLERR | POLLHUP | POLLNVAL` on the UDP socket as "recreate", which is the intended fix for the top-priority spin. libctru's `poll.h` defines those bits as `0x08 // probably`, `0x04 // unknown ???` and `0x20`. If SOCU reports an error with any other bit, `poll2` returns a positive count, no branch in `nsPoll2` matches, it returns 0, and `WHILE_POLL2` re-polls at once: the spin that finding 1 of the earlier review described is still possible. The 1 s timeout does not help, because the call returns immediately.

Cheap closure that does not depend on the bit values: at the end of `nsPoll2`, if `nready > 0` and neither `pi[0].revents` nor `pi[1].revents` matched a handled case, log the two `revents` values once and `svcSleepThread(100 ms)` before returning 0. Same for the TCP side with `pi[0].revents` set but no `POLLIN/HUP/ERR/NVAL`.

### 3. A persistent error turns into an SD write storm (medium, confirmed)

`main_nwm.c:262-290`, `main_menu.c:1001-1016, 1213-1222`

In a persistent error state the NWM ns thread logs on every pass: "nwm_recv_sock poll error" every 100 ms, or "socket poll failed" / "nwm socket failed" every 100 to 500 ms. Each line is a synchronous IPC from NWM's top-priority ns thread to the menu port thread (also priority 0x10, core 1), and each contains "error" or "fail", so each sets `nsLogUrgent`. The menu main loop then does `FSUSER_OpenFileDirectly`, `FSFILE_GetSize`, `FSFILE_Write(FS_WRITE_FLUSH)` and `FSFILE_Close` on the next 100 ms tick, so up to ten open/write/close cycles a second on the same SD card the game is installed on, for as long as the error lasts. That is not a freeze, but it adds SD contention and port-thread load exactly when Wi-Fi is already failing, which is the opposite of what the build is trying to measure.

Fix: rate-limit the urgent flush (not more than once per second), and in `nsPoll2` log the first occurrence of each error and then only every Nth (or once per 5 s).

### 4. SD writes now sit in the loop that handles X+Y (medium, confirmed)

`main_menu.c:1203-1225`

`nsLogFlush` runs from `mainThread`'s 100 ms loop, the same loop that checks `getKeys() == NTRMenuHotkey` and calls `lockCpuClock`. The flush before `showMainMenu` was removed, which is good, but a flush still runs in that loop every 2 s and on every urgent line. If FS or the SD card stalls (a slow card, FS busy with the game's own reads, or the FS wedge that the earlier review mentioned), `FSFILE_Write` blocks the main thread and X+Y is not checked until it returns. The console does not freeze, but "X+Y does nothing" becomes a possible new symptom, which muddies the very X+Y freeze being chased.

Also worth knowing: while the NTR menu is open, `showMainMenu` runs on this thread, so no flush happens; the 8 KB log buffer fills and later lines are dropped (the drop count is logged afterwards).

Fix: move `nsLogFlush` to its own low-priority thread (0x3f) that sleeps 100 ms and flushes; the main loop then only sets flags.

Related, low and inferred: the menu is injected into Home Menu and writes through Home Menu's own `fs:USER` session (`fsUseSession(HomeFSUHandleAddr)`, thread-local override). The kernel serialises requests on a session and Home Menu is suspended while a game runs, so I expect no conflict, but this is the first time NTR uses that session continuously rather than only at boot.

### 5. `nwmPause` timeout leaves a stale ready signal (low, confirmed)

`thread_main.rs:210-231, 476-489`

`RESTART_READY_EVENT` is a one-shot event. After a 5 s timeout in `nwmPause`, the main loop later reaches `pause()`, signals READY with nobody waiting, and the event stays signalled. The next `nwmPause` (next sleep) then returns immediately without the Remote Play threads having paused. Harmless in practice because `set_reset_threads()` has been called and the threads stop within a frame, but it makes every later pause "early". Fix: `svcClearEvent(RESTART_READY_EVENT)` at the top of `nwmPause`, before `set_reset_threads()`.

### 6. The 10 s idle disconnect applies to every port 8000 client (low, confirmed)

`rt.c:157-186`

`rtRecvSocket` now returns -1 after 10 s with no bytes. NTR-VC sends a heartbeat every 250 ms, so it is never affected. Any other client that connects to port 8000 and sits idle for more than 10 s (an NTR debugger or memory-reading tool that only talks on demand) is now dropped and must reconnect. On the 5000+pid ports the loop only calls `rtRecvSocket` after `poll2` reports data, so nothing changes there. Behaviour change, not a bug; worth remembering if a tool starts "losing connection".

### 7. Items from the earlier review that are still open (low, confirmed)

* `vars.rs:281-325`: `init_syn_handles` still returns `None` at six points without closing the handles it already created. `d56355a` only added `cleanup_syn_handles` for the `init_seg_mem_handles` failure. With init retried every second, a persistent failure leaks a few handles per retry.
* `thread_audio.rs:25-26`, `thread_main.rs:410-421`: the audio thread still polls every 1.63 ms at priority 0x10 on core 1.
* No periodic NWM status line (frames, packets, QoS, last ack age), so the log will show when things stopped, not which counter stopped first.
* The menu's port 8000 ns thread (priority 0x10) now polls `recv` 20 times a second with 50 ms sleeps instead of blocking. Negligible load, but new.

## Things I checked that are fine

* The SD log tee is called only with `debugBufferLock` held (`nsDbgPutc`, `nsDbgCpy`, `nsDbgLn` are reached from `nsDbgPrint2` and `nsDbgPrintVerboseVABuf`, both under the lock; `nsLogFlush` takes the lock to swap buffers). The SD write itself happens outside the lock and only on the menu main thread, never on an NWM thread or the port thread.
* No lock is held across a sleep in the new code. `rp_udp_output` still releases the KCP lock before sleeping (unchanged). `JoinThread::drop` holds nothing.
* The watchdog, pause and restart log lines are printed from the NWM main thread via the usual IPC path; the only print under `NwmCbLock` is the once-per-session "kcp session_established" line.
* `nsLogFormatTime` is a correct civil-from-days conversion; the `up N s` stamp cannot overflow.
* The VBlank timeout cannot miss a reset: `do_screen` checks `reset_threads()` at the top of each loop, and on a timeout the capture simply runs on the current framebuffer.
* The TCP side of `nsPoll2` returns 1 on `POLLERR/POLLNVAL` so `accept` or `recv` sees the error and the socket is closed; `POLLHUP` on the client socket leads to a zero-byte `recv` and a close, as before.
* `rtAcquireLock` is a `LightLock`, so a print from a high-priority thread cannot spin behind the main thread's buffer swap.

## Suggested order

1. Reword the claim: the restart hang is detected, not fixed (finding 1). Keep the build as the diagnostic build it is.
2. Add the "unhandled revents" sleep in `nsPoll2` (finding 2) and rate-limit repeated error lines and the urgent flush (finding 3). Both are a few lines.
3. Move the SD flush to its own low-priority thread (finding 4).
4. `svcClearEvent` in `nwmPause` (finding 5), then the carried-over items (finding 7).
