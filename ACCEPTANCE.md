# Acceptance checklist (blueprint section 18, the four 2.0 security fixes, and the three review findings)

Checked on the dev machine (Windows 11, WSL2 Ubuntu 24.04, Docker Engine 29.1.3, strace 6.13, Python 3.12.3).
**Every number below comes from one run bound to one commit:**

| | |
|---|---|
| Commit tested | `6783e23ed84cebda7e62fb15a4e3f1a7a593f32b` (branch `security-fixes-and-redesign`) |
| Source hash | `0da2af7ee8e9b549eda4572c0b5ae51888e14800c054ffff1e9aa370f4751d89` (`bash scripts/source_hash.sh` on that commit prints the same value; docs and generated evidence are excluded from it) |
| Working tree | clean when the run started; source hash identical before and after the run |
| Results | doctor 9/9 PASS, **287 tests passed, none skipped**, 3-repeat benchmark, audit chain OK over 626 entries |
| Raw output | [`evidence/validation.log`](evidence/validation.log), re-runnable with `bash scripts/validate.sh` (refuses a dirty tree) |
| Independent check | `.github/workflows/ci.yml` rebuilds the sandbox and runs doctor, the full suite (fails if any test is skipped) and the benchmark on GitHub. **It has not run yet**, so this file does not claim it passes |

