# ISSUE-006 — transfer workers: failed I/O exits without main-loop propagation

State: Published
Mode: Report
Target: Pull request comment
Location: https://github.com/esnet/iperf/pull/1654#issuecomment-5288277182
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

- [N] No publication blocker remains. Maintainer response or a revised candidate addressing the reproduced regressions is pending.
- [N] Receive-worker, UDP/SCTP failure, bidirectional, FreeBSD, and macOS verification remains for any revised candidate.

## Resume

Index: Monitor PR 1654 response
Next: Monitor PR #1654 for maintainer response or a revised worker-propagation candidate.
Done when: Maintainers respond, the candidate changes, or the pull request reaches an observable disposition.

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

## Draft

Target: Pull request comment on https://github.com/esnet/iperf/pull/1654.

````markdown
Hi, thanks for working on this.

I reproduced the underlying missing worker-to-main-loop propagation on current [`master@c9b7422`](https://github.com/esnet/iperf/commit/c9b74229d0d9bfec6d2307b66b43c29a7665ad0b), then replayed all four commits from this PR through [`d7ab071`](https://github.com/esnet/iperf/tree/d7ab0713d95ab163825a75d8b9772092ca49117d) onto that revision. The replay needed only one mechanical conflict resolution in the expanded `iperf_error.c` switch.

The fault injector keeps the control socket and second data stream live, selects the first data socket after the control connection, lets 31 `write()` calls succeed, then returns `-1` with `errno=EIO` for that socket. I ran it against client and server sender workers with `-t 4 -i 1 -P 2`; the server case used `-R`.

The PR detects both worker failures, but I found these regressions:

1. **The originating stream error is replaced.** The worker correctly reports `unable to write to stream socket: Input/output error`, but the main loop overwrites `IESTREAMWRITE/EIO` with `IEPTHREADNOTRUNNING`. The client-facing result becomes `a thread stopped running unexpectedly: `, including a trailing colon without native error detail. The client exits 1 after about one second but emits the first interval twice before cleanup.

2. **The server continues after destructive cleanup.** On the server-worker injection, the client receives the generic `SERVER_ERROR`, while the server calls `cleanup_server(test)` and then continues its event loop. The observed tail was:

```text
State set to SERVER_ERROR
All threads stopped
All threads stopped
select failed: Bad file descriptor
```

This replaces the root transfer error with a secondary `select()` failure and executes cleanup twice.

3. **A partial `pthread_create()` failure now crashes.** `thread_number` is assigned before `pthread_create()`, while cleanup treats `thread_number > 0` as proof that the thread exists. Faulting the second creation with `EAGAIN` made the reconstructed PR exit 139. Current master under the same injector cleanly reports `unable to create thread` and exits 1. Keeping `thread_created` separate from diagnostic numbering avoids canceling or joining an uncreated thread.

4. **Normal reverse teardown emits false worker failures.** Successful persistent-server forward/reverse runs with both `-P 2` and `-P 1` returned 0, but each reverse teardown logged errors such as:

```text
Server Worker Thread 1 FD 5 failed - unable to write to stream socket: Bad file descriptor
```

The worker is classifying an expected teardown race as a transfer failure.

There are two related ownership problems in the current approach:

- `running_threads` is role-global `static volatile`, not test-owned. Workers modify it under `running_mutex`, but main loops read it without that mutex; `volatile` does not provide synchronization.
- Workers call `iperf_err()` directly, moving text, JSON-stream, logfile, and callback output into worker-thread context. In JSON streaming, the observed sequence was `start, interval, interval, error, end`; full JSON contained the generic error, two intervals, and an `end` object.

The reconstructed candidate built successfully, `make -s check` passed 5/5, and `test_commands.sh 127.0.0.1` exited 0. Those checks did not catch the partial-creation crash or the timing-sensitive reverse teardown diagnostics.

Would it make sense to keep worker completion as test- or stream-owned synchronized state containing the first exact `i_errno` and native `errno`, retain `thread_created` as independent lifecycle truth, and let only the owning main loop report the error and enter one cleanup path followed by an immediate return?

I checked the existing PR body and discussion; these current-master reproduction results and regressions were not already reported.

### Disclosure

Investigated thoroughly with GPT-5.6 (extra high reasoning effort), using [Oh My Pi](https://github.com/can1357/oh-my-pi) as the agent framework.

This report is not generic or unreviewed AI-generated output. Its claims were checked against the cited evidence, and it includes the relevant detail intended to help maintainers resolve the issue.

If reports like this are not useful to the project, please let me know and I will refrain from submitting similar ones. My intent is to help without wasting maintainer time or energy or discouraging their work.

Thank you for your work.
````
