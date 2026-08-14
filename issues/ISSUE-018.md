# ISSUE-018 — control receive: returning `EINTR` can yield a short count

State: Hold
Mode: Undecided
Target: Undecided
Location: Not published.
Priority: Low
Confidence: High
Type: reliability
Created: 2026-08-14
Updated: 2026-08-14
Source: `upstream/master@c9b74229d0d9bfec6d2307b66b43c29a7665ad0b`

## Root

Root [S]: `Nread()` delegates to `Nrecv()` in `src/net.c:400-415`. After readiness, `Nrecv()` breaks when `read()`/`recv()` reports `EINTR`, `EAGAIN`, or `EWOULDBLOCK` and returns `count - nleft` (`src/net.c:446-459`, `503-504`). A read-side `EINTR` after partial progress therefore has the generic helper's positive-short result. By contrast, an interrupted initial or post-progress `select()` returns `NET_HARDERROR` (`src/net.c:434-444`, `493-501`), so the result depends on whether the signal reaches the wait or read phase.

## Reach and impact

Reach [A]: This path requires an embedding with a returning, non-`SA_RESTART` signal handler whose delivery makes the underlying `read()` or `recv()`—not its preceding `select()`—return `EINTR` while a control frame is in progress. Handler disposition, signal targeting, timing, and the resulting `errno` path remain unexecuted.
Impact [S]: `JSON_read()` uses exact `Nread()` counts for its four-byte length and payload and parses only when `rc == hsize` (`src/iperf_api.c:3211-3253`). A short JSON control read is rejected rather than parsed as JSON.
Impact [N]: No embedded returning-handler reproduction, end-to-end exchange outcome, affected-integration count, or user impact is measured.

## Evidence

- [S] `src/net.c:419-505` — after readiness, a read-side `EINTR`, `EAGAIN`, or `EWOULDBLOCK` breaks the loop and the helper returns the accumulated count; an interrupted `select()` returns `NET_HARDERROR`.
- [S] `src/iperf_api.c:3211-3253` — `JSON_read()` requires an exact length before decoding and parses a payload only when its returned count equals `hsize`; a short payload warns and is rejected.
- [O] Ubuntu 24.04.4 x64, iperf 3.21+ built from the recorded revision: direct `Nread` over a socketpair with bytes at `0/4/8/12` seconds returned `rc=3`, `ABC`, `10.002s`. This is a timeout-path observation separately owned by ISSUE-014, not an `EINTR` reproduction and not evidence of this record's reach or impact.

## Prior art

Coverage [O]: Directly reviewed `https://github.com/esnet/iperf/issues/1986`, active `https://github.com/esnet/iperf/pull/1654`, and related `https://github.com/esnet/iperf/pull/1861` on 2026-08-14. They own worker-failure propagation, a distinct root from read-side interruption handling. [N] Broader current external search and an embedded returning-handler reproduction remain absent.

## Direction

Direction [S]: Preserve `Nrecv`'s generic short-count contract and `JSON_read()`'s fixed-frame exact-count check. Source inspection alone does not justify changing generic `Nrecv()` retry semantics. If an embedded returning-handler harness establishes a required different outcome, assign that policy to the affected fixed-frame caller or an explicit API boundary after reviewing every `Nread` caller.

Rejected [S]: Accepting or parsing a short JSON control frame is wrong because `JSON_read()` already makes exact count the parsing boundary. Retrying `EINTR` in generic `Nrecv()` would change the shared helper's behavior for every caller. [A] Requiring `SA_RESTART` or swallowing signals is not a correction: signal disposition belongs to the embedding and the required application policy is unverified.

## Bounds

- Preserve [S]: `Nrecv`'s generic EOF, timeout, `EAGAIN`, `EWOULDBLOCK`, and partial-count behavior; the current `select()`-interruption hard-error result; and `JSON_read()`'s four-byte-length then exact-payload framing.
- Preserve [S]: Wire fields and JSON parsing eligibility. Do not change timeout constants or signal disposition without a separately owned contract.
- Exclude [S]: A generic I/O retry-policy rewrite, accepting incomplete JSON, signal-handler installation, and unmeasured claims about embedded user impact.

## Verification

Test decision: none. No gates run. Planned focused harnesses, not an existing-test update:

- [N] Use an embedded control fixture with a returning non-`SA_RESTART` handler timed during header and payload reads; separately force delivery during the preceding readiness wait and during `read()`/`recv()`.
- [N] Record signal disposition, delivery phase, `Nread` return count, `errno`, JSON caller outcome, and elapsed timeout behavior.
- [N] Prove incomplete JSON remains unparsed and compare every affected generic `Nread` caller before selecting any interruption-policy change.

## Missing

- [N] An observed returning-handler `EINTR` path during a control-frame read, including header and payload behavior.
- [N] Measured embedded-integration reach and end-to-end outcome.
- [N] Broader current prior-art coverage and any target that owns this root.
- [N] A user-selected Mode and Target after the above evidence exists.

## Resume

Index: Reproduce returning signal control read
Next: Reproduce a control-frame `Nread()` interruption in an embedded returning-handler harness.
Done when: The harness records header and payload return values, delivery phase, JSON caller outcome, signal disposition, and timeout result.

## API and compatibility

Callers [S]: `JSON_read()` uses `Nread()` for parameter and result frames (`src/iperf_api.c:2550`, `3009`, `3211-3253`); other callers also consume `Nread()`'s generic return values.

Contract [S]: `Nrecv()` returns accumulated bytes after its read-side soft exits. `JSON_read()` separately owns fixed-frame completeness by parsing only exact header and payload counts.

Compatibility: No wire or API change is selected. Any future correction must retain generic partial-count behavior unless a specifically affected public caller receives an explicit, verified contract.

Migration: None.