| # | Item | Status | Evidence |
|---|---|---|---|
| 1 | `shadowrun doctor` passes all checks | PASS | 9/9, and it gates every run (fail closed, tested) |
| 2 | Sandbox cannot reach the real internet | PASS | doctor check 3: `1.1.1.1` unreachable, `example.com` does not resolve |
| 3 | strace logs a canary `openat` and an outbound `connect` | PASS | doctor check 5 (tracer sidecar) and `tests/test_pipeline_e2e.py` |
| 4 | Sink stores sandbox traffic; no host port published | PASS | `docker port sink` prints nothing (`evidence/verify_demo_twice.txt`) |
| 5 | Hash mismatch rejected at install | PASS | `test_tampered_pin_is_rejected`: pip refuses, `HASH_MISMATCH`, BLOCK |
| 6 | No package code runs on the host | PASS (by construction) | fetch is HTTP + hashing; fixtures are assembled by `sdist_tool.py`, never built with `python -m build`. Not independently audited beyond that |
| 7 | Detector fires every rule on fixtures; pip baseline has zero findings | PASS | `tests/test_detector.py`, `test_trace_completeness.py`, `test_syscall_coverage.py`; both baselines recaptured through the sidecar tracer: complete, zero findings, no evasion syscalls |
| 8 | Strictest decision wins; errors fail closed | PASS | `tests/test_policy.py`; doctor-failure, unknown-package, parser-crash, failed-tracer e2e tests |
| 9 | 6 packages built with SIMULATED headers and plausible names | PASS | `tests/test_packages.py`, `packages/dist/INDEX.md` |
| 10 | M1 and M2 blocked with canary evidence; B1 to B3 allowed | PASS | e2e per manifest; benchmark matrix |
| 11 | Time bomb slip reported honestly | PASS | M3 ALLOW in every run; stated in README, results notes and demo script |
| 12 | Agent is blocked, switches package, finishes | PASS | `test_agent_blocked_then_switches_and_finishes`, `evidence/demo_run.txt` |
| 13 | Audit chain verifies and a tamper is detected | PASS | `tests/test_audit.py`; `audit verify` OK over 626 entries (validation) and 486 (demo run) |
| 14 | Dashboard works end to end; loading/empty/error states | PASS | 35 dashboard tests; redesign driven in the in-app browser (earlier session: 1440, 768, 375 px, no horizontal scroll). The Docker-down error path was exercised only through tests |
| 15 | Templates are autoescaped (XSS test passes) | PASS | `test_untrusted_strings_are_escaped_everywhere`; no `innerHTML`, no inline handlers (tested) |
| 16 | Benchmark table has counts, false positives, latency and the disclaimer | PASS | `eval/results/results.md`: no guard 0/3, name check 1/3, Shadow-Run 2/3 attacks stopped, 0/3 benign stopped |
| 17 | Demo runs twice without manual database fixes | PASS | `scripts/verify_demo_twice.sh`, both runs exit 0 (`evidence/verify_demo_twice.txt`) |
| 18 | **Fix 1:** one honest, consistent health status | PASS | `tests/test_health.py` (14) and 9 dashboard consistency tests. The original screenshot mismatch was not reproduced; causes found by code reading |
| 19 | **Fix 2:** the package cannot alter the trace evidence | PASS | `tests/test_evidence_isolation.py` (27): trace lives in another container's mount namespace; hostile wipe-your-own-trace package still BLOCKED with an intact trace; ownership/mode/hash checks |
| 20 | **Fix 3:** downloads only from the intended registry | PASS | `tests/test_fetcher.py` (68): allowlist, lookalike hosts, credentials, alternate IP spellings, hostile links and redirects, private/loopback resolution, DNS rebinding pinned to one IP |
| 21 | **Fix 4:** missing or unparsed evidence is uncertainty | PASS | `tests/test_trace_completeness.py` (22) and e2e: empty, malformed, truncated, unresumed, cut-off strings, missing pip exec, missing completion marker, failed tracer, parser crash all give a denied, indeterminate verdict |
| 22 | **Review 1:** root tracer privileges reduced or documented | PASS, residual risk documented | package container: host uid, `CapEff = CapBnd = 0`, no root process. Tracer sidecar: root with **only** `SYS_PTRACE` (no `SETUID`/`SETGID`), no network, no host mounts. Container settings asserted in `test_container_privileges_are_what_the_design_says`. Root + `CAP_SYS_PTRACE` in a shared PID namespace remains a risk against kernel/runtime bugs (README section 4, `CLAUDE.md`, `sandbox.py`); rootless Docker or gVisor is roadmap |
| 23 | **Review 2:** no unmonitored bypass syscalls | PASS, with stated limits | `tests/test_syscall_coverage.py` (43) and doctor check 9: `openat2`, `creat`, `sendto`, `linkat`, `memfd_create`, `execveat`, `io_uring_setup`, `mount` are recorded **and** detected inside the real sandbox. Not covered: read/write contents, `bind`, `socket` options, I/O through `io_uring` where the runtime allows it (Docker's default seccomp denied it here) |
| 24 | **Review 3:** validation bound to the tested source | PASS | `scripts/validate.sh` refused a dirty tree (exit 2, shown in the session), records commit, tree hash and source hash before and after, and fails if the source changed. The old log's "1 uncommitted file" was the log file itself, created before the status check |

Targeted regression run: [`evidence/regression_four_fixes.txt`](evidence/regression_four_fixes.txt).
Fresh demo with one consistent state: [`evidence/demo_status.txt`](evidence/demo_status.txt) (doctor check `#bab2caf9c9`
gates every run listed) and [`evidence/split_screen_demo.gif`](evidence/split_screen_demo.gif) (replay assembled from
real captures).

## Known gaps

- **Latency doubled:** median decision 2.6 s to 5.3 s (max 5.7 s), because `strace --seccomp-bpf` is refused when
  attaching to a running process. Breakdown in README section 9.
- M3 time bomb is a missed attack (needs a clock-shifting deep mode).
- No real-install step, approval UI, MODIFY, cache or npm support (roadmap).
- The benchmark is 6 simulated packages, 3 repeats. It demonstrates the mechanism, not real-world detection rates.
- The split-screen "recording" is a replay assembled from real screenshots and terminal output, not a live screen
  capture; no screen recorder (ffmpeg, OBS, Chrome) was available.
- The demo evidence (`demo_*`, GIF) was captured on commit `115b66d`; the only later source change is the `regression_four_fixes.sh` helper; the validation log is the
  authoritative record of what was tested.
- The CI workflow is untested until it runs on GitHub.
- The registry allowlist governs the fetcher. The doctor's registry probe and the agent talk to fixed localhost
  constants.
- A package can still force a denied verdict (for example by exhausting the trace size cap, or by keeping a process
  alive until the timeout); it cannot force an allow.
- `AF_UNIX` connects are parsed but not flagged; symlinks are not resolved; traces over 64 MB are refused
  (indeterminate verdict).
