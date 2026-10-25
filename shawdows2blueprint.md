# Shadow-Run One-Day Blueprint (shawdows2blueprint.md)

Behavioural Verification Gate for AI Coding Agents. Self-contained build spec for a **one-day** demo prototype, written for Claude Code. This is a cut-down version of the full blueprint (`shadowblueprint.md`); where the two differ, this file wins.

Stack: Python 3.10+, FastAPI, Pydantic v2, Docker SDK on WSL2/Linux, strace (inside the container), pypiserver, SQLite, YAML policy, Jinja + vendored HTMX/Chart.js dashboard, pytest.

Honest claim: Shadow-Run **reduces the chance and the blast radius** of a malicious install. It does not prove safety. All numbers come from a small, self-written, simulated benchmark.

---

## 1. Scope

### 1.1 Build (in priority order)

1. Docker analysis sandbox on an internal network, fake secrets, a logging sink, and a `doctor` self-test.
2. strace parser and detector.
3. HTTP fetch of the package, SHA-256 hash pin, offline install in the sandbox.
4. Policy engine: ALLOW, BLOCK, ESCALATE (ESCALATE = denied, no approval UI).
5. SQLite storage and a hash-chained audit log.
6. Six simulated packages on a local pypiserver.
7. CLI `shadowrun pip install` and a scripted agent.
8. Dashboard: Live Gate page and Split-Screen Demo page.
9. Mini benchmark: no guard vs name check vs Shadow-Run.

### 1.2 Not built (roadmap, say so in the pitch)

MODIFY, quick/deep modes and the verdict cache, the other evasive packages, replay, approval UI, LLM explainer, Claude Code hook, shell commands, re-scan, a contained "real install" step, npm, MCP server, gVisor/Firecracker.

### 1.3 Cut order if you run late

1. Benchmark repeats go from 3 to 1.
2. Audit hash chain becomes a plain log table.
3. The dashboard becomes the split-screen page only.
4. If strace fails after about 3 hours: **sink-only detection** (catches sending, not reading). State this in the pitch.

---

## 2. Architecture

### 2.1 Containers

| Role | Network | Home | strace | Used for |
|---|---|---|---|---|
| `analysis` | `shadowrun_net` (internal: reaches only the sink) | per-run fake home with canaries and decoys | yes | Shadow-Run's check |
| `exposed` | `shadowrun_net` | same fake home | no | `no_guard` and `name_check` baselines and the unguarded half of the demo |
| `sink` | `shadowrun_net` only | n/a | n/a | Logs every request it receives |

No package code ever runs on the host. Fetching is HTTP plus hashing only.

### 2.2 Pipeline for `pip install <spec>`

1. Parse the spec and resolve the version from the local registry.
2. Fetch the file over HTTP from pypiserver (`GET /simple/<normalized-name>/`, PEP 503 normalization: lowercase, runs of `-_.` become `-`). **Never use `pip download`**: for source packages it can run `setup.py` on the host.
3. SHA-256 the file.
4. Create canaries; start the `analysis` container with the file mounted read-only.
5. Install offline inside the container, hash-required, under strace.
6. Read the strace log and the sink log; build findings.
7. Policy decides; write the run, findings and audit entries.
8. Return the verdict and timings. `decision_ms` is receive-to-decision.

### 2.3 Repo layout

```
shadow-run/                  # keep in the WSL filesystem (~/shadow-run), not /mnt/c
├── CLAUDE.md
├── shawdows2blueprint.md
├── pyproject.toml  .env.example  .gitignore
├── shadowrun/
│   ├── cli.py  config.py  models.py  fetcher.py  sandbox.py  canary.py
│   ├── monitor.py  detector.py  policy.py  audit.py  pipeline.py  db.py  api.py
├── sink/                    # app.py + Dockerfile
├── docker/sandbox.Dockerfile
├── packages/src/…  packages/dist/  packages/build_all.py
├── policies/default.yaml
├── agent/scripted_agent.py
├── dashboard/templates/  dashboard/static/
├── eval/baselines.py  eval/run_eval.py  eval/results/
├── scripts/up.sh  scripts/vendor_assets.sh
├── .data/                   # fetch cache, traces, sink log (gitignored)
└── tests/fixtures/          # recorded strace + sink logs
```

