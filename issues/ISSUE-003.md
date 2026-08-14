# ISSUE-003 — interval reporting: flush policy executes once per stream line

State: Hold
Mode: Undecided
Target: Undecided
Location: Not published.
Priority: Medium
Confidence: Medium
Type: performance
Created: 2026-08-14
Updated: 2026-08-14
Source: `upstream/master@c9b74229d0d9bfec6d2307b66b43c29a7665ad0b`

## Root

Root [S]: `print_interval_results` owns an output-policy decision and calls `iflush` after each individual stream result when a logfile or `--forceflush` is active, although the documented visibility unit is the reporting interval and the caller still emits remaining stream and summary lines.

## Reach and impact

Reach [S]: Every text interval report iterates matching streams and calls `print_interval_results`; with N streams, one reporter callback can acquire the print mutex and call `fflush` N times before the interval summary is complete. The path applies to explicit logfiles even without `--forceflush`.
Impact [N]: Flush, lock, stdio, write-syscall, storage, and throughput costs are not measured. The static upper bound is 128 stream-line flushes per callback and callbacks as often as every 0.1 seconds; actual write behavior depends on libc, destination, buffering, and filesystem.

## Evidence

- [S] `src/iperf_api.c:4142-4144` — `iperf_print_intermediate` calls `print_interval_results` inside the stream loop.
- [S] `src/iperf_api.c:4165-4233` — sum formatting occurs after stream lines, so each current flush precedes completion of the interval report.
- [S] `src/iperf_api.c:4931-4933` — the per-stream helper calls `iflush` for every logfile or force-flush output.
- [S] `src/iperf_api.c:5729-5749` — `iflush` locks `print_mutex`, calls `fflush`, and unlocks it.
- [S] `src/iperf_locale.c:122-124` — CLI help defines `--forceflush` as forcing output at every interval, not every stream line.
- [S] `src/iperf.h:470-476` — report intervals can be 0.1 seconds and stream count can be 128.
- [S] `src/iperf_api.c:3262-3284` — JSON streaming already serializes and flushes one complete event independently.
- [N] No syscall count, write count, reporter CPU, or visibility test has been run.

## Prior art

Coverage: GitHub issues(open+closed), PRs(open+closed+merged), and commit search for `forceflush`, `fflush`, interval reporter, and performance; checked=2026-08-14.
Gaps: GitHub Discussions, `iperf-dev` archives, releases, libc/filesystem-specific reports, and current unpublished branches remain unchecked.

- No direct upstream issue, PR, or commit candidate was found with the recorded searches.

Target fit: Undecided — target depends on measured syscall reduction and proof that one interval-boundary flush preserves established visibility and failure semantics.

## Direction

Move text flush policy to `iperf_print_intermediate`, after all stream and summary output for one accepted interval. Route text exits that have emitted output through one conditional flush. Leave `JSONStream_Output` ownership unchanged and avoid adding an interval flush to monolithic JSON output.

## Bounds

- Preserve: `--forceflush` interval visibility, logfile visibility, line order, partial output on reporter errors, print mutex serialization, JSON streaming's event flush, callback behavior, and output bytes.
- Exclude: Changing default buffering, adding fsync, changing `iperf_printf`, removing locking, batching multiple intervals, or redesigning reporter callbacks.
- Cost: Centralizing the flush condition requires careful control flow for early returns; a naive single tail call can leave already-written error-path output buffered.

## Verification

- Count `fflush` calls and resulting writes for 1, 2, 8, and 128 streams at 1.0 and 0.1 second intervals with stdout, regular logfile, and `--forceflush`.
- Require one text flush per completed interval and no extra monolithic-JSON interval flush.
- Observe the logfile during a running test to prove all stream and summary lines become visible at each interval boundary.
- Inject or trigger reporter error paths after partial output and confirm emitted diagnostics remain visible with equivalent timing.
- Compare complete text and JSON outputs byte-for-byte; run `make check` and `test_commands.sh` after focused scenarios pass.

## Missing

- [N] Baseline `fflush`/write counts, reporter CPU, latency, and variance on representative outputs.
- [N] Candidate measurement and interval-visibility/error-path guard.
- [N] Confirmation whether logfile-per-line visibility is relied upon beyond the documented interval contract.
- [N] GitHub Discussions, mailing-list, releases, and active-branch prior-art coverage.
- [N] User selection of Report or Pull request mode and exact target.

## Resume

Index: Measure interval flush costs
Next: Trace flush and write calls for increasing stream counts while observing interval-boundary logfile visibility.
Done when: Baseline evidence quantifies duplicate flush work and establishes the exact visibility contract a correction must preserve.

## Performance evidence

Workload: Planned text reporter matrix over stream count, report interval, stdout/logfile, and `--forceflush`; Linux localhost test with identical traffic parameters.
Baseline [N]: Not measured.
Candidate [N]: Not implemented or measured.
Guard [N]: Interval visibility, partial-error output, and byte equivalence not tested.
Boundary [N]: Repeated `fflush` calls are source-proven; resulting syscalls and user-visible cost are environment-dependent and unmeasured.

## API and compatibility

Callers [S]: Default reporter callback through client/server reporter timers and libiperf users retaining the default reporter.
Contract [S]: CLI help promises forced visibility at every interval; `JSONStream_Output` separately owns per-event JSON visibility.
Compatibility: Preserve interval output timing, output bytes, callback API, locking, and errors across text, logfile, JSON, and JSON-stream modes.
Migration: None.
