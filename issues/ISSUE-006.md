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
- [O] PR #1654 reconstruction: its four commits at `d7ab0713d95ab163825a75d8b9772092ca49117d` replayed onto current master after one mechanical `iperf_error.c` conflict; the candidate built successfully.
- [O] Candidate client failure: the injected client exited 1 after about one second, but replaced `IESTREAMWRITE/EIO` with `IEPTHREADNOTRUNNING`, emitted two copies of the first interval, and sent `IPERF_DONE`; the server logged repeated late receives in `IPERF_DONE` before cleanup.
- [O] Candidate server failure: the reverse client exited 1 after `SERVER_ERROR`, but the server replaced `IESTREAMWRITE/EIO`, called cleanup twice, continued into the loop with closed descriptors, and ended with `select failed: Bad file descriptor`; the one-off server process still exited 0 under the existing CLI server contract.
- [O] Candidate thread-creation regression: faulting the second client `pthread_create()` caused exit 139. Current master under the same injector cleanly reported `unable to create thread` and exited 1.
- [O] Candidate normal reverse regression: four successful forward/reverse tests across `-P 2` and `-P 1` all returned 0, but each reverse server teardown logged false `Server Worker Thread ... Bad file descriptor` errors.
- [O] Candidate JSON behavior: full JSON exited 1 with generic `a thread stopped running unexpectedly`, two intervals, and an `end` object; streaming JSON emitted `start,interval,interval,error,end`, with only its diagnostic event retaining the originating `IESTREAMWRITE/EIO` text.

## Prior art

Coverage [O]: Direct-read https://github.com/esnet/iperf/issues/1986, https://github.com/esnet/iperf/pull/1654, and https://github.com/esnet/iperf/pull/1861 on 2026-08-14; source anchors were checked at the recorded canonical revision.

Gaps [N]: GitHub Discussions, `iperf-dev` archives, releases, commits outside the cited PRs, and unpublished branches remain unchecked. Receive-worker, UDP/SCTP failure, bidirectional, and supported-platform candidate behavior remain untested because the TCP sender candidate already violates root error, cleanup, creation, and normal reverse contracts.

- https://github.com/esnet/iperf/issues/1986 — Existing symptom thread; #1654 targets the missing worker-to-main-loop propagation. Do not open a duplicate thread.
- https://github.com/esnet/iperf/pull/1654 — Active but currently unsafe correction: reconstructed current-master testing proves error replacement, double cleanup, a creation-failure crash, and false normal-teardown diagnostics.
- https://github.com/esnet/iperf/pull/1861 — Related diagnostics: visibility is useful, but printing an error alone does not stop the test.

Target fit: Pull request comment on #1654 is recommended. The current-master reconstruction supplies four concrete regressions and a deterministic reproducer that materially advance the active correction. Mode and Target remain user-unselected; do not publish without user selection and exact-draft approval.

## Direction

Track #1654 rather than open competing work, but reject its current counter/thread-number ownership. A correction should store each worker's terminal outcome in test- or stream-owned synchronized state, preserve the first `IESTREAMREAD`/`IESTREAMWRITE` and native `errno`, retain `thread_created` as lifecycle truth independent of diagnostic numbering, and return immediately after one owning cleanup. Do not use role-global counters, overwrite the root error with `IEPTHREADNOTRUNNING`, treat expected teardown syscalls as worker failures, or continue the server loop after cleanup.

## Bounds

- Preserve: current control-state protocol, `i_errno`/`iperf_strerror()` diagnostics, TCP/UDP/SCTP negotiation, reverse, bidirectional, parallel, JSON, JSON-stream, timing, and normal worker completion.
- Preserve: error cleanup cancels and joins only created workers; a worker failure must not cause duplicate reports, duplicate `SERVER_ERROR`, or a second cleanup.
- Exclude: a global worker-error redesign, new timeout policy, transport-specific retries, and PR #1861-style diagnostics as a replacement for propagation.
- Preserve: supported Ubuntu Linux, FreeBSD, and macOS worker/cancellation behavior and successful mixed-version control operation; no wire or public API change is implied.
- Cost [N]: Counter synchronization, ordering with worker creation/cancellation, and mixed failure races require focused validation; their portability and runtime cost are not measured.

## Verification

- [O] The reconstructed candidate detects both injected sender failures and makes both clients exit 1 within about one second.
- [O] It fails the required contracts: originating error preservation, exactly-once cleanup, safe partial thread creation, quiet successful reverse teardown, and server terminal return ownership.
- [O] Normal forward/reverse `-P 1`/`-P 2`, normal full/streaming JSON, `make -s check` (5/5), and isolated `test_commands.sh 127.0.0.1` completed; the command script's SCTP and IPv6 scenarios remained unavailable in this build/IPv4 namespace.
- [O] The normal-path command suite did not emit worker-failure diagnostics, but focused short reverse server reuse did, proving a timing-sensitive teardown regression not covered by the repository checks.
- Test decision: none — verification replayed the external PR in an isolated worktree and used an external preload injector; no owned source or tests changed.

## Missing

- [N] An exact #1654 comment draft and user-selected Report/PR-comment target.
- [N] Maintainer response or a revised candidate addressing the four reproduced regressions.
- [N] Receive-worker, UDP/SCTP failure, bidirectional, FreeBSD, and macOS verification for any revised candidate.
- [N] Prior-art coverage for Discussions, mailing list, releases, broader commit history, and unpublished work.

## Resume

Index: Draft PR 1654 findings
Next: Prepare an exact PR #1654 comment with the injector contract, four reproduced regressions, and bounded ownership direction.
Done when: The complete comment and exact target are ready for user review without publishing.

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
