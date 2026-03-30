# SLI / SLO Ops Demo Guide

This guide walks through the SLI/SLO Ops story for the coffeeshop application running on the **coffeeshop** HCP spoke cluster, managed by ACM and delivered via Argo CD from this Git repository.

---

## What This Demonstrates

**SLI/SLO Ops** is the practice of defining, delivering, and enforcing Service Level Objectives as code across a fleet of clusters — the same GitOps principles that provision the clusters also govern their reliability contracts.

This demo shows three things working together:

1. **SLOs as code in Git** — PrometheusRule objects in `slo/` define what "good" means for the coffeeshop service (availability and latency targets, error budgets, and burn-rate alerts).
2. **Argo CD delivers the rules** — The hub Argo CD instance detects the spoke cluster via ACM's GitOpsCluster integration and automatically syncs the `slo/` directory to the coffeeshop cluster.
3. **ACM enforces the prerequisites and detects drift** — ACM Policies ensure user workload monitoring is running on every spoke, and report a compliance violation if the SLO rules are ever deleted or fail to sync.

---

## Architecture

```
Git repository (slo/)
        │
        │  auto-sync (Argo CD)
        ▼
  slo-coffeeshop Application  ──────────────────────────────────────┐
  (hub Argo CD)                                                      │
        │ registered via                                             │
        │ spoke-gitopscluster + spoke-placement                      │
        ▼                                                            ▼
  coffeeshop cluster                                        ACM hub
  ┌──────────────────────────────┐              ┌──────────────────────────────┐
  │  Namespace: coffeeshop       │              │  Policy: slo-infrastructure  │
  │  ┌────────────────────────┐  │◄─ enforce ───│  (enables UWM on all spokes) │
  │  │ PrometheusRule         │  │              ├──────────────────────────────┤
  │  │ coffeeshop-slo-        │  │              │  Policy: slo-verify-         │
  │  │ availability           │  │◄─ inform ────│  coffeeshop                  │
  │  ├────────────────────────┤  │  (checks     │  (compliance = rules exist)  │
  │  │ PrometheusRule         │  │   existence) └──────────────────────────────┘
  │  │ coffeeshop-slo-latency │  │
  │  └────────────────────────┘  │
  │  User Workload Prometheus     │
  │  evaluates rules every 30s    │
  └──────────────────────────────┘
```

---

## SLOs Defined

| SLO | Target | Error Budget | Metric |
|---|---|---|---|
| Availability | 99.9% of requests succeed (non-5xx) | ~40 min/28 days | `http_requests_total` |
| Latency | 95% of requests served within 500 ms | 5% of requests | `http_request_duration_seconds` |

Both SLOs use **multi-burn-rate alerting** — instead of a single threshold, three alert levels fire based on how fast the error budget is being consumed:

| Alert level | Burn rate | Meaning | Action |
|---|---|---|---|
| Critical fast | 14.4× | 2% of budget gone in 1 hour | Page on-call now |
| Critical slow | 6× | 5% of budget gone in 6 hours | Page on-call soon |
| Warning | 1× | Budget on track to exhaust by month end | Create a ticket |

---

## Demo Script

### Step 1 — Show the SLOs in Git

**Talking point:** "Our SLO targets live in the same Git repository as the cluster definitions. Any engineer can open a pull request to change an SLO, it goes through normal code review, and the change is automatically deployed to the right cluster."

Open the two files in the IDE or on GitHub:
- [`slo/coffeeshop-slo-availability.yaml`](../slo/coffeeshop-slo-availability.yaml)
- [`slo/coffeeshop-slo-latency.yaml`](../slo/coffeeshop-slo-latency.yaml)

Point out:
- The SLO target and error budget are declared as comments at the top of each file.
- The `spec.groups` section contains **recording rules** (pre-computed ratios over 5m, 30m, 1h, 6h, 3d windows) and **alerting rules** (multi-burn-rate pairs).
- No imperative steps — just YAML committed to Git.

---

### Step 2 — Show ACM Policy Compliance

**Talking point:** "ACM acts as the compliance layer. It ensures every spoke cluster has the monitoring infrastructure ready to evaluate our SLOs, and it tells us immediately if Argo CD ever fails to deliver the rules."

In the **ACM console → Governance → Policies**, show:

| Policy | Expected status |
|---|---|
| `slo-infrastructure` | Compliant on all spokes — user workload monitoring is enabled |
| `slo-verify-coffeeshop` | Compliant on coffeeshop — both PrometheusRules exist |

```bash
# Hub cluster — verify both policies on the command line
oc get policy slo-infrastructure slo-verify-coffeeshop -n acm-policies

# Check propagated status per cluster
oc get policy.open-cluster-management.io -A | grep slo
```

Point out:
- `slo-infrastructure` uses `enforce` — ACM will re-create the monitoring ConfigMap and namespace if someone deletes them.
- `slo-verify-coffeeshop` uses `inform` — it never creates the PrometheusRules itself, but it raises a violation the moment Argo CD diverges from Git. This is the **GitOps drift detector**.

---

### Step 3 — Show the Argo CD Application

**Talking point:** "Argo CD is the delivery mechanism. It watches the `slo/` directory and syncs any change to the coffeeshop cluster within seconds of a Git push."

In the **Argo CD UI → Applications**, open `slo-coffeeshop`:
- Status: `Synced / Healthy`
- Source: `slo/` path in this repository
- Destination: `coffeeshop` cluster, `coffeeshop` namespace

```bash
# Hub cluster
oc get application slo-coffeeshop -n openshift-gitops -o wide
```

Point out the destination cluster is the **spoke** (coffeeshop), not the hub — the hub Argo CD is deploying cross-cluster via the ACM GitOpsCluster integration.

