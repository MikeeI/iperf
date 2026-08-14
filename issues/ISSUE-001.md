# ISSUE-001 — JSON lifecycle: rendered output survives persistent server reset

State: Published
Mode: Pull request
Target: New pull request
Location: https://github.com/esnet/iperf/pull/2067
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
- [N] GitHub Discussions and `iperf-dev` searches for `json_output_string`, JSON memory leaks, and persistent-server memory found no direct candidate on 2026-08-14.
- [N] Upstream issue, PR, and commit searches for `json_output_string reset leak OR memory` and `json_output_string reset` returned no direct candidate on 2026-08-14.
- [O] PR #2067 status on 2026-08-14: open and mergeable at `3bb3582a128c6f90f8a368118f8271ea93ad4d42`, but GitHub reports `mergeStateStatus=UNSTABLE`; no reviews, review requests, comments, or status-check rollup exist.
- [O] GitHub Actions run https://github.com/esnet/iperf/actions/runs/31757951157 completed immediately with `action_required` and zero jobs, so no upstream CI test has executed.

## Prior art

Coverage: GitHub issues(open+closed), PRs(open+closed+merged), and commit search; checked=2026-08-14.
Gaps: Release notes and active unpublished branches remain unchecked.

- `https://github.com/esnet/iperf/pull/1098` — Related; introduced streaming JSON and deliberately discards streamed interval objects, but does not own the persistent full-output result-string reset.
- `https://github.com/esnet/iperf/pull/1463` — Related; JSON logfile error-path fix, not this retained result-string lifecycle.
- `https://github.com/esnet/iperf/pull/1712` — Distinct; frees strings replaced within parallel-stream setup, not the completed full JSON document replaced between persistent-server tests.
- `https://github.com/esnet/iperf/pull/2034` — Distinct; fixes client file-descriptor leaks, not server JSON heap ownership.

Target fit: New pull request — runtime evidence confirms linear retained memory, the correction is lifecycle-local, and searched upstream work does not own this reset omission.

## Direction

At the reset lifecycle owner, release `test->json_output_string` with the existing destructor's `free`-and-NULL pattern before the next server run. Do not combine this fix with cJSON allocator ownership changes, JSON serialization refactoring, or output-format changes.

## Bounds

- Preserve: Full JSON bytes, callback timing, getter validity through the completed run, JSON streaming behavior, one-off server behavior, error reporting, and mixed-version protocol behavior.
- Exclude: Direct ownership transfer from `cJSON_Print`, serialization redesign, interval retention changes, and unrelated JSON leaks.
- Cost: One reset cleanup operation and an explicit result lifetime ending at `iperf_reset_test`; callers retaining the internal getter pointer across reset must not be treated as supported without contract evidence.

## Verification

- [O] On the baseline, full JSON warm runs 20–40 retained 628 KiB total and heap RSS, or 31.4 KiB/run.
- [O] On the candidate, full JSON total RSS at runs 0, 20, 40, and 60 was `3672,4268,4576,4592 KiB`; heap RSS was `12,320,360,376 KiB`.
- [O] Candidate warm runs 20–40 retained 40 KiB, or 2.0 KiB/run; runs 40–60 retained 16 KiB, or 0.8 KiB/run.
- [O] `jq` normalization replacing scalar values with their JSON types produced identical sorted trees for baseline and candidate one-off full JSON documents.
- [O] Two candidate `--json-stream --json-stream-full-output` runs emitted JSON accepted by `jq empty`.
- [O] `make -s check` passed all five repository tests.
- [O] `test_commands.sh 127.0.0.1` completed with status 0 in an isolated network namespace; IPv6 cases were unavailable with the IPv4-only target.

## Missing

- [N] GitHub marks the Build Test `action_required`, but exposes zero jobs and no exact cause; no CI result or maintainer review exists yet.
- [N] FreeBSD and macOS runtime verification is unperformed; the correction uses the existing portable `free`-and-NULL ownership pattern.
- [N] Release-note and active unpublished-branch prior-art coverage remains unchecked.

