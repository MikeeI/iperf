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

[S] `test->pidfile` is heap-owned from `strdup(optarg)` in the `--pidfile` parser case (`src/iperf_api.c:1698-1700`), but `iperf_free_test()` never frees it (`src/iperf_api.c:3567-3701`).

[S] With `HAVE_SSL`, `test->server_authorized_users` is heap-owned by both the public setter (`src/iperf_api.c:766-769`) and `--authorized-users-path` parser case (`src/iperf_api.c:1757-1759`), but `iperf_free_test()` omits it while freeing the other listed authentication fields (`src/iperf_api.c:3598-3616`).

[S] The root is two terminal destructor omissions: the test object owns these final allocations, but its sole public destructor does not release them.

## Reach and impact

[S] The normal CLI path calls `iperf_delete_pidfile()` after client/server work (`src/main.c:194,203`) and later calls `iperf_free_test(test)` (`src/main.c:124-129`). `iperf_delete_pidfile()` only unlinks the pathname; it does not free `test->pidfile`: `src/iperf_api.c:5431-5440`.

[S] Libiperf exposes `iperf_free_test()` as the termination call (`src/iperf_api.h:284-289`; `src/libiperf.3:18-22,67-103`), so embedding applications retain these allocations for a test object's lifetime even when they follow the documented destruction path.

[S] This is normally at most one final `pidfile` allocation and, with `HAVE_SSL`, one final authorized-users allocation per test object unless replacement leaks occur; those repeated-replacement losses are separately owned by ISSUE-025.

[N] Allocation size, long-lived embedding frequency, and operational effect are unmeasured. No user-visible bug reproduction is recorded.

## Evidence

[S] `iperf_free_test()` explicitly frees many top-level strings and client authentication/RSA values before freeing settings and the test (`src/iperf_api.c:3578-3629,3693-3701`), establishing field-by-field destruction ownership while leaving both subject fields absent.

[S] The parser validates `test->server_authorized_users` as a readable server file and keeps its string for later use (`src/iperf_api.c:1876-1906`), confirming that it is test-owned state rather than a transient parser buffer.

[S] Excluded: `test->diskfile_name` is assigned directly from `optarg`, not duplicated (`src/iperf_api.c:1582-1584`). It is argv-aliased storage and must not be freed by `iperf_free_test()`.

[O] Recorded baseline: `make check` — `5/5 pass`. It does not inspect final test-object ownership under the destructor.

## Prior art

[S] Recorded audit candidates are https://github.com/esnet/iperf/issues/1986, active https://github.com/esnet/iperf/pull/1654, and https://github.com/esnet/iperf/pull/1861 for worker-failure propagation; https://github.com/esnet/iperf/pull/1709 introduced the 8-KiB parameter receive cap. None owns final destruction of these two fields.

[N] No targeted current upstream issue/PR search for `pidfile` or `server_authorized_users` destructor ownership is recorded here. No external target is selected.

## Direction

[A] In `iperf_free_test()`, release and clear `test->pidfile`; under `HAVE_SSL`, release and clear `test->server_authorized_users` alongside its peer authentication fields.

[A] Keep pathname unlinking owned by `iperf_delete_pidfile()`; destruction should release only the stored path string and must not add a new filesystem-unlink side effect.

## Bounds

[S] Preserve `iperf_delete_pidfile()` return/error semantics, explicit CLI pidfile lifecycle, public destructor signature, and OpenSSL feature guards.

[S] Do not free `diskfile_name`: it aliases argv storage. Do not combine this final-value cleanup with ISSUE-025's per-replacement behavior.

[N] The correction must not claim to reclaim process-exit memory or alter auth-file validation semantics without measurement.

## API and compatibility

Callers [S]: public libiperf callers use `iperf_free_test()` after `iperf_new_test()` (`src/iperf_api.h:275-289`; `src/libiperf.3:67-103`); OpenSSL users may configure authorized users through the public setter in `src/iperf_api.h:230-238`.

Contract [S]: `iperf_free_test()` is documented to free test resources; its existing caller-visible behavior does not unlink a pidfile.

Compatibility: Keep the `void` destructor API and all file/auth behavior; only release memory that the test itself allocated.

Migration: None.

## Verification

Test decision: none. This is a ledger-only change; no source, test, validator, or gate was run.

[A] In a non-SSL build, configure a heap-owned pidfile path, perform any required explicit `iperf_delete_pidfile()` call, then free the test under a leak detector and verify no unlink behavior changes.

[A] In an OpenSSL build, configure authorized users through both setter and parser routes, free the test under a leak detector, and verify the auth path is freed while `diskfile_name` remains caller/argv-owned.

## Missing

[N] Targeted upstream prior-art currentness, dynamic destructor reproduction in non-SSL and OpenSSL builds, retained-allocation measurement, and a maintainer-selected correction/target remain unresolved.

## Resume

Index: Confirm destructor ownership
Next: At `upstream/master@c9b74229d0d9bfec6d2307b66b43c29a7665ad0b`, inspect the two destructor paths under a leak detector in non-SSL and OpenSSL builds while confirming pidfile unlink behavior is unchanged.
Done when: Leak evidence distinguishes the two final owned values from argv aliases and confirms a destructor-only correction has no filesystem or authentication regression.
