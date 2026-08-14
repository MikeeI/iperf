# ISSUE-026 — Test destruction: pidfile and authorized-users survive free

State: Hold
Mode: Undecided
Target: Undecided
Location: Not published.
Priority: Low
Confidence: High
Type: reliability
Created: 2026-08-14
Updated: 2026-08-14
Source: `upstream/master@c9b74229d0d9bfec6d2307b66b43c29a7665ad0b`

## Root

[S] The terminal field-release inventory in `iperf_free_test()` omits two heap-owned test members: `test->pidfile`, assigned by `strdup(optarg)` for `--pidfile` (`src/iperf_api.c:1698-1700`), and, with `HAVE_SSL`, `test->server_authorized_users`, assigned by its public setter and `--authorized-users-path` parser (`766-769,1757-1759`).

[S] `iperf_free_test()` frees neither member while releasing peer fields and then the test (`src/iperf_api.c:3567-3701`). The one root is incomplete terminal destruction of final test-owned fields, not repeated replacement behavior.

## Reach and impact

[S] The normal CLI path calls `iperf_delete_pidfile()` after client/server work (`src/main.c:194,203`) and later calls `iperf_free_test(test)` (`124-129`). `iperf_delete_pidfile()` unlinks the pathname only; it does not release `test->pidfile` (`src/iperf_api.c:5431-5440`).

[S] Libiperf exposes `iperf_free_test()` as the terminal resource disposer (`src/iperf_api.h:284-289`; `src/libiperf.3:67-103`). Embedding applications therefore retain the omitted allocation for a test object's lifetime even when following the normal destructor path.

[S] The bound is one final `pidfile` allocation and, in an SSL build, one final authorized-users allocation per object. Prior allocations lost by repeated replacement are owned by ISSUE-025.

[N] Allocation size, long-lived embedding frequency, operational effect, and user-visible impact are unmeasured.

## Evidence

[S] `iperf_free_test()` explicitly releases top-level strings, authentication fields, RSA keys, output text, JSON output, and settings before `free(test)` (`src/iperf_api.c:3578-3701`), but contains no release of either subject member.

[S] The authorized-users parser validates the stored path as a readable server file and retains it for later use (`src/iperf_api.c:1876-1906`), confirming test ownership rather than a transient parse buffer.

[S] Excluded: `test->diskfile_name` is assigned directly from `optarg`, not duplicated (`src/iperf_api.c:1582-1584`). It aliases argv storage, is not owned by the test, and must not be freed by `iperf_free_test()`.

## Prior art

[S] Recorded audit candidates are https://github.com/esnet/iperf/issues/1986, active https://github.com/esnet/iperf/pull/1654, and https://github.com/esnet/iperf/pull/1861 for worker-failure propagation; https://github.com/esnet/iperf/pull/1709 introduced the 8-KiB parameter receive cap. None owns final destruction of these two fields.

[N] No targeted current upstream issue/PR search for `pidfile` or `server_authorized_users` destructor ownership is recorded. No external target is selected.

## Direction

[A] In `iperf_free_test()`, free `test->pidfile` and, under `HAVE_SSL`, `test->server_authorized_users` alongside their comparable owned fields. This terminal correction releases storage only; it does not add pathname unlinking or alter authorized-users validation.

## Bounds

[S] Preserve `iperf_delete_pidfile()` return/error semantics, the explicit CLI pidfile lifecycle, public destructor signature, and OpenSSL feature guards.

[S] Do not free `diskfile_name`: it is argv-aliased and non-owned. Do not combine final-value destruction with ISSUE-025's per-replacement behavior.

[N] This bounded final-destructor correction does not claim to reclaim process-exit memory, change reset/reuse behavior, or alter auth-file validation.

## API and compatibility

Callers [S]: public libiperf callers dispose a test through `iperf_free_test()` (`src/iperf_api.h:275-289`; `src/libiperf.3:67-103`); OpenSSL callers can set authorized users through `src/iperf_api.h:230-238`.

Contract [S]: `iperf_free_test()` releases test resources and does not unlink a pidfile.

Compatibility: Keep the `void` destructor API and all file/auth behavior; release only storage allocated by the test.

Migration: None.

## Verification

Test decision: none. This is a ledger-only change; no source, test, validator, or gate was run.

[N] In a non-SSL build, create/default a test, configure exactly one heap-owned pidfile path, exercise the explicit `iperf_delete_pidfile()` path with a real pidfile, then dispose the test under a leak detector. Verify the allocation is released and unlink behavior is unchanged.

[N] In an OpenSSL build, use the authorized-users setter and parser in separate single-assignment scenarios, then dispose each test under a leak detector. Verify the final auth-path allocation is released while `diskfile_name` remains unallocated by and untouched through the destructor.

[N] No OOM injection is needed for this root: it concerns successful final allocations reaching the terminal destructor. Reuse and duplicate-assignment checks belong to ISSUE-025; do not use them to inflate this one-final-value finding.

## Missing

[N] No non-SSL pidfile or SSL authorized-users dynamic destructor result, pidfile-unlink regression result, targeted upstream prior-art currentness, retained-allocation measurement, or maintainer-selected correction/target is recorded.

## Resume

Index: Confirm final destructor fields

Next: Leak-check one pidfile case in a non-SSL build and one authorized-users case in an OpenSSL build.

Done when: Dynamic results confirm the two final owned values are released, `diskfile_name` stays argv-owned, and pidfile unlink/auth behavior is unchanged.
