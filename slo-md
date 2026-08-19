# SLO Dashboard Detailed Guide

File: `slo-dashboard.yaml`
Dashboard title: `D1C SLO Metrics (v2 Industry Standard)`
UID: `d1c-slo-metrics-v2`

## What this dashboard is for
- Tracks SLO health using industry-standard SRE signals: availability, latency, error rate, and error-budget burn.
- Supports both incident response (fast detection) and daily/weekly reliability management (trends and scorecards).
- Uses HTTP request metrics from Mimir and excludes actuator/health probe routes to avoid noisy/non-user traffic.

## Global filters / variables
- `Mimir Datasource` (`datasource_mimir`): `prometheus`
- `Namespace` (`namespace`): `label_values(target_info, k8s_namespace_name)`
- `App / Job` (`service`): `label_values(http_server_request_duration_bucket, job)`
- `Latency Target (ms)` (`slo_latency_ms`): `200,500,1000,2000`
- `Availability Target (%)` (`slo_availability_pct`): `99,99.5,99.9,99.95,99.99`
- `Deployment` (`deployment`): `label_values(kube_deployment_metadata_generation{namespace=~"$namespace"}, deployment)`

## Panel-by-panel explanation
### SLO Overview (Engineering Objectives)
#### `Availability SLI (30d)` (id: `101`, type: `gauge`)
- **What it shows:** Event-based availability SLI over 30 days vs SLO target variable.
- **Why important:** Primary reliability KPI: shows user-visible success level against target and is used for release/risk decisions.
- **How it works (query logic):**
  - `A` (Availability 30d):

    ```promql
    (sum(increase(http_server_request_duration_count{job=~"$service", http_route!~"/actuator.*|/health.*|/healthz|/ready|/readiness|/live|/liveliness|/ping", http_response_status_code!~"5.."}[30d])) / clamp_min(sum(increase(http_server_request_duration_count{job=~"$service", http_route!~"/actuator.*|/health.*|/healthz|/ready|/readiness|/live|/liveliness|/ping"}[30d])), 1))
    ```
- **How to read:** higher is better; compare against `Availability Target (%)`.

#### `Error Budget Remaining` (id: `102`, type: `gauge`)
- **What it shows:** Google SRE style: remaining monthly error budget for selected availability SLO.
- **Why important:** Critical for SRE alerting: this tells you how fast reliability budget is being consumed and whether you are heading toward an SLO breach.
- **How it works (query logic):**
  - `A` (Budget left):

    ```promql
    clamp(1 - ((1 - (sum(increase(http_server_request_duration_count{job=~"$service", http_route!~"/actuator.*|/health.*|/healthz|/ready|/readiness|/live|/liveliness|/ping", http_response_status_code!~"5.."}[30d])) / clamp_min(sum(increase(http_server_request_duration_count{job=~"$service", http_route!~"/actuator.*|/health.*|/healthz|/ready|/readiness|/live|/liveliness|/ping"}[30d])), 1))) / (1 - ($slo_availability_pct / 100))), 0, 1)
    ```

#### `Error Rate (5m)` (id: `103`, type: `stat`)
- **What it shows:** Current 5xx ratio (SLO error SLI).
- **Why important:** Direct failure signal: helps detect regressions and incident impact quickly.
- **How it works (query logic):**
  - `A` (Error rate):

    ```promql
    sum(rate(http_server_request_duration_count{job=~"$service", http_route!~"/actuator.*|/health.*|/healthz|/ready|/readiness|/live|/liveliness|/ping", http_response_status_code=~"5.."}[5m])) / clamp_min(sum(rate(http_server_request_duration_count{job=~"$service", http_route!~"/actuator.*|/health.*|/healthz|/ready|/readiness|/live|/liveliness|/ping"}[5m])), 1e-9)
    ```
- **How to read:** lower is better; spikes often correlate with deploys or dependency outages.

#### `Latency p95 vs Target` (id: `104`, type: `stat`)
- **What it shows:** p95 / target (>1 means missing latency SLO).
- **Why important:** User experience signal: high latency can violate SLO even before hard failures occur.
- **How it works (query logic):**
  - `A` (p95 / target):

    ```promql
    (histogram_quantile(0.95, sum by (le) (rate(http_server_request_duration_bucket{job=~"$service", http_route!~"/actuator.*|/health.*|/healthz|/ready|/readiness|/live|/liveliness|/ping"}[5m])))) / ($slo_latency_ms / 1000)
    ```
