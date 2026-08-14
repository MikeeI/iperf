# ISSUE-002 — server output: result packaging repeatedly rescans accumulated text

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

Root [S]: Textual `--get-server-output` packaging first computes the exact total length, then appends every captured line with `strncat`; each append rescans the growing destination, making assembly superlinear in captured output instead of copying each known line once.

## Reach and impact

Reach [S]: Server-side `iperf_printf` captures one allocation per rendered line when the client requests server output. At `TEST_END`, the server calls `iperf_exchange_results`, whose `send_results` path concatenates every captured text line before serializing the control JSON.
Impact [N]: Packaging work is Θ(B·L) for B total bytes and L lines, worst-case Θ(B²) for similarly sized lines. End-to-end completion delay and the report size at which this becomes user-visible are not measured; ordinary short tests may see negligible cost.

## Evidence

- [S] `src/iperf_api.c:5714-5717` — server `iperf_printf` duplicates each line and appends it to `server_output_list` when server output is requested.
- [S] `src/iperf_api.c:2908-2912` — `send_results` performs a first pass that already knows the exact aggregate byte count.
- [S] `src/iperf_api.c:2914-2919` — the second pass calls `strncat` for each line and recomputes each line length; `strncat` must locate the current destination terminator before appending.
- [S] `src/iperf_api.c:2921-2922` — the assembled text is copied into cJSON before the temporary buffer is freed.
- [S] `src/iperf_api.c:2403-2419` — server result exchange reaches `send_results` once per completed test.
- [S] `src/iperf_server_api.c:267-281` — `TEST_END` runs final statistics/reporting and result exchange.
- [S] `src/iperf_api.c:4142-4144` — interval reporting emits per-stream lines, so line count scales with intervals and streams.
- [N] No baseline timing, CPU profile, or report-size curve exists yet.

## Prior art

Coverage: GitHub issues(open+closed), PRs(open+closed+merged), and commit search; checked=2026-08-14.
Gaps: GitHub Discussions, `iperf-dev` archives, releases, and current unpublished branches remain unchecked.

- `https://github.com/esnet/iperf/pull/1863` — Related; proposed replacing `strncat` with `strlcat` for safety. Maintainers declined the platform-specific dependency. It did not remove repeated destination scans or present a performance curve.
- `https://github.com/esnet/iperf/issues/1954` — Distinct; asks for client output at the server, the opposite data direction.
- `https://github.com/esnet/iperf/pull/1927` — Related; documents when `--get-server-output` becomes available, but does not change packaging complexity.
- `https://github.com/esnet/iperf/pull/1779` — Related; bounds received server-output JSON size, not local textual assembly cost.

Target fit: Undecided — a portable cursor-based correction could be a focused pull request, but representative measurements and complete prior-art review are still required.

## Direction

Keep the existing exact-size pass. In the second pass, maintain a destination cursor, compute each source length once, copy with `memcpy`, advance the cursor, and terminate once. Use only standard C/POSIX facilities already accepted by the project; do not introduce `strlcat`, a string-builder abstraction, or a dependency.

## Bounds

- Preserve: Captured line order, every byte and newline, terminating NUL, cJSON field name and escaping, allocation-failure behavior, server/client result state, and protocol compatibility.
- Exclude: Changing line capture, streaming server output during the test, bounding requested output, replacing cJSON, or rewriting reporting.
- Cost: Small pointer/length bookkeeping in one packaging function; review must verify overflow handling and no write beyond the precomputed allocation.

## Verification

- Generate controlled captured reports with fixed line size and increasing line counts; measure `send_results` CPU and completion latency.
- Fit baseline and candidate time against total bytes and line count; require the candidate to scale linearly within measurement variance.
- Compare `server_output_text` byte-for-byte for empty, one-line, multi-line, long, and maximum representative reports.
- Exercise allocation failure where practical and normal TCP, UDP, reverse, bidirectional, and parallel-stream result exchange.
- Run `make check` and `test_commands.sh` after focused result-exchange scenarios pass.

## Missing

- [N] Representative baseline curve showing when repeated scans become material.
- [N] Candidate curve and byte-equivalence guard.
- [N] GitHub Discussions, mailing-list, releases, and active-branch prior-art coverage.
- [N] User selection of Report or Pull request mode and exact target.

## Resume

Index: Measure output assembly scaling
Next: Build a deterministic captured-output benchmark that varies line count at fixed line size and records packaging CPU and terminal latency.
Done when: Baseline measurements establish the complexity curve, variance, and smallest materially affected workload.

## Performance evidence

Workload: Planned server result exchange with textual `--get-server-output`; fixed-size lines; increasing intervals and stream counts; Linux localhost control connection.
Baseline [N]: Not measured.
Candidate [N]: Not implemented or measured.
Guard [N]: Exact returned-text equivalence not tested.
Boundary [N]: Superlinear source structure is proven; representative end-to-end impact remains unmeasured.

## API and compatibility

Callers [S]: Server `iperf_exchange_results` and clients requesting textual server output.
Contract [S]: The control payload must contain the complete captured server text in original order; PR #1927 confirms availability only after test completion.
Compatibility: Preserve payload field, bytes, timing boundary, errors, and mixed-version interoperability.
Migration: None.
