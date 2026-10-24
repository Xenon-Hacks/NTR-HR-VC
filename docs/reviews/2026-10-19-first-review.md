# NTR-HR (3DS side) review: tonight's Remote Play death

Reviewed: `D:\Claude\ntr-hr-full-width` at `1db8a6f` (the SD-log build, `BootNTR-HR-sdlog.cia`) and `d56355a` (Oct 8 fixes, matches the installed `BootNTR-HR-streamfix.cia`).
Compared against: NTR-VC 2.4.0 viewer source (`ntrviewer-hr-frame-share.zip` at tag 2.4.0) and tonight's viewer log `D:\Claude\NTR-VC-2.4.0\ntrviewer_log_20261018_114754.txt`.

The CIA was not converted to XCI. XCI is a Switch cartridge format, and this CIA is our own unencrypted build with its full source on this PC, so I reviewed the source directly. I did not change or rebuild anything.

## What tonight's viewer log shows

The viewer started at 23:47:54 (log time 00:00:00).

* **12:22:23** (00:10:17 real time): a 5.5 s gap in top-screen frames, then recovery.
* **12:22:54**: packets drop to a fifth of normal. The last UDP packet arrives at about **12:22:49 (00:10:43)**, and the DIAG STALL line fires at 12:22:59.
* **12:23 to 12:45**: zero packets for 23 minutes. There is no `kcp session_established` after the first one, so the 3DS never restarted the stream, even though the viewer keeps nudging the handshake every 250 ms.
* **No TCP line at all**: no `tcp recv error`, no `heart beat send failed`, no `disconnected`, no reconnect attempt and no ping diagnosis. The port 8000 control link was never declared dead (see V1 and V2).

Two dips before the death point to the radio link getting worse, or NWM getting starved, before it died. The 3DS-side stream then never came back. Every likely cause below fits that pattern: one bad event, and recovery code that hangs or spins instead of retrying.

The installed CIA's `ntr.hr.nwm.bin` contains the Oct 8 strings ("Nwm init failed, retrying" and the others), so tonight's failure happened with those fixes in place. **The new SD-log build fixes none of the items below.** It only adds logging.

---

## Findings, most likely cause of tonight's failure first

### 1. The NWM control thread busy-spins at top priority when `poll` reports an error (high)

`source/nwm/main_nwm.c:258-275`, `source/ns.c:419-465`

`nsPoll2` polls the TCP socket and the KCP UDP socket (port 8000) with an infinite timeout. It only acts on `POLLIN` (UDP) and `POLLIN|POLLHUP` (TCP). If poll returns with only `POLLERR` or `POLLNVAL` set, which is what an error state such as a dropped Wi-Fi link or a reset SOC looks like, `nsPoll2` returns 0. `WHILE_POLL2` treats 0 as "loop again" and calls poll again at once. The error flag is still set, so the loop never sleeps.

