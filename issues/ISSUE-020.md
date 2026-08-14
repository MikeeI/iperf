# ISSUE-020 — stream adoption: failed construction loses data socket

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

[S] Client and server both own a connected/accepted data socket until stream adoption. The client returns after `iperf_new_stream(test, s, ...) == NULL` without closing `s` (`src/iperf_client_api.c:123-170`); the server does the same after either `iperf_common_sockopts(test, s) < 0` or `iperf_new_stream(test, s, ...) == NULL` (`src/iperf_server_api.c:774-876`). [S] `iperf_new_stream()` can fail before or after assigning `sp->socket = s`; its error chain closes stream-owned temporary resources but never `s`, then returns `NULL` (`src/iperf_api.c:4990-5084`). It transfers the socket into `test->streams` only after `iperf_init_stream()` succeeds and `iperf_add_stream()` runs (`src/iperf_api.c:5063-5068`, `5185-5211`).

This is one constructor/adoption-boundary root across both roles, not separate client and server findings.

## Reach and impact

[S] The client path is reachable after `test->protocol->connect()` succeeds; the server path is reachable after `test->protocol->accept()` succeeds. The pre-adoption failures are client stream construction, server common-socket-option setup, and server stream construction (`src/iperf_client_api.c:123-170`, `src/iperf_server_api.c:774-876`). [S] Constructor failures include stream/result allocation, temporary-buffer setup, disk-file setup, entropy, and stream initialization (`src/iperf_api.c:4990-5068`). Each listed error path returns with `s` neither closed nor added to `test->streams`. [N] Failure frequency, descriptor-count growth, process or embedded-server exhaustion, and user impact are unmeasured. [A] An embedding that continues after such returns can accumulate those unowned descriptors.

[S] Explicit pre-adoption closes establish caller ownership: the client closes `s` on post-connect congestion-option failures (`src/iperf_client_api.c:128-150`); the server closes `s` on post-accept user-timeout and congestion-option failures (`src/iperf_server_api.c:788-841`). [S] After successful insertion, stream lifecycle owners close `sp->socket` in `iperf_client_end()` (`src/iperf_client_api.c:557-574`) and server cleanup (`src/iperf_server_api.c:337-345`, `506-514`).

## Evidence

- [S] Only two in-tree callers invoke `iperf_new_stream()`: `src/iperf_client_api.c:168` and `src/iperf_server_api.c:873`.
- [S] The client returns on a null stream; the server returns on failed common socket options or a null stream; none of those paths closes `s`. The preexisting nearby option-failure paths preserve `errno` around their direct close.
- [S] `iperf_new_stream()` assigns `s` to the stream at `src/iperf_api.c:5038-5039`, but all error labels through `src/iperf_api.c:5070-5084` omit that descriptor.
- [S] `iperf_add_stream()` inserts the stream only on the successful path (`src/iperf_api.c:5063-5068`, `5185-5211`), and later cleanup closes only sockets reachable through `test->streams`.
- [O] Historical repository baseline: `make check` returned `5/5 pass`; it does not force a post-connect/pre-adoption failure and is not evidence for this ownership defect.

## Prior art

[O] Audit-supplied candidates https://github.com/esnet/iperf/issues/1986, active https://github.com/esnet/iperf/pull/1654, related https://github.com/esnet/iperf/pull/1861, and merged https://github.com/esnet/iperf/pull/1709 classify as worker-failure propagation or parameter-length handling. They do not own the connected/accepted socket before stream adoption. [N] No verified external thread or pull request currently owns this lifecycle root.

## Direction

[A] Make the existing pre-adoption owners close `s` on every local error after connection/acceptance and before successful insertion: client null-stream return; server common-socket-option failure and null-stream return. Preserve `errno` and the selected `i_errno`; success stays unchanged. These changes are coupled by the single ownership rule and should land together.

Rejected: make `iperf_new_stream()` unconditionally close `s` on failure. `src/iperf_api.h:296-303` declares this constructor for external consumers, while in-tree callers demonstrably close sockets themselves before successful adoption; changing the callee's failure ownership would silently alter that boundary and risks caller double-close. Rejected: fix only the client, only the server constructor call, or omit the server common-socket-option path; each leaves the same pre-adoption loss route.

## Bounds

[S] Preserve direct caller ownership before insertion, transfer ownership only after `iperf_add_stream()`, and preserve the existing post-adoption close paths. [S] Keep the original setup error (`i_errno`) and native `errno` visible after cleanup; do not mask it with `close()` failure. The caller-side cleanup changes no control data, so same- and mixed-version peers retain existing successful stream behavior. Excluded scope: redesigning the public constructor contract, stream-list cleanup, or protocol behavior.

## API and compatibility

Callers [S]: in-tree callers are `iperf_create_streams()` and the server `CREATE_STREAMS` path; `iperf_new_stream()` is declared in `src/iperf_api.h:296-303`.

Contract [S]: before successful insertion, client/server callers already close their local `s` on intervening socket-option errors; after insertion, stream lifecycle code closes `sp->socket`.

Compatibility: a caller-side failure cleanup preserves successful stream behavior, wire protocol, and the constructor's existing external failure-ownership boundary.

Migration: None.

## Verification

Test decision: none; no gates ran for this ledger edit. Any later correction must force a client `iperf_new_stream()` failure, server `iperf_common_sockopts()` failure, and server `iperf_new_stream()` failure after their sockets exist; each failed descriptor must close while the original error survives, and successful streams must still close exactly once.

## Missing

[N] No deterministic reproduction or descriptor-count measurement on either role; no evidence of descriptor accumulation or exhaustion in an embedded or persistent server; no current external target.

## Resume

Index: Audit pre-adoption socket cleanup
Next: Force each pre-adoption client/server failure while measuring descriptor ownership.
Done when: Client stream construction, server common-socket-option, and server stream-construction failures each close the unadopted descriptor exactly once and retain the owning error.