- **How to read:** p95/p99 crossing target indicates degraded user experience and SLO risk.

### Error Budget Burn Rate (Multi-Window — Industry Standard)
#### `Burn Rate 1h (fast)` (id: `111`, type: `stat`)
- **What it shows:** Fast burn: how quickly budget is consuming over 1h. >14.4 ≈ exhausts 30d budget in ~2h (page-worthy).
- **Why important:** Critical for SRE alerting: this tells you how fast reliability budget is being consumed and whether you are heading toward an SLO breach.
- **How it works (query logic):**
  - `A` (1h burn):

    ```promql
    clamp_min((1 - (sum(increase(http_server_request_duration_count{job=~"$service", http_route!~"/actuator.*|/health.*|/healthz|/ready|/readiness|/live|/liveliness|/ping", http_response_status_code!~"5.."}[1h])) / clamp_min(sum(increase(http_server_request_duration_count{job=~"$service", http_route!~"/actuator.*|/health.*|/healthz|/ready|/readiness|/live|/liveliness|/ping"}[1h])), 1))) / (1 - ($slo_availability_pct / 100)), 0)
    ```
- **How to read:** `>1` means burning budget faster than sustainable pace; very high short-window burn means urgent action needed.

#### `Burn Rate 6h` (id: `112`, type: `stat`)
- **What it shows:** Medium window burn rate for ticket/page decisioning.
- **Why important:** Critical for SRE alerting: this tells you how fast reliability budget is being consumed and whether you are heading toward an SLO breach.
- **How it works (query logic):**
  - `A` (6h burn):

    ```promql
    clamp_min((1 - (sum(increase(http_server_request_duration_count{job=~"$service", http_route!~"/actuator.*|/health.*|/healthz|/ready|/readiness|/live|/liveliness|/ping", http_response_status_code!~"5.."}[6h])) / clamp_min(sum(increase(http_server_request_duration_count{job=~"$service", http_route!~"/actuator.*|/health.*|/healthz|/ready|/readiness|/live|/liveliness|/ping"}[6h])), 1))) / (1 - ($slo_availability_pct / 100)), 0)
    ```
- **How to read:** `>1` means burning budget faster than sustainable pace; very high short-window burn means urgent action needed.

#### `Burn Rate 3d (slow)` (id: `113`, type: `stat`)
- **What it shows:** Slow burn over 3d — catches sustained degradation.
- **Why important:** Critical for SRE alerting: this tells you how fast reliability budget is being consumed and whether you are heading toward an SLO breach.
- **How it works (query logic):**
  - `A` (3d burn):

    ```promql
    clamp_min((1 - (sum(increase(http_server_request_duration_count{job=~"$service", http_route!~"/actuator.*|/health.*|/healthz|/ready|/readiness|/live|/liveliness|/ping", http_response_status_code!~"5.."}[3d])) / clamp_min(sum(increase(http_server_request_duration_count{job=~"$service", http_route!~"/actuator.*|/health.*|/healthz|/ready|/readiness|/live|/liveliness|/ping"}[3d])), 1))) / (1 - ($slo_availability_pct / 100)), 0)
    ```
- **How to read:** `>1` means burning budget faster than sustainable pace; very high short-window burn means urgent action needed.

#### `Burn Rate 30d` (id: `114`, type: `stat`)
- **What it shows:** Monthly burn — should stay near 1.0 if on track.
- **Why important:** Critical for SRE alerting: this tells you how fast reliability budget is being consumed and whether you are heading toward an SLO breach.
- **How it works (query logic):**
  - `A` (30d burn):

    ```promql
    clamp_min((1 - (sum(increase(http_server_request_duration_count{job=~"$service", http_route!~"/actuator.*|/health.*|/healthz|/ready|/readiness|/live|/liveliness|/ping", http_response_status_code!~"5.."}[30d])) / clamp_min(sum(increase(http_server_request_duration_count{job=~"$service", http_route!~"/actuator.*|/health.*|/healthz|/ready|/readiness|/live|/liveliness|/ping"}[30d])), 1))) / (1 - ($slo_availability_pct / 100)), 0)
    ```
