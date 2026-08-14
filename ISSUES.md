# Issue and Pull Request Tracking

Read this index at the start of every agent session before repository work.
`FORMAT.md` owns research, lifecycle, drafting, implementation, and publication rules.
Each linked `issues/ISSUE-NNN.md` is the complete authoritative record for one root cause.
This file owns `Next finding ID` and projects current issue-file state.
`Next` is a 2–6 word projection of the issue record's `Resume/Next`.
When a row disagrees with its issue file, correct the row from the issue file in the same task.

Next finding ID: ISSUE-006

## Active

| ID | Finding | State | Mode | Target | Priority | Next | Location |
| --- | --- | --- | --- | --- | --- | --- | --- |
| [ISSUE-001](issues/ISSUE-001.md) | JSON lifecycle: rendered output survives persistent server reset | Hold | Undecided | Undecided | High | Reproduce JSON reset leak | Not published. |
| [ISSUE-002](issues/ISSUE-002.md) | server output: result packaging repeatedly rescans accumulated text | Hold | Undecided | Undecided | Medium | Measure output assembly scaling | Not published. |
| [ISSUE-003](issues/ISSUE-003.md) | interval reporting: flush policy executes once per stream line | Hold | Undecided | Undecided | Medium | Measure interval flush costs | Not published. |
| [ISSUE-004](issues/ISSUE-004.md) | interval state: sole retained result node is reallocated every callback | Hold | Undecided | Undecided | Medium | Profile interval allocator churn | Not published. |
| [ISSUE-005](issues/ISSUE-005.md) | JSON reporting: unused human-readable units are formatted every interval | Hold | Undecided | Undecided | Medium | Profile JSON formatting waste | Not published. |

## Terminal

| ID | Finding | State | Mode | Target | Priority | Outcome | Location |
| --- | --- | --- | --- | --- | --- | --- | --- |