---

## 3. Safety rules (override everything)

1. Test packages are inert. Each source file starts with `# SIMULATED TEST FIXTURE. INERT OUTSIDE THE SHADOW-RUN SANDBOX.` Package **names are plausible** (no "SIMULATED" in names, so the name-check baseline cannot cheat); the word SIMULATED lives in the description and manifest.
2. Payloads only read fake files inside the sandbox and only send them to the hostname `sink`. Nothing works against a real host.
3. `shadowrun_net` is `internal=True`. `doctor` must pass before any run; if it fails, refuse to run (fail closed).
4. No Docker socket in containers. Non-root user, `cap_drop ALL` plus `SYS_PTRACE` (analysis only), `no-new-privileges`, memory/CPU/pids limits, hard timeout, container always removed.
5. Package names, metadata and sink contents are untrusted: Jinja autoescape everywhere.
6. Any error ends in BLOCK/ESCALATE, never ALLOW. No LLM is in the decision path.

---

## 4. Sandbox specification

### 4.1 Images

- `shadowrun-sandbox`: python:3.11-slim, `strace`, `git`, pip, **`setuptools` and `wheel` preinstalled**.
- `shadowrun-sink`: python:3.11-slim, FastAPI + uvicorn running `sink/app.py`.
- pypiserver on the host: `pypi-server run -p 8080 -a . -P . packages/dist`.

Prebuild images in `scripts/up.sh`; never build during a timed run.

### 4.2 Install command inside the container

```
strace -f --seccomp-bpf -ttt -yy -s 200 \
  -e trace=openat,open,connect,execve,rename,unlink,mkdir,symlink,chmod \
  -o /out/trace.log \
  pip install --no-index --no-build-isolation --disable-pip-version-check --no-cache-dir -q \
              --find-links /cache --no-deps --require-hashes \
              -r /work/req.txt --target /work/site
```

- `/work/req.txt`: `<name>==<version> --hash=sha256:<sha256>`.
- `--no-build-isolation` is **required**; without it pip tries to download setuptools and fails offline.
- Drop `--seccomp-bpf` if doctor shows it fails.
- A hash mismatch makes pip refuse; record `HASH_MISMATCH`.

### 4.3 Docker settings (`containers.run`)

| Setting | Value |
|---|---|
| network | `shadowrun_net` |
| user | host `uid:gid` (so the bind-mounted home is writable) |
| read_only | `True` |
| tmpfs | `/tmp` and `/work` with `rw,exec,nosuid,size=128m` (**`exec` required**; tmpfs defaults to `noexec`) |
| capabilities | drop `ALL`; add `SYS_PTRACE` for analysis |
| security_opt | `no-new-privileges:true` |
| limits | 512m memory, 1 CPU, 128 pids |
| volumes | `.data/fetch/<sha>/` → `/cache` (ro); per-run temp dir → `/home/sandbox` (rw, deleted after); `.data/trace/<run>/` → `/out` (rw) |
| env | `HOME=/home/sandbox`, fake `USER`, `GIT_AUTHOR_EMAIL` |
| timeout | 30 s; on expiry kill and add `SANDBOX_TIMEOUT` |

Do not nest mounts under `/home/sandbox`. Remove the container and temp dir in a `finally` block.

### 4.4 `shadowrun doctor` (each check prints PASS/FAIL; any FAIL aborts)

1. Docker reachable; images present.
2. `shadowrun_net` exists and `Internal == true`.
3. A throwaway container cannot reach `1.1.1.1:443` or resolve/reach `example.com`.
4. The same container can reach `sink:9000`.
5. strace works in the container and records a known `openat`; `--seccomp-bpf` works (else disable it).
6. Rootfs is read-only; `/tmp` is executable.
7. pypiserver reachable from the host; the sink data directory is writable.

