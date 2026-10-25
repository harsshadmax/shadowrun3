# Shadow-Run

Behavioural verification gate for AI coding agents: a package install is first run inside an isolated, instrumented Docker sandbox; the observed behaviour decides ALLOW / BLOCK / ESCALATE. Full spec: `shawdows2blueprint.md` (this file wins over any older blueprint).

## Honest claim
Shadow-Run reduces the chance and blast radius of a malicious install. It does not prove safety. All numbers come from a small, self-written, simulated benchmark. Never claim more.

## Safety rules (override everything)
1. Test packages are inert, start with `# SIMULATED TEST FIXTURE. INERT OUTSIDE THE SHADOW-RUN SANDBOX.`; names are plausible, "SIMULATED" lives in description/manifest.
2. Payloads only read fake files in the sandbox and only send to hostname `sink`.
3. `shadowrun_net` is `internal=True`. `doctor` must pass before any run; otherwise refuse (fail closed).
4. No Docker socket in containers; mem/CPU/pids limits; hard timeout; always remove containers; no-new-privileges everywhere. Two privilege domains, never mixed:
   - **package container**: non-root host uid, `cap_drop ALL` (CapEff = CapBnd = 0), read-only root, internal network. No root process and no capability inside it.
   - **tracer sidecar**: the ONLY root component. `cap_drop ALL` + `SYS_PTRACE` and nothing else (no SETUID/SETGID), NO network, read-only root, no host mounts; joins the package container's PID namespace and keeps the trace in its own root-only tmpfs.
   Residual risk (documented in `shadowrun/sandbox.py`): root + CAP_SYS_PTRACE in a shared PID namespace is not a hard boundary against a kernel/runtime vulnerability. Rootless Docker / userns remap / gVisor would shrink it (roadmap). Do not add capabilities, host mounts or network to the tracer without updating this rule and the doctor checks.
7. What is not traced cannot be detected: the strace filter is built from `shadowrun/monitor.py` syscall groups, and doctor check 9 proves that calls bypassing openat()/connect() (openat2, creat, sendto, linkat, memfd_create, execveat, io_uring_setup, mount) are collected AND detected. When adding a detection, add the syscall to its group and to that probe.
8. Validation evidence must name the commit it tested: `scripts/validate.sh` refuses a dirty tree and records the commit and a source hash (`scripts/source_hash.sh`).
5. Package names, metadata, sink contents are untrusted: Jinja autoescape everywhere.
6. Any error ends in BLOCK/ESCALATE, never ALLOW. No LLM in the decision path.

## Rules of work
- Never run package code on the host. Fetch = HTTP + hash only (never `pip download`).
- Fail closed.
- Run tests (`pytest`) after every change.
- If Docker/strace behaviour is uncertain, run a small experiment instead of guessing.
- Work step by step per blueprint section 14; each step ends with its check, real output shown, and one commit `step-N: ...`.
- Keep the repo in the WSL filesystem (`~/shadowrun2.0`).

## Layout
`shadowrun/` (cli, config, models, fetcher, sandbox, canary, monitor, detector, policy, audit, pipeline, db, api), `sink/`, `docker/`, `packages/`, `policies/default.yaml`, `agent/`, `dashboard/`, `eval/`, `scripts/`, `.data/` (gitignored), `tests/fixtures/`.

## Commands
- `pytest` — tests
- `python -m shadowrun.cli doctor | up | down | serve | eval | reset`
- `python -m shadowrun.cli pip install <spec>` (exit 0 ALLOW, 2 BLOCK, 3 ESCALATE)
