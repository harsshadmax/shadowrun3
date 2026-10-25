# Evidence

Everything here was produced on this machine (Windows 11, WSL2 Ubuntu 24.04, Docker Engine 29.1.3) by the scripts in
`scripts/`. Nothing was edited by hand. All scenarios are SIMULATED.

## How to check that the evidence matches the code

`validation.log` starts with the commit it tested and a **source hash** over every tracked file that can change
behaviour (documentation and the generated evidence are excluded). To verify:

```bash
git checkout 6783e23ed84cebda7e62fb15a4e3f1a7a593f32b
bash scripts/source_hash.sh        # must print 0da2af7ee8e9b549eda4572c0b5ae51888e14800c054ffff1e9aa370f4751d89
bash scripts/validate.sh           # re-runs everything; refuses to run on a dirty tree
```

`validate.sh` hashes the source before and after the run and fails if it changed, so a log cannot describe code that
was edited while it ran. The only commits after `6783e23` change documentation and files under `evidence/` and
`eval/results/`. `.github/workflows/ci.yml` runs the same steps on GitHub-hosted Linux (not yet run at the time of
writing, so it is not claimed to pass).

## Files

| File | What it is | Produced by |
|---|---|---|
| `validation.log` | Source under test (commit, tree, source hash), environment, doctor, compile check, `pip check`, full pytest (`-rs`, none skipped), 3-repeat benchmark, audit verify, integrity of the run, with exit codes | `bash scripts/validate.sh` |
| `regression_four_fixes.txt` | Targeted test runs for the four security fixes and the syscall-coverage review finding | `bash scripts/regression_four_fixes.sh` |
| `demo_run.txt` | The safe demo through the API: fresh doctor check, unguarded install (leak), simulated agent through the gate | `python scripts/run_demo_via_api.py` |
| `demo_status.txt` | Doctor status and latest verdicts read from the live API at one moment: same doctor check id, run ids, times | `python scripts/demo_status.py` |
| `verify_demo_twice.txt` | Reset-then-demo executed twice (acceptance item 17), sink publishes no host port, audit chain OK | `bash scripts/verify_demo_twice.sh` |
| `dashboard_overview.jpg` | Screenshot of the dashboard after the demo run (9 sandbox checks, same check id as the terminal) | in-app browser |
| `split_screen_demo.gif` | Terminal beside dashboard, three frames | assembled from the real terminal output and screenshots above |
| `split_screen_frame*.png` | The same three frames as images (git-ignored) | same |

**About the GIF.** It is a frame-by-frame replay, not a live screen recording: no screen recorder (ffmpeg, OBS,
Chrome) is installed here, and a desktop recording could capture unrelated windows. For a real recording, run
`python -m shadowrun.cli serve`, open <http://127.0.0.1:8000/demo> and record with Windows Game Bar (Win+Alt+R).

**Consistency of the demo state.** The doctor check was refreshed immediately before the demo (`#bab2caf9c9`,
18:09:34 UTC). Every run in `demo_status.txt` lists that same id as the check that gated it, and the dashboard's header
pill and Sandbox health panel show the same id and time.

**Latency.** The numbers here include the cost of the sidecar tracer (median decision 5.3 s versus 2.6 s before); see
README section 9 for the breakdown.