---

## 5. Canaries and sink

### 5.1 Fake home (per run, in a temp dir)

- **Canaries (alarm):** `.ssh/id_rsa`, `.aws/credentials`, `.env`, `.npmrc`. Each contains a unique token `CANARY-<run_token>-<kind>`. Formats look plausible but are generated, never real.
- **Decoys (no alarm):** `.gitconfig`, `.bash_history`, `.bashrc`, `.profile`, a tiny fake project. Benign tools like `git` read decoys, so decoys must never trigger findings.
- The home is **writable** so persistence attempts succeed inside the throwaway home and are observed; it does not look read-only to a sandbox detector.

### 5.2 Sink

- Container `sink` on `shadowrun_net` only. **Nothing is published to the host** (internal networks cannot publish ports).
- Listens on `0.0.0.0:9000`, accepts any method and path, appends one JSON line per request to `/data/sink.jsonl`: `ts, source_ip, method, path, headers, body` (body truncated to 64 KB), flushing each write.
- Host directory `.data/sink/` is mounted at `/data`; the gateway reads `sink.jsonl` from disk.

### 5.3 Attribution

After starting the analysis container, read its IP on `shadowrun_net` (`container.attrs`). Sink events for a run are those from that source IP in the run window. `CANARY_EXFIL` fires when a `CANARY-<run_token>-*` string appears in a sink event body, path or headers.

---

## 6. Detector

### 6.1 Parser (`monitor.py`)

Parse `trace.log` into events with `ts, pid, syscall, args, ret, kind, path/addr/port/argv`. Kinds: `FILE_OPEN`, `FILE_WRITE` (open flags include `O_WRONLY`, `O_RDWR`, `O_CREAT`, `O_TRUNC`), `NET_CONNECT`, `PROC_EXEC`. Merge `<unfinished ...>` / `<... resumed>` lines per pid. Parse `connect` (`sa_family`, `htons(port)`, `inet_addr`, IPv6, `AF_UNIX`) and `execve` (binary + argv). Unknown lines count in `parse_warnings`; never crash. **Detect on attempts, regardless of return value.**

### 6.2 Rules

| Rule | Trigger | Severity |
|---|---|---|
| `CANARY_EXFIL` | Run token found in sink data | critical |
| `HASH_MISMATCH` | pip rejected the file | critical |
| `CANARY_READ` | Open of a canary file by package code | high |
| `NET_OUTBOUND` | `connect` to anything except loopback and the resolver `127.0.0.11` (the install is offline, so no connect is legitimate; the sink counts) | high |
| `PERSISTENCE_WRITE` | Write to `.bashrc`, `.profile`, autostart, cron paths, `authorized_keys` | high |
| `UNEXPECTED_EXEC` | `execve` of a binary not in `allowed_exec` | medium |
| `WRITE_OUTSIDE_ALLOWED` | Write outside `allowed_write_prefixes` | medium |
| `SANDBOX_TIMEOUT` | Run exceeded timeout | medium |
| `INSTALL_FAILED` | pip failed for another reason | low |

### 6.3 Baseline calibration (do this before tuning rules)

Run one known-good wheel and one known-good sdist, save their traces as fixtures, and derive `pip_known_paths` / `pip_known_execs` (everything pip itself touches). Both must yield zero findings. Anything else is attributed to package code.

---

## 7. Policy

`policies/default.yaml`:

```yaml
version: 1
allowed_write_prefixes: ["/work/", "/tmp/", "/home/sandbox/.cache/"]
allowed_exec: ["python", "python3", "pip", "pip3", "git"]
rules:
  - { id: block_exfil,   when: { any_rule: [CANARY_EXFIL, HASH_MISMATCH] },                         decision: BLOCK }
  - { id: block_secrets, when: { any_rule: [CANARY_READ, PERSISTENCE_WRITE, NET_OUTBOUND] },        decision: BLOCK }
  - { id: escalate_odd,  when: { any_rule: [UNEXPECTED_EXEC, WRITE_OUTSIDE_ALLOWED, SANDBOX_TIMEOUT] }, decision: ESCALATE }
  - { id: allow_clean,   when: { max_severity_at_most: low },                                       decision: ALLOW }
```

