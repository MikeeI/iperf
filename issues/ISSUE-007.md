# ISSUE-007 — server duration timer: frees live stream before join

State: Hold
Mode: Undecided
Target: Undecided
Location: Not published.
Priority: High
Confidence: High
Type: reliability
Created: 2026-08-14
Updated: 2026-08-14
Source: `upstream/master@c9b74229d0d9bfec6d2307b66b43c29a7665ad0b`

## Root

Root [S]: `server_timer_proc` closes and frees each stream without first joining its created worker. The worker retains `sp` and dereferences it in its loop, so the timer path supplies no join-before-free lifetime relation. A worker dereference after the free requires a scheduling interleaving [A]; it is not an observed crash.

## Reach and impact

Reach [S]: This is server-only. It requires a duration-enabled test, `test->done == 0` when the server duration timer fires, and a created stream worker. It can involve TCP, UDP, or SCTP where configured; direction and parallelism select or multiply workers but do not change the timer owner.

Impact [S]: The timer closes `sp->socket` and calls `iperf_free_stream(sp)` without setting `sp->done`, canceling, or joining `sp->thr`. The existing `cleanup_server` path demonstrates the required join-before-close ordering.

Impact [N]: No current-master crash, sanitizer result, stale-FD failure, data corruption, or frequency is observed. The actual use-after-free requires the stated scheduling interleaving [A].

## Evidence

- [O] Currentness: `git diff --quiet upstream/master -- src/iperf_client_api.c src/iperf_server_api.c src/iperf_api.c` returned `0` on 2026-08-14; these local source anchors equal `upstream/master@c9b74229d0d9bfec6d2307b66b43c29a7665ad0b`.
- [S] `src/iperf_server_api.c:317-349` — `server_timer_proc` sets `test->done`, sends `SERVER_ERROR`, closes each stream socket, removes the stream from the list, and calls `iperf_free_stream(sp)`; it does not set `sp->done`, cancel, or join `sp->thr`.
- [S] `src/iperf_server_api.c:374-395` — a duration test creates that timer for `test->duration + test->omit + 40` seconds, so it can fire while the test still has live worker objects.
- [S] `src/iperf_server_api.c:68-110,953-969` — each worker receives `sp`, repeatedly dereferences it, and is created for every stream; its function has no ownership transfer that would make timer free safe.
- [S] `src/iperf_server_api.c:459-521,995-1007` — the normal server cleanup joins workers before closing remaining stream/control FDs, but it runs only after the main loop exits; this is the correct ordering that the timer bypasses.
- [O] https://github.com/esnet/iperf/issues/1977 was open when read on 2026-08-14. Its reporter observed a UDP `--bidir -P 16` intermittent failure on 3.18; a maintainer described cancel-before-buffer-free and a join as the lifetime boundary, but did not prove this recorded timer path.
- [O] https://github.com/esnet/iperf/issues/1760 was closed when read on 2026-08-14. Its older multi-thread client recycle segfault is an excluded client stale-FD/repeated-cleanup topic, not this server timer's object-lifetime root.
- [O] https://github.com/esnet/iperf/issues/753 and https://github.com/esnet/iperf/issues/751 were closed when read on 2026-08-14. Their historical server-duration/control `EBADF` reports are excluded control-FD ownership topics, not proof that the timer frees a live worker object.
- [O] https://github.com/esnet/iperf/pull/859 was closed when read on 2026-08-14. Its merged control-channel timeout change is excluded from this worker-object lifetime root.

## Prior art

Coverage [O]: GitHub issue searches `"server_timer_proc"` (returned #753) and `"pthread_join"` (returned #1977 and #1760), plus a PR search `"pthread_join"` (no result), were run for `esnet/iperf` on 2026-08-14. Direct-read #1977, #1760, #753, #751, and #859; source anchors were checked at the recorded canonical revision.

Gaps [N]: GitHub Discussions, `iperf-dev` archives, releases, broader commit history, platform-specific reports, and unpublished branches remain unchecked. No current-master race-assisted reproduction links a sanitizer failure to the timer's join-before-free omission.

- https://github.com/esnet/iperf/issues/1977 — Same join-before-release invariant, but incomplete currentness: its maintainer comment proposes a join; its version-specific UDP report does not establish this server duration-timer root. It is the only plausible existing-thread target if fresh evidence adds a timer-specific current source/reproduction fact.
- https://github.com/esnet/iperf/issues/1760 — Excluded historical client recycle/stale-FD topic; it has a different stated owner.
- https://github.com/esnet/iperf/issues/753 and https://github.com/esnet/iperf/issues/751 — Excluded historical server control-FD timing faults; neither establishes worker-object reclamation.
- https://github.com/esnet/iperf/pull/859 — Excluded control-timeout correction, not worker lifetime management.

Target fit: #1977 may accept a useful comment only after a current-master reproduction or sanitizer/source demonstration establishes this timer-specific root. Mode and Target remain user-unselected.

## Direction

Keep stream destruction out of `server_timer_proc`. The timer must transfer terminal control to an existing server lifecycle path that marks/cancels and joins every created worker before closing or freeing its stream object. Preserve the timer's terminal notification and exactly-once cleanup. Do not change ordinary `TEST_END` FD ordering, client teardown, control-FD `EBADF` handling, worker-failure detection in ISSUE-006, cancellation policy, or add a broad thread abstraction.

## Bounds

- Preserve: the server duration timer's terminal notification, `TEST_END`, `EXCHANGE_RESULTS`, `DISPLAY_RESULTS`, and `IPERF_DONE` control behavior; final text/JSON reports; server reuse; and one close per data/control FD after worker lifetime ends.
- Preserve: TCP, UDP, SCTP where configured; forward, reverse, bidirectional, parallel, omission, byte/block termination, and server callback behavior.
- Preserve: supported Ubuntu Linux, FreeBSD, and macOS pthread semantics and successful mixed-version control operation; no wire or public API change is required.
- Exclude: ordinary `TEST_END` close ordering, all client teardown, test-duration policy, the 40-second server grace calculation, transport retry policy, and the distinct silent-worker-error propagation in ISSUE-006.
- Cost [N]: Moving timer cleanup onto the ordered lifecycle path changes wait timing and may expose a truly non-cancelable worker; supported-platform behavior and report-ordering effects are unmeasured.

## Verification

- [A] Hold a server data worker in `iperf_send_mt` or `iperf_recv_mt`, keep `test->done == 0` until the duration timer fires, and run under AddressSanitizer; require the worker to be joined before its `sp` is freed and one terminal server cleanup.
- [A] Trace the timer path's worker cancellation/join, stream close/free, and terminal control notification; require exactly-once cleanup without changing ordinary `TEST_END` or client ordering.
- [A] Exercise duration-enabled TCP, UDP, and SCTP where configured; forward, reverse, bidirectional; `-P 1` and `-P 2`; server reuse; JSON; and JSON streaming.
- Test decision: none — this ledger-only assignment changed no source or tests; no gate was run.

## Missing

- [N] A deterministic current-master timer reproduction with a live worker and sanitizer/trace result.
- [N] Supported-platform validation and the exact observable impact of the scheduling-dependent object-lifetime breach.
- [N] Prior-art coverage for Discussions, mailing list, releases, broader commits, and unpublished work.
- [N] User selection of Mode and Target.

## Resume

Index: Reproduce timer free race
Next: Hold one current-master worker across server-duration expiry under AddressSanitizer, then capture its join/free ordering and final server state.
Done when: A trace or sanitizer result establishes whether the timer frees `sp` before a live worker ends and whether #1977 gains material evidence.
