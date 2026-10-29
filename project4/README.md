# Project 4 — Prometheus Node Exporter Rollout (Ansible Role)

Rolls out Prometheus Node Exporter to every Linux server across four environments (**dev, qa, uat, prod**) using a reusable Ansible role, with per-environment configuration, version promotion, canary deployment to prod, and a one-command rollback.

## Scenario
The company uses Datadog as its primary monitoring platform. Leadership approved Prometheus + Grafana as a secondary, open-source monitoring stack. Node Exporter must be deployed on all Linux machines in all environments without affecting Datadog or production workloads.

## Status
| Phase | Status |
|-------|--------|
| Role built (install, verify, remove) | ✅ Done |
| dev deployed (1.8.2 → upgraded in place to 1.9.1) | ✅ Done |
| qa / uat / prod deployed on 1.8.2 | ⏳ |
| 1.9.1 promoted dev → qa → uat → prod | ⏳ |
| Prometheus + Grafana on control node | ⏳ |
| Rollback tested | ⏳ |

## Architecture
```