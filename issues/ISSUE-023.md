# ISSUE-023 — Defaults initialization: freed protocol nodes remain linked

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

[S] `iperf_defaults()` initializes `testp->protocols` at `src/iperf_api.c:3498-3501`, allocates and inserts TCP at `3502-3514`, then frees TCP if the second `protocol_new()` for UDP fails at `3516-3520` without unlinking that list node.

[S] With `HAVE_SCTP_H`, it inserts UDP at `3522-3530` and calls `set_protocol(testp, Ptcp)` at `3532` before the third `protocol_new()` at `3534-3552`. That failure frees both TCP and UDP while both remain linked and leaves `testp->protocol` pointing at freed TCP.

[S] `iperf_free_test()` assumes every `test->protocols` entry is live: it removes and frees each at `3641-3646`. Its traversal dereferences and frees a dangling node after either rollback. A first TCP allocation failure has an empty list and is outside this root.

## Reach and impact

[S] `iperf_defaults(struct iperf_test *)` returns `-1` without freeing `testp` on these paths, while `iperf_free_test(struct iperf_test *)` is the public disposer (`src/iperf_api.h:282-289`). A libiperf caller that retains a new test after the failed defaults call and then disposes it reaches the dangling traversal; an SCTP caller can also observe the stale selected-protocol pointer before disposal.

[S] Direct callers include `src/main.c:115`, `src/t_api.c:73`, `examples/mic.c:37`, `examples/mis.c:36`, and the `src/libiperf.3` examples at `75` and `93`. `main.c:115` ignores the return value; that caller contract defect is adjacent, not this rollback owner.

[N] No second- or third-allocation fault injection, memory-error trace, crash frequency, or production impact is recorded. Ordinary successful defaults and the first allocation failure are not claimed to fail.

## Evidence

[S] `protocol_new()` returns `NULL` only when `malloc(sizeof(struct protocol))` fails (`src/iperf_api.c:3406-3418`), and `protocol_free()` is raw `free()` (`3420-3424`). Neither rollback removes the linked allocations before freeing them.

[S] The conditional third-allocation path exists only under `#if defined(HAVE_SCTP_H)` (`3431-3433`, `3534-3552`); a non-SCTP build exposes only the second-allocation rollback.

[S] The SCTP-only stale selection is independent evidence of premature publication: `set_protocol(testp, Ptcp)` precedes the third allocation (`3532-3539`).

## API and compatibility

[S] These failures return `-1` with `testp` retained, and `iperf_free_test()` is the exported resource disposer. The correction must make that retained object safe to dispose and must leave no selected-protocol pointer to freed storage.

[A] The public examples call `iperf_defaults()` but do not state post-failure disposal semantics. Keep test-object ownership with the caller rather than making defaults free its argument.

[S] Preserve TCP/UDP/SCTP callback wiring, list order, default TCP selection, failure return value, and wire behavior.

## Prior art

[O] GitHub searches on 2026-08-14: upstream issues for `"iperf_defaults"` returned five results and pull requests returned three; searches for `"protocol_new"` returned zero issues and zero pull requests.

[S] Open https://github.com/esnet/iperf/issues/2063 concerns stale `i_errno` causing later `iperf_parse_arguments()` calls to fail. Its stated root, affected flow, and proposed reset are distinct from protocol allocation rollback and dangling list nodes.

[A] The remaining broad `iperf_defaults` search hits were not shown by their titles to own protocol allocation cleanup; that is insufficient to treat them as duplicates. No direct upstream issue or pull request is established.

[N] Re-search current upstream issues, pull requests, and their diffs for `protocol_new`, `iperf_defaults`, and allocation cleanup immediately before choosing an external target.

## Direction

[A] Keep protocol objects local until all enabled protocols have allocated and been configured. On any failure, free only local, unlinked objects; on success, link TCP, UDP, and optional SCTP in the existing order and then select TCP. This is smaller and safer than making `iperf_free_test()` recognize poisoned nodes.

## Bounds

[S] Do not skip nodes in `iperf_free_test()` or free `testp` inside `iperf_defaults()`; either changes the caller's ownership boundary instead of repairing the failed initialization transaction.

[S] In the SCTP failure branch, both the protocol list and `testp->protocol` must remain free of failed allocations.

[A] Missing `i_errno` assignment and `main.c` ignoring the defaults return are distinct error-reporting and caller-contract questions.

## Verification

Test decision: none. This is a ledger-only change; no source, test, validator, or gate was run.

[N] Use a focused libiperf fault-injection harness that fails exactly the second `protocol_new()` allocation; require `iperf_defaults()` to return `-1`, then call `iperf_free_test()` under a memory-error detector and verify no invalid traversal or free.

[N] In an SCTP-enabled build, fail exactly the third `protocol_new()` allocation; verify the same disposal path and that `testp->protocol` is not stale. In a non-SCTP build, record that this branch is compiled out.

[N] After a correction, exercise successful defaults in both configurations and verify the TCP/UDP/SCTP list order and default TCP selection.

## Missing

[N] No exact second-allocation fault-injection result and no SCTP third-allocation result establish the runtime invalid access at `upstream/master@c9b74229d0d9bfec6d2307b66b43c29a7665ad0b`.

[N] No current targeted upstream prior-art search or documented post-failure disposal contract establishes an external target.

## Resume

Index: Inject protocol rollback failures

Next: Fail the second `protocol_new()` allocation, dispose the retained test under a memory-error detector, and record the result.

Done when: The second-allocation result establishes or disproves invalid disposal; the SCTP third-allocation and stale-selection scope are separately recorded.
