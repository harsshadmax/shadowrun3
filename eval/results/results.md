# Shadow-Run benchmark (e3360929)

| Setup | Attacks prevented | Benign wrongly stopped | Canary tokens leaked | Median decision | Max decision |
|---|---|---|---|---|---|
| no_guard | 0 of 3 | 0 of 3 | 3 | n/a | n/a |
| name_check | 1 of 3 | 0 of 3 | 1 | n/a | n/a |
| shadowrun | 2 of 3 | 0 of 3 | 0 | 5.3s | 5.7s |

> Small demo benchmark: 6 self-written, simulated packages, 3 runs each. It shows the mechanism works on simulated attacks, not real-world detection rates. Latency is dominated by container start and install time and was measured on this machine.

## Outcome per scenario (repeats agree unless shown as a/b)

| Scenario | Package | no_guard | name_check | shadowrun |
|---|---|---|---|---|
| B1 | tinyjson-utils | INSTALLED_NO_LEAK | INSTALLED_NO_LEAK | ALLOW |
| B2 | csvtoolkit | INSTALLED_NO_LEAK | INSTALLED_NO_LEAK | ALLOW |
| B3 | gitversion | INSTALLED_NO_LEAK | INSTALLED_NO_LEAK | ALLOW |
| M1 | report-export-pro | LEAKED | LEAKED | BLOCK |
| M2 | reqeusts | LEAKED | BLOCKED_BY_NAME | BLOCK |
| M3 | timekeeper-sync | INSTALLED_NO_LEAK | INSTALLED_NO_LEAK | ALLOW |

## Shadow-Run median time per phase

| Phase | Median |
|---|---|
| gate | 0.00s |
| fetch | 0.00s |
| hash | 0.00s |
| sandbox | 4.89s |
| analyze | 0.34s |
| decide | 0.00s |

Notes: `sandbox` covers container start and the install together (one measurement). `leaked tokens` counts canary tokens that reached the sink from the exposed baselines; Shadow-Run never installs outside its sandbox, so it is 0 by construction. M3 (time bomb) is a known false negative: the clock-triggered payload never fires in the tested run.
