# SLI / SLO Ops Demo Guide

This guide walks through the SLI/SLO Ops story for the coffeeshop application running on the **coffeeshop** HCP spoke cluster, managed by ACM and delivered via Argo CD from this Git repository.

---

## What This Demonstrates

**SLI/SLO Ops** is the practice of defining, delivering, and enforcing Service Level Objectives as code across a fleet of clusters — the same GitOps principles that provision the clusters also govern their reliability contracts.

This demo shows three things working together:

1. **SLOs as code in Git** — PrometheusRule objects in `slo/` define what "good" means for the coffeeshop service (availability and stability targets, error budgets, and burn-rate alerts).
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
  │  Namespace: openshift-       │              │  Policy: slo-infrastructure  │
  │  monitoring                  │◄─ enforce ───│  (enables UWM on all spokes) │
  │  ┌────────────────────────┐  │              ├──────────────────────────────┤
  │  │ PrometheusRule         │  │              │  Policy: slo-verify-         │
  │  │ coffeeshop-slo-        │  │◄─ inform ────│  coffeeshop                  │
  │  │ availability           │  │  (checks     │  (checks openshift-monitoring│
  │  ├────────────────────────┤  │   existence) │   for both PrometheusRules)  │
  │  │ PrometheusRule         │  │              └──────────────────────────────┘
  │  │ coffeeshop-slo-        │  │
  │  │ latency (stability)    │  │
  │  └────────────────────────┘  │
  │  Platform Prometheus         │
  │  (prometheus-k8s) evaluates  │
  │  rules — has kube-state-     │
  │  metrics access              │
  └──────────────────────────────┘
```

---

## SLOs Defined

| SLO | Target | Error Budget | Metric source |
|---|---|---|---|
| Availability | 99.9% of scrape intervals have at least one pod Ready | ~40 min/28 days | `kube_pod_status_ready` (kube-state-metrics) |
| Stability | Fewer than 1 container restart per hour on average | ~672 restarts/28 days | `kube_pod_container_status_restarts_total` (kube-state-metrics) |

Both metrics come from **kube-state-metrics**, which is always available in OpenShift — no `ServiceMonitor` configuration on the application is required.

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
- The `spec.groups` section contains **recording rules** (pre-computed ratios/rates over 5m, 30m, 1h, 6h, 3d windows) and **alerting rules** (multi-burn-rate pairs).
- Both SLOs use **kube-state-metrics** — no application-level instrumentation required.
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
- Destination: `coffeeshop` cluster, `openshift-monitoring` namespace

```bash
# Hub cluster
oc get application slo-coffeeshop -n openshift-gitops -o wide
```

Point out the destination cluster is the **spoke** (coffeeshop), not the hub — the hub Argo CD is deploying cross-cluster via the ACM GitOpsCluster integration.

---

### Step 4 — Verify the Rules on the Spoke Cluster

**Talking point:** "The rules are now live on the spoke cluster's Prometheus instance, evaluating every 30 seconds against kube-state-metrics data — no application changes needed."

```bash
# Switch to coffeeshop cluster kubeconfig
export KUBECONFIG=./kubeconfig   # extracted via: oc extract -n local-cluster secret/coffeeshop-admin-kubeconfig --to=.

# Confirm both PrometheusRules exist in openshift-monitoring
oc get prometheusrule -n openshift-monitoring | grep coffeeshop

# Confirm the platform Prometheus pods are running
oc get pods -n openshift-monitoring -l app.kubernetes.io/name=prometheus
```

---

### Step 5 — Show SLI Metrics and Error Budget in Prometheus

**Talking point:** "The recording rules pre-compute the error ratios continuously. We can query the current error rate and remaining error budget in real time."

The PrometheusRules live in `openshift-monitoring` and are evaluated by the **platform Prometheus** (`prometheus-k8s`). Their results flow through the Thanos Querier, which means they are visible in the OpenShift console without any project-selector restriction. Use one of the two options below.

#### Option A — OpenShift console (best for presentations)

**1. Get the coffeeshop cluster console URL** (run on the hub):

```bash
oc get hostedcluster coffeeshop -n local-cluster \
  -o jsonpath='{.status.conditions[?(@.type=="Available")].message}'

