# ISSUE-008 — client completion: failed final control write is reported as success

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

Root [S]: On `DISPLAY_RESULTS`, `iperf_handle_message_client` ignores `iperf_client_end`'s return. If its final `IPERF_DONE` control write fails, `iperf_set_send_state` has already changed local state to `IPERF_DONE`; the handler returns success, the run loop exits normally, and the skipped end-path control close/error return is not propagated to the caller.

Root boundary [S]: This is the ignored terminal-write return, not ISSUE-009's recovery failure: the false-success path arises when the final `IPERF_DONE` write is the only failure.

## Reach and impact

Reach [S]: Any completed client data mode can enter `DISPLAY_RESULTS` and call `iperf_client_end` with a live control socket. The failure requires the final one-byte `IPERF_DONE` write on the TCP control channel to fail after results display; it is independent of TCP/UDP/SCTP data transport. Forward, reverse, bidirectional, and parallel runs all have this final control transition; parallelism changes stream cleanup quantity, not the trigger.

Impact [S]: `iperf_set_send_state` sets `i_errno = IESENDMESSAGE` after the failed write but first changes the local state. Because the handler discards `iperf_client_end`'s `-1`, the loop condition becomes false and the ordinary tail can return `0` after receiver cleanup and output finishing. `iperf_client_end` returns early before `iperf_sync_close_socket`, so the intended final control close is skipped.

Impact [N]: No injected current-master run establishes CLI exit status, JSON error output, peer-visible close behavior, duplicated reporting during an attempted fix, or production frequency. Do not label the source flow as an observed user-visible failure.

## Evidence

- [O] Currentness: `git diff --quiet upstream/master -- src/iperf_client_api.c src/iperf_server_api.c src/iperf_api.c` returned `0` on 2026-08-14; these local source anchors equal `upstream/master@c9b74229d0d9bfec6d2307b66b43c29a7665ad0b`.
- [S] `src/iperf_client_api.c:369-373` — the `DISPLAY_RESULTS` handler calls `iperf_client_end(test)` and discards its result before returning success.
- [S] `src/iperf_client_api.c:558-591` — `iperf_client_end` closes stream FDs and reports, then returns `-1` if its final `iperf_set_send_state(test, IPERF_DONE)` fails; that early return bypasses `iperf_sync_close_socket`.
- [S] `src/iperf_api.c:2063-2073` — `iperf_set_send_state` calls `iperf_set_test_state(test, state)` before `Nwrite`; a failing write sets `i_errno = IESENDMESSAGE` and returns `-1`.
- [S] `src/iperf_client_api.c:656,723-728,839-880` — the run loop stops when the locally changed state equals `IPERF_DONE`, then performs normal receiver cleanup/output and returns `0` unless a later operation fails.
- [S] `src/iperf_client_api.c:882-921` — merely checking the ignored return is insufficient: failure cleanup calls `iperf_client_end` again after the first call has already closed data FDs and reported, risking duplicate finalization; on that failed first call, `iperf_sync_close_socket` did not run and `test->ctrl_sck` remains unmarked.
- [S] `src/net.c:877-887` — `iperf_sync_close_socket` shuts down, drains, and closes the supplied FD; it has no test-state side effect, so its owner must mark the FD after use if a later cleanup can revisit it.
- [S] `src/main.c:201-202` and `src/iperf_api.h:372` — CLI maps a negative `iperf_run_client` return with `i_errno` to terminal error output; the ignored failure bypasses that contract.
- [O] https://github.com/esnet/iperf/issues/677 was closed when read on 2026-08-14. It reported the former opposite ordering in `iperf_client_end`—control socket closed before `IPERF_DONE`—and is fixed in the current source's send-then-close order.
- [O] https://github.com/esnet/iperf/pull/597 was merged when read on 2026-08-14. It added control-close ownership in `iperf_client_end`, the historical change that #677 corrected for state ordering.
- [O] https://github.com/esnet/iperf/issues/1405 and https://github.com/esnet/iperf/pull/1406 were closed when read on 2026-08-14. They fix JSON exit status for an earlier client failure path, not this failed final completion write.
- [O] https://github.com/esnet/iperf/pull/2034 was merged when read on 2026-08-14. It hardens client buffer-FD closing in `iperf_client_end`, but does not propagate the final state-write failure or own its partial-finalization state.

## Prior art

Coverage [O]: GitHub issue search `"iperf_set_send_state" "IPERF_DONE"` returned #677; issue search `"iperf_client_end"` returned #1545, #1991, #1405, #1023, #677, and #217; PR search `"iperf_client_end"` returned #2034, #1406, #1139, #1128, and #597 on 2026-08-14. Direct-read #677, #1405, #1991, #1545, #2034, #1406, #1139, #1128, and #597; source anchors were checked at the recorded canonical revision.

Gaps [N]: GitHub Discussions, `iperf-dev` archives, releases, broader commit history, active unpublished branches, and an injected final-write failure remain unchecked. Search candidates #1545, #1991, #1023, #217, #1139, and #1128 do not state this final completion return path; no exact current open owner was found.

