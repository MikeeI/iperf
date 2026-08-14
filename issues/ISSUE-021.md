# ISSUE-021 — Server lifecycle: stale stream descriptor after TEST_END

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

[S] `iperf_handle_message_server()` handles normal `TEST_END` in `src/iperf_server_api.c:268-286`: it clears the fd sets and calls `close(sp->socket)` for every stream at lines 272-276, but leaves each `sp->socket` unchanged. After result exchange and `iperf_set_send_state(test, DISPLAY_RESULTS)`, it invokes `test->on_test_finish(test)` at lines 278-285.

[S] `iperf_set_send_state()` changes the local state at `src/iperf_api.c:2064-2074`. When the client subsequently sends `IPERF_DONE`, `iperf_run_server()` exits its loop and calls `cleanup_server()` at `src/iperf_server_api.c:992-1001`; that cleanup treats every nonnegative stale `sp->socket` as open and calls `close()` again before assigning `-1` at `src/iperf_server_api.c:506-514`.

[A] A finish callback that opens a descriptor after the first close can receive the same numeric descriptor; the later cleanup then closes the callback-owned descriptor. This is sequential descriptor reuse, not a data race or VM scheduling assumption.

[S] `CLIENT_TERMINATE` also leaves the closed numeric value in `sp->socket` (`src/iperf_server_api.c:289-307`), but it invokes no finish callback or other source-proven descriptor-adopting handoff before `cleanup_server()`. It is excluded: without a separate reuse event, its later second close is only no-reuse `EBADF` noise, not this callback-reuse root.

## Reach and impact

[S] `iperf_set_on_test_finish_callback()` is public at `src/iperf_api.h:228` and stores arbitrary callback code at `src/iperf_api.c:627-631`. Embedded server users that register it are affected during ordinary successful completion.

[S] The default `iperf_on_test_finish()` at `src/iperf_api.c:1045-1048` is empty, so a stock CLI completion does not itself demonstrate descriptor loss. The finding does not claim observed CLI harm.

[N] Frequency and user-visible impact are unmeasured. A reused fd can be a file, socket, pipe, or other callback resource; the source does not establish which resource or workload will hit it.

## Evidence

[S] `server_timer_proc()` and `cleanup_server()` both close then assign `sp->socket = -1` (`src/iperf_server_api.c:337-349`, `506-514`). The normal `TEST_END` loop lacks that invalidation before it reaches public callback code.

[S] The client handles `DISPLAY_RESULTS` by invoking its finish callback and then `iperf_client_end()` at `src/iperf_client_api.c:369-373`; `iperf_client_end()` sends `IPERF_DONE` at `src/iperf_client_api.c:557-591`. This supplies the normal path from the server callback to final server cleanup.

[O] Historical repository baseline: `make check` returned `5/5 pass`; it does not allocate a callback descriptor between the two server close sites and is not evidence for this reuse sequence.

## API and compatibility

[S] The callback setter is public API; callback order is observable. The correction must retain the current normal-completion order—data-stream close, result exchange, `DISPLAY_RESULTS`, then `on_test_finish`—and invalidate each closed stream descriptor before callback code runs.

[A] No wire format, CLI option, result payload, or mixed-version behavior needs to change. The intended library behavior change is only that final cleanup no longer owns an fd closed and subsequently reused by callback code.

## Prior art

[O] GitHub searches on 2026-08-14: upstream issues for `"on_test_finish"` returned #1838 and #8; `server close callback file descriptor` returned none; upstream pull requests for `"on_test_finish"` returned #1508 and closed #1339; `"on_test_finish" socket` returned none.

[S] Merged https://github.com/esnet/iperf/pull/1508 introduced the public callback setters, including `on_test_finish`, for library embedding. It is provenance for the exposed callback boundary, not a fix for server stream-descriptor ownership.

[A] https://github.com/esnet/iperf/issues/8 requests callbacks for state changes; its title does not establish this close/reuse root. No inspected issue or pull request owns the normal-`TEST_END` stale-descriptor path.

[N] Recheck current upstream discussion and pull-request diffs immediately before any external target decision; search results alone do not prove absence of a newer direct fix.

## Direction

In the normal `TEST_END` loop only, assign `sp->socket = -1` immediately after `close(sp->socket)` and before result exchange or `on_test_finish`. Preserve existing close, reporting, state, and callback order. Rejected: patch `CLIENT_TERMINATE` merely because it later double-closes a stale value; no callback/handoff creates a source-proven reuse window there, so that no-reuse `EBADF` path is outside this record. Rejected: move `on_test_finish`, delay the close, or add a generic close wrapper; each changes lifecycle semantics or expands scope instead of repairing this owner.

## Bounds

[S] Keep the established `-1` sentinel and `cleanup_server()`'s existing nonnegative guard. Do not move `on_test_finish`, suppress it, or delay stream close; those change public lifecycle semantics instead of fixing ownership.

[S] Preserve normal result exchange and `IPERF_DONE` handling. Exclude `CLIENT_TERMINATE`, `src/iperf_client_api.c`, and unrelated descriptor/error-path cleanup: none supplies this record's source-proven callback reuse boundary.

[A] Do not infer the callback-reuse outcome from VM tests or callbacks that allocate no descriptor; the decisive case is numeric fd reuse between the first close and `cleanup_server()`.

## Verification

Test decision: none; this ledger-only finding changes no source or tests.

[N] Before implementation or publication, run a focused libiperf server completion scenario that registers `on_test_finish`, records a descriptor opened by that callback after normal `TEST_END`, then verifies with `fcntl(fd, F_GETFD)` after server cleanup that the callback descriptor remains open.

[N] Repeat the callback-resource scenario on Ubuntu Linux, FreeBSD, and macOS where libiperf server embedding is supported; do not substitute network-throughput or VM-only runs for descriptor-lifecycle evidence.

## Missing

[N] No reproduction has yet shown callback descriptor reuse or the final erroneous close on `upstream/master@c9b74229d0d9bfec6d2307b66b43c29a7665ad0b`.

[N] No direct upstream target is established; current prior-art coverage requires a pre-publication recheck.

## Resume

Index: Invalidate terminal stream descriptors

Next: Reproduce normal server completion with a registered finish callback that opens and retains a descriptor.

Done when: The callback descriptor is shown reused from a closed stream and then closed by uncorrected final cleanup, or the current source disproves that sequence.
