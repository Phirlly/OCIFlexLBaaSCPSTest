# OCI Flex LBaaS CPS Test Suite (Locust)

This folder contains the notebooks and automation used to run high-concurrency CPS and TLS-handshake stress tests on Oracle Cloud Infrastructure (OCI), including direct backend tests, single-LB tests, multi-LB scaling tests, and OKE-backed variants.

Published outcome (post-test): **High-Concurrency Load Balancing on Oracle Cloud Infrastructure** (April 6, 2026, by Tony Markel, Ivan Pavlovic, and Adekola Okunola).

## Why This Exists

The goal of this test suite is to measure how OCI load balancing behaves under extreme connection churn:

- New connections per second (CPS) capacity.
- TLS handshake overhead impact (TLS 1.2 and TLS 1.3).
- Latency and failure behavior as CPS increases.
- Scale-out predictability when adding load balancers and distributing traffic.

The key scaling result from the published analysis: moving from one LB to two LBs (with larger generator/backend fleets) delivered near-linear growth from ~60k CPS to ~120k CPS.

## Folder Contents

| File | Purpose |
|---|---|
| `Generators_For_CPS_Test_v1.01.ipynb` | Generator-only build and prep flow: Terraform, host prep, Locust deployment, start/stop, teardown. |
| `OCI_Direct_CPS_Test.ipynb` | Baseline direct path (generator -> backend HTTPS, no LB) for CPS/throughput comparisons. |
| `OCI_Flex_LBaaS_CPS_Test_Single(TCP TLS Offloading With Stateless Rules).ipynb` | Single OCI Flexible LB test with TCP:443 listener, TLS offload, stateless NSG model. |
| `OCI_Flex_LBaaS_CPS_Test_Combined(IPv6-IPv4 TCP TLS Offload, Stateless).ipynb` | Combined topology notebook supporting Single or Multi LB with IPv6 frontend VIPs and IPv4 backends. |
| `OCI_Flex_LBaaS_CPS_Test_On_OKE_v1.01.ipynb` | OKE backend variant (pods/services) with optional generator orchestration. |
| `OCI_Flex_LBaaS_CPS_Test_On_OKE_v1.02.ipynb` | Improved OKE flow (more robust deploy, worker orchestration, cleanup, teardown). |

## Architecture and Test Model

Common characteristics for the stateless LB runs:

- Region used during published testing: OCI San Jose (`us-sanjose-1`).
- Load balancer: OCI Flexible Load Balancer, `min=8000 Mbps`, `max=8000 Mbps`.
- Listener: TCP `443` with TLS offload at LB.
- TLS profile: TLS 1.2/TLS 1.3, `oci-modern-ssl-cipher-suite-v1`, ECDSA P-256 certificate.
- Backend protocol: HTTP/1.1 on port `80`.
- Source context propagation: Proxy Protocol v2 LB -> backend.
- Network security posture: stateless NSG rules with explicit symmetric ingress/egress.
- Dual-stack coverage included: IPv6 frontend VIP -> IPv4 backends (plus IPv4-only paths).

Why stateless rules are central in this suite:

- CPS-heavy patterns create very high connection churn.
- Stateful policy introduces connection-tracking pressure.
- Stateless rules remove that pressure point when return-path rules are defined correctly.

Why ECDSA is used in these scenarios:

- CPS mode forces frequent full handshakes (`Connection: close`).
- ECDSA P-256 lowers per-handshake cryptographic overhead relative to RSA.
- This helps expose routing/control-plane limits rather than certificate math bottlenecks.

## Methodology

- Load tool: distributed Locust across generator instances.
- Request pattern: CPS mode forces short-lived connections and full TLS handshakes.
- Sanity validation: pre-tier HTTPS check expecting HTTP `200`.
- Soak behavior: each tier includes a steady hold period.
- Host tuning: sysctl and system tuning applied to increase high-concurrency headroom.
- Artifacting: run outputs exported to local `results/` and per-run CSV/JSON summaries.

## Representative Published Results

The published report includes full stateless results and highlights linear scale-out behavior. Representative data points:

| Generators | LB Count | Backends | Target CPS | Observed RPS | Avg Time (ms) | Failures | Nines |
|---:|---:|---:|---:|---:|---:|---:|---:|
| 5 | 1 | 5 | 60000 | 58694 | 17.10 | 0.0000% | 100.0000% |
| 10 | 2 | 10 | 120000 | 118940 | 18.92 | 0.0002% | 99.9998% |

Interpretation:

- Doubling LB count plus balanced client/backend scale produced near 2x CPS at similar latency class.
- The test data supports predictable fan-out scaling for CPS-heavy workloads.

## Full Stateless Result Table (Published)

