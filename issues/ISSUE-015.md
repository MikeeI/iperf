# ISSUE-015 — control errors: short error words are decoded

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

Root [S]: In the client `SERVER_ERROR` branch, `src/iperf_client_api.c:392-407` treats only negative `Nread` returns as a control-read failure. `Nread` delegates to `Nrecv`, which deliberately returns `count - nleft`, including positive short counts, after EOF, timeout, `EINTR`, `EAGAIN`, or `EWOULDBLOCK` (`src/net.c:405-505`). The branch therefore passes an incomplete four-byte network-order error word to `ntohl` and assigns its partly stale/uninitialized value to global `i_errno`.

## Reach and impact

Reach [S]: This is reached when a client receives the `SERVER_ERROR` control state and either of its two following four-byte words is short; both reads at `src/iperf_client_api.c:393` and `src/iperf_client_api.c:398` have the same `< 0` predicate.
Impact [O]: An incomplete first word can be reported as a varying unknown `i_errno` instead of the control-read failure; identical three-byte payloads produced codes `408`, `289`, and `468` in three observed runs. A short second word can likewise be decoded as `errno`, producing a misleading server-error diagnostic.
Impact [N]: Frequency on ordinary blocking TCP control connections is not measured. The failure needs a peer close, timeout, or other short-read condition after control-state receipt.

## Evidence

- [S] `src/iperf_client_api.c:392-407` — each `SERVER_ERROR` word is decoded with `ntohl` after `Nread(...) < 0`, not after an exact four-byte result.
- [S] `src/iperf_client_api.c:302-307` declares automatic `int32_t err` without initialization, so a short first read leaves part of the decoded object indeterminate.
- [S] `src/net.c:415-505` — `Nrecv` returns `count - nleft` after EOF or a read-side `EINTR`, `EAGAIN`, or `EWOULDBLOCK`; it also returns that count when a readiness or overall timeout ends the loop. The result can therefore be a positive short count.
- [S] `src/iperf_server_api.c:466-471` — the server attempts to send `i_errno` and `errno` as two network-order 32-bit words after `SERVER_ERROR`; a complete frame requires both words.
- [O] Direct `Nread` socketpair probe with bytes at `0/4/8/12` seconds returned `rc=3`, `ABC`, `10.002s`. This timeout-path observation is separately owned by ISSUE-014; it demonstrates that `Nread` can return a positive short count, not a `SERVER_ERROR` reproduction.
- [O] The focused `SERVER_ERROR` control harness delivered the identical truncated three-byte error word in three runs; decoded unknown `i_errno` values were `408`, `289`, and `468`.
- [O] Environment: Ubuntu 24.04.4 x64; iperf 3.21+ build from `upstream/master@c9b74229d0d9bfec6d2307b66b43c29a7665ad0b`.

## Prior art

Coverage [O]: Directly reviewed `https://github.com/esnet/iperf/issues/1986`, active `https://github.com/esnet/iperf/pull/1654`, related active `https://github.com/esnet/iperf/pull/1861`, and merged `https://github.com/esnet/iperf/pull/1709` on 2026-08-14.
Gaps [N]: GitHub-wide symbol/history search, GitHub Discussions, `iperf-dev` archives, releases, and unpublished active branches remain unchecked.

- [S] `https://github.com/esnet/iperf/issues/1986`, `https://github.com/esnet/iperf/pull/1654`, and `https://github.com/esnet/iperf/pull/1861` own worker-thread failure detection/reporting, not short fixed-width `SERVER_ERROR` word decoding; they are related failure-reporting context, not a duplicate or target.
- [S] Merged `https://github.com/esnet/iperf/pull/1709` added exact-size validation to JSON receive framing, but does not cover the two `SERVER_ERROR` reads; it is relevant prior correction style, not a direct owner.
- [N] No direct upstream issue, PR, or commit for these two predicates is recorded yet.

Target fit [N]: Undecided until broader prior-art coverage and an exact-count candidate verify that no active change owns this correction.

## Direction

Direction [S]: At `src/iperf_client_api.c:393` and `src/iperf_client_api.c:398`, require `Nread(...) == sizeof(err)` before either `ntohl`. Map every other result—negative, zero, or positive short—to existing `IECTRLREAD`. Keep `Nread`'s partial-count contract intact because the fixed-width control-frame owner, not the generic transport helper, owns completeness.
Rejected [S]: Changing `Nread`/`Nrecv` to erase positive short counts would alter the generic transport contract for unrelated callers; fixed-width control decoding is the narrower owner.

## Bounds

- Preserve [S]: Complete `SERVER_ERROR` state handling; the two network-order four-byte words; complete-frame `i_errno`, `errno`, and diagnostic text; TCP control flow; and mixed-version iperf3 interoperability.
- Change [S]: Incomplete `SERVER_ERROR` words fail as `IECTRLREAD` rather than being decoded as server-provided values.
- Exclude [S]: Changing `Nread`/`Nrecv` return semantics, retrying or buffering incomplete words, reformatting error strings, worker-thread failure handling, JSON framing, and unrelated control states.
- Cost [N]: The complete-frame and truncated-frame behavior matrix has not yet been run against a candidate on Linux, FreeBSD, and macOS.

## Verification

Test decision: none. Planned focused harnesses, not an existing-test update:

- [N] Deliver a complete `SERVER_ERROR` frame and assert unchanged decoded `i_errno`, `errno`, and error text.
- [N] Truncate each word independently at `0`, `1`, `2`, and `3` bytes, including the observed three-byte first-word case; assert client failure and `i_errno == IECTRLREAD` without any `ntohl`-derived code.
- [N] Exercise EOF, per-read timeout, overall timeout, `EINTR`, `EAGAIN`, and `EWOULDBLOCK` exits where the harness can control them; record result counts and diagnostics.
- [N] Run the focused control-channel harness on Ubuntu Linux, FreeBSD, and macOS to distinguish portable framing behavior from platform timing.

## Missing

- [N] Exact-count candidate results for complete and independently truncated first/second words.
- [N] Supported-platform harness results and failure timing.
- [N] Broader upstream prior-art coverage.
- [N] User selection of Report or Pull request mode and exact target.

## Resume

Index: Verify short error rejection
Next: Run an exact-count candidate against complete and independently truncated `SERVER_ERROR` words.
Done when: Complete frames retain their decoded values and every short word produces `IECTRLREAD` in recorded harness output.

## Bug reproduction

Environment: Ubuntu 24.04.4 x64; iperf 3.21+ build from `upstream/master@c9b74229d0d9bfec6d2307b66b43c29a7665ad0b`.

Reproduction: Feed the client control handler `SERVER_ERROR`, then the same three-byte prefix of its first required four-byte error word and no fourth byte; repeat with the identical payload.

Actual [O]: The partial word was decoded as unknown `i_errno` values `408`, `289`, and `468` across three otherwise identical runs.

Expected [S]: A `SERVER_ERROR` error word is exactly four bytes. Any other `Nread` result must fail the control read as `IECTRLREAD` and must not be decoded.

## API and compatibility

Callers [S]: The client `SERVER_ERROR` control-state receiver in `src/iperf_client_api.c:392-407`; server cleanup attempts the paired words at `src/iperf_server_api.c:466-471`.

Contract [S]: After `SERVER_ERROR`, the client expects an iperf error and a system error as two network-order four-byte words. It may decode either word only when its `Nread` result is exactly four bytes.

Compatibility: Complete control frames retain identical wire bytes and decoded behavior. Incomplete frames deliberately change from accidental value decoding to the established control-read failure.

Migration: None.