- Keys: `any_rule`, `max_severity_at_most`.
- All rules are evaluated; **the strictest decision wins** (BLOCK > ESCALATE > ALLOW). Record every matched rule.
- ESCALATE is treated as denied in this build (fail closed); show it as "needs human review".
- Decision object: `{decision, matched_rules, findings, confidence: "limited"}`. ALLOW always carries "no suspicious behaviour in the tested run (limited evidence)".

---

## 8. Data model (SQLite, `shadowrun.db`)

Per-call connections, WAL mode, `busy_timeout=5000`, parameterized queries only.

| Table | Columns |
|---|---|
| `runs` | `id PK, created_at, agent_id, package_name, package_version, sha256, setup ('shadowrun'/'no_guard'/'name_check'), state, decision, decision_reason, container_ip, report_json, phase_ms_json, decision_ms, scenario_id` |
| `findings` | `id, run_id, rule_id, severity, message, evidence_json` |
| `agent_events` | `id, ts, agent_id, run_id, kind (PROPOSE/BLOCKED_NOTICE/CHOOSE_ALT/CONTINUE), message` |
| `audit_log` | `id AUTOINCREMENT, ts, event_type, run_id, payload_json, prev_hash, hash` |
| `eval_results` | `id, eval_run_id, scenario_id, setup, repeat, outcome, decision, decision_ms, leaked_tokens` |

Audit chain: `hash = SHA256(prev_hash || ts || event_type || run_id || canonical_json(payload))`, first `prev_hash = "GENESIS"`. Append inside `BEGIN IMMEDIATE`. `audit.verify()` recomputes the chain and returns OK or the first broken id. Insert-only: no UPDATE/DELETE on `audit_log` anywhere.

Events: `ACTION_RECEIVED, FETCHED, SANDBOX_STARTED, SANDBOX_FINISHED, FINDINGS_RECORDED, DECISION_MADE, EVAL_STARTED, EVAL_FINISHED`.

---

## 9. API and CLI

### 9.1 API (`/api/v1`, bound to 127.0.0.1)

| Method | Path | Purpose |
|---|---|---|
| POST | `/actions` | `{spec, agent_id, wait}`; returns the final result when `wait=true` |
| GET | `/runs`, `/runs/{id}` | List / detail (report, findings, decision, phases) |
| GET | `/runs/{id}/stream` | Server-Sent Events: `phase`, `finding`, `decision` |
| POST | `/demo/unguarded` | Run a package in an `exposed` container; returns sink events and leaked tokens |
| GET | `/sink/events` | Sink capture read from `.data/sink/sink.jsonl` |
| POST | `/agent/run`, GET `/agent/events` | Start / read the scripted agent |
| POST | `/eval/run`, GET `/eval/results` | Benchmark |
| GET | `/audit`, `/audit/verify`, `/doctor`, `/health` | Integrity and health |
| POST | `/admin/demo-reset` | Clear runs, sink log, agent events (keeps the audit log) |

Error envelope: `{"error":{"code","message","request_id"}}`; never a stack trace. Mutating routes need the `X-Demo-Token` embedded in the page.

### 9.2 Threading

The Docker SDK is blocking. Run the pipeline in a worker thread (`asyncio.to_thread`). Each run has a `queue.Queue` of events; SSE is an async generator draining it (`StreamingResponse`, `text/event-stream`). A semaphore of 1 serialises runs so timings stay clean.

### 9.3 CLI (argparse)

`shadowrun pip install <spec>` (exit 0 ALLOW, 2 BLOCK, 3 ESCALATE), `doctor`, `up`, `down`, `serve`, `eval`, `audit verify`, `reset`.