- **How to read:** `>1` means burning budget faster than sustainable pace; very high short-window burn means urgent action needed.

#### `Burn Rate Over Time (1h / 6h / 3d)` (id: `115`, type: `timeseries`)
- **What it shows:** Google SRE multi-window burn-rate chart for SLO alerting context.
- **Why important:** Critical for SRE alerting: this tells you how fast reliability budget is being consumed and whether you are heading toward an SLO breach.
- **How it works (query logic):**
  - `A` (1h):

    ```promql
    clamp_min((1 - (sum(increase(http_server_request_duration_count{job=~"$service", http_route!~"/actuator.*|/health.*|/healthz|/ready|/readiness|/live|/liveliness|/ping", http_response_status_code!~"5.."}[1h])) / clamp_min(sum(increase(http_server_request_duration_count{job=~"$service", http_route!~"/actuator.*|/health.*|/healthz|/ready|/readiness|/live|/liveliness|/ping"}[1h])), 1))) / (1 - ($slo_availability_pct / 100)), 0)
    ```
  - `B` (6h):

    ```promql
    clamp_min((1 - (sum(increase(http_server_request_duration_count{job=~"$service", http_route!~"/actuator.*|/health.*|/healthz|/ready|/readiness|/live|/liveliness|/ping", http_response_status_code!~"5.."}[6h])) / clamp_min(sum(increase(http_server_request_duration_count{job=~"$service", http_route!~"/actuator.*|/health.*|/healthz|/ready|/readiness|/live|/liveliness|/ping"}[6h])), 1))) / (1 - ($slo_availability_pct / 100)), 0)
    ```
  - `C` (3d):

    ```promql
    clamp_min((1 - (sum(increase(http_server_request_duration_count{job=~"$service", http_route!~"/actuator.*|/health.*|/healthz|/ready|/readiness|/live|/liveliness|/ping", http_response_status_code!~"5.."}[3d])) / clamp_min(sum(increase(http_server_request_duration_count{job=~"$service", http_route!~"/actuator.*|/health.*|/healthz|/ready|/readiness|/live|/liveliness|/ping"}[3d])), 1))) / (1 - ($slo_availability_pct / 100)), 0)
    ```
  - `D` (budget pace (=1)):

    ```promql
    1
    ```
- **How to read:** `>1` means burning budget faster than sustainable pace; very high short-window burn means urgent action needed.

#### `Budget Consumed (30d)` (id: `116`, type: `gauge`)
- **What it shows:** Fraction of monthly error budget already used.
- **Why important:** Operational visibility: contributes context for reliability triage and daily SRE review.
- **How it works (query logic):**
  - `A` (Consumed):

    ```promql
    clamp((1 - (sum(increase(http_server_request_duration_count{job=~"$service", http_route!~"/actuator.*|/health.*|/healthz|/ready|/readiness|/live|/liveliness|/ping", http_response_status_code!~"5.."}[30d])) / clamp_min(sum(increase(http_server_request_duration_count{job=~"$service", http_route!~"/actuator.*|/health.*|/healthz|/ready|/readiness|/live|/liveliness|/ping"}[30d])), 1))) / (1 - ($slo_availability_pct / 100)), 0, 2)
    ```

### Golden Signals (RED + Latency)
#### `Request Rate (RPS)` (id: `121`, type: `timeseries`)
- **What it shows:** Throughput golden signal.
- **Why important:** Traffic context: needed to interpret error/latency changes and understand blast radius.
- **How it works (query logic):**
  - `A` (All):

    ```promql
    sum(rate(http_server_request_duration_count{job=~"$service", http_route!~"/actuator.*|/health.*|/healthz|/ready|/readiness|/live|/liveliness|/ping"}[5m]))
    ```
  - `B` (5xx):

    ```promql
    sum(rate(http_server_request_duration_count{job=~"$service", http_route!~"/actuator.*|/health.*|/healthz|/ready|/readiness|/live|/liveliness|/ping", http_response_status_code=~"5.."}[5m]))
    ```

