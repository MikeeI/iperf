# ISSUE-009 — control cleanup: preserve primary diagnostic

State: Hold
Mode: Undecided
Target: Undecided
Location: Not published.
Priority: High
Confidence: High
Type: correctness
Created: 2026-08-14
Updated: 2026-08-14
Source: `upstream/master@c9b74229d0d9bfec6d2307b66b43c29a7665ad0b`

## Root

[S] `i_errno` is the primary diagnostic selected before cleanup. Its value must survive fallible error reporting and terminal teardown. `cleanup_server()` starts its `SERVER_ERROR` exchange before capturing `i_errno` (`src/iperf_server_api.c:460-500`): the save is at line 477, after `iperf_set_send_state(test, SERVER_ERROR)` at line 467.

[S] `iperf_set_send_state()` writes the state and replaces `i_errno` with `IESENDMESSAGE` when that `Nwrite()` fails (`src/iperf_api.c:2063-2073`). Thus a failed `SERVER_ERROR` state write causes server cleanup to retain the secondary code instead of the originating one.

[S] The client has the same policy failure in a different order. `cleanup_and_fail` saves and restores `i_errno` around thread cancellation, then calls unchecked `iperf_client_end()` (`src/iperf_client_api.c:882-919`). `iperf_client_end()` sends `IPERF_DONE` for a positive state and propagates a failed state write through `iperf_set_send_state()` (`src/iperf_client_api.c:557-591`); the later JSON error string therefore can describe `IESENDMESSAGE`, not the original failure.

[S] This is one cross-role error-preservation root, not two unrelated control-send failures: recovery code lets a secondary cleanup/control-write failure replace the primary diagnostic it is meant to report.

## Reach and impact

[S] Server reach requires a non-`IENONE` error, an open control socket, and failure while sending `SERVER_ERROR`. Client reach requires the `cleanup_and_fail` path, a positive test state, and failure while sending `IPERF_DONE`.

[S] The affected contract is error identity. Server cleanup can retain `IESENDMESSAGE` rather than the initiating error, and the client's final JSON `"error"` can be derived from that secondary code.

[N] Failure frequency, user-visible wording, and the proportion of runs with a failed control write are not measured.

[S] Do not claim that a peer receives an incorrect `SERVER_ERROR` frame when the state-byte write itself fails: that failure prevents the subsequent payload write in the shown branch.

## Evidence

[S] Server order: `cleanup_server()` tests `i_errno`, calls `iperf_set_send_state()` at `src/iperf_server_api.c:465-474`, and only then snapshots `i_errno` at line 477. The shared sender assigns `IESENDMESSAGE` on its failed write at `src/iperf_api.c:2066-2071`.

[S] Client order: `cleanup_and_fail` restores `i_errno` at line 913, invokes `iperf_client_end()` at line 915 without checking its result, and builds JSON from the post-call `i_errno` at lines 916-919. `iperf_client_end()` returns `-1` after the state-send failure at `src/iperf_client_api.c:579-583`.

[S] Public ownership is explicit: `iperf_set_send_state()` and `iperf_client_end()` are declared in `src/iperf_api.h:343,375-376`.

[N] No controlled `Nwrite()` failure was injected in this audit; the control-flow consequences above are source-proven, not an observed end-user reproduction.

## Prior art

[S] Recorded direct-read coverage on 2026-08-14: https://github.com/esnet/iperf/issues/1986, active https://github.com/esnet/iperf/pull/1654, and related https://github.com/esnet/iperf/pull/1861 own worker-failure propagation.

[S] Classification: worker failure propagation is not this record's root. This record owns preservation of a pre-existing primary `i_errno` across control reporting and cleanup; no provided candidate demonstrates that exact policy failure.

[N] Target fit: none of those candidates is selected for a comment or pull-request extension. Mode and Target remain user decisions.

[N] A current upstream search specifically for primary-diagnostic clobbering is not recorded; that gap keeps this finding on Hold.

## Direction

At each role-specific recovery boundary, snapshot the primary `i_errno` before its first fallible recovery I/O, use that snapshot for the `SERVER_ERROR` payload and final JSON diagnostic, and restore it after fallible teardown.

Keep the policy local to `cleanup_server()` and `cleanup_and_fail`; `iperf_set_send_state()` must retain its direct-caller `IESENDMESSAGE` contract. Do not add a generic wrapper that hides state-specific error serialization.

This excludes ISSUE-008's `DISPLAY_RESULTS` success path: it has no earlier diagnostic to preserve and must propagate its discarded final-write failure.

Do not convert a cleanup-write failure into success or invent a secondary-diagnostic channel in this correction.

## Bounds

- Preserve the successful `SERVER_ERROR` state/error/errno wire sequence, `iperf_set_send_state()`'s `IESENDMESSAGE` contract for direct callers, existing `-1` failure returns, and thread/socket cleanup ordering.
- Preserve protocol states, public declarations, JSON schema keys, normal-success output, supported Ubuntu Linux, FreeBSD, and macOS behavior, and successful mixed-version negotiation.
- Do not redesign the global error model or alter unrelated `Nwrite()` handling.

## Shared change pressure

Copies [S]: Two role-specific recovery paths carry the same primary-diagnostic invariant: server `cleanup_server()` (`src/iperf_server_api.c:460-500`) and client `cleanup_and_fail` (`src/iperf_client_api.c:882-919`), both through shared `iperf_set_send_state()` (`src/iperf_api.c:2063-2073`).

Pressure [S]: Any future control-send failure path must preserve the initiating error while performing secondary recovery I/O.

Drift [S]: The server saves too late; the client restores before a later fallible terminal call and does not restore afterward.

Owner: The error-preservation policy at role-specific recovery boundaries, using the shared sender's existing error contract.

Cost: Keep the policy local to recovery paths; a generic wrapper is unjustified unless it can preserve this exact ordering without hiding state-specific serialization.

## API and compatibility

Callers [S]: `iperf_set_send_state()`, `iperf_client_end()`, and `iperf_run_server()` are libiperf API declarations in `src/iperf_api.h:343,375-379`; CLI error/JSON presentation consumes their outcomes.

Contract [S]: A recovery path must report the operation that failed before cleanup, not overwrite it with a later best-effort control-write failure.

Compatibility: Preserve state-byte framing, error-code framing, existing public signatures, and client/server behavior on successful control writes.

Migration: None.

## Verification

Test decision: none. This ledger-only change adds no test.

[A] Inject an `Nwrite()` failure while the server sends `SERVER_ERROR` after a known primary `i_errno`; assert the caller retains the original code after cleanup and that no claim is made about an unsent payload.

[A] Inject an `Nwrite()` failure for the client's `IPERF_DONE` during `cleanup_and_fail`; assert `iperf_run_client()` remains failing and its final JSON error names the original primary error rather than `IESENDMESSAGE`.

[A] Run the same server/client control paths without injection; assert their current successful state/error framing and cleanup behavior remain unchanged.

## Missing

- [N] Controlled fault-injection results for both roles; a current upstream search and classification for this exact root; user selection of Mode and Target; and an external draft.

## Resume

Index: Preserve primary cleanup error

Next: Inject failures in the `SERVER_ERROR` and `IPERF_DONE` state-write paths at the recorded revision.

Done when: The two runs retain their initiating `i_errno` through teardown with captured outputs and exit results, or source evidence disproves either reach condition.
