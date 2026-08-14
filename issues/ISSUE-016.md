# ISSUE-016 — JSON control frames: short writes are accepted

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

Root [S]: `JSON_write` considers both length-prefix and JSON-body writes successful unless `Nwrite` returns a negative value (`src/iperf_api.c:3170-3191`). `Nwrite` can return a positive partial count after one or more bytes were written and a later `EINTR`, `EAGAIN`, or `EWOULDBLOCK` ends its loop (`src/net.c:648-679`). Thus `JSON_write` returns success after emitting a truncated length prefix or body.

## Reach and impact

Reach [S]: `send_parameters` and `send_results` call `JSON_write` through the control socket (`src/iperf_api.c:2531` and `src/iperf_api.c:2964`) and treat its zero result as successful transmission.
Trigger [A]: A false success requires progress followed by a returning signal-handler interruption, or a valid nonblocking control-socket embedding whose later write reports `EAGAIN`/`EWOULDBLOCK`. An interruption before any byte is written returns `NET_SOFTERROR` and is already rejected.
Impact [S]: The receiver requires an exact four-byte length and exact JSON body before parsing (`src/iperf_api.c:3211-3253`), so a partial prefix or body cannot represent a complete JSON frame.
Impact [A]: Depending on what arrives next, the peer can reject the incomplete frame or consume later control bytes as the unfinished frame; the current sender can nevertheless advance as if its parameters or results were delivered.
Impact [N]: No end-to-end partial-write reproduction, affected embedding count, or user-visible consequence is measured.

## Evidence

- [S] `src/net.c:648-679` — after partial progress, `Nwrite` returns `count - nleft` for `EINTR`, `EAGAIN`, or `EWOULDBLOCK`; it returns `NET_SOFTERROR` only when no progress has occurred or `write` returns zero.
- [S] `src/iperf_api.c:3177-3189` — `JSON_write` sends a network-order four-byte `hsize` then `hsize` JSON bytes, but tests both `Nwrite` calls only with `< 0`.
- [S] `src/iperf_api.c:2531-2534` maps a negative `JSON_write` result to `IESENDPARAMS`; `src/iperf_api.c:2964-2967` maps it to `IESENDRESULTS`.
- [S] `src/iperf_api.c:3211-3253` accepts JSON only after exact header and body counts, warning on a short body; the sender-side check is asymmetric.
- [S] Merged `https://github.com/esnet/iperf/pull/1709` added exact receive-size validation and the 8-KiB parameter receive cap. The cap remains receiver-only: `JSON_write` does not enforce it and does not require exact sender counts.
- [A] A valid nonblocking embedding or returning signal handler can create the positive-partial `Nwrite` path; this trigger has not been run against `JSON_write`.

## Prior art

Coverage [O]: Directly reviewed merged `https://github.com/esnet/iperf/pull/1709`, `https://github.com/esnet/iperf/issues/1986`, active `https://github.com/esnet/iperf/pull/1654`, and related active `https://github.com/esnet/iperf/pull/1861` on 2026-08-14.
Gaps [N]: GitHub-wide search for `JSON_write`/`Nwrite` partial results, GitHub Discussions, `iperf-dev` archives, releases, commits outside the reviewed PRs, and unpublished active branches remain unchecked.

- [S] `https://github.com/esnet/iperf/pull/1709` is directly related receive-side framing work: it verifies header size and caps parameter JSON, but it neither checks sender exact counts nor owns the post-progress partial-write path.
- [S] `https://github.com/esnet/iperf/issues/1986`, `https://github.com/esnet/iperf/pull/1654`, and `https://github.com/esnet/iperf/pull/1861` own worker-thread failure propagation/reporting, not JSON control-frame write completeness; they are not duplicates or targets.
- [N] No direct upstream issue, PR, or commit for `JSON_write` accepting a positive partial `Nwrite` result is recorded yet.

Target fit [N]: Undecided pending a controlled trigger, end-to-end consequence, broader prior-art coverage, and user mode selection.

## Direction

Direction [S]: Keep `Nwrite` unchanged and make `JSON_write` the sole owner of JSON-frame completeness: require the header result to equal `sizeof(nsize)` and the body result to equal `hsize`; return `-1` for every other result. Existing `send_parameters` and `send_results` then retain their respective `IESENDPARAMS` and `IESENDRESULTS` mappings. This preserves all complete-frame bytes and avoids changing partial-progress semantics required by unrelated `Nwrite` callers.
Rejected [S]: Changing `Nwrite` to turn all partial progress into a generic error would alter unrelated callers' partial-write semantics; `JSON_write` alone owns this fixed-frame success condition.

## Bounds

- Preserve [S]: The length-prefixed JSON wire format, network byte order, complete-frame bytes, `cJSON_PrintUnformatted` output, parameter/result payload schemas, receiver limits, and existing caller-specific error mapping.
- Change [S]: A partial header or body becomes a failed `JSON_write` instead of false success; no retry is added after a partial frame because retrying onto an unknown peer frame boundary would create a separate protocol decision.
- Exclude [S]: Changes to `Nwrite`, its non-JSON callers, retries, buffering, signal disposition, socket blocking mode, the 8-KiB receive cap, JSON schema/precision, and generic control-message writes.
- Cost [N]: The controlled interruption/nonblocking matrix and any observable peer outcome remain unmeasured.

## Verification

Test decision: none. Planned focused harnesses, not an existing-test update:

- [N] Use a socketpair or loopback control fixture to force a header write and a body write to return each positive count from `1` through `count - 1`, followed by a returning-signal `EINTR` or `EAGAIN`/`EWOULDBLOCK`; assert `JSON_write == -1`.
- [N] Exercise `send_parameters` and `send_results`; assert their existing mappings remain `IESENDPARAMS` and `IESENDRESULTS` when the partial write is rejected.
- [N] Capture complete parameter and result frames before and after the candidate; require byte-for-byte equality and successful `JSON_read` parsing.
- [N] Confirm receiver behavior for a deliberately truncated header and body without claiming an unmeasured production consequence.
- [N] Run the focused fixture on Ubuntu Linux, FreeBSD, and macOS with each available controlled trigger; record where signal and nonblocking behavior differs.

## Missing

- [N] A controlled positive-partial `Nwrite` reproduction through `JSON_write`.
- [N] End-to-end peer outcome for partial header and body paths.
- [N] Complete-frame wire-equivalence results.
- [N] Supported-platform trigger results and broader upstream prior-art coverage.
- [N] User selection of Report or Pull request mode and exact target.

## Resume

Index: Force partial JSON writes
Next: Run a controlled JSON control-frame harness that produces post-progress short header and body writes.
Done when: The harness records the positive partial count, baseline false success, candidate `JSON_write == -1`, caller mapping, and complete-frame wire equivalence.

## API and compatibility

Callers [S]: `send_parameters` at `src/iperf_api.c:2531-2534` and `send_results` at `src/iperf_api.c:2964-2967` consume `JSON_write` success/failure.

Contract [S]: `JSON_write` sends one network-order four-byte length followed by exactly that many JSON bytes; `JSON_read` requires both exact counts before parsing (`src/iperf_api.c:3211-3253`).

Compatibility: Every complete control frame remains byte-identical. Only a previously accepted incomplete transmission becomes a caller-visible send failure; no mixed-version format or migration change is introduced.

Migration: None.