# Or directly from the ACM console:
# Infrastructure → Clusters → coffeeshop → "Open console" link
```

**2. Log in** to the coffeeshop cluster console as `kubeadmin` (password from the `coffeeshop-kubeadmin-password` secret):

```bash
# Retrieve kubeadmin password (run on the hub)
oc extract -n local-cluster secret/coffeeshop-kubeadmin-password --to=-
```

**3. Switch to the Administrator perspective** — in the top-left corner of the console, click the perspective switcher (it may say "Developer") and select **Administrator**.
The Administrator perspective has the full PromQL query interface. The Developer perspective has a simplified metrics browser that does not accept raw PromQL.

**4. No project selection needed** — the recording rules produced by the platform Prometheus (`prometheus-k8s`) are visible globally through the Thanos Querier. You do not need to switch to a specific project. The namespace is embedded inside the PromQL expressions themselves (e.g. `namespace="coffeeshop-dev"`).

**5. Navigate to Observe → Metrics** in the left sidebar.

**6. Type the query directly** — you will see an expression input field (the placeholder text reads something like *"Expression (press Shift+Enter for newline)"*). Paste a query from the table below and press **Enter** or click **Run queries**. Switch between the **Table** and **Graph** tabs to show different views.

---

#### Pre-flight check — verify kube-state-metrics is scraping coffeeshop-dev

The SLI recording rules use `kube_pod_status_ready` and `kube_pod_container_status_restarts_total`. These come from kube-state-metrics and do not require any `ServiceMonitor` on the application. Run this first to confirm data is available:

```promql
kube_pod_status_ready{namespace="coffeeshop-dev", pod=~"coffeeshop-.*"}
```

- **Returns rows with value 0 or 1** → kube-state-metrics is working. The SLO queries below will all produce data immediately.
- **Returns "No datapoint found"** → the `coffeeshop-dev` namespace or `coffeeshop-.*` pods don't exist yet. Check `oc get pods -n coffeeshop-dev` on the coffeeshop cluster.

---

#### Show the alerting rules are loaded

In the console: go to **Observe → Alerting → Alerting rules tab**.
- Use the search box to filter by `coffeeshop`.
- You should see all six SLO alerting rules (`CoffeeshopAvailabilityCriticalFastBurn`, `CoffeeshopStabilityCriticalFastBurn`, etc.) with state **Inactive** (correct — pods are healthy).
- These rules appear here because they live in `openshift-monitoring` and are evaluated by the platform Prometheus.

**Talking point:** "The rules are loaded and being evaluated every 30 seconds against live kube-state-metrics data. Inactive means we are within our SLO right now."

---

**Queries to run during the demo:**

| What to show | Expression | What the audience sees |
|---|---|---|
| Pod readiness now | `kube_pod_status_ready{namespace="coffeeshop-dev", pod=~"coffeeshop-.*", condition="true"}` | 1 for each ready pod |
| Not-ready ratio (5 min) | `job:coffeeshop_pod_notready:ratio_rate5m` | Near 0 — all pods healthy |
| Remaining error budget | `job:coffeeshop_availability_slo:error_budget_remaining` | Close to 1.0 — budget is full |
| Restart rate (5 min) | `job:coffeeshop_restart_rate:rate5m` | Near 0 — no crash loops |
| Active SLO alerts | `ALERTS{service="coffeeshop"}` | Empty — no alerts firing |

**Talking point for each result:**
- Pod readiness = `1` → "Every coffeeshop pod is Ready right now. The SLI is 100%."
- Not-ready ratio near `0` → "Over the last 5 minutes there has been effectively no downtime."
- Error budget near `1.0` → "We have our full error budget. No incidents have consumed it this month."
- Restart rate near `0` → "No crash loops. The application is stable."
- Empty alerts → "No burn-rate alerts are firing. We are comfortably within both SLOs."

#### Option B — Port-forward (best for CLI-heavy demos)

The recording rules are evaluated by the **platform Prometheus** (`prometheus-k8s`), not the user workload Prometheus. Port-forward accordingly:

```bash
# Ensure KUBECONFIG points to the coffeeshop cluster
export KUBECONFIG=./kubeconfig   # extracted via: oc extract -n local-cluster secret/coffeeshop-admin-kubeconfig --to=.

# Port-forward the platform Prometheus pod to localhost:9090
oc port-forward -n openshift-monitoring \
  $(oc get pod -n openshift-monitoring \
    -l app.kubernetes.io/name=prometheus -o name | head -1) \
  9090:9090
```

Open **http://localhost:9090** in a browser. Click on the **Graph** tab, paste each expression into the input field, and press **Execute**.

---

**PromQL expressions for both options:**

```promql
# Pod readiness — 1 = ready, 0 = not ready
kube_pod_status_ready{namespace="coffeeshop-dev", pod=~"coffeeshop-.*", condition="true"}

# Not-ready ratio over 5 minutes (should be 0 when all pods are healthy)
job:coffeeshop_pod_notready:ratio_rate5m

# Remaining availability error budget (1.0 = full, 0.0 = exhausted)
job:coffeeshop_availability_slo:error_budget_remaining

# Container restart rate over 5 minutes (should be 0 when stable)
job:coffeeshop_restart_rate:rate5m

# All currently firing SLO alerts
ALERTS{service="coffeeshop"}
```

---

### Step 6 — Simulate a Burn-Rate Alert (Optional Live Demo)

**Talking point:** "Let's show what happens when the service degrades. Watch how fast the burn-rate alerts respond compared to a simple threshold alert."

```bash
# On the coffeeshop cluster — take the app offline
# The app runs in coffeeshop-dev; replace "coffeeshop" with the actual Deployment name if different
oc scale deployment coffeeshop --replicas=0 -n coffeeshop-dev
```

Wait approximately 5–10 minutes for both the 5-minute and 1-hour recording rule windows to accumulate enough not-ready time, then query Prometheus:

```promql
# Pod readiness should now be 0 (no pods running)
kube_pod_status_ready{namespace="coffeeshop-dev", pod=~"coffeeshop-.*", condition="true"}

