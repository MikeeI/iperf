# ISSUE-005 — JSON reporting: unused human-readable units are formatted every interval

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

Root [S]: Interval stream and sum reporting calls `unit_snprintf` to build human-readable byte and bandwidth strings before branching between JSON and text output; JSON consumes only raw numeric values, so the formatted strings are dead work in JSON modes.

## Reach and impact

Reach [S]: Every accepted reporting interval formats two unit strings for each reported stream and another pair for each sum. With 128 streams at 0.1-second intervals, the stream path alone can execute 2,560 unnecessary `unit_snprintf` calls per second, plus sum formatting, on both client and server reporters using JSON.
Impact [N]: Reporter CPU, total process CPU, throughput, and end-to-end latency are not measured. Cost should be most visible with high stream count and short report intervals; normal one-stream, one-second tests may remain below noise.

## Evidence

- [S] `src/iperf_api.c:4881-4888` — `print_interval_results` computes `ubuf` and `nbuf` before choosing JSON or text output.
- [S] `src/iperf_api.c:4895-4928` — JSON branches serialize raw bytes, duration, and `bandwidth * 8`; only text branches consume the formatted buffers.
- [S] `src/iperf_api.c:4188-4190` — interval sum reporting similarly formats both buffers before its output branch.
- [S] `src/iperf_api.c:4196-4228` — JSON sums use raw numbers; text sums consume `ubuf` and `nbuf`.
- [S] `src/units.c:291-372` — `unit_snprintf` performs case checks, adaptive scaling, floating conversion, branch selection, and `snprintf`.
- [S] `src/iperf_api.c:2025-2028` — argument validation explicitly warns that `-f` report formatting is ignored with JSON output.
- [S] `src/iperf_api.c:4142-4144` and `src/iperf.h:470-476` — reporting traverses streams with up to 128 streams and 0.1-second intervals.
- [N] No reporter profile or before/after measurement exists yet.

## Prior art

Coverage: GitHub issues(open+closed), PRs(open+closed+merged), and commit search for `unit_snprintf`, JSON output, interval formatting, and performance; checked=2026-08-14.
Gaps: GitHub Discussions, `iperf-dev` archives, releases, and current unpublished branches remain unchecked.

- `https://github.com/esnet/iperf/pull/1098` — Related; defines JSON streaming and interval-event output, but does not remove human-format conversion from JSON reporting.
- No direct upstream issue, PR, or commit candidate was found with the recorded symbol/performance searches.

Target fit: Undecided — likely a small pull request only if profiling proves the formatting cost exceeds noise under a representative supported workload.

## Direction

Keep shared numeric calculations required by both outputs. Move only human-readable buffer creation, including `unit_snprintf` and text-only role labels where proven unused, into the text branch. Avoid duplicating bandwidth or timestamp calculations and preserve one owner for each JSON number.

## Bounds

- Preserve: Every JSON key and numeric value, floating precision, text output bytes, `-f` text behavior, timestamps, omitted flags, TCP/UDP/SCTP fields, bidirectional role labels, JSON streaming, and callback behavior.
- Exclude: Replacing `iperf_json_printf`, changing JSON precision or schema, caching interval values, changing unit conversion generally, and optimizing final summaries without separate evidence.
- Cost: Branch-local declarations may be needed to prevent uninitialized text buffers; the change must not duplicate shared arithmetic or create divergent JSON/text formulas.

## Verification

- Profile reporter CPU with JSON output at 1, 8, and 128 streams and 1.0/0.1-second intervals; separate transfer-worker CPU from reporter CPU.
- Count `unit_snprintf` calls in baseline and candidate; require zero interval-path calls in JSON modes and unchanged calls in text modes.
- Compare complete plain JSON and JSON-stream output byte-for-byte under TCP, UDP, reverse, bidirectional, omitted intervals, retransmits, and loss.
- Compare text output across every unit format accepted by `-f`.
- Run `make check`, `test_commands.sh`, and focused client/server scenarios after output guards pass.

## Missing

- [N] Baseline call counts, reporter CPU, total CPU, throughput, and variance.
- [N] Candidate measurement proving benefit above noise.
- [N] Full JSON/text byte-equivalence matrix.
- [N] GitHub Discussions, mailing-list, releases, and active-branch prior-art coverage.
- [N] User selection of Report or Pull request mode and exact target.

## Resume

Index: Profile JSON formatting waste
Next: Profile interval reporting and count `unit_snprintf` calls across JSON/text, stream-count, and interval-frequency boundaries.
Done when: Baseline evidence quantifies JSON-only dead formatting and identifies a workload with repeatable material cost.

## Performance evidence

Workload: Planned TCP/UDP localhost reporting matrix over output mode, stream count, and interval frequency on the recorded Linux environment.
Baseline [N]: Not measured.
Candidate [N]: Not implemented or measured.
Guard [N]: JSON/text byte equivalence and protocol-mode coverage not tested.
Boundary [N]: Dead JSON-path formatting is source-proven; meaningful process-level performance value remains unmeasured.
