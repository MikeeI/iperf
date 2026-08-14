# ISSUE-012 — server control EOF: handler return conflicts with error state

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

[S] The server control-EOF branch in `iperf_handle_message_server()` records `IECTRLCLOSE`, sets `IPERF_DONE`, and returns `0`; a non-EOF control-read failure sets `IERECVMESSAGE` and returns `-1` (`src/iperf_server_api.c:239-259`). The outer server loop treats only a negative handler result as an immediate error (`src/iperf_server_api.c:763-769`).

[S] The handler itself marks this ambiguity with `// XXX: Need to rethink how this behaves to fit API` immediately before the control read (`src/iperf_server_api.c:248-249`).

[A] The intended outer return contract remains unresolved. The source does not by itself establish whether EOF is a recoverable completed session, a server-run failure, or a condition that must be surfaced differently by the CLI or public API.

## Reach and impact

[S] Reach requires EOF on a readable server control socket. The outer loop invokes `iperf_handle_message_server()` and does not take its immediate error return for this `0` result (`src/iperf_server_api.c:763-769`).

[S] The handler-set `IPERF_DONE` ends the loop; the normal tail then finishes JSON if enabled, flushes, cleans up, and normally returns `0` unless a separate tail operation fails (`src/iperf_server_api.c:621-1007`).

[N] No observation measures the public `iperf_run_server()` result, CLI exit behavior, persistent-server restart behavior, or user-visible harm after control EOF. The direct handler result alone is not proof of a user-visible bug.

## Evidence

[S] Canonical source audit: `upstream/master@c9b74229d0d9bfec6d2307b66b43c29a7665ad0b`; `src/iperf_server_api.c:239-259,621-1007`.

[O] Recorded direct control-EOF probe result: `iperf_handle_message_server()` returned `0` while global `i_errno=109` (`IECTRLCLOSE`).

[S] The source proves the local zero-return/error-state combination and its normal outer-loop tail. It does not prove the intended public or CLI meaning of that outcome.

## API and compatibility

Callers [S]: The server loop invokes `iperf_handle_message_server()` at `src/iperf_server_api.c:763-769`; external CLI and public `iperf_run_server()` return semantics remain untraced.

Contract [O]: The recorded handler probe returns `0` with `i_errno=IECTRLCLOSE`.

Compatibility: Preserve the diagnostic `i_errno` mapping, control-channel cleanup, and any intentional ability to serve or report a subsequent client session on Ubuntu Linux, FreeBSD, and macOS.

Migration: None unless traced public callers require a documented behavior change.

## Prior art

[S] Worker-failure propagation is separately owned by https://github.com/esnet/iperf/issues/1986, active https://github.com/esnet/iperf/pull/1654, and related https://github.com/esnet/iperf/pull/1861. They are adjacent lifecycle work, not evidence that they own this EOF-return contradiction.

[A] No direct external issue, pull request, or comment has been verified for this exact control-EOF return contract. Target remains `Undecided`.

## Direction

First establish the required `iperf_run_server()` and CLI outcome after a disconnected control session, including whether the server intentionally returns to an idle state for another client.

Changing only the handler return code is unsound: it would select a public lifecycle policy from an internal probe without establishing caller expectations.

## Bounds

[S] Keep the existing control protocol, `i_errno`/`iperf_strerror()` diagnostic path, cleanup ordering, and normal server-session completion behavior.

[A] Preserve potential repeated-server recovery unless traced callers and observed behavior disprove it.

[S] Keep public signatures and mixed-version control framing unchanged; no platform-specific control-socket policy is proposed.

[N] No source change, external draft, mode selection, or target selection is authorized by this record.

## Verification

Test decision: none. No tests or gates were run for this tracker-only update.

[S] After the outer contract is established, induce server control EOF; capture handler, `iperf_run_server()`, and CLI results; then run a second client connection to prove the documented restart or terminal-failure path.

## Missing

Public repeated-server return semantics; traced outer caller behavior; observed CLI/API outcomes after control EOF and a subsequent-client attempt; and direct prior-art search for this exact root cause.

## Resume

Index: Define EOF return contract
Next: Trace the control-EOF result through `iperf_run_server()` and its CLI/public callers, including one subsequent-client attempt.
Done when: The observed outer result and repeated-server behavior establish whether `0` is intentional recovery or masked failure.