| Generators | Generator Config | LBaaS | Backends | Backend Config | CPS | RPS | Avg Time (ms) | Failures | Nines |
|---:|---|---:|---:|---|---:|---:|---:|---:|---:|
| 1 | 16/64 | 0 | 1 | 16/64 | 10000 | 9889 | 21.00 | 0.0000% | 100.0000% |
| 1 | 16/64 | 1 | 1 | 16/64 | 10000 | 9954 | 9.72 | 0.0000% | 100.0000% |
| 1 | 16/64 | 1 | 1 | 16/64 | 12500 | 12176 | 23.36 | 0.0000% | 100.0000% |
| 1 | 16/64 | 1 | 2 | 16/64 | 12500 | 12200 | 24.33 | 0.0000% | 100.0000% |
| 1 | 16/64 | 1 | 2 | 16/64 | 15000 | 13577 | 86.25 | 0.0000% | 100.0000% |
| 2 | 16/64 | 1 | 1 | 16/64 | 20000 | 19702 | 11.91 | 0.0000% | 100.0000% |
| 2 | 16/64 | 1 | 1 | 16/64 | 25000 | 24781 | 29.29 | 0.0006% | 99.9994% |
| 2 | 16/64 | 1 | 2 | 16/64 | 25000 | 24153 | 22.11 | 0.0010% | 99.9990% |
| 2 | 16/64 | 1 | 2 | 16/64 | 30000 | 26286 | 79.36 | 0.0000% | 100.0000% |
| 3 | 16/64 | 1 | 1 | 16/64 | 20000 | 19900 | 5.83 | 0.0000% | 100.0000% |
| 3 | 16/64 | 1 | 1 | 16/64 | 25000 | 24832 | 8.32 | 0.0004% | 99.9996% |
| 5 | 16/64 | 1 | 5 | 16/64 | 50000 | 49500 | 10.26 | 0.0001% | 99.9999% |
| 5 | 16/64 | 1 | 5 | 16/64 | 60000 | 58694 | 17.10 | 0.0000% | 100.0000% |
| 5 | 16/64 | 1 | 10 | 16/64 | 50000 | 49430 | 9.61 | 0.0001% | 99.9999% |
| 5 | 16/64 | 1 | 10 | 16/64 | 60000 | 58711 | 17.77 | 0.0000% | 100.0000% |
| 10 | 16/64 | 1 | 6 | 16/64 | 50000 | 49960 | 5.85 | 0.0004% | 99.9996% |
| 10 | 16/64 | 1 | 6 | 16/64 | 60000 | 59881 | 5.47 | 0.0002% | 99.9998% |
| 10 | 16/64 | 2 | 10 | 16/64 | 120000 | 118940 | 18.92 | 0.0002% | 99.9998% |

Column notes:

- `Failures` is request failure percentage from the test run.
- `Nines` is the corresponding request success percentage.

## Execution Pattern Inside the Notebooks

Most notebooks follow this sequence:

1. Install minimal Python dependencies.
2. Set central variables and OCI credentials.
3. Use widget dashboard to select region, compartment, shapes, mode, and tuning parameters.
4. Render cloud-init and Terraform files.
5. `terraform init/apply` and capture outputs.
6. Run connectivity/sanity checks.
7. Prepare generators and Locust runtime.
8. Start Locust master/workers (tmux-managed).
9. Execute warm-up and full suites (CPS or throughput mode).
10. Export and summarize artifacts.
11. Stop processes and optionally destroy infrastructure.

## Locust Behavior Used for High CPS

The notebooks intentionally use request semantics that maximize connection churn in CPS mode:

- `Connection: close` headers in CPS requests.
- Tight wait/connect/read tuning via environment variables.
- Distributed workers across generator fleet.
- Optional per-VIP naming in combined/multi-LB scenarios for visibility.

## Kernel and Host Tuning

The tests include host-level network and file-descriptor tuning (for both backends and generators), including:

- backlog and queue limits,
- ephemeral port range,
- `tcp_tw_reuse`,
- `somaxconn`,
- `fs.file-max`.

These settings are applied to increase concurrency headroom and reduce false bottlenecks during stress tiers.

## Practical Takeaways

- Stateless NSG policies are important for CPS-heavy patterns.
- Generator capacity must be sized so client limits do not mask LB limits.
- Tier soak periods help verify steady-state behavior.
- LB fan-out is an effective and predictable strategy for higher aggregate CPS.

## Safety and Cleanup

- Notebooks include teardown guards such as `TEARDOWN_CONFIRM`.
- OKE variants include additional run flags to gate infra apply, deploy, and locust phases.
- Validate teardown settings before execution to avoid orphaned resources.
