# ISSUE-001 — JSON lifecycle: rendered output survives persistent server reset

State: Hold
Mode: Undecided
Target: Undecided
Location: Not published.
Priority: High
Confidence: High
Type: performance
Created: 2026-08-14
Updated: 2026-08-14
Source: `upstream/master@c9b74229d0d9bfec6d2307b66b43c29a7665ad0b`

## Root

Root [S]: `iperf_json_finish` stores a heap copy in `test->json_output_string`, but `iperf_reset_test` does not release that test-owned result before the persistent server reuses the same `iperf_test`; the next full JSON result overwrites the only pointer to the prior allocation.

## Reach and impact

Reach [S]: `main` repeatedly calls `iperf_run_server(test)` and `iperf_reset_test(test)` on one object; every completed persistent-server test using full JSON output reaches `iperf_json_finish`. Plain `-J` and `--json-stream-full-output` render the full document; default `--json-stream` sets `print_full_json = 0` and does not create this allocation.
Impact [N]: Retained heap is structurally unbounded in completed full-JSON test count and proportional to each rendered document, including its stream and interval data. RSS slope, allocator-retained bytes, and outage threshold are not measured.

## Evidence

- [S] `src/main.c:169-193` — the server loop reuses one `iperf_test`, then calls `iperf_reset_test` after each run.
- [S] `src/iperf_server_api.c:995-1001` — a successful JSON server run calls `iperf_json_finish` before cleanup and return.
- [S] `src/iperf_api.c:5486-5518` — full-output modes render `json_top`, duplicate the rendered string, and assign it to `test->json_output_string`.
- [S] `src/iperf_api.c:5535-5539` — JSON-tree pointers are cleared after rendering, but `json_output_string` remains owned by the test.
- [S] `src/iperf_api.c:3667-3669` — final `iperf_free_test` releases `json_output_string`.
- [S] `src/iperf_api.c:3706-3838` — `iperf_reset_test` releases streams, timers, title, extra data, and captured server lines, but not `json_output_string`.
- [S] `src/iperf_api.c:326-329` and `src/iperf_api.h:158-161` — the public getter returns the internal pointer; no copy or caller ownership transfer is declared.
- [N] Upstream issue, PR, and commit searches for `json_output_string reset leak OR memory` and `json_output_string reset` returned no direct candidate on 2026-08-14.

## Prior art

Coverage: GitHub issues(open+closed), PRs(open+closed+merged), and commit search; checked=2026-08-14.
Gaps: GitHub Discussions, `iperf-dev` archives, releases, and runtime reproduction remain unchecked.

- `https://github.com/esnet/iperf/pull/1098` — Related; introduced streaming JSON and deliberately discards streamed interval objects, but does not own the persistent full-output result-string reset.
- `https://github.com/esnet/iperf/pull/1463` — Related; JSON logfile error-path fix, not this retained result-string lifecycle.
- `https://github.com/esnet/iperf/pull/2034` — Distinct; fixes client file-descriptor leaks, not server JSON heap ownership.

Target fit: Undecided — current source supports a focused pull request if runtime reproduction confirms linear retained memory and no active upstream implementation owns the correction.

## Direction

At the reset lifecycle owner, release `test->json_output_string` with the existing destructor's `free`-and-NULL pattern before the next server run. Do not combine this fix with cJSON allocator ownership changes, JSON serialization refactoring, or output-format changes.

## Bounds

- Preserve: Full JSON bytes, callback timing, getter validity through the completed run, JSON streaming behavior, one-off server behavior, error reporting, and mixed-version protocol behavior.
- Exclude: Direct ownership transfer from `cJSON_Print`, serialization redesign, interval retention changes, and unrelated JSON leaks.
- Cost: One reset cleanup operation and an explicit result lifetime ending at `iperf_reset_test`; callers retaining the internal getter pointer across reset must not be treated as supported without contract evidence.

## Verification

- Start one persistent `src/iperf3 -s -J`; run many identical short clients; record process RSS and allocated-live bytes after every completed test.
- Repeat with `--json-stream` and `--json-stream-full-output` to prove the exact affected-mode boundary.
- Compare baseline and candidate memory slope under identical build, commands, run count, and environment.
- Capture each emitted JSON document and compare it byte-for-byte before and after the correction.
- Exercise successful completion, client failure, one-off server, callback output, getter-before-reset, and repeated reset.
- Run `make check` and `test_commands.sh` after the focused scenario passes.

## Missing

- [N] Deterministic runtime reproduction with command, environment, per-run retained bytes, RSS slope, and variance.
- [N] Candidate measurement proving flat retained-memory slope and unchanged JSON output.
- [N] GitHub Discussions, mailing-list, release-note, and active-branch prior-art coverage.
- [N] User selection of Report or Pull request mode and exact target.

## Resume

Index: Reproduce JSON reset leak
Next: Run a persistent full-JSON server against repeated identical clients while recording per-run RSS and heap ownership.
Done when: Baseline evidence shows whether retained memory grows linearly and identifies the allocation stack and affected JSON modes.

## Performance evidence

Workload: Planned persistent Linux server on the recorded revision; identical short localhost clients; plain `-J`, default `--json-stream`, and `--json-stream-full-output` tested separately.
Baseline [N]: Not measured.
Candidate [N]: Not implemented or measured.
Guard [N]: JSON byte equivalence and getter/reset lifetime not tested.
Boundary [N]: Source proves lost ownership across resets; user-visible memory slope and operational impact remain unmeasured.

## API and compatibility

Callers [S]: CLI persistent server loop and libiperf users that call `iperf_run_server`, read `iperf_get_test_json_output_string`, then call `iperf_reset_test`.
Contract [S]: The getter returns test-owned internal storage; `iperf_reset_test` begins a new test lifecycle and already invalidates other test-owned result state.
Compatibility: Keep the result valid until reset; after reset, expose no stale pointer. Preserve wire protocol and emitted JSON.
Migration: None.