#### `Availability SLI (short window)` (id: `122`, type: `timeseries`)
- **What it shows:** Rolling availability vs SLO target.
- **Why important:** Primary reliability KPI: shows user-visible success level against target and is used for release/risk decisions.
- **How it works (query logic):**
  - `A` (Availability):

    ```promql
    (1 - sum(rate(http_server_request_duration_count{job=~"$service", http_route!~"/actuator.*|/health.*|/healthz|/ready|/readiness|/live|/liveliness|/ping", http_response_status_code=~"5.."}[5m])) / clamp_min(sum(rate(http_server_request_duration_count{job=~"$service", http_route!~"/actuator.*|/health.*|/healthz|/ready|/readiness|/live|/liveliness|/ping"}[5m])), 1e-9))
    ```
  - `B` (SLO target):

    ```promql
    $slo_availability_pct / 100
    ```
- **How to read:** higher is better; compare against `Availability Target (%)`.

#### `Latency p50 / p95 / p99 vs SLO` (id: `123`, type: `timeseries`)
- **What it shows:** Latency SLI percentiles vs target.
- **Why important:** User experience signal: high latency can violate SLO even before hard failures occur.
- **How it works (query logic):**
  - `A` (p50):

    ```promql
    histogram_quantile(0.5, sum by (le) (rate(http_server_request_duration_bucket{job=~"$service", http_route!~"/actuator.*|/health.*|/healthz|/ready|/readiness|/live|/liveliness|/ping"}[5m])))
    ```
  - `B` (p95):

    ```promql
    histogram_quantile(0.95, sum by (le) (rate(http_server_request_duration_bucket{job=~"$service", http_route!~"/actuator.*|/health.*|/healthz|/ready|/readiness|/live|/liveliness|/ping"}[5m])))
    ```
  - `C` (p99):

    ```promql
    histogram_quantile(0.99, sum by (le) (rate(http_server_request_duration_bucket{job=~"$service", http_route!~"/actuator.*|/health.*|/healthz|/ready|/readiness|/live|/liveliness|/ping"}[5m])))
    ```
  - `D` (SLO target):

    ```promql
    $slo_latency_ms / 1000
    ```
- **How to read:** p95/p99 crossing target indicates degraded user experience and SLO risk.

#### `Error Rate vs SLO (0.1%)` (id: `124`, type: `timeseries`)
- **What it shows:** Error SLI with typical 0.1% line for 99.9% availability.
- **Why important:** Direct failure signal: helps detect regressions and incident impact quickly.
- **How it works (query logic):**
  - `A` (Error rate):

    ```promql
    sum(rate(http_server_request_duration_count{job=~"$service", http_route!~"/actuator.*|/health.*|/healthz|/ready|/readiness|/live|/liveliness|/ping", http_response_status_code=~"5.."}[5m])) / clamp_min(sum(rate(http_server_request_duration_count{job=~"$service", http_route!~"/actuator.*|/health.*|/healthz|/ready|/readiness|/live|/liveliness|/ping"}[5m])), 1e-9)
    ```
  - `B` (0.1% line):

    ```promql
    0.001
    ```
  - `C` (Allowed error ratio):

    ```promql
    1 - ($slo_availability_pct / 100)
    ```
- **How to read:** lower is better; spikes often correlate with deploys or dependency outages.

### Good vs Bad Events (SLI building blocks)
#### `Good Requests vs Bad (5xx) Events` (id: `131`, type: `timeseries`)
- **What it shows:** Industry-standard event-based SLI components: good and bad request rates.
- **Why important:** Operational visibility: contributes context for reliability triage and daily SRE review.
- **How it works (query logic):**
  - `A` (Good):

    ```promql
    sum(rate(http_server_request_duration_count{job=~"$service", http_route!~"/actuator.*|/health.*|/healthz|/ready|/readiness|/live|/liveliness|/ping", http_response_status_code!~"5.."}[5m]))
    ```
  - `B` (Bad (5xx)):

    ```promql
    sum(rate(http_server_request_duration_count{job=~"$service", http_route!~"/actuator.*|/health.*|/healthz|/ready|/readiness|/live|/liveliness|/ping", http_response_status_code=~"5.."}[5m]))
    ```

#### `Top Jobs Missing Latency SLO (p95)` (id: `132`, type: `bargauge`)
- **What it shows:** Jobs whose p95 exceeds latency target.
- **Why important:** User experience signal: high latency can violate SLO even before hard failures occur.
- **How it works (query logic):**
  - `A` ({{job}}):

    ```promql
    topk(10, histogram_quantile(0.95, sum by (le, job) (rate(http_server_request_duration_bucket{job=~"$service", http_route!~"/actuator.*|/health.*|/healthz|/ready|/readiness|/live|/liveliness|/ping"}[5m]))) * 1000)
    ```
