# ISSUE-014 — Nrecv: select timeout has platform-dependent deadline

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

[S] `Nrecv` declares two distinct policies: after partial data it requires continued progress within `nread_read_timeout` seconds, and it maintains an approximate total limit with `ftimeout` plus `nread_overall_timeout` (`src/net.c:75-76,471-490`). The recorded values are 10 seconds per continued-progress wait and approximately 30 seconds overall.

[S] `Nrecv` initializes `struct timeval timeout` once, then reuses it for every `select()` (`src/net.c:422,437,493-500`). On Linux, `select()` mutates that object to the unslept remainder, so later partial-progress waits inherit a shrinking timeout instead of a fresh 10 seconds.

[S] The separate `ftimeout` check remains present, but it cannot restore the consumed per-gap timeout. The defect is reuse of the mutable per-gap `timeval`, not absence of the approximate overall limit.

## Reach and impact

[S] The `Nread` control-frame paths at `src/iperf_api.c:2550,3009,3211-3233` can receive a payload in fragments and inherit `Nrecv`'s timeout behavior.

[O] On Linux, a direct `Nread` socketpair reproduction with bytes arriving at 0/4/8/12 seconds returned only the first three bytes after about 10 seconds.

[S] Each recorded four-second gap is within the intended 10-second continued-progress wait, and the 12-second schedule is within the approximate 30-second total limit. The intended policy therefore accepts all four bytes.

[N] This establishes a control-read deadline inconsistency only; throughput, measurement-data, and deployment-frequency impact were not measured.

## Evidence

[S] Canonical source audit: `upstream/master@c9b74229d0d9bfec6d2307b66b43c29a7665ad0b`; `src/net.c:75-76,419-505`; `src/iperf_api.c:2550,3009,3211-3233`.

[O] Environment: Ubuntu 24.04.4 x64; iperf 3.21+ build from the recorded revision.

[O] Direct `Nread` socketpair probe: peer bytes at 0/4/8/12 seconds; result `rc=3`, `ABC`, `10.002s`.

[S] Linux mutation of the supplied `select()` timeout leaves less than 10 seconds for later in-loop waits. `Nrecv` does not reinitialize that timeout, while its independent `ftimeout` is checked only before each later wait (`src/net.c:422,471-500`).

[A] FreeBSD and macOS results were not observed in this audit. Their `select()` timeout handling must not define this local control-read policy.

## Bug reproduction

Environment: Ubuntu 24.04.4 x64; iperf 3.21+ build from `upstream/master@c9b74229d0d9bfec6d2307b66b43c29a7665ad0b`.

Reproduction: Call `Nread` over a socketpair with `nread_read_timeout=10` while the peer writes successive bytes at 0, 4, 8, and 12 seconds.

Actual [O]: `rc=3`, data `ABC`, elapsed `10.002s`; Linux's cumulatively consumed per-gap timeout expires before the fourth byte arrives.

Expected [S]: `rc=4`, data `ABCD`, near 12 seconds. Every inter-byte gap is under 10 seconds and the complete schedule is under the approximate 30-second total limit, so the fourth byte must be accepted under the documented two-timeout policy.

## API and compatibility

Callers [S]: `src/iperf_api.c:2550,3009,3211-3233` reaches `Nrecv` through the control-read path.

Contract [O]: Linux currently returns the partial three-byte result at `10.002s` in the recorded schedule.

Compatibility: Preserve partial-read/error conventions, control-frame format, mixed-version framing, a fresh 10-second continued-progress wait after each partial read, and the separate approximate 30-second overall control-read limit.

Migration: None.

## Prior art

[S] Worker-failure propagation is separately owned by https://github.com/esnet/iperf/issues/1986, active https://github.com/esnet/iperf/pull/1654, and related https://github.com/esnet/iperf/pull/1861. That work is adjacent lifecycle context, not prior art for `Nrecv` deadline ownership.

[A] No verified external issue, pull request, or comment owns this Linux/FreeBSD timeout-semantic divergence. Target remains `Undecided`.

## Direction

[S] Reinitialize the per-gap `struct timeval` from `nread_read_timeout` immediately before every `Nrecv` `select()`. Retain the existing separately monotonic `ftimeout` initialization and pre-wait check after the first partial read as the approximate `nread_overall_timeout` limit.

[S] Do not redefine the behavior as one cumulative 10-second deadline, and do not rearm a fresh 30-second overall timeout for each fragment. The correction is a fresh per-gap wait plus the existing separate approximate overall bound.

[A] Preserve the current partial-result and interruption behavior after tracing all `Nrecv` exits; the record does not authorize a broader I/O retry-policy change.

## Bounds

[S] Preserve the 10-second continued-progress wait and approximate 30-second total bound at the listed control-read call sites, along with existing byte-order and frame handling.

[S] Support Ubuntu Linux, FreeBSD, and macOS without deriving correctness from `select()` timeout side effects. The local scheduling fix must leave mixed-version control framing and accepted payload bytes unchanged.

[N] No performance claim is made. No source change, external draft, mode selection, or target selection is authorized by this record.

## Verification

Test decision: none. No tests or gates were run for this tracker-only update.

[S] A focused future socketpair check on Ubuntu Linux, FreeBSD, and macOS must schedule bytes at 0/4/8/12 seconds with `nread_read_timeout=10` and assert `rc=4`, `ABCD`, near 12 seconds. A second incomplete-frame schedule with progress gaps under 10 seconds must show the independently retained overall limit ends the read at approximately 30 seconds rather than being rearmed for every fragment.

## Missing

Observed FreeBSD and macOS schedule results; all `Nrecv` partial-read/interruption exit contracts; direct prior-art search for this exact timeout-policy defect; and a candidate-change result proving both the 12-second four-byte schedule and the approximate 30-second overall bound.

## Resume

Index: Verify per-gap timeout policy

Next: Run the 0/4/8/12-second socketpair schedule on FreeBSD and macOS, then trace every `Nrecv` exit against the separate per-gap and overall timeout comments.

Done when: Supported-platform evidence distinguishes timeout-mutation effects from the intended fresh 10-second progress wait and approximate 30-second overall bound.
