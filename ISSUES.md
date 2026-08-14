# Issue and Pull Request Tracking

Read this index at the start of every agent session before repository work.
`FORMAT.md` owns research, lifecycle, drafting, implementation, and publication rules.
Each linked `issues/ISSUE-NNN.md` is the complete authoritative record for one root cause.
This file owns `Next finding ID` and projects current issue-file state.
`Next` is a 2–6 word projection of the issue record's `Resume/Next`.
When a row disagrees with its issue file, correct the row from the issue file in the same task.

Next finding ID: ISSUE-027

## Active

| ID | Finding | State | Mode | Target | Priority | Next | Location |
| --- | --- | --- | --- | --- | --- | --- | --- |
| [ISSUE-001](issues/ISSUE-001.md) | JSON lifecycle: rendered output survives persistent server reset | Published | Pull request | New pull request | High | Monitor PR 2067 | https://github.com/esnet/iperf/pull/2067 |
| [ISSUE-002](issues/ISSUE-002.md) | server output: result packaging repeatedly rescans accumulated text | Hold | Undecided | Undecided | Medium | Measure output assembly scaling | Not published. |
| [ISSUE-003](issues/ISSUE-003.md) | interval reporting: flush policy executes once per stream line | Hold | Undecided | Undecided | Medium | Measure interval flush costs | Not published. |
| [ISSUE-004](issues/ISSUE-004.md) | interval state: sole retained result node is reallocated every callback | Hold | Undecided | Undecided | Medium | Profile interval allocator churn | Not published. |
| [ISSUE-005](issues/ISSUE-005.md) | JSON reporting: unused human-readable units are formatted every interval | Hold | Undecided | Undecided | Medium | Profile JSON formatting waste | Not published. |
| [ISSUE-006](issues/ISSUE-006.md) | transfer workers: failed I/O exits without main-loop propagation | Hold | Undecided | Undecided | High | Reproduce worker exit | Not published. |
| [ISSUE-007](issues/ISSUE-007.md) | server duration timer: frees live stream before join | Hold | Undecided | Undecided | High | Reproduce timer free race | Not published. |
| [ISSUE-008](issues/ISSUE-008.md) | client completion: failed final control write is reported as success | Hold | Undecided | Undecided | High | Inject final control failure | Not published. |
| [ISSUE-009](issues/ISSUE-009.md) | control cleanup: preserve primary diagnostic | Hold | Undecided | Undecided | High | Preserve primary cleanup error | Not published. |
| [ISSUE-010](issues/ISSUE-010.md) | server JSON finish: terminal failure skips cleanup | Hold | Undecided | Undecided | High | Inject JSON cleanup failure | Not published. |
| [ISSUE-011](issues/ISSUE-011.md) | server workers: pthread-attribute cleanup falls through | Hold | Undecided | Undecided | Medium | Inject pthread attribute failures | Not published. |
| [ISSUE-012](issues/ISSUE-012.md) | server control EOF: handler return conflicts with error state | Hold | Undecided | Undecided | Medium | Define EOF return contract | Not published. |
| [ISSUE-013](issues/ISSUE-013.md) | parameter frames: sender omits 8-KiB limit | Hold | Undecided | Undecided | High | Validate encoded parameter size | Not published. |
| [ISSUE-014](issues/ISSUE-014.md) | Nrecv: select timeout has platform-dependent deadline | Hold | Undecided | Undecided | High | Verify per-gap timeout policy | Not published. |
| [ISSUE-015](issues/ISSUE-015.md) | control errors: short error words are decoded | Hold | Undecided | Undecided | High | Verify short error rejection | Not published. |
| [ISSUE-016](issues/ISSUE-016.md) | JSON control frames: short writes are accepted | Hold | Undecided | Undecided | Medium | Force partial JSON writes | Not published. |
| [ISSUE-017](issues/ISSUE-017.md) | control sockets: valid high descriptors exceed `fd_set` | Hold | Undecided | Undecided | High | Verify high control fds | Not published. |
| [ISSUE-018](issues/ISSUE-018.md) | control receive: returning `EINTR` can yield a short count | Hold | Undecided | Undecided | Low | Reproduce returning signal control read | Not published. |
| [ISSUE-019](issues/ISSUE-019.md) | result JSON: receiver length is uncapped | Hold | Undecided | Undecided | Low | Measure result-frame allocation reach | Not published. |
| [ISSUE-020](issues/ISSUE-020.md) | stream adoption: failed construction loses data socket | Hold | Undecided | Undecided | High | Audit pre-adoption socket cleanup | Not published. |
| [ISSUE-021](issues/ISSUE-021.md) | Server lifecycle: stale stream descriptor after TEST_END | Hold | Undecided | Undecided | High | Invalidate terminal stream descriptors | Not published. |
| [ISSUE-022](issues/ISSUE-022.md) | UDP GSO/GRO: effective mmap length is lost | Hold | Undecided | Undecided | High | Record effective mmap length | Not published. |
| [ISSUE-023](issues/ISSUE-023.md) | Defaults initialization: freed protocol nodes remain linked | Hold | Undecided | Undecided | High | Inject protocol rollback failures | Not published. |
| [ISSUE-024](issues/ISSUE-024.md) | JSON lifecycle: failed aggregates lack rollback | Hold | Undecided | Undecided | Medium | Inject aggregate ownership failures | Not published. |
| [ISSUE-025](issues/ISSUE-025.md) | API setters: owned values leak on replacement | Hold | Undecided | Undecided | Medium | Exercise replacement ownership matrix | Not published. |
| [ISSUE-026](issues/ISSUE-026.md) | Test destruction: pidfile and authorized-users survive free | Hold | Undecided | Undecided | Low | Confirm final destructor fields | Not published. |

## Terminal

| ID | Finding | State | Mode | Target | Priority | Outcome | Location |
| --- | --- | --- | --- | --- | --- | --- | --- |
