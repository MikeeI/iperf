# ISSUE-011 — server workers: pthread-attribute cleanup falls through

State: Hold
Mode: Undecided
Target: Undecided
Location: Not published.
Priority: Medium
Confidence: High
Type: reliability
Created: 2026-08-14
Updated: 2026-08-14
Source: `upstream/master@c9b74229d0d9bfec6d2307b66b43c29a7665ad0b`

## Root

[S] `iperf_run_server()` owns worker-attribute lifecycle, worker startup, and the terminal handoff to `cleanup_server()`. Both `pthread_attr_init()` and `pthread_attr_destroy()` error branches set `i_errno` and call `cleanup_server()`, but neither returns (`src/iperf_server_api.c:953-977`).

[S] The single root is missing terminal control flow after attribute-lifecycle cleanup. On init failure, execution next passes the attribute whose initialization failed to `pthread_create()` and can later reach `pthread_attr_destroy()`; it must instead stop before either operation.

[S] On destroy failure, cleanup has already run but execution next resets receive-progress bookkeeping and can re-enter the outer server loop. This is a separate reach consequence of the same missing-return root, not an assertion that init and destroy failures have the same libc behavior.

## Reach and impact

[S] Init reach requires `pthread_attr_init()` to return nonzero after the server enters `CREATE_STREAMS`. The following `SLIST_FOREACH` passes `&attr` to every `pthread_create()` (`src/iperf_server_api.c:953-970`).

[S] Destroy reach requires successful worker creation followed by nonzero `pthread_attr_destroy()`. Its branch calls cleanup, then falls through to the receive-progress update and the outer loop (`src/iperf_server_api.c:974-992`).

[S] `cleanup_server()` cancels or joins created workers, closes owned descriptors, and cancels timers, but does not set `IPERF_DONE` or return from its caller (`src/iperf_server_api.c:476-552`). It cannot make either error branch terminal.

[N] pthread attribute failures are not measured. Do not claim a deterministic crash, a specific pthread error, or a platform-independent invalid-object symptom without fault injection; the source proves fallthrough, not a particular libc response.

## Evidence

[S] Init branch: `pthread_attr_init(&attr) != 0` sets `IEPTHREADATTRINIT` and calls `cleanup_server(test)` at `src/iperf_server_api.c:953-958`; the following statement creates workers with `&attr` at lines 960-966.

[S] Destroy branch: `pthread_attr_destroy(&attr) != 0` sets `IEPTHREADATTRDESTROY` and calls cleanup at lines 974-977; the next statements update progress values at lines 979-981 instead of returning failure.

[S] The existing adjacent `pthread_create()` failure branch proves the intended terminal shape: it sets `IEPTHREADCREATE`, calls `cleanup_server()`, and immediately returns `-1` (`src/iperf_server_api.c:961-965`).

[S] `cleanup_server()` closes descriptors and cancels timers but does not reset the server state or terminate the caller (`src/iperf_server_api.c:476-552`), so it cannot substitute for the missing return.

[A] No pthread failure was injected in this audit; these are source-proven lifecycle defects, not observed runtime failures.

## Prior art

[S] Recorded prior-art coverage: https://github.com/esnet/iperf/issues/1986, active https://github.com/esnet/iperf/pull/1654, and related https://github.com/esnet/iperf/pull/1861 own worker-failure propagation.

[S] Classification: this root is server-owned pthread-attribute terminal control flow, not worker failure propagation. The recorded candidates do not demonstrate ownership of the missing returns.

[A] A current upstream search for `pthread_attr_init`/`pthread_attr_destroy` failure handling is not recorded here; that currentness/target gap keeps the finding on Hold.

## Direction

[S] Make both attribute-error branches terminal: retain their assigned `i_errno`, call `cleanup_server()` once, and immediately return `-1`.

[S] The init-failure branch must not create workers with or destroy an unsuccessfully initialized `attr`. The destroy-failure branch must not re-enter timer/select processing after cleanup.

[S] Follow the neighboring `pthread_create()` error branch's ownership pattern rather than introducing a new cleanup abstraction or changing worker success behavior.

## Bounds

[S] Preserve successful pthread attribute setup, worker creation, worker cancellation/join ordering, existing error values, control protocol, and normal server-loop behavior on Ubuntu Linux, FreeBSD, and macOS.

[S] Preserve public signatures and mixed-version client/server behavior: only existing local attribute-error branches become terminal.

## API and compatibility

Callers [S]: `iperf_run_server()` is public in `src/iperf_api.h:379`; the CLI calls it in the server loop at `src/main.c:171-185`.

Contract [S]: Any failed worker-attribute lifecycle operation is a terminal server-run failure after owned cleanup, consistent with the adjacent `pthread_create()` failure contract.

Compatibility: Preserve public signatures, `i_errno` families, successful wire behavior, and mixed-version client/server operation. Only the existing error branch becomes terminal.

Migration: None.

## Verification

Test decision: none. No tests or gates were run for this tracker-only update.

[A] Inject a nonzero `pthread_attr_init()` result after the server reaches `CREATE_STREAMS`; assert `iperf_run_server()` returns `-1`, no `pthread_create()` or `pthread_attr_destroy()` receives the invalid `attr`, and owned resources are cleaned.

[A] Inject a nonzero `pthread_attr_destroy()` result after at least one worker starts; assert workers are canceled/joined, `iperf_run_server()` returns `-1`, and no subsequent `select()`/server-loop iteration occurs after cleanup.

[A] Run the same worker-start path without injection; assert normal worker execution, JSON/non-JSON server output as applicable, and current successful completion remain unchanged.

## Missing

[A] Controlled init/destroy fault-injection results, a current upstream search/classification for this exact root, and a user-selected Mode, Target, and external draft are required before publication.

## Resume

Index: Inject pthread attribute failures

Next: Fault-inject `pthread_attr_init()` and `pthread_attr_destroy()` failures in the server worker-start path.

Done when: Both branches return `-1` immediately after one cleanup, with call-order evidence that invalid attributes and post-cleanup loop work are absent.