# Not-ready ratio should now be close to 1.0
job:coffeeshop_pod_notready:ratio_rate5m

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
oc scale deployment coffeeshop --replicas=1 -n coffeeshop-dev
```

---

### Step 7 — Demonstrate GitOps Drift Detection

**Talking point:** "What happens if someone manually deletes the SLO rules from the cluster — perhaps thinking they're cleaning up? ACM detects it instantly, and Argo CD restores them."

```bash
# On the coffeeshop cluster — delete one PrometheusRule manually
oc delete prometheusrule coffeeshop-slo-availability -n openshift-monitoring
```

Within ~1 minute:
1. **Argo CD** detects the drift and re-applies `coffeeshop-slo-availability.yaml` (self-heal is enabled).
2. **ACM** `slo-verify-coffeeshop` policy briefly shows NonCompliant, then returns to Compliant once Argo CD restores the rule.

```bash
# Watch Argo CD restore the rule
oc get prometheusrule coffeeshop-slo-availability -n openshift-monitoring --watch
```

This closes the demo loop: **Git is the single source of truth, and any deviation from it is detected and corrected automatically.**

---

## Quick Reference — PromQL Queries

```promql
# ── Availability (pod readiness) ─────────────────────────────────────────────

# Current pod readiness — 1 per ready pod, 0 per not-ready pod
kube_pod_status_ready{namespace="coffeeshop-dev", pod=~"coffeeshop-.*", condition="true"}

# Not-ready ratio (5-min window) — 0 when healthy, 1 when all pods down
job:coffeeshop_pod_notready:ratio_rate5m

# Not-ready ratio (1-hour window)
job:coffeeshop_pod_notready:ratio_rate1h

# Remaining availability error budget (28-day) — 1.0 = full, 0 = exhausted
job:coffeeshop_availability_slo:error_budget_remaining

# ── Stability (container restart rate) ───────────────────────────────────────

# Restart rate (5-min window) — should be 0 restarts/second when stable
job:coffeeshop_restart_rate:rate5m

# Restart rate (1-hour window)
job:coffeeshop_restart_rate:rate1h

# Total restarts in the last 28 days
job:coffeeshop_restart_count:increase28d

# ── Alerts ───────────────────────────────────────────────────────────────────

# All currently active SLO alerts
ALERTS{service="coffeeshop"}

# Critical burn-rate alerts only
ALERTS{service="coffeeshop", severity="critical"}

# Availability alerts only
ALERTS{service="coffeeshop", slo="availability"}

# Stability alerts only
ALERTS{service="coffeeshop", slo="stability"}
```

---

## Troubleshooting During the Demo

| Symptom | Likely cause | Fix |
|---|---|---|
| `slo-verify-coffeeshop` NonCompliant | PrometheusRules not deployed | Check `oc get application slo-coffeeshop -n openshift-gitops`; confirm rules exist with `oc get prometheusrule -n openshift-monitoring \| grep coffeeshop` |
| Argo CD Application `Unknown` / `Missing` destination | coffeeshop cluster secret not created yet | Wait 2 min after `oc apply -k argocd/`, re-check `oc get secret -n openshift-gitops -l argocd.argoproj.io/secret-type=cluster` |
| `job:coffeeshop_pod_notready:ratio_rate5m` returns "No datapoint found" | PrometheusRules not in `openshift-monitoring`, or first evaluation interval not yet complete | Confirm `oc get prometheusrule coffeeshop-slo-availability -n openshift-monitoring`; if it exists, wait 30–60 s for the first evaluation interval |
| `kube_pod_status_ready{namespace="coffeeshop-dev"}` returns "No datapoint found" | Namespace or pods do not exist | Run `oc get pods -n coffeeshop-dev` on the coffeeshop cluster to confirm the app is deployed |
| Alerting rules not visible in console | PrometheusRules in wrong namespace | Rules must be in `openshift-monitoring`; check `oc get prometheusrule -n openshift-monitoring \| grep coffeeshop` |
| Prometheus UI not reachable via port-forward | Wrong namespace used | Port-forward from `openshift-monitoring` (platform Prometheus), not `openshift-user-workload-monitoring`: `oc port-forward -n openshift-monitoring $(oc get pod -n openshift-monitoring -l app.kubernetes.io/name=prometheus -o name \| head -1) 9090:9090` |
| Burn-rate alert does not fire after scaling to 0 | Both windows (5m and 1h) need elevated not-ready ratio | Wait the full 5–10 minutes; the 1-hour window must also exceed the threshold before the critical alert fires |
