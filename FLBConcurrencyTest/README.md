#  OCI Flex Load Balancer Session Concurrency Validation

## Purpose
This notebook (`FLBConcurrencyTest.ipynb`) validates OCI Flexible Load Balancer behavior for long-lived session concurrency with:

- HTTP/2 preferred
- HTTP/1.1 support
- lifecycle-driven session close (~20 min)
- mostly idle sessions with periodic 5KB uploads (~every 50s)

---

## What It Provisions
Through generated Terraform, the notebook creates:

- VCN/subnets/route tables/NSGs (stateless rules)
- service gateway route for private subnet service access
- OCI Flex LB listeners:
  - `443` (`h2-443`) for HTTP/2 preferred
  - `8443` (`https-h1-8443`) for HTTPS/H1 compatibility
  - optional `9443` (`tcp-ppv2-9443`)
- backend instances (Nginx endpoints: `/healthz`, `/upload_5k`)
- generator instances for Locust workers

---

## Test Methodology

## 1) Model and math
Core model:

- one session maps to one long-lived connection
- session closes by lifecycle TTL (not random abort)
- each live session sends one ~5KB heartbeat every ~50s

Formulas used in the notebook:

- `opens_per_sec = TARGET_CONCURRENCY / AVG_SESSION_SEC`
- `steady_heartbeat_rps = users / HEARTBEAT_INTERVAL_SEC`
- `ui_spawn_rate = users / AVG_SESSION_SEC`

Default spec values:

- `AVG_SESSION_SEC = 1200`
- `HEARTBEAT_INTERVAL_SEC = 50`
- `UPLOAD_BYTES = 5120`
- `PROTOCOL_PROFILE = http2_preferred`
- `H2_CLIENT_PERCENT = 100`

---

## 2) Execution flow
1. Cells 1-3: dependency install + central config + interactive selections.
2. Cells 4-6: cloud-init generation + `main.tf` + `terraform.tfvars`.
3. Cell 7: `terraform init/validate/plan/apply`.
4. Cell 8: collect runtime outputs (LB IPs, generator IPs, backend IPs, host URL).
5. Cell 9: preflight (generator readiness, backend health/upload checks).
6. Cell 10: protocol sanity (H1/H2 checks against listeners).
7. Cell 11: prepare generators and write workload.
8. Cell 12: launch Locust (master + workers, UI/headless).
9. Cell 13: controlled stop + artifact capture.
10. Cell X: guarded teardown (`terraform destroy`).

---

## Locust Setup (and Why)

## Runtime setup
- Locust runs inside a dedicated venv on each generator (`/home/opc/locustwork/.venv`).
- Master/worker processes are launched via `tmux` (fallback `nohup`).
- Worker count is planned per host from CPU (`nproc`) to maximize host utilization safely.

Why:
- venv avoids system Python package conflicts.
- tmux/nohup keeps load processes stable across SSH disconnects.
- per-host worker planning avoids under/over-driving a generator.

## Workload behavior
Generated `locustfile.py` includes:

- `H1SessionUser` (`HttpUser`) for HTTP/1.1 path
- `H2SessionUser` (`User` + `httpx(http2=True)`) for HTTP/2 path
- weighted H1/H2 mix from `PROTOCOL_PROFILE` + `H2_CLIENT_PERCENT`
- `constant_pacing(HEARTBEAT_SEC)` for heartbeat timing control
- TTL lifecycle stop (`StopUser`) at session deadline
- optional TTL jitter (`SESSION_TTL_JITTER_*`)
- H2 client limits:
  - `max_connections=1`
  - `max_keepalive_connections=1`

Why:
- `constant_pacing` enforces periodic heartbeat cadence.
- TTL stop models application lifecycle-driven close behavior.
- TTL jitter prevents synchronized close spikes.
- single-connection H2 limits preserve one-session-one-connection fidelity.

---

## What Is Measured

Notebook outputs capture:

- generator readiness and runtime verification
- backend `/healthz` and `/upload_5k` readiness
- H1/H2 protocol sanity evidence
- Locust run stats (users, RPS, failures, latency)
- launch/worker state and cleanup status
- stop-state verification (before/after process counts)

Note:
- OCI Monitoring LB timeseries export is not auto-collected in a dedicated default cell.

---

## Artifacts
Run artifacts are written to:

`FLBConcurrencyTest/results/<RUN_ID>/`

Key files:

- `selected_outputs.json`
- `cell9_preflight_summary.json`
- `cell10_protocol_sanity_summary.json`
- `cell11_prep_summary.json`
- `cell12_launch_summary.json`
- `cell13_stop_summary.json`
- `cell13_master_status_before_after.json`
- `cell13_workers_status_before_after.json`
- `cell13_logs_manifest.json`

---

## Correct Run Order
1. Run Cells 1-12.
2. Drive test from UI/headless with math-derived users/spawn.
3. Run Cell 13 for controlled stop and artifact preservation.
4. Run Cell X only when ready to destroy infrastructure.