- **How to read:** p95/p99 crossing target indicates degraded user experience and SLO risk.

### Per-Service SLO Compliance
#### `Per-Job SLO Scorecard` (id: `141`, type: `table`)
- **What it shows:** Availability 7d/30d, error rate, p95, burn 1h — engineering standup view.
- **Why important:** Prioritization view: identifies which services need immediate remediation.
- **How it works (query logic):**
  - `A` (avail_7d):

    ```promql
    sum by (job) (increase(http_server_request_duration_count{job=~"$service", http_route!~"/actuator.*|/health.*|/healthz|/ready|/readiness|/live|/liveliness|/ping", http_response_status_code!~"5.."}[7d])) / clamp_min(sum by (job) (increase(http_server_request_duration_count{job=~"$service", http_route!~"/actuator.*|/health.*|/healthz|/ready|/readiness|/live|/liveliness|/ping"}[7d])), 1)
    ```
  - `B` (avail_30d):

    ```promql
    sum by (job) (increase(http_server_request_duration_count{job=~"$service", http_route!~"/actuator.*|/health.*|/healthz|/ready|/readiness|/live|/liveliness|/ping", http_response_status_code!~"5.."}[30d])) / clamp_min(sum by (job) (increase(http_server_request_duration_count{job=~"$service", http_route!~"/actuator.*|/health.*|/healthz|/ready|/readiness|/live|/liveliness|/ping"}[30d])), 1)
    ```
  - `C` (error_rate):

    ```promql
    sum by (job) (rate(http_server_request_duration_count{job=~"$service", http_route!~"/actuator.*|/health.*|/healthz|/ready|/readiness|/live|/liveliness|/ping", http_response_status_code=~"5.."}[5m])) / clamp_min(sum by (job) (rate(http_server_request_duration_count{job=~"$service", http_route!~"/actuator.*|/health.*|/healthz|/ready|/readiness|/live|/liveliness|/ping"}[5m])), 1e-9)
    ```
  - `D` (p95):

    ```promql
    histogram_quantile(0.95, sum by (le, job) (rate(http_server_request_duration_bucket{job=~"$service", http_route!~"/actuator.*|/health.*|/healthz|/ready|/readiness|/live|/liveliness|/ping"}[5m])))
    ```
- **How to read:** use this for prioritization—focus first on highest-traffic services with worst SLO indicators.

### Saturation / Dependency Signals
#### `DB Pool Pending (saturation)` (id: `151`, type: `timeseries`)
- **What it shows:** Dependency saturation often precedes SLO burn.
- **Why important:** Dependency/saturation signal: helps find root cause outside the app itself.
- **How it works (query logic):**
  - `A` ({{job}} / {{pool_name}}):

    ```promql
    sum by (job, pool_name) (db_client_connections_pending_requests{job=~"$service"})
    ```

#### `HTTP Client Upstream p95 (dependencies)` (id: `152`, type: `timeseries`)
- **What it shows:** Outbound dependency latency impacting user SLO.
- **Why important:** Dependency/saturation signal: helps find root cause outside the app itself.
- **How it works (query logic):**
  - `A` (p95 {{server_address}}):

    ```promql
    histogram_quantile(0.95, sum by (le, server_address) (rate(http_client_request_duration_bucket{job=~"$service"}[5m])))
    ```

## Suggested SRE workflow during incident
1. Check **Burn Rate 1h/6h** and **Error Rate** to confirm impact severity.
2. Validate **Availability SLI** and **Latency p95/p99** against targets.
3. Use **Top Jobs Missing Latency SLO** and **Per-Job SLO Scorecard** to identify affected services.
4. Correlate with **DB Pool Pending** and **HTTP Client Upstream p95** for dependency root cause.
5. Track recovery by watching burn rates return toward `1` and error rate fall.

## Notes / caveats
- If a panel shows `No data`, first verify selected `App / Job`, time range, and whether that service has traffic.
- Burn-rate math depends on `Availability Target (%)`; avoid setting exactly `100` to prevent divide-by-zero behavior.
- Namespace list depends on labels present in your Mimir data for `target_info`.