- https://github.com/esnet/iperf/issues/677 and https://github.com/esnet/iperf/pull/597 — Related historical control-close ordering; their completed correction is distinct from a failed write whose return is ignored.
- https://github.com/esnet/iperf/issues/1405 and https://github.com/esnet/iperf/pull/1406 — Related error-return handling in an initial/JSON client path; distinct lifecycle phase and closed.
- https://github.com/esnet/iperf/pull/2034 — Related `iperf_client_end` FD lifetime work; merged buffer-FD protection does not make partial finalization idempotent or propagate `IESENDMESSAGE`.
- https://github.com/esnet/iperf/issues/1545, https://github.com/esnet/iperf/issues/1991, https://github.com/esnet/iperf/issues/1023, https://github.com/esnet/iperf/issues/217, https://github.com/esnet/iperf/pull/1139, and https://github.com/esnet/iperf/pull/1128 — Distinct API, cancellation, leak, shutdown, or initial-connection topics; none owns the final `DISPLAY_RESULTS` write failure.

Target fit: No exact open thread was found. A new issue may be useful only after a minimal final-write-failure reproduction demonstrates a caller-visible result; Mode and Target remain user-unselected.

## Direction

At the client completion owner, propagate a failed final `IPERF_DONE` write into one error cleanup path while making partial completion exactly-once: preserve `IESENDMESSAGE`, close/mark the control FD exactly once, and avoid a second data-FD close or reporter callback when cleanup revisits completion. Do not apply only a one-line return check—the existing failure cleanup calls `iperf_client_end` again after the first partial end. Do not fold this into ISSUE-009: that record preserves an earlier diagnostic across fallible recovery, while this record propagates a terminal failure the success path ignored. Do not change generic `iperf_set_send_state` ordering without auditing every control state, because its local-before-write behavior is a shared state-machine contract.

## Bounds

- Preserve: final report content and single-callback behavior; JSON/JSON-stream error shaping; `TEST_END`, `EXCHANGE_RESULTS`, `DISPLAY_RESULTS`, and `IPERF_DONE` wire semantics; and the existing CLI/libiperf negative-return plus `i_errno` contract.
- Preserve: TCP control-channel framing and data behavior for TCP, UDP, SCTP, reverse, bidirectional, and parallel on supported Ubuntu Linux, FreeBSD, and macOS; successful mixed-version runs remain unchanged.
- Preserve: the existing cancellation/join sequence; the final-write correction must not add a duplicate worker cancellation, data-FD close, or reporter callback.
- Exclude: generic control-state refactoring, retrying a failed final write, changing server timeout policy, and unrelated descriptor-leak cleanup already addressed by #2034.
- Cost [N]: The partial-finalization state needs an explicit ownership boundary; exact output/close ordering and mixed-version impact are unmeasured.

## Verification

- [A] Inject `Nwrite` failure only for the client’s final one-byte `IPERF_DONE` after `DISPLAY_RESULTS`; require `iperf_run_client` to return `-1`, retain `IESENDMESSAGE`, emit one error result, and close the control FD once.
- [A] Instrument reporter callback and close calls; require no duplicate final report or FD close when failure cleanup runs after the partial end.
- [A] Compare successful TCP, UDP, and SCTP runs where configured; forward, reverse, bidirectional, `-P 1`/`-P 2`; text, JSON, and JSON streaming, proving unchanged final output and control sequence.
- [A] Cover a peer control close, initial connection failure, server error, and server reuse to distinguish the final-write path from existing error handling.
- Test decision: none — this ledger-only assignment changed no source or tests; no gate was run.

## Missing

- [N] A deterministic current-master injection of the final `IPERF_DONE` write failure with client/server output and return status.
- [N] A focused trace of callback, data-FD, and control-FD operations across partial completion and cleanup re-entry.
- [N] Current supported-platform, mixed-version, and transport-mode validation.
- [N] Prior-art coverage for Discussions, mailing list, releases, broader commits, and unpublished work.
- [N] User selection of Mode and Target.

## Resume

Index: Inject final control failure
Next: Force the one-byte client `IPERF_DONE` write to fail after `DISPLAY_RESULTS`, then capture return status, `i_errno`, JSON/text output, callback count, and both control closes.
Done when: The run demonstrates whether completion is falsely successful and supplies a bounded, exactly-once correction contract suitable for target selection.

## API and compatibility

Callers [S]: `iperf_run_client` is a public libiperf entry point; `src/main.c:201-202` reports its negative result with `iperf_strerror(i_errno)`.

Contract [S]: A failed terminal control write must remain distinguishable from successful completion to CLI and libiperf callers, while finalization closes each owned resource and emits each report at most once.

Compatibility: Keep the established control wire states and successful peer behavior. The correction is internal state/error ownership; it must not require a protocol version change.

Migration: None.
