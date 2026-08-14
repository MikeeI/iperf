# ISSUE-017 — control sockets: valid high descriptors exceed `fd_set`

State: Hold
Mode: Undecided
Target: Undecided
Location: Not published.
Priority: High
Confidence: Medium
Type: reliability
Created: 2026-08-14
Updated: 2026-08-14
Source: `upstream/master@c9b74229d0d9bfec6d2307b66b43c29a7665ad0b`

## Root

Root [S]: Control-read paths pass descriptors through `fd_set` macros without enforcing their defined range. `Nrecv` unconditionally calls `FD_SET(fd, ...)` before its initial and post-progress waits (`src/net.c:419-505`). More importantly for ordinary control flow, `iperf_run_client` retains any nonnegative caller-supplied `test->ctrl_sck` and calls `FD_SET` on it before message handling (`src/iperf_client_api.c:437-473`); the server does the same for an accepted control socket (`src/iperf_server_api.c:167-203`). A nonnegative open control descriptor `>= FD_SETSIZE` is valid to the operating system but outside the `fd_set` macro domain. A negative descriptor violates the same macro precondition, but it is invalid input rather than a valid-high-descriptor case.

## Reach and impact

Reach [S]: A libiperf caller can supply a connected control socket through public `iperf_set_control_socket`; `iperf_run_client` creates a replacement only when `ctrl_sck < 0`, then adds the retained descriptor to `test->read_set` (`src/iperf_api.c:455-459`, `src/iperf_client_api.c:437-473`). Separately, a server can accept a high-numbered control socket and add it to `test->read_set` (`src/iperf_server_api.c:167-203`). Both standard paths reach an `FD_SET` before `Nrecv`.
Reach [A]: A descriptor-heavy process, inherited descriptors, or an embedding can make a client-created or caller-supplied control socket `>= FD_SETSIZE`; no supported-platform harness has run that setup.
Impact [S]: POSIX defines `FD_SET` behavior as undefined when `fd < 0` or `fd >= FD_SETSIZE`; it does not promise a recoverable `select` error for either precondition. See `https://pubs.opengroup.org/onlinepubs/9699919799/functions/FD_CLR.html`.
Impact [N]: No valid-high-control-fd harness has recorded a concrete result on Ubuntu Linux, FreeBSD, or macOS, and no affected-embedding count is measured.

## Evidence

- [S] `src/iperf_client_api.c:437-473` — `iperf_run_client` retains a nonnegative supplied control socket and calls `FD_SET(test->ctrl_sck, &test->read_set)`; `src/iperf_client_api.c:686-729` later uses that set with `select`, `FD_ISSET`, and `FD_CLR`.
- [S] `src/iperf_server_api.c:167-203` — an accepted control socket is added with `FD_SET`; `src/iperf_server_api.c:741-769` tests and clears it through the server `select` loop.
- [S] `src/net.c:421-437` and `src/net.c:476-501` — `Nrecv` repeats an unbounded `FD_SET`/`select` readiness wait before its initial read and after partial progress.
- [S] `src/iperf_api.c:455-459` assigns the public setter argument directly to `ipt->ctrl_sck`; `src/iperf_api.h:179-182` exports that setter.
- [S] `https://pubs.opengroup.org/onlinepubs/9699919799/functions/FD_CLR.html` states that `FD_SET` behavior is undefined for a descriptor less than zero or greater than or equal to `FD_SETSIZE`.
- [S] `configure.ac:121-122` checks for `poll.h`, while `src/net.c:66-68` conditionally includes it and `src/net.c:89-125` already uses `poll` for `timeout_connect`; `poll` is the existing portability seam.
- [N] No valid `>= FD_SETSIZE` control-fd harness has run, and no supported-platform result is observed.

## Prior art

Coverage [O]: Directly reviewed merged `https://github.com/esnet/iperf/pull/1709`, `https://github.com/esnet/iperf/issues/1986`, active `https://github.com/esnet/iperf/pull/1654`, and related active `https://github.com/esnet/iperf/pull/1861` on 2026-08-14; also reviewed the POSIX `FD_SET` contract at `https://pubs.opengroup.org/onlinepubs/9699919799/functions/FD_CLR.html`.
Gaps [N]: GitHub-wide search for `FD_SETSIZE`, `FD_SET`, `Nrecv`, and high-fd libiperf use; GitHub Discussions; `iperf-dev` archives; releases; and unpublished active branches remain unchecked.

- [S] `https://github.com/esnet/iperf/pull/1709` owns JSON receive framing, not readiness multiplexing or high valid descriptors.
- [S] `https://github.com/esnet/iperf/issues/1986`, `https://github.com/esnet/iperf/pull/1654`, and `https://github.com/esnet/iperf/pull/1861` own worker-thread failure propagation/reporting, not public control-fd admission or `fd_set` bounds.
- [N] No direct upstream issue, PR, or commit for `iperf_set_control_socket` with valid descriptors `>= FD_SETSIZE` is recorded yet.