That thread runs **inside the NWM (Wi-Fi) sysmodule** at priority **0x10**, the highest user priority (`ns.c:516`, core -2, which is NWM's own core). A spin there starves NWM's own Wi-Fi threads and everything else on the system core. That fits all three symptoms: the 3DS drops off Wi-Fi, the game keeps running on its own core, and the X+Y menu (which needs HID, GSP and FS on the system core) freezes.

This is a confirmed code defect. Whether it fired tonight is inferred, because nothing was logged on the 3DS.

Fix: in `nsPoll2`, treat `POLLERR|POLLHUP|POLLNVAL` on the UDP socket as `-1` (close and recreate it), and treat them as `1` on the TCP socket (let `recv` fail and close it). Give `poll2` a finite timeout (for example 1000 ms). Make `WHILE_POLL2` sleep about 10 ms whenever it gets 0 several times in a row.

### 2. A stream restart can hang forever, and can take NWM down with it (high)

`thread_main.rs:477-485`, `utils.rs:209`, `thread_screen/safe_impl.rs:123`, `thread_screen/vars.rs:526-535`, `main_nwm.c:118-127`

The 3DS restarts the stream when a KCP timeout (2 s with no ack) sets `reset_threads`. The restart only finishes when every Remote Play thread exits, because `JoinThread::drop` waits with `-1`. Two of those threads can block with no timeout and no reset check:

* **Screen thread**: `wait_for_vblank` → `gspWaitForEvent` → `LightEvent_Wait` with no timeout. If NWM's GSP event relay misses VBlanks, the screen thread never sees the reset.
* **nwm/KCP thread**: every send goes through NWM's internal `nwmSendPacket`, called while holding `RELIABLE_STREAM_CB_LOCK`. If NWM's TX path blocks while the link is down, this thread never returns.

If either happens, the main loop never reaches `pause()`, so the stream never restarts. Tonight's log matches that: 23 minutes of handshake nudges and no new session.

Worse, `nwmPause` runs on **NWM's own notification thread**, which NTR hooks for sleep, wake and shutdown, and it waits for the restart with `-1`. A sleep-allowed or shutdown notification during a hung restart therefore wedges NWM itself for good.

Fix: give `wait_for_vblank` a timeout (`LightEvent_WaitTimeout`, 100 ms) and check `reset_threads()` in the loop. In `nwmPause`, wait a bounded time (for example 2 s), then log and carry on. If the restart has not finished after a few seconds, log which thread has not exited, so the next failure names the thread.

### 3. The port 8000 control socket is blocking, so the Oct 8 timeouts never apply to it (high, for "never recovers")

`source/ns.c:443-446`, `source/rt.c:156-212`

The menu process clears `O_NONBLOCK` on the accepted port 8000 socket. The Oct 8 5 s limits in `rtRecvSocket` and `rtSendSocket` only count `EWOULDBLOCK` and `EAGAIN`, and a blocking socket never returns those. If Wi-Fi drops without a FIN or RST, the menu's ns thread blocks in `recv` forever. That thread serves one client at a time and only calls `accept` again after the current client closes, so the viewer can never reconnect on port 8000 until the 3DS reboots. This bites as soon as Wi-Fi comes back.

Fix: keep the socket non-blocking (the 5 s logic then works), or set `SO_RCVTIMEO` and `SO_SNDTIMEO`. Also add an idle rule: no heartbeat for 5 s means close the socket and go back to `accept`.

### 4. Network errors in NWM pop a blocking message box in the menu (medium, a likely cause of the X+Y freeze)

`include/ui.h:14-18`, `source/ui.c:37-52`, `source/menu/ui_menu.c:315-335`, `source/menu/main_menu.c:726-737`

`showDbg` sends a synchronous IPC request to the menu, and the menu's port thread then shows a message box. That box calls `acquireVideo()`, which pauses the game if a game plugin PID is known, takes over the bottom screen, and waits for **B**. The NWM thread that sent the request is blocked the whole time.

Network failure paths use `showDbg`: `bind failed`, `listen failed`, `nwm socket failed`, `nwm_recv_sock bind failed` and `socket poll failed`. With the Oct 8 0.5 s retry, a persistent Wi-Fi failure can produce one message box after another. The menu's port thread is single-threaded, so while a box is up, every other debug print from NWM blocks behind it, including prints from the real-time KCP and nwm threads. Pressing X+Y while one of these boxes is up gives two threads drawing over each other and both reading keys, which looks like a freeze.

Fix: in the NWM payload, log network errors with `nsDbgPrint` only, never `showDbg` or `showMsg`. Keep message boxes for user-initiated actions in the menu process.

### 5. The audio capture thread wakes 613 times a second at top priority on the system core (medium, risk factor)

`thread_main.rs:412-425`, `thread_audio.rs:25-26`

This is NTR-HR+ code, not upstream. The thread runs on **core 1 (the system core)** at priority 0x10 and sleeps `AUDIO_POLL_NS` ≈ 1.63 ms (a third of a DSP frame) between polls, for the whole stream. Each wake is short, but it preempts NWM's own Wi-Fi threads more than 600 times a second for 12 hours. It does not fail on its own, but it adds latency to the Wi-Fi driver and makes finding 1 and finding 2 more likely.

Fix: poll at half a frame (about 2.4 ms). With two mix regions that still catches every frame. Also lower the priority below NWM's driver threads (for example 0x20), or move the thread to core 2 or 3, where the other Remote Play threads already run.

### 6. Tick arithmetic at the 32-bit wrap (low)

`thread_nwm/vars.rs:414-434, 600-612`, `ikcp.c:510`

The system tick truncated to u32 wraps every 16 s, and the signed differences are right only within ±8 s. If `RP_OUTPUT_NEXT_TICK` goes stale for more than 8 s (for example the thread was blocked), the difference flips sign. The 2 s KCP timeout is then missed and `rp_udp_output` can sleep for up to 8 s. It heals within 16 s, so it is not tonight's cause.

Separately, `congc.last_dur` is wrong when more than 16 s pass between congestion updates, so the QoS estimate is off after a stall. It is clamped to a quarter of max QoS, so it is not fatal.

Other counters (frame id u8, 12-bit packet and FEC ids, the 1-bit KCP conversation id, the DSP frame counter) are all compared wrap-safely.

### 7. Handle leaks on partial init failure (low)

`vars.rs:280-355`

`init_syn_handles` creates more than 10 handles and returns `None` partway through without closing what it already made. Since Oct 8, init is retried every second, so a persistent failure (handle-table pressure) leaks a few handles per retry until NWM runs out. I found no per-frame or per-restart leak in the normal path: DMA handles, the game-process handle, the overlay handle and thread handles are all closed.

### 8. Debug-line timestamps overflow after 68.7 s of uptime (logging quality)

`source/ns.c:117`, `source/ui.c:68`

`ticks * 1000000000` overflows u64 once uptime passes about 68.7 s, so every per-line timestamp in the debug title is garbage after the first minute. That includes the lines the new SD log records. Fix: compute milliseconds as `ticks / (SYSCLOCK_ARM11 / 1000)`, the way `nsLogFormatTime` already does.

---

## Viewer-side mismatches (NTR-VC 2.4.0)

**V1. Missing heartbeat replies go unnoticed.** `ntr_hb.c` `tcp_thread_func` sends a heartbeat every 250 ms and reads replies with a non-blocking `tcp_recv` that returns 0 when nothing came. Nothing tracks the last reply, so a 3DS that stops answering on port 8000 is never flagged. That is why tonight's log has no disconnect, no auto-reconnect and no ping diagnosis. Suggested fix: if no reply comes for 3 s, log "3DS stopped answering heartbeats", then `RESET_SOCKET()`, which also starts the reconnect and ping logic.

**V2. `socket_poll` waits for the wrong event and can spin.** In `ntr_common.h:54-71`, `tcp_send` calls `socket_poll` after `WSAEWOULDBLOCK`, but `socket_poll` waits for `POLLIN`. A full send buffer needs `POLLOUT`. If WSAPoll returns only `POLLERR` or `POLLHUP`, the loop spins without returning an error until the program exits or `kcp_restart` is set. Suggested fix: pass the events to wait for, and return false on `POLLERR|POLLHUP|POLLNVAL`.

**Protocol check.** The 901 packet layout matches what `nsHandleRemotePlay` reads: args[0..2] are mode, quality and QoS, args[3] is the guarding magic, args[4] is port plus flags, args[5] is audio (2 means in-band KCP audio) and args[6] is full width. The flag bits (bits 30 and 31 for KCP and delta, bit 29 and bits 27-28 for lossless and color) match `RP_CONFIG_FLAG_*` in `constants.h`. The heartbeat header is 84 bytes on both sides. I found no protocol mismatch.

The viewer only re-sends 901 when parameters change, at most once a second. Quality-only and QoS-only changes are applied live by `set_config`. Any other change, such as screen priority, restarts the whole stream on the 3DS.

---

## The new SD-card logging (`1db8a6f`)

It works well overall. The double buffer under `debugBufferLock`, the flush from the menu main loop every 2 s, the minute stamps and the 1 MB rotation all look correct. Notes:

1. **NWM lines only reach the SD card when debug mode is off.** That is the normal case (D-Left not held at boot). In debug mode, NWM prints stay in NWM's own RAM buffer and never reach `nsLogWrite`.
2. **NWM lines travel by synchronous IPC through the menu's port thread.** If that thread is busy with a message box (finding 4), NWM's real-time threads block on their next print. Fixing finding 4 fixes this too.
3. **Up to 2 s of lines are lost at a hard freeze**, and nothing is written if the system core is starved (finding 1), because the menu main loop does the writing. A cheap improvement: when a line contains "failed", "timeout" or "restart", ask for an early flush.
4. **`nsLogFlush(1)` runs right before `showMainMenu()`**, so an SD write is now on the X+Y path. If FS is wedged, X+Y now hangs there too. It is harmless in normal use, but consider dropping it.
5. **The per-line timestamps are broken after 68.7 s** (finding 8). The minute stamps cover for it, but the fix is one line.
6. **Several of the most useful lines only print in debug mode.** "Nwm main loop restarted" (`thread_main.rs:443`) and "kcp session_established" (`ikcp.c:597`) are both gated on `nsUseDbg`. Log them unconditionally (they are rare).

### Lines worth adding so the next failure explains itself

* **One status line from NWM every 60 s**: frames sent, UDP packets sent, current and max QoS, `n_snd`, the age of the last KCP ack in ms, restarts so far and the last restart reason. A failure then shows which counter stopped first.
* **Every stream restart, with its reason** ("KCP timeout", "KCP send next -0x..", "params changed", "nwm_syn wait failed"), plus a line when the restart finishes. A restart without a finish line names finding 2.
* **A watchdog line** when a Remote Play thread has not exited 3 s after a reset, naming the thread (screen, nwm, aux or audio).
* **Wi-Fi state changes** seen by NWM (link up or down), and every error branch in `nsPoll2` with its `revents` value. This would confirm or rule out finding 1 directly.
* **The SD-log timestamp of the last heartbeat from the PC**, so the 3DS log and the viewer log can be lined up.

---

## Optimization notes (no bugs)

* NWM's debug prints run synchronous IPC from real-time threads. A small shared-memory ring that the menu drains would remove those blocking points entirely.
* A finite `poll2` timeout also lets the ns thread do periodic housekeeping, such as the 60 s status line, without needing another thread.
* The per-packet header copy and IP checksum are cheap. I found no wasted encoder work worth changing.

## Suggested order of work

Each item is a separate small change in `ntr-hr-full-width`. The "NTR-VC 2.4.0 crash" thread owns that tree.

1. Finding 1 (`nsPoll2` error flags and a finite timeout) and finding 4 (no `showDbg` in NWM network paths).
2. Finding 2 (VBlank wait timeout and a bounded `nwmPause`), plus the restart-reason and watchdog log lines.
3. Finding 3 (non-blocking port 8000 socket with an idle timeout).
4. Finding 5 (audio thread priority, core and poll rate) and finding 8 (timestamp fix).
5. Viewer V1 and V2 in NTR-VC.
