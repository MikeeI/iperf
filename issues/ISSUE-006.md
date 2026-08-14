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

Impact [O]: With two TCP streams, a deterministic `EIO` from the first sender worker's 32nd data-socket `write()` caused that stream to report zero throughput for the remaining three one-second intervals while the sibling stream and both role main loops completed the configured four seconds. Client-worker and server-worker injections both produced normal summaries, `iperf Done`, and exit status 0; neither role reported the transfer error.

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
- [O] Client-worker injection: `LD_PRELOAD=/tmp/iperf-worker-fail.so IPERF_INJECT_ROLE=connect IPERF_INJECT_AFTER=32 src/iperf3 -c 127.0.0.1 -p 55221 -t 4 -i 1 -P 2` marked the first data socket, injected `EIO`, printed three subsequent `0.00 bits/sec` intervals for that stream, printed `iperf Done`, and exited 0 after 4.06 seconds.
- [O] Server-worker injection: the server used the same preload with `IPERF_INJECT_ROLE=accept`; `src/iperf3 -c 127.0.0.1 -p 55223 -R -t 4 -i 1 -P 2` observed the failed server-sender stream at `0.00 bits/sec` for the remaining intervals. Client and one-off server both exited 0 after normal result exchange.
- [O] Injector boundary: it identifies the second connected or accepted socket as the first data stream, lets 31 writes succeed, then returns `-1` with `errno=EIO` for that socket only; the control socket and second data stream remain live.

## Prior art

Coverage [O]: Direct-read https://github.com/esnet/iperf/issues/1986, https://github.com/esnet/iperf/pull/1654, and https://github.com/esnet/iperf/pull/1861 on 2026-08-14; source anchors were checked at the recorded canonical revision.

Gaps [N]: GitHub Discussions, `iperf-dev` archives, releases, commits outside the cited PRs, and unpublished branches remain unchecked. PR #1654 has not been rebased or run against the deterministic injector.

- https://github.com/esnet/iperf/issues/1986 — Existing symptom thread; #1654 targets the missing worker-to-main-loop propagation. Do not open a duplicate thread.
- https://github.com/esnet/iperf/pull/1654 — Active correction: it owns the direct propagation direction; opening a parallel issue or pull request would duplicate active work.
- https://github.com/esnet/iperf/pull/1861 — Related diagnostics: visibility is useful, but printing an error alone does not stop the test.

Target fit: A comment on #1654 is the smallest useful target after its current dirty diff is rebased and tested with this injector. The reproduction proves silent partial success on both roles, but does not yet prove #1654 preserves the originating stream error or avoids cleanup races. Mode and Target remain user-unselected.

## Direction

Track the active #1654 correction rather than design a competing path: it detects a worker that terminates on error from the owning role loop. A reproduction must capture the terminal `i_errno`; counter-based detection alone does not prove that the originating `IESTREAMREAD` or `IESTREAMWRITE` remains the caller-visible diagnostic. Do not substitute diagnostics alone, poll `pthread_kill`, suppress the error, or merge this work with teardown ordering in ISSUE-007.

## Bounds

- Preserve: current control-state protocol, `i_errno`/`iperf_strerror()` diagnostics, TCP/UDP/SCTP negotiation, reverse, bidirectional, parallel, JSON, JSON-stream, timing, and normal worker completion.
- Preserve: error cleanup cancels and joins only created workers; a worker failure must not cause duplicate reports, duplicate `SERVER_ERROR`, or a second cleanup.
- Exclude: a global worker-error redesign, new timeout policy, transport-specific retries, and PR #1861-style diagnostics as a replacement for propagation.
- Preserve: supported Ubuntu Linux, FreeBSD, and macOS worker/cancellation behavior and successful mixed-version control operation; no wire or public API change is implied.
- Cost [N]: Counter synchronization, ordering with worker creation/cancellation, and mixed failure races require focused validation; their portability and runtime cost are not measured.

## Verification

- [O] One client-sender and one server-sender hard failure were injected after two TCP workers started; both main loops continued through three zero-throughput intervals for the failed stream and returned success.
- [O] The unaffected stream, control exchange, result exchange, summaries, and normal four-second duration remained live, isolating worker-to-main-loop propagation from peer or control failure.
- [A] Apply the current #1654 direction and require each reproduced run to enter one terminal cleanup path, preserve the originating `IESTREAMWRITE`, emit no post-failure intervals or successful summary, and return nonzero.
- [A] Extend only after the TCP candidate passes: inject receive failure; cover UDP/SCTP where configured, forward/reverse/bidirectional, `-P 1`/`-P 2`, JSON modes, normal completion, server reuse, control failure, and worker-creation failure.
- Test decision: none — reproduction used an external preload injector and changed no repository source or tests.

## Missing

- [N] Verification that #1654's current dirty diff handles the reproduced client/server failures while preserving `IESTREAMWRITE` and exactly-once cleanup.
- [N] Receive-worker, UDP, SCTP, bidirectional, JSON, platform, and normal-path candidate coverage.
- [N] Prior-art coverage for Discussions, mailing list, releases, broader commit history, and unpublished work.
- [N] User selection of Mode and Target.

## Resume

Index: Verify PR 1654 propagation
Next: Rebase or reconstruct #1654's worker-propagation change on current master and run both deterministic TCP injections while tracing terminal error and cleanup.
Done when: Both roles terminate promptly with the originating stream error and one cleanup path, or a precise counter/error-preservation defect is recorded for #1654.

## Bug reproduction

Environment: iperf 3.21+ at `upstream/master@c9b74229d0d9bfec6d2307b66b43c29a7665ad0b`; Ubuntu 24.04.4 LTS; Linux 6.8.0-107-generic; x86_64; glibc; localhost TCP.
Injection: An `LD_PRELOAD` interposer marks the first data socket after the control connection and returns `-1/EIO` from its 32nd and later `write()` calls without closing that data socket or disturbing the control socket or sibling stream.
Client role: Normal server plus injected `-t 4 -i 1 -P 2` forward client.
Server role: Injected one-off server plus normal `-R -t 4 -i 1 -P 2` client.
Actual [O]: In each role, one worker silently stopped, its stream reported zero for intervals 1–4, the sibling stream continued, result exchange completed, and both processes returned success.
Expected: The owning role must detect the worker's hard transfer failure, stop the test promptly, preserve `IESTREAMWRITE`, run cleanup once, and return failure rather than a valid-looking partial result.

## API and compatibility

Callers [S]: `iperf_run_client` and `iperf_run_server` are public libiperf entry points; `src/main.c:171-174,201-202` maps their negative return and `i_errno` to CLI failure.

Contract [S]: A terminal transfer I/O failure is classified as `IESTREAMWRITE` or `IESTREAMREAD`; callers must continue receiving the existing return/error contract rather than a silent successful run.

Compatibility: Preserve the control wire format and successful mixed-version behavior. A peer without worker-failure propagation may still follow its current protocol path; the local process must not wait indefinitely for a worker it knows has failed.

Migration: None.