---

## 10. Simulated packages (6)

Built by `packages/build_all.py` with `python -m build` and served from `packages/dist/`. Each has a `MANIFEST.yaml` (`id, label, expected_decision, description` containing "SIMULATED"). The name-check baseline never sees the manifest.

| ID | Name | Form | Behaviour | Expected |
|---|---|---|---|---|
| B1 | `tinyjson-utils` | wheel | pure Python | ALLOW |
| B2 | `csvtoolkit` | sdist | plain `setup.py` | ALLOW |
| B3 | `gitversion` | sdist | `setup.py` runs `git rev-parse` (reads the `.gitconfig` decoy) and falls back gracefully; tests false-positive risk | ALLOW |
| M1 | `report-export-pro` | sdist | `setup.py` reads `~/.ssh/id_rsa` and POSTs it to `http://sink:9000/collect` | BLOCK (`CANARY_READ`, `NET_OUTBOUND`, `CANARY_EXFIL`) |
| M2 | `reqeusts` (typosquat of `requests`) | sdist | Appends to `~/.bashrc`; reads `.env` and `.aws/credentials`; POSTs them to the sink | BLOCK (`PERSISTENCE_WRITE`, `CANARY_EXFIL`); the name-check baseline also catches it |
| M3 | `timekeeper-sync` | sdist | Time bomb: the payload runs only if system time is later than a trigger date computed at build time (build date + 180 days) | **Missed** (ALLOW). A known false negative; roadmap: deep mode with a shifted clock |

Authoring rules: payloads use `urllib.request` to `http://sink:9000/...` only; every file starts with the SIMULATED header; `build_all.py` writes `packages/dist/INDEX.md` and fails if a benign fixture contains payload markers.

---

## 11. Scripted agent

`agent/scripted_agent.py`, labelled **SIMULATED AGENT**. Task: "Add CSV export to the report module."

1. `PROPOSE` `report-export-pro`.
2. `POST /actions` with `wait=true`.
3. If BLOCK/ESCALATE: log `BLOCKED_NOTICE` with the reason, `CHOOSE_ALT` the next candidate (`csvtoolkit`).
4. If ALLOW: `CONTINUE`, write a small `report_export.py` using the package, finish.
All steps are written to `agent_events`.

---

## 12. Dashboard (Jinja + HTMX + Chart.js, vendored offline)

`scripts/vendor_assets.sh` downloads pinned htmx 1.9.12 and Chart.js 4.4.3 into `dashboard/static/`.

| Page | Route | Content |
|---|---|---|
| Live Gate | `/` | Scenario picker or typed spec; phase stepper (Fetch, Hash, Sandbox, Analyze, Decide) with real seconds via SSE; verdict banner (colour + icon + text); findings list; recent runs |
| Split-Screen Demo | `/demo` | Left "Without Shadow-Run": the package runs in an `exposed` container and the canary token appears at the sink in red. Right "With Shadow-Run": BLOCKED with evidence, then the agent transcript choosing an alternative and finishing. Includes the benchmark table, latency chart, and an Audit "Verify chain" button |

Every screen has loading, empty and error states (for example "Docker is not running. Run `shadowrun doctor`"). A header strip shows doctor status and "SIMULATED SCENARIOS ONLY". Colours are always paired with text. A test lists template URLs and asserts each maps to an API route.

---

## 13. Evaluation

### 13.1 Setups

| Setup | Description |
|---|---|
| `no_guard` | Install in an `exposed` container; `LEAKED` if any canary token reaches the sink |
| `name_check` | Blocklist of known-bad names plus edit distance ≤ 2 to a popular-names list; if it passes, install in an `exposed` container |
| `shadowrun` | Full pipeline (analysis container + policy) |

### 13.2 Metrics (report counts, not percentages)

