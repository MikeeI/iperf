# ISSUE-025 — API setters: owned values leak on replacement

State: Hold
Mode: Undecided
Target: Undecided
Location: Not published.
Priority: Medium
Confidence: High
Type: API
Created: 2026-08-14
Updated: 2026-08-14
Source: `upstream/master@c9b74229d0d9bfec6d2307b66b43c29a7665ad0b`

## Root

[S] Fifteen public string/auth/RSA setter entry points overwrite an owned field with `strdup()` or a loaded `EVP_PKEY` without releasing the prior owner: `logfile` (`src/iperf_api.c:502-505`), `timestamp_format` (`598-601`), `server_hostname` (`663-666`), `tmp_template` (`669-672`), client username/password/public-key setters (`742-763`), server authorized-users/private-key setters (`766-787`), bind address/device (`791-800`), extra data (`821-824`), and congestion (`868-871`).

[S] A second successful setter call loses the preceding allocation or key object's only pointer. The root is missing replacement ownership at each configuration entrypoint, not omission of the final stored value from destruction.

[S] The same invariant is absent from repeatable parser assignments: `--extra-data`, `--title`, `--congestion`/`--linux-congestion`, `--pidfile`, `--logfile`, and, with `HAVE_SSL`, `--username`, `--rsa-public-key-path`, `--rsa-private-key-path`, and `--authorized-users-path` overwrite a field or deferred parser holder (`src/iperf_api.c:1142-1178,1525-1528,1676-1703,1744-1759`).

## Reach and impact

[S] The setters are public libiperf API (`src/iperf_api.h:188-238`), and libiperf documents a caller-owned test that is configured, run, reset, and freed (`src/libiperf.3:67-103`). A caller can configure a live test more than once before a run or between reused runs.

[S] Duplicate parser options re-enter these switch cases without rejection. The SSL `client_username`, `client_rsa_public_key`, and `server_rsa_private_key` parser holders are overwritten before their later transfer/load (`src/iperf_api.c:1744-1759,1843-1916`).

[S] Each successful replacement loses exactly one preceding successful allocation or key object. Repeated persistent-test configuration amplifies that loss; the bounded final `pidfile` and `server_authorized_users` omissions belong to ISSUE-026.

[N] No repeated-setter or duplicate-option leak measurement, allocation-failure result, or user-visible impact is recorded.

## Evidence

[S] Later field-specific cleanup can reclaim only the final pointer: `iperf_free_test()` frees `logfile`, `server_hostname`, `tmp_template`, client auth values, RSA keys, bind values, extra data, congestion, and timestamp format (`src/iperf_api.c:3578-3631`).

[S] `iperf_reset_test()` frees only a mutable subset, including congestion, client auth, title, and extra data (`src/iperf_api.c:3739-3744,3799-3828`); no listed setter releases the old value at assignment.

[S] The public setter declarations return `void` (`src/iperf_api.h:188-238`). Current string setters directly assign `strdup()` and key setters directly assign loader results, so an acquisition failure can overwrite a prior value without an API-visible error result.

## Prior art

[S] Recorded audit candidates are https://github.com/esnet/iperf/issues/1986, active https://github.com/esnet/iperf/pull/1654, and https://github.com/esnet/iperf/pull/1861 for worker-failure propagation; https://github.com/esnet/iperf/pull/1709 introduced the 8-KiB parameter receive cap. None owns repeated setter or option-value replacement.

[N] No targeted current upstream issue/PR search for these setter and parser owners is recorded. No external target is selected.

## Direction

[A] At each owner, acquire a string or key into a local candidate, then release the exact prior owner (`free` or `EVP_PKEY_free`) only after the candidate succeeds and install it. Do not add a generic helper: string duplication and RSA loading have distinct error and release contracts.

[A] Because the public setters cannot report acquisition failure and no source contract says a failed setter clears a valid prior value, preserve that prior value on a failed candidate unless compatibility evidence selects another policy. For parser-owned fields and deferred SSL holders, release the superseded local/field on every successful duplicate and give failed acquisition an explicit parser error path rather than silently losing either value.

## Bounds

[S] Preserve public setter signatures, copied-input/key-loading behavior, `HAVE_SSL` guards, parser spellings, final selected values after successful duplicate options, and normal output.

[S] ISSUE-026 owns only destruction of the final `pidfile` and `server_authorized_users` allocations. This record owns a prior value lost by replacement, including repeated authorized-users setter/parser assignment.

[N] Do not merge cJSON lifecycle cleanup, generic allocation policy, or final destructor omissions into this replacement-owner finding.

## Shared change pressure

Copies [S]: 15 public setters and 9 direct parser assignments overwrite an owned string/key or deferred owned parser value without releasing its predecessor.

Pressure [S]: Every repeated configuration path needs exactly one current owned value; the string and RSA sites share that invariant but retain type-specific allocation and release behavior.

Drift [S]: Existing reset/free cleanup releases selected final values, while no entrypoint handles a superseded owner.

Owner: Each setter and parser-value owner in `src/iperf_api.c`, with local type-specific replacement.

Cost: Explicit local candidates and releases at the existing owners; a shared wrapper would obscure the distinct `strdup()` and `EVP_PKEY` contracts.

## API and compatibility

Callers [S]: libiperf consumers call the setter declarations in `src/iperf_api.h:188-238`; CLI users reach the parser assignments in `src/iperf_api.c:1142-1178`.

Contract [S]: Setters are `void` and copy/load into test-owned storage; duplicate parser options overwrite the prior selected value when acquisition succeeds.

Compatibility: Keep source/binary API, option grammar, successful last-value-wins behavior, and normal output unchanged.

Migration: None.

## Verification

Test decision: none. This is a ledger-only change; no source, test, validator, or gate was run.

[N] In one live test, call each of the 15 setters twice with distinct valid strings or keys; verify the second value is effective and a leak detector reports no predecessor after `iperf_free_test()`. Cover RSA setters in an OpenSSL build.

[N] In fresh CLI/parser scenarios, repeat each of the 9 direct options, complete parsing, and verify the final value while a leak detector reports no superseded direct field or deferred SSL holder.

[N] Fault-inject the second `strdup()` and each key-load acquisition after a successful first setter call. Record whether the retained value remains usable and the compatible failure policy before implementation; separately fault-inject duplicate parser acquisition and verify its selected error cleanup.

[N] Reconfigure and reset a persistent test repeatedly, then dispose it, to distinguish per-replacement losses from ISSUE-026 final-value omissions.

## Missing

[N] No complete 15-setter/9-option repetition matrix, second-acquisition fault-injection result, persistent-reuse result, retained-allocation measurement, targeted upstream prior-art currentness, or maintainer-selected correction/target is recorded.

## Resume

Index: Exercise replacement ownership matrix

Next: Call the 15 setters and 9 duplicate options twice under a leak detector, including OpenSSL key paths.

Done when: Results distinguish successful replacement leaks, second-acquisition behavior, parser cleanup, and final-value destructor omissions.
