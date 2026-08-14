# ISSUE-004 — interval state: sole retained result node is reallocated every callback

State: Hold
Mode: Undecided
Target: Undecided
Location: Not published.
Priority: Medium
Confidence: High
Type: performance
Created: 2026-08-14
Updated: 2026-08-14
Source: `upstream/master@c9b74229d0d9bfec6d2307b66b43c29a7665ad0b`

## Root

Root [S]: `add_to_interval_list` intentionally retains only the latest interval result, yet every callback after the first removes and frees the sole node, allocates a replacement, copies the payload, and inserts it again instead of reusing the already owned storage.

## Reach and impact

Reach [S]: Client and server statistics timers invoke `iperf_stats_callback`; it traverses every stream and calls `add_to_interval_list` once. At the configured limits of 128 streams and a 0.1-second interval, steady state performs up to 1,280 frees and 1,280 mallocs per second across each process running that workload.
Impact [N]: Allocator calls, cache effects, reporter CPU share, latency, and throughput impact are not measured. Memory cardinality is already bounded to one node per stream; this finding concerns avoidable churn, not growth.

## Evidence

- [S] `src/iperf_api.c:3299-3305` — the function documents one retained result, then removes and frees the existing tail node.
- [S] `src/iperf_api.c:3307-3309` — every call allocates, copies, and inserts a new node; allocation failure is not checked.
- [S] `src/iperf_api.c:3896-3997` — `iperf_stats_callback` invokes the function once for every stream on each statistics tick.
- [S] `src/iperf_client_api.c:190-198,237-245` — client periodic stats timer reaches the callback.
- [S] `src/iperf_server_api.c:352-360,398-406` — server periodic stats timer reaches the callback.
- [S] `src/iperf.h:94-126` — the payload contains a `TAILQ_ENTRY`; overwriting an inserted node wholesale would corrupt queue links.
- [S] `src/iperf_api.c:4938-4957` — stream teardown owns and frees retained interval nodes.
- [S] `src/iperf.h:470-476` — allowed pressure is 0.1-second intervals and 128 streams.
- [N] No allocator profile or callback timing exists yet.

## Prior art

Coverage: GitHub issues(open+closed), PRs(open+closed+merged), and commit search for `add_to_interval_list`, `interval_results`, malloc, reuse, and performance; checked=2026-08-14.
Gaps: GitHub Discussions, `iperf-dev` archives, releases, allocator-specific reports, and current unpublished branches remain unchecked.

- No direct upstream issue, PR, or commit candidate was found with the recorded searches.

Target fit: Undecided — a focused pull request is plausible if allocator and CPU measurements show a stable benefit without queue or output regressions.

## Direction

Allocate only when the queue is empty. Otherwise remove the existing sole node, copy the new payload while it is detached, and insert the same node again. Preserve `iperf_free_stream` as the only terminal owner. Add explicit allocation-failure handling consistent with the surrounding callback contract only if that error contract can be defined without broadening scope.

## Bounds

- Preserve: Exactly one latest interval per stream, interval values and timestamps, text/JSON/JSON-stream output, `TAILQ` integrity, callback ordering, stream cleanup, OOM behavior, and custom-data semantics.
- Exclude: Retaining interval history, embedding the interval struct directly into another object, changing callback APIs, pooling across streams, or changing statistics cadence.
- Cost: Detach-copy-reinsert is required because direct `memcpy` over an inserted node would overwrite live `TAILQ_ENTRY` links.

## Verification

- Count malloc/free calls attributed to `add_to_interval_list` for 1, 8, and 128 streams at 1.0 and 0.1 second intervals.
- Measure stats-callback CPU and end-to-end throughput with identical builds and workloads; report variance.
- Compare every emitted interval object and text line before and after the change for TCP, UDP, reverse, bidirectional, JSON, and JSON streaming.
- Exercise first interval, repeated intervals, omit reset, final interval, stream cleanup, and a forced allocation failure on first creation.
- Run ASan/UBSan focused scenarios, `make check`, and `test_commands.sh` after behavioral guards pass.

## Missing

- [N] Baseline allocator-call counts, callback CPU, throughput, and variance.
- [N] Candidate measurements and exact interval-output equivalence.
- [A] Intended OOM behavior of `add_to_interval_list`; current unchecked malloc may be a separate correctness concern.
- [N] GitHub Discussions, mailing-list, releases, and active-branch prior-art coverage.
- [N] User selection of Report or Pull request mode and exact target.

## Resume

Index: Profile interval allocator churn
Next: Attribute allocator calls and callback CPU to `add_to_interval_list` across stream-count and interval-frequency boundaries.
Done when: Baseline measurements establish whether reuse produces a material, repeatable improvement and document current first-allocation behavior.

## Performance evidence

Workload: Planned TCP and UDP localhost matrix with 1, 8, and 128 streams and 1.0/0.1-second stats intervals; text and JSON reporters separated.
Baseline [N]: Not measured.
Candidate [N]: Not implemented or measured.
Guard [N]: Interval-value, queue-integrity, cleanup, and output equivalence not tested.
Boundary [N]: The allocator-operation upper bound is source-derived; real CPU and throughput value remain unmeasured.
