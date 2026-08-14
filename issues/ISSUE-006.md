# ISSUE-006 — transfer workers: failed I/O exits without main-loop propagation

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

Root [S]: A client or server stream worker returns `NULL` immediately after `iperf_send_mt` or `iperf_recv_mt` fails. No per-worker terminal outcome reaches the role main loop, so it does not transition the test to error cleanup solely because the worker died.

## Reach and impact

Reach [S]: Both roles create one worker for every established stream at `TEST_RUNNING`; the worker loop covers sender and receiver streams. Therefore TCP, UDP, and SCTP runs can reach this seam once streams exist; reverse changes which role sends, bidirectional creates both directions, and parallel mode multiplies workers. The control path is separate from the failed data worker.

Impact [S]: A hard stream read/write failure maps to `IESTREAMREAD` or `IESTREAMWRITE`, but the failed worker exits without telling the main loop to stop. The main loop can continue timing/reporting until an independent control, timeout, or lifecycle event intervenes.

Impact [N]: Reproduction, affected duration, displayed zero-throughput intervals, exit status, and behavior by transport/mode are not measured on the recorded revision. Do not treat the source proof as an observed hang.

## Evidence

- [O] Currentness: `git diff --quiet upstream/master -- src/iperf_client_api.c src/iperf_server_api.c src/iperf_api.c` returned `0` on 2026-08-14; these local source anchors equal `upstream/master@c9b74229d0d9bfec6d2307b66b43c29a7665ad0b`.
- [S] `src/iperf_client_api.c:55-97` — `iperf_client_worker_run` routes `iperf_send_mt`/`iperf_recv_mt` failure to `cleanup_and_fail`, whose only action is `return NULL`.
- [S] `src/iperf_server_api.c:68-110` — `iperf_server_worker_run` has the same silent terminal path.
- [S] `src/iperf_api.c:2191-2259` — a hard `sp->snd` failure sets `i_errno = IESTREAMWRITE` and returns the negative result; a soft error only breaks the burst.
- [S] `src/iperf_api.c:2261-2281` — a negative `sp->rcv` result sets `i_errno = IESTREAMREAD` and returns it.
- [S] `src/iperf_client_api.c:723-728,738-761` — the client main loop reacts to control messages and creates workers, but no worker completion/error state is checked after creation.
- [S] `src/iperf_server_api.c:927-977` — the server likewise creates every stream worker after entering `TEST_RUNNING`, with no worker outcome propagated to the event loop.
- [S] `src/main.c:171-174,201-202` and `src/iperf_api.h:372,379` — CLI and libiperf callers rely on `iperf_run_server`/`iperf_run_client` negative return plus `i_errno` for terminal failure reporting.
- [O] https://github.com/esnet/iperf/issues/1986 was open when read on 2026-08-14. It records the sudden-client-disconnect server hang; its reporter-visible symptom alone does not establish this record's worker-to-main-loop root.
- [O] https://github.com/esnet/iperf/pull/1654 was open when read on 2026-08-14. It proposes a shared running-worker counter decremented on worker error and checked by each main loop.
- [O] https://github.com/esnet/iperf/pull/1861 was open when read on 2026-08-14. It adds a diagnostic for worker failure, not the missing terminal propagation.

## Prior art

Coverage [O]: Direct-read https://github.com/esnet/iperf/issues/1986, https://github.com/esnet/iperf/pull/1654, and https://github.com/esnet/iperf/pull/1861 on 2026-08-14; source anchors were checked at the recorded canonical revision.

Gaps [N]: GitHub Discussions, `iperf-dev` archives, releases, commits outside the cited PRs, and unpublished branches remain unchecked. No fresh local reproduction yet establishes whether new evidence would materially advance the existing thread.

- https://github.com/esnet/iperf/issues/1986 — Existing symptom thread; #1654 targets the missing worker-to-main-loop propagation. Do not open a duplicate thread.
- https://github.com/esnet/iperf/pull/1654 — Active correction: it owns the direct propagation direction; opening a parallel issue or pull request would duplicate active work.
- https://github.com/esnet/iperf/pull/1861 — Related diagnostics: visibility is useful, but printing an error alone does not stop the test.

Target fit: #1654 is the canonical active correction; do not create a new issue or pull request. Comment on #1986 only if a current-master reproduction adds a precise failure path or outcome absent from #1654. Mode and Target remain user-unselected.

## Direction

Track the active #1654 correction rather than design a competing path: it detects a worker that terminates on error from the owning role loop. A reproduction must capture the terminal `i_errno`; counter-based detection alone does not prove that the originating `IESTREAMREAD` or `IESTREAMWRITE` remains the caller-visible diagnostic. Do not substitute diagnostics alone, poll `pthread_kill`, suppress the error, or merge this work with teardown ordering in ISSUE-007.

## Bounds

- Preserve: current control-state protocol, `i_errno`/`iperf_strerror()` diagnostics, TCP/UDP/SCTP negotiation, reverse, bidirectional, parallel, JSON, JSON-stream, timing, and normal worker completion.
- Preserve: error cleanup cancels and joins only created workers; a worker failure must not cause duplicate reports, duplicate `SERVER_ERROR`, or a second cleanup.
- Exclude: a global worker-error redesign, new timeout policy, transport-specific retries, and PR #1861-style diagnostics as a replacement for propagation.
- Preserve: supported Ubuntu Linux, FreeBSD, and macOS worker/cancellation behavior and successful mixed-version control operation; no wire or public API change is implied.
- Cost [N]: Counter synchronization, ordering with worker creation/cancellation, and mixed failure races require focused validation; their portability and runtime cost are not measured.

## Verification

- [A] Inject one hard `sp->snd` and one hard `sp->rcv` failure after workers start; require both `iperf_run_client` and `iperf_run_server` to enter one terminal cleanup path and return the captured terminal error without further interval output.
- [A] Repeat the injection for TCP, UDP, and SCTP where configured; cover forward, reverse, bidirectional, and `-P 1`/`-P 2` to prove stream-count and direction boundaries.
- [A] Confirm normal completion, server reuse, control-channel failure, JSON, JSON streaming, and a worker-creation failure retain their existing state/error behavior.
- Test decision: none — this ledger-only assignment changed no source or tests; no gate was run.

## Missing

- [N] A minimal current-master reproduction of hard transfer I/O failure and the client/server result.
- [N] Verification that #1654's active diff handles the reproduced failure without a counter/cleanup race.
- [N] Prior-art coverage for Discussions, mailing list, releases, broader commit history, and unpublished work.
- [N] User selection of Mode and Target.

## Resume

Index: Reproduce worker exit
Next: Inject one hard send or receive failure after two current-master workers start, then capture both roles' state transition, final output, and return status.
Done when: The recorded run shows whether a dead worker independently causes terminal cleanup and identifies evidence that would add value to #1986/#1654.

## API and compatibility

Callers [S]: `iperf_run_client` and `iperf_run_server` are public libiperf entry points; `src/main.c:171-174,201-202` maps their negative return and `i_errno` to CLI failure.

Contract [S]: A terminal transfer I/O failure is classified as `IESTREAMWRITE` or `IESTREAMREAD`; callers must continue receiving the existing return/error contract rather than a silent successful run.

Compatibility: Preserve the control wire format and successful mixed-version behavior. A peer without worker-failure propagation may still follow its current protocol path; the local process must not wait indefinitely for a worker it knows has failed.

Migration: None.
