# ISSUE-022 — UDP GSO/GRO: effective mmap length is lost

State: Hold
Mode: Undecided
Target: Undecided
Location: Not published.
Priority: High
Confidence: High
Type: reliability
Created: 2026-08-14
Updated: 2026-08-14
Source: `upstream/master@c9b74229d0d9bfec6d2307b66b43c29a7665ad0b`

## Root

[S] `iperf_new_stream()` computes a local effective mapping length in `src/iperf_api.c:5020-5031`: start with `test->settings->blksize`, then raise `size` to `gso_bf_size` or `gro_bf_size` for UDP when the corresponding option is enabled; it uses `size` for both `ftruncate()` and `mmap()`.

[S] That effective length is not retained in `struct iperf_stream`. `iperf_free_stream()` unmaps with the base `test->settings->blksize` at `src/iperf_api.c:4937-4957`, and the post-map rollback does the same at `src/iperf_api.c:5070-5084`. Thus both teardown paths can unmap less than the mapping created at `src/iperf_api.c:5031`.

[S] `src/iperf.h:196-201` owns the negotiated GSO/GRO settings; `src/iperf.h:468-499` defines `MAX_UDP_BLOCKSIZE` as `(65535 - 8 - 20)` and aliases both `GSO_BF_MAX_SIZE` and `GRO_BF_MAX_SIZE` to it. The current defaults assign those maxima at `src/iperf_api.c:3472-3477`, so a UDP GSO/GRO stream can map more than its ordinary block size.

[S] The root is lost allocation metadata, not GSO/GRO feature negotiation. Recomputing a size during free from mutable test settings would create a second, non-authoritative lifetime rule instead of retaining the exact successful `mmap()` request.

## Reach and impact

[S] Reach requires a UDP stream whose enabled GSO or GRO buffer requirement exceeds `test->settings->blksize`; ordinary TCP/SCTP streams and UDP streams without such an increase use equal map and unmap arguments. [S] `--gsro` sets both flags even when the local client lacks offload support so it can request server support (`src/iperf_api.c:1801-1840`), while the manual says actual offload support is currently Linux-only (`src/iperf3.1:517-526`). Effective map-size selection therefore does not by itself prove active offload or limit the local mapping path to Linux.

[N] The unmapping residue, virtual-memory cost, RSS effect, failure threshold, and frequency are unmeasured. Do not claim a throughput regression: this is a resource-lifetime finding.

[N] Page rounding can hide the defect when the page-rounded base and effective lengths coincide; it can leave a tail mapping when they differ. The exact result has not been measured on any supported platform.

## Evidence

[S] The same mismatch exists on the normal destructor and `err_exit_munmap_buffer` rollback, so fixing only `iperf_free_stream()` leaves failure cleanup wrong.

[S] `iperf_new_stream()` is publicly declared at `src/iperf_api.h:297` and `iperf_free_stream()` at `src/iperf_api.h:315`; their source implementation owns the allocation/teardown pair.

- [O] Historical repository baseline: `make check` returned `5/5 pass`; it does not assert `munmap()` extent after a GSO/GRO-enlarged map or post-map allocation failure and is not evidence for this lifetime defect.

## API and compatibility

Callers [S]: `iperf_new_stream()` and `iperf_free_stream()` are public pointer APIs (`src/iperf_api.h:297,315`), while `src/Makefile.am:8` installs only `iperf_api.h`; `struct iperf_stream` storage remains internal in `src/iperf.h`.

Contract [S]: preserve negotiated GSO/GRO values, payload initialization at `src/iperf_api.c:5057-5064`, and existing CLI/wire behavior. The stream alone must retain its actual mapping length for teardown.

Compatibility: one internal mapping-length member changes no wire field or installed stream layout. Same- and mixed-version peers keep their existing GSO/GRO negotiation; the corrected process alone uses its local successful `mmap()` length.

Migration: None.

## Prior art

[O] GitHub searches on 2026-08-14 for `(GSO OR GRO OR mmap OR munmap) UDP` returned upstream issue #2037 and pull requests #1925, #2007, #2033, #2038, and other UDP work. The result set was reviewed for direct mapping-lifetime ownership.

[S] Merged https://github.com/esnet/iperf/pull/1925 added Linux UDP GSO/GRO and `--gsro`; its feature scope explains the current settings and enlarged buffer path. Merged https://github.com/esnet/iperf/pull/2007 is a follow-up to #1925 whose listed files are `src/iperf_api.c`, `src/iperf_udp.c`, and `src/iperf_util.c`; neither inspected record identifies the map-length teardown invariant.

[A] Open https://github.com/esnet/iperf/issues/2037 and https://github.com/esnet/iperf/pull/2038 propose separate GSO/GRO controls, not this allocation-lifetime fix. Redirecting this finding there would mix an independent CLI-policy change with a narrow resource correction.

[S] #1925 reports bare-metal throughput and a FreeBSD/Ubuntu VM interoperability check; those observations do not measure `mmap()`/`munmap()` extent, page rounding, or cleanup. VM success is not evidence that the host mapping lifetime is correct.

[N] Recheck current GSO/GRO issues and pull-request diffs before selecting a target; no inspected direct owner of the lost effective mapping length is established.

## Direction

Add one internal `struct iperf_stream` member as the sole owner of the successful mapping length. Store the computed effective `size` immediately after successful `mmap()`, then use that member for both `iperf_free_stream()` and `err_exit_munmap_buffer`. Retain the existing base block size for payload generation and I/O; do not recalculate teardown length from GSO/GRO descriptors. Rejected: fix only `iperf_free_stream()`; the post-map rollback retains the same mismatch. Rejected: recompute from mutable test settings during teardown; it creates a second lifetime rule rather than retaining the successful mapping request.

## Bounds

[S] Preserve `ftruncate()` and `mmap()`'s current effective-size selection, the GSO/GRO negotiation fields, and the existing cleanup order for map, temporary buffer fd, disk file, results, timer, and stream.

[S] Cover normal free and every post-map rollback; pre-map failures have no mapping to unmap. Do not change `GSO_BF_MAX_SIZE`, `GRO_BF_MAX_SIZE`, UDP packet sizing, feature availability, offload policy, or the existing cross-version negotiation fields.

[N] Platform variance remains a verification boundary: the manual's Linux-only offload statement does not establish `munmap()` extent on Linux, FreeBSD, or macOS. Record the negotiated flags and effective mapping length for each tested role; do not treat a VM or throughput result as mapping-lifetime evidence.

## Verification

Test decision: none; no gates ran for this ledger edit.

[N] Before implementation or publication, exercise UDP GSO and UDP GRO configurations where the effective buffer exceeds `blksize`; record `mmap()` and normal/rollback `munmap()` lengths, then verify whether a mapping tail remains. Include same-page and page-crossing cases to prevent page rounding from masking the defect.

[N] Test Linux, FreeBSD, and macOS role/peer combinations only to the extent their negotiated `gso`/`gro` settings enter the enlarged-map path; do not infer that path solely from local offload support. Verify same- and mixed-version exchanges retain their existing negotiated flags and wire data.

## Missing

[N] No runtime mapping measurement, rollback injection, residual-map observation, or role/platform/mixed-version negotiation matrix exists for `upstream/master@c9b74229d0d9bfec6d2307b66b43c29a7665ad0b`.

[N] No direct upstream target is established.

## Resume

Index: Record effective mmap length

Next: Measure normal and rollback unmapping after a UDP GSO/GRO stream maps more than `blksize`.

Done when: The recorded map and unmap extents demonstrate the residual mapping boundary and identify the applicable supported-platform scope.
