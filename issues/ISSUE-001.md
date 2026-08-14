# ISSUE-001 — JSON lifecycle: rendered output survives persistent server reset

State: Hold
Mode: Pull request
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
Impact [O]: With identical 32-stream, 1-second localhost tests, full JSON increased warm server RSS and heap by about 31.4 KiB per completed run from runs 20–40. One representative rendered server document was 32,134 bytes. Default JSON streaming stayed nearly flat over the same warm interval. Outage timing still depends on test rate, document size, allocator, and memory limit.

## Evidence

- [S] `src/main.c:169-193` — the server loop reuses one `iperf_test`, then calls `iperf_reset_test` after each run.
- [S] `src/iperf_server_api.c:995-1001` — a successful JSON server run calls `iperf_json_finish` before cleanup and return.
- [S] `src/iperf_api.c:5486-5518` — full-output modes render `json_top`, duplicate the rendered string, and assign it to `test->json_output_string`.
- [S] `src/iperf_api.c:5535-5539` — JSON-tree pointers are cleared after rendering, but `json_output_string` remains owned by the test.
- [S] `src/iperf_api.c:3667-3669` — final `iperf_free_test` releases `json_output_string`.
- [S] `src/iperf_api.c:3706-3838` — `iperf_reset_test` releases streams, timers, title, extra data, and captured server lines, but not `json_output_string`.
- [S] `src/iperf_api.c:326-329` and `src/iperf_api.h:158-161` — the public getter returns the internal pointer; no copy or caller ownership transfer is declared.
- [O] `src/iperf3 -s -J -p 55201`; clients=`src/iperf3 -c 127.0.0.1 -p 55201 -t 1 -i 0.1 -P 32 -J --logfile /dev/null`; environment=iperf 3.21+, Ubuntu 24.04.4 LTS, Linux 6.8.0-107-generic, x86_64 — `pmap -x` total RSS was 3,708 KiB at run 0, 4,684 KiB at run 10, 5,072 KiB at run 20, and 5,700 KiB at run 40.
- [O] The process heap mapping RSS was 12 KiB at run 0, 700 KiB at run 10, 984 KiB at run 20, and 1,612 KiB at run 40; runs 20–40 added 628 KiB, or 31.4 KiB/run.
- [O] A matching one-off server document from `src/iperf3 -s -1 -J -p 55205 --logfile /tmp/iperf-json-size.json` was 32,134 bytes by `wc -c`, matching the warm retained-memory slope.
- [O] Control=`src/iperf3 -s --json-stream -p 55203` with the same clients — total RSS was 3,708 KiB at run 0, 4,500 KiB at run 20, and 4,540 KiB at run 40; heap RSS was 12, 248, and 288 KiB. The warm 20–40 interval added only 2.0 KiB/run.
- [N] Upstream issue, PR, and commit searches for `json_output_string reset leak OR memory` and `json_output_string reset` returned no direct candidate on 2026-08-14.

## Prior art

Coverage: GitHub issues(open+closed), PRs(open+closed+merged), and commit search; checked=2026-08-14.
Gaps: GitHub Discussions, `iperf-dev` archives, releases, and active unpublished branches remain unchecked.

- `https://github.com/esnet/iperf/pull/1098` — Related; introduced streaming JSON and deliberately discards streamed interval objects, but does not own the persistent full-output result-string reset.
- `https://github.com/esnet/iperf/pull/1463` — Related; JSON logfile error-path fix, not this retained result-string lifecycle.
- `https://github.com/esnet/iperf/pull/2034` — Distinct; fixes client file-descriptor leaks, not server JSON heap ownership.

Target fit: New pull request recommended — runtime evidence confirms linear retained memory, the fix is lifecycle-local, and searched upstream work does not own this reset omission. Exact Target remains user-unselected.

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

- [N] Candidate measurement proving flat retained-memory slope and unchanged JSON output.
- [N] GitHub Discussions, mailing-list, release-note, and active-branch prior-art coverage.
- [N] Exact Target selection; Mode is user-selected Pull request.

## Resume

Index: Create JSON reset branch
Next: Create a clean contribution branch from the recorded upstream revision and record the bounded reset-cleanup implementation scope.
Done when: The contribution worktree is based on `upstream/master@c9b74229d0d9bfec6d2307b66b43c29a7665ad0b` with only ISSUE-001 source scope authorized.

## Bug reproduction

Environment: iperf 3.21+ from `upstream/master@c9b74229d0d9bfec6d2307b66b43c29a7665ad0b`; Ubuntu 24.04.4 LTS; Linux 6.8.0-107-generic; x86_64; glibc; localhost.
Reproduction: Start `src/iperf3 -s -J -p 55201`; run 40 sequential `src/iperf3 -c 127.0.0.1 -p 55201 -t 1 -i 0.1 -P 32 -J --logfile /dev/null`; sample `pmap -x <server-pid>` after runs 0, 10, 20, and 40.
Actual [O]: Warm runs 20–40 increased total and heap RSS by 628 KiB, 31.4 KiB/run. A representative result document was 32,134 bytes.
Expected: `iperf_reset_test` must release prior test-owned rendered output before the same server object begins another test; retained memory must not grow with completed test count.

## Performance evidence

Workload: Persistent Linux server; identical localhost clients using `-t 1 -i 0.1 -P 32 -J`; RSS sampled after 0, 10, 20, and 40 completed tests. Default `--json-stream` repeated for 0, 20, and 40 as the non-full-output control.
Baseline [O]: Full JSON total RSS=`3708,4684,5072,5700 KiB`; heap RSS=`12,700,984,1612 KiB`. Warm runs 20–40 retained 31.4 KiB/run; representative JSON size=32,134 bytes.
Candidate [N]: Not implemented or measured.
Guard [O]: Default `--json-stream` warm runs 20–40 added 40 KiB total RSS and heap, or 2.0 KiB/run; full JSON added 628 KiB over the same run interval.
Boundary [O]: Linear retained memory is reproduced for plain full JSON on this Linux/glibc workload. `--json-stream-full-output`, candidate behavior, other allocators, and operational exhaustion time remain unmeasured.

## API and compatibility

Callers [S]: CLI persistent server loop and libiperf users that call `iperf_run_server`, read `iperf_get_test_json_output_string`, then call `iperf_reset_test`.
Contract [S]: The getter returns test-owned internal storage; `iperf_reset_test` begins a new test lifecycle and already invalidates other test-owned result state.
Compatibility: Keep the result valid until reset; after reset, expose no stale pointer. Preserve wire protocol and emitted JSON.
Migration: None.