---

### Step 4 — Verify the Rules on the Spoke Cluster

**Talking point:** "The rules are now live on the spoke cluster's Prometheus instance, evaluating against real application traffic."

```bash
# Switch to coffeeshop cluster kubeconfig
export KUBECONFIG=./kubeconfig   # extracted via: oc extract -n local-cluster secret/coffeeshop-admin-kubeconfig --to=.

# Confirm both PrometheusRules exist
oc get prometheusrule -n coffeeshop

# Confirm user workload monitoring pods are running
oc get pods -n openshift-user-workload-monitoring
```

---

### Step 5 — Show SLI Metrics and Error Budget in Prometheus

**Talking point:** "The recording rules pre-compute the error ratios continuously. We can query the current error rate and remaining error budget in real time."

Open the **user workload Prometheus UI** on the coffeeshop cluster:

```bash
# Get the Prometheus route (coffeeshop cluster)
oc get route -n openshift-user-workload-monitoring
```

Paste these queries into the Prometheus expression browser:

```promql
# Current 5-minute availability error ratio (should be near 0)
job:coffeeshop_http_errors:ratio_rate5m

# Remaining error budget for availability (1.0 = full, 0.0 = exhausted)
job:coffeeshop_availability_slo:error_budget_remaining

# p95 request latency over the last 5 minutes
job:coffeeshop_latency_p95:rate5m

# All currently firing SLO alerts
ALERTS{slo=~"availability|latency", service="coffeeshop"}
```

---

### Step 6 — Simulate a Burn-Rate Alert (Optional Live Demo)

**Talking point:** "Let's show what happens when the service degrades. Watch how fast the burn-rate alerts respond compared to a simple threshold alert."

```bash
# On the coffeeshop cluster — take the app offline
oc scale deployment coffeeshop --replicas=0 -n coffeeshop
```

Wait approximately 5–10 minutes, then query Prometheus:

```promql
# Error ratio should now be 1.0 (100% errors — no successful requests)
job:coffeeshop_http_errors:ratio_rate5m

# Check for firing alerts (CoffeeshopAvailabilityCriticalFastBurn should appear)
ALERTS{severity="critical", slo="availability"}
```

In the **Alertmanager UI**, show the `CoffeeshopAvailabilityCriticalFastBurn` alert with:
- `severity: critical`
- `slo: availability`
- `service: coffeeshop`

**Key talking point — why multi-burn-rate is better:**
> "A traditional alert fires when the error rate crosses a static threshold, which generates noise for short spikes. This alert fires only when BOTH the 5-minute and 1-hour windows are elevated simultaneously. That means the service has been degraded long enough to genuinely threaten the monthly error budget — a real incident, not a blip."

```bash
# Restore the application
oc scale deployment coffeeshop --replicas=1 -n coffeeshop
```

---

### Step 7 — Demonstrate GitOps Drift Detection

**Talking point:** "What happens if someone manually deletes the SLO rules from the cluster — perhaps thinking they're cleaning up? ACM detects it instantly, and Argo CD restores them."

```bash
# On the coffeeshop cluster — delete one PrometheusRule manually
oc delete prometheusrule coffeeshop-slo-availability -n coffeeshop
```

Within ~1 minute:
1. **Argo CD** detects the drift and re-applies `coffeeshop-slo-availability.yaml` (self-heal is enabled).
2. **ACM** `slo-verify-coffeeshop` policy briefly shows NonCompliant, then returns to Compliant once Argo CD restores the rule.

```bash
# Watch Argo CD restore the rule
oc get prometheusrule coffeeshop-slo-availability -n coffeeshop --watch
```

This closes the demo loop: **Git is the single source of truth, and any deviation from it is detected and corrected automatically.**

---

## Quick Reference — PromQL Queries

```promql
# ── Availability ─────────────────────────────────────────────────────────────

# Error ratio (5-min window) — should be < 0.001 (0.1%) for SLO compliance
job:coffeeshop_http_errors:ratio_rate5m

# Error ratio (1-hour window)
job:coffeeshop_http_errors:ratio_rate1h

# Remaining error budget (28-day window) — 1.0 = full, 0 = exhausted
job:coffeeshop_availability_slo:error_budget_remaining

# ── Latency ──────────────────────────────────────────────────────────────────

# Fraction of requests exceeding 500 ms (5-min window) — should be < 0.05 (5%)
job:coffeeshop_latency_errors:ratio_rate5m

# p95 latency in seconds (5-min window) — should be < 0.5
job:coffeeshop_latency_p95:rate5m

# Remaining latency error budget
job:coffeeshop_latency_slo:error_budget_remaining

# ── Alerts ───────────────────────────────────────────────────────────────────

# All currently active SLO alerts
ALERTS{service="coffeeshop"}

# Critical burn-rate alerts only
ALERTS{service="coffeeshop", severity="critical"}
```

---

## Troubleshooting During the Demo

| Symptom | Likely cause | Fix |
|---|---|---|
| `slo-verify-coffeeshop` NonCompliant | PrometheusRules not deployed | Check `oc get application slo-coffeeshop -n openshift-gitops` |
| Argo CD Application `Unknown` / `Missing` destination | coffeeshop cluster secret not created yet | Wait 2 min after `oc apply -k argocd/`, re-check `oc get secret -n openshift-gitops -l argocd.argoproj.io/secret-type=cluster` |
| Recording rules return `no data` | User workload monitoring pods not ready | `oc get pods -n openshift-user-workload-monitoring`; may take 3–5 min after enabling |
| Burn-rate alert does not fire | App is not actually serving traffic | Ensure the coffeeshop app is running and a ServiceMonitor is scraping it |
