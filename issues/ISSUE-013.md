# ISSUE-013 — parameter frames: sender omits 8-KiB limit

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

[S] `MAX_PARAMS_JSON_STRING` defines the 8-KiB parameter-frame receive limit (`src/iperf.h:460`). `get_parameters()` passes that limit to `JSON_read()`, whose `hsize <= max_size` predicate rejects a larger parameter payload (`src/iperf_api.c:2543-2553,3196-3253`).

[S] `send_parameters()` builds final JSON and delegates its length-prefixed transmission to generic `JSON_write()` without applying that predicate first (`src/iperf_api.c:2425-2537,3170-3191`).

[S] This is an asymmetric parameter-frame boundary: a client can transmit a final encoded parameter frame that the peer is required to reject.

## Reach and impact

[S] Any client-controlled combination of serialized parameter fields can exceed the receiver's 8-KiB frame cap; `--title` is a reproduced input, not the only contributor, and its option parser imposes no encoded-size bound (`src/iperf_api.c:1676-1679`).

[O] A 9000-character `--title` produced a 9135-byte parameter frame rejected by the server, so one ordinary CLI option can make both endpoints fail before a measurement starts.

[N] Prevalence across deployments and the number of other reachable oversized parameter combinations were not measured.

## Evidence

[S] Canonical source audit: `upstream/master@c9b74229d0d9bfec6d2307b66b43c29a7665ad0b`; `src/iperf.h:460`; `src/iperf_api.c:2425-2537,2543-2553,3170-3253`.

[S] `JSON_read()` accepts a nonzero frame only when `max_size == 0 || hsize <= max_size`; for parameter frames, `max_size` is `MAX_PARAMS_JSON_STRING` (`src/iperf_api.c:2550,3211-3247`).

[O] Environment: Ubuntu 24.04.4 x64; iperf 3.21+ build from the recorded revision.

[O] With a 9000-character `--title`, the encoded parameter frame was 9135 bytes. Server result: `warning: JSON data length overflow - 9135 bytes JSON size is not allowed`; client and server then failed.

[S] Merged https://github.com/esnet/iperf/pull/1709 introduced the receiver cap. The audited sender path still has no matching final-encoded-size check.

## Bug reproduction

Environment: Ubuntu 24.04.4 x64; iperf 3.21+ build from `upstream/master@c9b74229d0d9bfec6d2307b66b43c29a7665ad0b`.

Reproduction: Start an iperf server. Run a client with `--title` set to 9000 characters and capture the encoded parameter-frame length and server output.

Actual [O]: The client sends a 9135-byte parameter frame; the server emits `warning: JSON data length overflow - 9135 bytes JSON size is not allowed`; both sides fail.

Expected: The sender validates the final encoded parameter-frame byte size against the receiver's existing acceptance predicate before sending, rejects an oversized frame locally, and does not transmit a frame the peer will reject.

## API and compatibility

Callers [S]: `send_parameters()` sends client parameters through `JSON_write()` (`src/iperf_api.c:2425-2537`); `get_parameters()` receives them through `JSON_read(..., MAX_PARAMS_JSON_STRING)` (`src/iperf_api.c:2543-2553`).

Contract [S]: The parameter receiver accepts only the exact `JSON_read()` predicate, including its equality boundary. `JSON_write()` is also used for results (`src/iperf_api.c:2964`), so its generic behavior is not the parameter-limit owner.

Compatibility: Preserve the current receiver limit and mixed-version negotiation. Validate final serialized bytes, not title character count or an estimated JSON size: UTF-8 encoding, escaping, and other parameter fields change the wire size.

Migration: None. A new sender rejects an already-invalid parameter payload locally; an older sender remains subject to the existing peer check, and accepted frames remain wire-identical.

## Prior art

[S] Merged https://github.com/esnet/iperf/pull/1709 owns the origin of the receiver cap, not sender enforcement. Its history rules out the inflated claim that the cap itself is newly discovered.

[A] No verified active external work owns the remaining sender/receiver asymmetry. Target remains `Undecided`; do not redirect to #1709 without proving that it still owns this missing enforcement.

## Direction

[S] In `send_parameters()`, validate the final encoded parameter JSON with the receiver's exact existing acceptance predicate before its length/payload write; reject the oversize payload locally without transmitting it.

[S] Do not raise the receiver cap or add a parameter cap to generic `JSON_write()`. That function also sends result JSON, so a generic cap would change non-parameter behavior, including the intentional large-`--get-server-output` compatibility pressure owned separately by ISSUE-019.

[A] Reuse the established local error mapping if one fits the sender boundary; otherwise choose the client-facing error contract only after tracing existing parameter-serialization failures.

## Bounds

[S] Keep the 8-KiB receiver boundary, frame format, field set, and peer negotiation behavior unchanged.

[S] Cover every field contributing to final parameter JSON rather than special-casing `--title`; leave result-frame sizing and `--get-server-output` behavior outside this finding.

[N] The cap's suitability for larger future metadata was not evaluated and is excluded from this finding.

## Verification

Test decision: none. No tests or gates were run for this tracker-only update.

[S] A focused future check must prove that a frame at the receiver's exact equality boundary still sends, while the same predicate rejects an oversized final encoding locally before any parameter frame reaches the server. Repeat the 9000-character `--title` case and preserve the receiver boundary exactly.

## Missing

The sender's exact existing error/`i_errno` contract; receiver equality-boundary behavior; current external search for sender enforcement; and an observed local-rejection result after a candidate change.

## Resume

Index: Validate encoded parameter size
Next: Trace sender serialization and existing parameter-send errors to select the exact receiver predicate and local error mapping.
Done when: A bounded sender-side check is specified that rejects the 9135-byte frame before transmission without changing receiver acceptance.
