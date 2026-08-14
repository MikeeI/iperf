# ISSUE-010 — server JSON finish: terminal failure skips cleanup

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

[S] `iperf_run_server()` owns the terminal server lifecycle and normally calls `cleanup_server()` after JSON finalization. Its JSON branch returns immediately when `iperf_json_finish()` fails (`src/iperf_server_api.c:995-1002`), bypassing that cleanup owner.

[S] `iperf_json_finish()` can return `-1` when `cJSON_Print(test->json_top)` or the subsequent `strdup()` returns `NULL` (`src/iperf_api.c:5486-5539`). The root is therefore a terminal-error branch that skips required cleanup, not ordinary JSON serialization quality.

## Reach and impact

[S] Reach is a server run with JSON output that enters the full JSON rendering path (`print_full_json` remains true) and suffers either allocation failure. Normal `--json` reaches that path; `--json-stream` reaches it only when full output remains enabled (`src/iperf_api.c:5486-5519`).

[S] Direct libiperf callers receive `-1` from `iperf_run_server()` without the stream, control/listener, timer, and congestion cleanup owned by `cleanup_server()` (`src/iperf_server_api.c:506-551`).

[S] In the CLI loop, `main()` retries `iperf_json_finish()` after the `iperf_run_server()` error, then calls `iperf_reset_test()` (`src/main.c:171-185`). If the retry succeeds after a one-shot allocation fault, reset frees streams but `iperf_free_stream()` does not close `sp->socket` (`src/iperf_api.c:3713-3718,4938-4957`), while reset overwrites control/listener descriptors with `-1` (`src/iperf_api.c:3755-3759`). The skipped server cleanup can therefore leave descriptors open in a continuing server process.

[N] Allocator-failure frequency, the number of leaked descriptors per affected run, and end-user incidence are unmeasured. Do not claim a normal-workload leak or process-lifetime leak: a process that exits releases descriptors.

## Evidence

[S] The server terminal sequence is exact: `iperf_json_finish()` at `src/iperf_server_api.c:995-998`, `iflush()` at line 1000, then `cleanup_server()` at line 1001. The early return skips both following operations.

[S] The explicit allocation-failure returns are `cJSON_Print(...) == NULL` at `src/iperf_api.c:5513-5516` and `strdup(...) == NULL` at lines 5517-5521. Neither path reaches `cJSON_Delete()` or the pointer reset at lines 5535-5539 on that invocation.

[S] `cleanup_server()` closes every live stream socket, the control socket, listener, and protocol listener, and cancels timers (`src/iperf_server_api.c:506-551`). `iperf_reset_test()` is not an equivalent cleanup for open stream sockets because `iperf_free_stream()` closes buffer and disk descriptors but not `sp->socket` (`src/iperf_api.c:4943-4957`).

[N] No allocator fault was injected in this audit. The conditional resource-retention path is source-proven; no production or observed descriptor leak is claimed.

## Prior art

[S] Recorded direct-read coverage on 2026-08-14: https://github.com/esnet/iperf/issues/1986, active https://github.com/esnet/iperf/pull/1654, and related https://github.com/esnet/iperf/pull/1861 own worker-failure propagation.

[S] Classification: those candidates do not own a server terminal JSON-allocation failure that bypasses `cleanup_server()`. No candidate is selected as a comment or pull-request target for this root.

[N] A current upstream search for JSON-finalization cleanup failures is not recorded; that target/currentness gap keeps the finding on Hold.

## Direction

Route every terminal `iperf_json_finish()` status through one `iperf_run_server()` epilogue that invokes `cleanup_server()` exactly once before returning the saved JSON-finalization status.

Do not depend on `main()`'s second `iperf_json_finish()` call as lifecycle recovery; it can remain a presentation decision only after the server-run owner has released its resources.

If JSON serialization needs a distinct `i_errno` mapping, scope that diagnostic decision separately; it is not a prerequisite for fixing the skipped cleanup branch.

## Bounds

- Preserve normal JSON and JSON-stream output selection, callback behavior, one-off/server-loop behavior, stream/control wire protocol, public return signatures, and successful terminal ordering except that cleanup must also occur on a JSON-finalization failure.
- Preserve supported Ubuntu Linux, FreeBSD, and macOS behavior and successful mixed-version control operation; no wire or public API change is required.
- Do not claim to make allocation failure recoverable, change cJSON allocation policy, add a generic deferred-cleanup framework, or alter client JSON paths outside this server terminal branch.

## API and compatibility

Callers [S]: `iperf_run_server()` and `iperf_json_finish()` are public declarations in `src/iperf_api.h:379,389`; `src/main.c:171-185` is the CLI server-loop consumer.

Contract [S]: A nonzero JSON-finalization result must not suppress the server-run function's owned resource teardown.

Compatibility: Keep current JSON fields, state framing, public signatures, and return-failure semantics; only make cleanup unconditional at this terminal ownership boundary.

Migration: None.

## Verification

Test decision: none. This ledger-only change adds no test.

[A] Fault-inject `cJSON_Print()` allocation failure in a one-off JSON server run; assert `iperf_run_server()` returns failure and that every stream socket, `ctrl_sck`, `listener`, `prot_listener`, and owned timer is cleaned before return.

[A] Independently fault-inject the `strdup()` allocation failure after successful cJSON rendering and assert the same cleanup invariant and first-failure return.

[A] Repeat without injection for normal `--json` and JSON-stream full-output paths; assert unchanged output and successful lifecycle. Exercise the CLI retry path separately to prove it cannot leave descriptors behind after a one-shot fault.

## Missing

- [N] Two allocator-fault-injection outcomes; a current upstream search and classification for this exact root; user selection of Mode and Target; and an external draft.

## Resume

Index: Inject JSON cleanup failure

Next: Inject one-shot `cJSON_Print()` and `strdup()` allocation failures at the server terminal JSON path.

Done when: Each injected path demonstrates that `iperf_run_server()` releases its owned resources before returning its failure status, with normal JSON paths unchanged.
