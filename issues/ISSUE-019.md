# ISSUE-019 — result JSON: receiver length is uncapped

State: Hold
Mode: Undecided
Target: Undecided
Location: Not published.
Priority: Low
Confidence: Medium
Type: reliability
Created: 2026-08-14
Updated: 2026-08-14
Source: `upstream/master@c9b74229d0d9bfec6d2307b66b43c29a7665ad0b`

## Root

[S] `get_results()` calls `JSON_read(test->ctrl_sck, 0)` (`src/iperf_api.c:2976-3012`). In `JSON_read()`, a positive `uint32_t` network length passes the optional maximum whenever `max_size == 0`; `hsize + 1` is then used as the allocation size before parsing (`src/iperf_api.c:3196-3229`). Result frames therefore have no receiver-selected size limit below the protocol's 32-bit offered length, except that an overflowing `hsize + 1` yields no allocation.

## Reach and impact

[S] A peer-controlled result length can request allocation before JSON validation; `calloc()` failure rejects it, but a successful allocation can consume the advertised amount. [A] Hostile-peer availability depends on deployment trust and authentication. [N] No controlled oversized-result run, allocation measurement, or production impact is recorded.

[S] Unbounded results are intentional for `--get-server-output`: on the server, `test->get_server_output` accumulates every output line and adds the complete text to the result JSON (`src/iperf_api.c:2898-2923`); the documented option gets server output (`src/iperf3.1:499-503`). [S] The offered length is 32-bit, and `Nread()` bounds readiness/progress by `nread_read_timeout` and `nread_overall_timeout` (`src/net.c:419-505`), but neither bound makes a large permitted allocation cheap or establishes measured harm. Priority remains Low; confidence is Medium because hostile reach and practical frame sizes are unmeasured.

## Evidence

- [S] `src/iperf_api.c:2976-3012` selects `max_size == 0` only for results; parameter receipt uses `JSON_read(test->ctrl_sck, MAX_PARAMS_JSON_STRING)` (`src/iperf_api.c:2549-2553`).
- [S] `src/iperf_api.c:3196-3253` decodes a four-byte length, applies the optional maximum, allocates, reads exactly that length, and parses only an exact read.
- [S] `src/iperf_api.c:2898-2923` serializes the whole requested server-output list into the result JSON rather than imposing a local output cap.
- [O] Ubuntu 24.04.4 x64, iperf 3.21+ built from the recorded revision: a 9000-character `--title` produced a 9135-byte parameter frame; the server rejected it with `warning: JSON data length overflow - 9135 bytes JSON size is not allowed`, followed by client/server failure. This demonstrates the parameter receiver cap, not oversized-result behavior.
- [O] Historical repository baseline: `make check` returned `5/5 pass`; it does not exercise an oversized result frame or establish this resource boundary.

## Prior art

[O] Closed https://github.com/esnet/iperf/issues/1776 reported the 8-KiB `--get-server-output` JSON receiver failure. Merged https://github.com/esnet/iperf/pull/1779 deliberately removed that result cap to preserve complete large server output; it is the owning compatibility precedent, not a bounded-result design. Merged https://github.com/esnet/iperf/pull/1709 introduced the 8-KiB parameter receive cap; sender enforcement remains absent. Classification: relevant parameter-framing precedent, but it does not cap `get_results()` because that caller deliberately passes zero. [N] No audited prior art establishes a compatible result-size bound, negotiated result limit, or result-streaming design.

## Direction

No ready fix: an arbitrary result receiver cap is invalid without protocol negotiation or streaming because it can reject intentional complete `--get-server-output` results from compatible peers. [A] A viable correction must first define an advertised/negotiated result maximum or versioned chunked/streamed results, including failure behavior when peers lack it; only then can allocation policy be bounded safely.

Rejected: copy `MAX_PARAMS_JSON_STRING` into `get_results()`. The parameter cap's purpose and successful-result contract differ, and a unilateral cap breaks existing large server output. Rejected: claim the 32-bit field or read timeouts alone resolves allocation exposure; both still admit a large allocation request.

## Bounds

[S] Preserve the current four-byte network length, exact-payload check, JSON result fields, and the complete `--get-server-output` behavior restored by https://github.com/esnet/iperf/pull/1779 across mixed iperf3 versions unless an intentional protocol change is approved. [S] Do not impose a sender-only or receiver-only arbitrary maximum that produces asymmetric success/failure. Excluded scope: parameter-cap sender enforcement; it is distinct from this result-receive root.

## API and compatibility

Callers [S]: `get_results()` is the in-tree result receiver; `JSON_read()` also serves parameter receipt with `MAX_PARAMS_JSON_STRING` (`src/iperf_api.c:2549-2553`, `2976-3012`).

Contract [S]: results use a 32-bit offered length; when `max_size` is zero, its optional maximum does not reject a positive value before allocation, and requested server output is embedded in that result JSON.

Compatibility: preserve complete results, including `--get-server-output`, for same- and mixed-version peers; https://github.com/esnet/iperf/pull/1779 establishes that a unilateral receiver cap is incompatible. A changed framing or limit needs explicit peer agreement.

Migration: Requires protocol negotiation or a versioned streamed/chunked result path; none is designed or authorized.

## Verification

Test decision: none; no gates ran for this ledger edit. Any later direction needs a controlled peer declaring boundary-sized and oversized result frames, a representative large `--get-server-output` exchange, allocation/failure observations, and same- and mixed-version behavior for peers without the new agreement.

## Missing

[N] No representative result-frame-size distribution, hostile-peer allocation measurement, negotiated/streaming design, compatibility experiment beyond https://github.com/esnet/iperf/pull/1779, or current external target.

## Resume

Index: Measure result-frame allocation reach
Next: Measure result sizes and allocation outcomes for controlled peer lengths and representative `--get-server-output` runs.
Done when: Recorded frame sizes, allocation/failure behavior, deployment trust boundary, and compatibility requirements can accept or reject a negotiated/streaming direction.