## Resume

Index: Await PR 2067 CI action
Next: Monitor Actions run #31757951157 and PR #2067 for workflow execution, review, or maintainer feedback.
Done when: CI executes, maintainers respond, or the pull request reaches an observable disposition.

## Bug reproduction

Environment: iperf 3.21+ from `upstream/master@c9b74229d0d9bfec6d2307b66b43c29a7665ad0b`; Ubuntu 24.04.4 LTS; Linux 6.8.0-107-generic; x86_64; glibc; localhost.
Reproduction: Start `src/iperf3 -s -J -p 55201`; run 40 sequential `src/iperf3 -c 127.0.0.1 -p 55201 -t 1 -i 0.1 -P 32 -J --logfile /dev/null`; sample `pmap -x <server-pid>` after runs 0, 10, 20, and 40.
Actual [O]: Warm runs 20–40 increased total and heap RSS by 628 KiB, 31.4 KiB/run. A representative result document was 32,134 bytes.
Expected: `iperf_reset_test` must release prior test-owned rendered output before the same server object begins another test; retained memory must not grow with completed test count.

## Performance evidence

Workload: Persistent Linux server; identical localhost clients using `-t 1 -i 0.1 -P 32 -J`; RSS sampled after completed tests. Default `--json-stream` was the non-full-output control.
Baseline [O]: Full JSON total RSS=`3708,4684,5072,5700 KiB` at runs 0, 10, 20, and 40; heap RSS=`12,700,984,1612 KiB`. Warm runs 20–40 retained 31.4 KiB/run; representative JSON size=32,134 bytes.
Candidate [O]: Full JSON total RSS=`3672,4268,4576,4592 KiB` at runs 0, 20, 40, and 60; heap RSS=`12,320,360,376 KiB`. Warm slopes were 2.0 KiB/run for runs 20–40 and 0.8 KiB/run for runs 40–60.
Guard [O]: Default baseline `--json-stream` warm runs 20–40 added 40 KiB total RSS and heap, or 2.0 KiB/run. Baseline and candidate one-off full JSON documents had identical sorted JSON structure and scalar types.
Boundary [O]: The candidate removes the document-sized linear retained-memory slope on this Linux/glibc workload. Residual allocator RSS, other allocators, and FreeBSD/macOS runtime behavior remain outside the measurement.

## API and compatibility

Callers [S]: CLI persistent server loop and libiperf users that call `iperf_run_server`, read `iperf_get_test_json_output_string`, then call `iperf_reset_test`.
Contract [S]: The getter returns test-owned internal storage; `iperf_reset_test` begins a new test lifecycle and already invalidates other test-owned result state.
Compatibility: Keep the result valid until reset; after reset, expose no stale pointer. Preserve wire protocol and emitted JSON.
Migration: None.

## Implementation

Branch: `fix/json-output-reset`
Base: `upstream/master@c9b74229d0d9bfec6d2307b66b43c29a7665ad0b`
Scope: Release and clear the test-owned rendered JSON string in `iperf_reset_test`; no serialization, wire, output, or API changes.
Commit: `3bb3582a128c6f90f8a368118f8271ea93ad4d42`
Push: `MikeeI/iperf:fix/json-output-reset`
Checks:
- `persistent -J baseline/candidate RSS sampling` → warm retained-memory slope changed from 31.4 KiB/run to 2.0 then 0.8 KiB/run.
- `jq -S 'walk(...)' baseline candidate | diff` → identical JSON structure and scalar types.
- `--json-stream --json-stream-full-output` twice, then `jq empty` → valid streamed JSON.
- `make -s check` → 5/5 tests passed.
- `test_commands.sh 127.0.0.1` in an isolated network namespace → status 0; IPv6 unavailable for the IPv4-only target.

## Draft

Target: New pull request to `esnet/iperf:master` from `MikeeI:fix/json-output-reset`.