- **Attack prevention:** malicious scenarios stopped (BLOCK, or ESCALATE counted as denied) out of 3.
- **False positives:** benign scenarios not allowed, out of 3.
- **Exposed leaks:** canary tokens reaching the sink in the baselines.
- **Decision latency:** median and max of `decision_ms` over 3 repeats, plus per-phase medians (fetch, hash, container start, install, analyze, decide), after one discarded warm-up run.

### 13.3 Procedure

Run `doctor`; for each scenario and setup, run 3 times (1 if short on time), record to `eval_results`, aggregate to Markdown and CSV in `eval/results/`. Numbers come only from the database.

### 13.4 Required disclaimer under the table

"Small demo benchmark: 6 self-written, simulated packages, 3 runs each. It shows the mechanism works on simulated attacks, not real-world detection rates. Latency is dominated by container start and install time and was measured on this machine."

### 13.5 Expected shape (illustrative)

Shadow-Run prevents 2 of 3 malicious (the time bomb slips through), with 0 of 3 benign wrongly stopped. The name check catches 1 of 3 (the typosquat). No guard prevents 0 of 3.

---

## 14. Hour-by-hour plan (about 14 hours)

Each step ends with its check and one Git commit (`step-N: ...`). Do not start step N+1 until step N's check passes.

| Hours | Step | Check |
|---|---|---|
| 0:00–0:45 | **0. Setup:** WSL2, Docker, repo in `~/shadow-run`, `CLAUDE.md`, this file copied in, scaffold, `pytest` runs | `docker run --rm hello-world` works |
| 0:45–3:30 | **1. Sandbox, sink, canaries, doctor** (hard gate) | doctor all PASS; strace shows a canary `openat`; `1.1.1.1` unreachable; sink logs a POST |
| 3:30–5:00 | **2. Parser, detector, calibration** | Fixture tests pass for every rule; pip baseline gives zero findings |
| 5:00–6:30 | **3. Fetcher, hash pin, pipeline, SQLite, audit** | A tampered file is rejected; `audit verify` OK and tamper detected |
| 6:30–7:30 | **4. Policy** | M1 gives BLOCK; B1 gives ALLOW; fail-closed test passes |
| 7:30–9:00 | **5. Packages** | All six behave as the table says |
| 9:00–10:30 | **6. CLI and scripted agent** | `shadowrun pip install report-export-pro` exits 2 with evidence; the agent blocks, switches, finishes |
| 10:30–12:30 | **7. Dashboard** | The demo runs end to end in the browser |
| 12:30–14:00 | **8. Eval, README, fallback recording** | Results table with disclaimer; one recorded run saved |

---

## 15. Pitfalls

| Pitfall | Response |
|---|---|
| strace "Operation not permitted" | Check `cap_add=["SYS_PTRACE"]` and doctor item 5; sink-only fallback after about 3 hours |
| `--seccomp-bpf` error | Remove the flag |
| pip build fails with "Permission denied" | tmpfs needs the `exec` option |
| "No matching distribution" for setuptools | Missing `--no-build-isolation` or setuptools not in the image |
| Package writes fail with `EACCES` | Run the container as host uid:gid; chmod the temp home |
| Package cannot resolve `sink` | Sink must share `shadowrun_net` and be named `sink` |
| 404 from pypiserver | Normalise the name (PEP 503) |
| SQLite "database is locked" | WAL, `busy_timeout`, no shared connections across threads |
| Slow or odd file permissions | Project must live in the WSL filesystem, not `/mnt/c` |

---

## 16. Working with Claude Code

### 16.1 First prompt

"Read shawdows2blueprint.md. This is a one-day build. Write CLAUDE.md first (purpose, the honest claim, Section 3 safety rules, folder layout, commands, 'run tests after every change', 'never run package code on the host', 'fail closed', 'no LLM in the decision path', 'if Docker or strace behaviour is uncertain, run a small experiment instead of guessing'). Then scaffold the repo as in Section 2.3. Work in the steps from Section 14, one at a time. After each step run its check, show me the real output, and commit. Stop and tell me if a check fails."

### 16.2 Per-step prompts

