# ISSUE-024 — JSON lifecycle: failed aggregates lack rollback

State: Hold
Mode: Undecided
Target: Undecided
Location: Not published.
Priority: Medium
Confidence: High
Type: reliability
Created: 2026-08-14
Updated: 2026-08-14
Source: `upstream/master@c9b74229d0d9bfec6d2307b66b43c29a7665ad0b`

## Root

[S] `iperf_json_start()` publishes `test->json_top` and child aliases while constructing the aggregate, then returns `-1` after a later `cJSON_CreateObject()` or `cJSON_CreateArray()` failure without deleting the already-built root (`src/iperf_api.c:5444-5465`).

[S] It also ignores each `cJSON_AddItemToObject()` result. That cJSON operation can fail while duplicating an object key and leaves the item unconsumed (`src/cjson.c:2022-2057`), so `iperf_json_start()` can return success with a child alias detached from the root. `iperf_json_finish()` likewise ignores attachment failure for `test->json_server_output` (`src/iperf_api.c:5478-5481`).

[S] `iperf_json_finish()` deletes the root and clears aliases only on its success path. `cJSON_Print()` or `strdup()` failure returns before that cleanup (`5513-5539`). The one root is incomplete cJSON aggregate ownership: an allocation or attachment failure leaves the installed root or a detached child without an error-path owner.

## Reach and impact

[S] `iperf_json_start()` and `iperf_json_finish()` are public (`src/iperf_api.h:387-389`). Direct libiperf callers can receive either failure and then reset or dispose the test without an intervening aggregate destructor.

[S] CLI error handling can mask, but not disprove, the reach: a client startup failure returns from `iperf_run_client()` (`src/iperf_client_api.c:627-629`) and `main` calls `iperf_errexit()`, whose JSON path calls `iperf_json_finish()` (`src/main.c:124-127`, `src/iperf_error.c:126-130`). Server startup returns `-2` after `cleanup_server()` (`src/iperf_server_api.c:590-594`), and the CLI loop also calls `iperf_json_finish()` on JSON errors (`src/main.c:169-185`). A second finish that succeeds can release the root; it does not make the direct-caller or repeated-finish failure paths safe.

[S] The documented libiperf server pattern repeatedly calls `iperf_run_server()` then `iperf_reset_test()` on one object (`src/libiperf.3:87-102`), while `iperf_reset_test()` neither deletes the root nor clears the JSON aliases (`src/iperf_api.c:3706-3838`). `iperf_free_test()` frees `json_output_string` but has no cJSON aggregate cleanup (`3567-3701`).

[N] This is allocation/attachment-failure reach. No fault-injection reproduction, retained-allocation count, user-visible impact, or normal-success leak is recorded.

## Evidence

[S] `iperf_json_start()` creates `json_top`, `json_start`, `json_connected`, `json_intervals`, and `json_end` and attaches each as it proceeds (`src/iperf_api.c:5444-5464`), but checks only creation and not attachment.

[S] `cJSON_AddItemToObject()` allocates the key before consuming its item; on that allocation failure it returns false before calling `add_item_to_array()` (`src/cjson.c:2022-2057`). The source therefore does not prove that every published alias belongs to `json_top`.

[S] On a normal finish, `iperf_json_finish()` deletes only `json_top` and clears `json_top`, all child aliases, and `json_server_output` (`src/iperf_api.c:5535-5539`). That cleanup cannot run after the two render/copy early returns.

## Prior art

[S] Recorded audit candidates are https://github.com/esnet/iperf/issues/1986, active https://github.com/esnet/iperf/pull/1654, and https://github.com/esnet/iperf/pull/1861 for worker-failure propagation; https://github.com/esnet/iperf/pull/1709 introduced the 8-KiB parameter receive cap. None owns failed aggregate construction, attachment, or finish cleanup.

[N] No targeted current upstream issue/PR search for `iperf_json_start`, `iperf_json_finish`, or cJSON attachment failure is recorded. No external target is selected.

## Direction

[A] Make aggregate ownership transactional: construct locally, check every cJSON attachment, and delete the root plus only still-detached children on any construction failure; publish test aliases only after a complete aggregate exists.

[A] In `iperf_json_finish()`, distinguish whether `json_server_output` was attached, then funnel render/copy failures through one cleanup path that deletes the root and clears aliases. Give `iperf_free_test()` the same ownership-aware terminal backstop. Never delete child aliases independently when their root owns them.

## Bounds

[S] Preserve successful JSON document shape, callback/output timing, JSON-stream reference behavior, and existing negative returns from `iperf_json_start()` and `iperf_json_finish()`.

[S] Excluded: ISSUE-001 owns the rendered `test->json_output_string` reset leakage, including replacement of a prior rendered string after persistent reuse. This record must not alter that result-string lifetime.

[S] Excluded: `JSONStream_Output()`'s local event on `cJSON_PrintUnformatted()` failure (`src/iperf_api.c:3262-3286`) and the transient `cJSON_CreateString(test->server_output_text)` passed to that reference wrapper (`5498`) have separate one-event ownership paths; neither establishes retained test-aggregate reach.

[N] Do not merge these low-frequency or one-off paths into this aggregate rollback finding, or claim a performance effect without measurement.

## API and compatibility

Callers [S]: `iperf_run_client()` and `iperf_run_server()` call the JSON lifecycle functions at `src/iperf_client_api.c:627-629,870-872` and `src/iperf_server_api.c:590-594,995-997`; `iperf_json_start()` and `iperf_json_finish()` are also declared in `src/iperf_api.h:387-389`.

Contract [S]: The lifecycle functions return negative status on setup/render failure; successful JSON output and JSON-stream callback behavior are currently controlled by `src/iperf_api.c:5444-5539`.

Compatibility: Preserve exported signatures, negative returns, JSON schema, callback/output timing, and cJSON child ownership.

Migration: None.

## Verification

Test decision: none. This is a ledger-only change; no source, test, validator, or gate was run.

[N] With a cJSON allocation hook, fail each `iperf_json_start()` object/array creation and each object-key allocation used for attachment. Require the documented failure result where applicable, dispose the test under a cJSON-aware leak detector, and record that no root, detached child, stale alias, invalid delete, or double free remains.

[N] Build a test aggregate with detached `json_server_output`, inject its object-key attachment failure, and separately inject `cJSON_Print()` and the rendered-string `strdup()` failures. Exercise `iperf_json_finish()`, `iperf_reset_test()`, and `iperf_free_test()` after each boundary.

[N] Run the documented persistent server reset shape after a failed JSON operation, then a succeeding JSON run; verify no stale aliases, aggregate leak, or double free. This does not test ISSUE-001's rendered-output reset leak.

## Missing

[N] No per-boundary cJSON creation, attachment, print, or copy fault-injection result establishes which installed or detached tree survives cleanup at `upstream/master@c9b74229d0d9bfec6d2307b66b43c29a7665ad0b`.

[N] No reuse result, targeted upstream prior-art currentness, retained-allocation measurement, or maintainer-selected correction/target is recorded.

## Resume

Index: Inject aggregate ownership failures

Next: Fault-inject one aggregate creation or attachment boundary, then inspect cJSON reachability after finish, reset, and free.

Done when: Per-boundary results distinguish root-owned from detached trees and identify the smallest compatible cleanup owner.