Target fit [N]: Undecided until a high-fd harness establishes supported-platform behavior, broader prior-art coverage is complete, and the user selects a mode.

## Direction

Direction [S]: The smallest complete valid-high-control-fd correction must remove every `fd_set` operation that can receive `ctrl_sck`, not only the two readiness probes in `Nrecv`: client setup and its event loop, server acceptance and its event loop, and `Nrecv`. A `Nrecv`-only conversion leaves the normal client and server paths outside the macro domain before that helper runs. Use the existing `poll` portability seam for the affected readiness owners while preserving valid-descriptor `Nrecv` results: initial timeout `0`, positive short counts after progress, EOF handling, and `NET_HARDERROR`.
Boundary [S]: Reject a negative descriptor as `NET_HARDERROR` before a `poll` wait so `poll` cannot silently ignore invalid input. Do not extend that invalid-input check to valid descriptors `>= FD_SETSIZE`.
Rejected [S]: Rejecting `fd >= FD_SETSIZE` before `FD_SET` avoids undefined behavior only by breaking the valid-high-descriptor case. Replacing only `Nrecv` likewise is incomplete. Negative-descriptor handling is a separate invalid-input boundary and must not determine the valid-high-descriptor design.

## Bounds

- Preserve [S]: A nonnegative valid control descriptor, including one `>= FD_SETSIZE`; public `iperf_set_control_socket` retention; control framing; initial and post-partial readiness timeouts; EOF/positive-short returns; and native hard-error handling for valid descriptors.
- Preserve [S]: Existing lower-fd control behavior and `Nrecv`'s timeout return of `0` before data and partial-count return after some data.
- Change [S]: A negative descriptor receives deterministic `NET_HARDERROR` rather than being passed to an undefined `fd_set` macro or silently ignored by `poll`.
- Exclude [S]: Control-socket ownership/closing policy, descriptor allocation, protocol-wire changes, and conversion of unrelated socket helpers that cannot receive `ctrl_sck`.
- Cost [N]: A poll-based event-loop mapping must preserve readiness ordering, error events, and timeout conversion on Ubuntu Linux, FreeBSD, and macOS.

## Verification

Test decision: none. Planned focused harnesses, not an existing-test update:

- [N] Discover `FD_SETSIZE` at compile time; create a connected control socket duplicated to a descriptor `>= FD_SETSIZE` within `RLIMIT_NOFILE`, set it through `iperf_set_control_socket`, and run the client control exchange so setup and event-loop paths are covered.
- [N] Run that high-fd exchange on Ubuntu Linux, FreeBSD, and macOS. Record only the observed baseline result; do not infer a particular undefined-behavior shape in advance.
- [N] On the candidate, verify low- and high-valid-fd control exchanges; initial timeout, partial-read timeout, EOF, read-side `EINTR`, and closed-descriptor hard-error behavior; and negative-fd `NET_HARDERROR`.
- [N] Compare complete control-message exchanges and partial `Nread` return counts before and after the readiness-owner change.
- [N] Configure and compile on every supported platform to prove the `HAVE_POLL_H` boundary instead of assuming Linux `FD_SETSIZE` or `poll` behavior generalizes.

## Missing

- [N] Reproduction with a valid high control descriptor and an observed baseline result on each supported platform.
- [N] Candidate results covering every control-socket `fd_set` owner, including readiness/error/timeout mapping.
- [N] `HAVE_POLL_H` supported-platform configuration evidence and affected-embedding reach.
- [N] Broader upstream prior-art coverage.
- [N] User selection of Report or Pull request mode and exact target.

## Resume

Index: Verify high control fds
Next: Run an end-to-end valid-high-control-fd harness through the public setter on each supported platform.
Done when: Recorded results establish baseline behavior and show a candidate preserves low/high valid control exchanges without any `fd_set` operation on `ctrl_sck`.

## API and compatibility

Callers [S]: libiperf consumers can call public `iperf_set_control_socket(struct iperf_test *ipt, int ctrl_sck)` (`src/iperf_api.h:179-182`, `src/iperf_api.c:455-459`); `iperf_run_client`, `iperf_accept`, their event loops, and `Nread`/`Nrecv` subsequently use the control descriptor.

Contract [S]: `iperf_run_client` preserves any nonnegative supplied control descriptor, but current `fd_set` operations impose an undocumented and undefined `FD_SETSIZE` boundary. `Nrecv` owns valid-descriptor readiness and partial-count behavior; negative descriptors are invalid input.

Compatibility: Preserve complete control-wire behavior and lower-fd operation while allowing valid high control descriptors without an `fd_set` bound. The correction must not treat an invalid negative descriptor as a valid high descriptor.

Migration: None.