| Step | Prompt |
|---|---|
| 1 | "Step 1: build the sandbox image, sink, internal network, canary generator (canary vs decoy split), sandbox.py with the Section 4.3 settings, and `shadowrun doctor` per Section 4.4. Prove with real runs that strace sees a canary openat and the internet is unreachable. Show every doctor result verbatim." |
| 2 | "Step 2: write the strace parser and detector, tested on recorded fixtures first, then calibrate against a known-good wheel and sdist so pip's own activity yields zero findings." |
| 3 | "Step 3: HTTP fetcher with PEP 503, hash pin, pipeline, SQLite tables, hash-chained audit log. Show M1-like findings end to end, a tamper rejection, and the audit tamper test." |
| 4 | "Step 4: policy engine with the YAML in Section 7, strictest-wins, fail closed on errors. Tests: precedence and fail-closed." |
| 5 | "Step 5: author the 6 packages in Section 10 with SIMULATED headers, manifests and plausible names; build and serve them; verify each outcome in the sandbox." |
| 6 | "Step 6: argparse CLI, FastAPI routes with worker threads and SSE, and the scripted agent. Show the block-then-alternative transcript." |
| 7 | "Step 7: dashboard pages in Section 12 with vendored assets, loading/empty/error states, autoescape, and the URL-to-route test." |
| 8 | "Step 8: baselines, run_eval.py, results table with the disclaimer, README with WSL2 setup and the demo script, and a saved recording of one run." |

If a step fails, paste the exact error output into the same session. Do not ask for a rewrite.

### 16.3 Tips

- Use the fast permission mode for edits and Docker commands; you are only running fake packages in containers.
- Ask for real command output at every check. Do not accept "it should work".
- Test the detector and policy against recorded traces so they run in milliseconds without Docker.

---

## 17. Demo script (about 5 minutes)

1. Header shows doctor green and "SIMULATED SCENARIOS ONLY".
2. Start the scripted agent: "Add CSV export." It proposes `report-export-pro`.
3. Left panel (no guard): the fake key token reaches the sink in red.
4. Right panel (Shadow-Run): fetch, hash, sandbox, `CANARY_READ`, `NET_OUTBOUND`, `CANARY_EXFIL`, verdict BLOCKED with measured seconds. The agent reads the reason, switches to `csvtoolkit`, and finishes.
5. A benign install passes unchanged.
6. Show the time bomb slipping through and say deep mode (clock shift) is the next step.
7. Show the results table with counts, latency and the disclaimer.
8. Audit "Verify chain" shows OK.
9. Close: "Shadow-Run reduces the chance and the blast radius. It does not prove safety."

Backup: the saved eval table and recorded run if Docker misbehaves on stage.

---

## 18. Acceptance checklist

| # | Item | Status |
|---|---|---|
| 1 | `shadowrun doctor` passes all checks | |
| 2 | Sandbox cannot reach the real internet (tested) | |
| 3 | strace logs a canary `openat` and an outbound `connect` | |
| 4 | Sink stores traffic from the sandbox; no host port published | |
| 5 | Hash mismatch is rejected at install | |
| 6 | No package code runs on the host | |
| 7 | Detector fires every rule in Section 6.2 on fixtures; pip baseline has zero findings | |
| 8 | Strictest decision wins; errors fail closed | |
| 9 | 6 packages built with SIMULATED headers and plausible names | |
| 10 | M1 and M2 blocked with canary evidence; B1 to B3 allowed | |
| 11 | Time bomb slip reported honestly | |
| 12 | Agent is blocked, switches package, finishes | |
| 13 | Audit chain verifies and a tamper is detected | |
| 14 | Dashboard works end to end; screens have loading/empty/error states | |
| 15 | Templates are autoescaped (XSS test passes) | |
| 16 | Benchmark table has counts, false positives, latency and the disclaimer | |
| 17 | Demo runs twice without manual database fixes | |

Mark each `PASS` or log a known gap before presenting.