Title: `Free rendered JSON output when resetting server tests`

````markdown
_PLEASE NOTE the following text from the iperf3 license.  Submitting a
pull request to the iperf3 repository constitutes "[making]
Enhancements available...publicly":_

```
You are under no obligation whatsoever to provide any bug fixes, patches, or
upgrades to the features, functionality or performance of the source code
("Enhancements") to anyone; however, if you choose to make your Enhancements
available either publicly, or directly to Lawrence Berkeley National
Laboratory, without imposing a separate written license agreement for such
Enhancements, then you hereby grant the following license: a non-exclusive,
royalty-free perpetual license to install, use, modify, prepare derivative
works, incorporate into other computer software, distribute, and sublicense
such enhancements or derivative works thereof, in binary and source code form.
```

_The complete iperf3 license is available in the `LICENSE` file in the
top directory of the iperf3 source tree._

* Version of iperf3 (or development branch, such as `master` or
  `3.1-STABLE`) to which this pull request applies: `master` at `c9b74229d0d9bfec6d2307b66b43c29a7665ad0b`

* Issues fixed (if any): None.

* Brief description of code changes (suitable for use as a commit message):
  Release the test-owned rendered JSON string when resetting a server test.

## Summary

A persistent server reuses one `iperf_test` across runs. Full JSON rendering stores a heap copy in `test->json_output_string`, but reset did not release it before the next result replaced the pointer. This change frees and clears that test-owned string in `iperf_reset_test`.

## Evidence

- [`iperf_json_finish`](https://github.com/esnet/iperf/blob/c9b74229d0d9bfec6d2307b66b43c29a7665ad0b/src/iperf_api.c#L5486-L5518) duplicates the rendered document into `json_output_string`.
- [`iperf_reset_test`](https://github.com/esnet/iperf/blob/c9b74229d0d9bfec6d2307b66b43c29a7665ad0b/src/iperf_api.c#L3706-L3838) did not release that string before persistent-server reuse.
- On Ubuntu 24.04.4/glibc, a persistent `-J` server receiving sequential one-second, 32-stream localhost tests retained 628 KiB over warm runs 20–40, or 31.4 KiB/run. A representative rendered document was 32,134 bytes.

## Changes

- Free and clear `json_output_string` at the reset lifecycle boundary.
- Preserve the rendered result through the completed run and leave JSON serialization, emitted output, callbacks, protocol behavior, and the public getter unchanged.

## Risks and boundaries

- `iperf_reset_test` already invalidates test-owned result state for the next run; this applies the same lifetime to the rendered string.
- The correction uses the destructor's existing `free`-and-NULL pattern and adds no new abstraction or platform-specific behavior.

## Verification

- Persistent full JSON candidate: warm RSS growth fell to 40 KiB over runs 20–40 and 16 KiB over runs 40–60, versus 628 KiB over baseline runs 20–40.
- Baseline and candidate one-off documents had identical sorted JSON structure and scalar types after normalizing live values.
- Two `--json-stream --json-stream-full-output` runs emitted JSON accepted by `jq empty`.
- `make -s check` — 5/5 tests passed.
- `test_commands.sh 127.0.0.1` — completed with status 0 in an isolated network namespace; IPv6 was unavailable for the IPv4-only target.

I checked the relevant issues, comments, pull requests, discussions, and `iperf-dev` search results; this pull request is not a duplicate.

### Disclosure

Investigated thoroughly with GPT-5.6 (extra high reasoning effort), using [Oh My Pi](https://github.com/can1357/oh-my-pi) as the agent framework.

This report is not generic or unreviewed AI-generated output. Its claims were checked against the cited evidence, and it includes the relevant detail intended to help maintainers resolve the issue.

If reports like this are not useful to the project, please let me know and I will refrain from submitting similar ones. My intent is to help without wasting maintainer time or energy or discouraging their work.

Thank you for your work.
````
