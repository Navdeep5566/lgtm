# PROD Platform IBM MQ Dashboard — Import & Migration Notes

Complete history of converting the New Relic dashboard **PROD MUOB/MUQ6 Platform IBM MQ** into a production Grafana dashboard on the LGTM stack (`Prod-mou-platform .yaml`).

| Field | Value |
|---|---|
| K8s resource | `GrafanaDashboard` / `prod-muq6-platform-ibm-mq` |
| Namespace | `observability` |
| Grafana UID | `prod-muq6-platform-ibm-mq` |
| Folder | `LGTM` |
| Datasources | `${datasource_mimir}` (metrics), `${datasource_loki}` (logs) |
| Current version | **1** (production) |

---

## 1. Starting point — New Relic JSON export

The original file was a **New Relic dashboard JSON export**, not a Grafana dashboard. It contained **26 widgets** across three NR pages:

| NR page | Widgets (examples) |
|---|---|
| **Queue Managers** | Queue Managers count, Connections, Errors, Channel Messages, Queues info table, FileSystem Usage, MQ Server Logs |
| **Queues** | Queue Depth, Attribute Usage, GET/PUT messages, Time Since GET/PUT, Oldest Message Age, Queue Time, Expired Messages |
| **Channels** | Channel Messages, Bytes Sent/Received, Current Instances, Time Since Message |

### NR query patterns (source of truth)

- Metrics filtered by `entity.type = 'IBM_MQ_MANAGER'` and host patterns like `'host.hostname' LIKE '%muobprodmq%'`
- Queue panels used NR metrics such as `ibmmq_queue_mqput_mqput1_count`, `ibmmq_queue_time_since_get`, `ibmmq_queue_time_since_put`
- Logs used NR Log query: `SELECT message FROM Log WHERE host.name = 'muobprodmq01' OR hostname = ... OR host = ...`

### First conversion (NR JSON → Grafana YAML)

| NR concept | Grafana equivalent |
|---|---|
| NRQL `FROM Metric select ...` | PromQL on Mimir |
| NR Log query on `host.name` | LogQL on Loki with `host_name` / `host` labels |
| NR billboard / line / bar / table | Grafana stat / timeseries / barchart / table |
| NR page layout (column/row) | Grafana `gridPos` (24-column grid) |
| Fixed NR account datasource | Template variables `${datasource_mimir}`, `${datasource_loki}` |

The file was wrapped as a **`GrafanaDashboard` CRD** (same pattern as `service-helath-sre-overview.yaml`, `sla-dashboard.yaml`, etc.) so it can be applied with:

```bash
kubectl apply -f "Prod-mou-platform .yaml"
```

### Problems with the original NR export file

- JSON was **truncated / invalid** (closed early; widgets like Queues info and FileSystem Usage were cut off)
- Widgets had to be **reconstructed** from the partial export plus NR query screenshots
- NR layout (column/row) does not map 1:1 to Grafana — panels were placed on a 24-column grid with row headers

---

## 2. Challenges encountered (why panels showed “No data”)

Each issue below blocked one or more panels until fixed in the YAML.

### Challenge 1 — Wrong metric prefix

| Assumption (wrong) | Reality in Mimir |
|---|---|
| `ibmq_*` (single `m`, from early NR naming) | **`ibmmq_*`** (double `m`) |

Explore proof:

```promql
count by (__name__) ({__name__=~"ibmq_.*"})   # → no data
topk(20, count by (__name__) ({qmgr=~".+"}))  # → ibmmq_channel_*, ibmmq_qmgr_*, ibmmq_queue_* (~174 series)
```

**Fix:** Renamed all panel queries and variables from `ibmq_*` → `ibmmq_*`.

---

### Challenge 2 — Wrong host / cluster filter

The NR dashboard filtered hosts like `%muobprodmq%`. The first Grafana draft used **`muq6produq`**, which is a **different cluster**. Metrics existed in Mimir but not for that host filter.

**Fix:** Removed hardcoded host filters for queue metrics; added dynamic Mimir variables where needed. Queue manager filtering uses the **`qmgr`** label on qmgr/channel metrics only.

---

### Challenge 3 — `qmgr` label missing on queue metrics

Explore showed queue series look like:

```promql
ibmmq_queue_depth{queue="DFT.ALERTS.EVENT.QUEUE"}
```

There is **no `qmgr` label** on `ibmmq_queue_*` series in this environment. Dashboard queries used `qmgr=~"$qmgr"` on queue panels, which **excluded every series**.

**Fix (v8):** All `ibmmq_queue_*` panels use **`{queue=~"$queue"}` only** — no `qmgr` filter on queue metrics.

Explore verification:

```promql
topk(30, count by (__name__) ({__name__=~"ibmmq_queue_.*"}))   # 30 metrics, 112 series
avg by (queue) (ibmmq_queue_mqput_mqput1_count)                 # 56 queues with data
```

---

### Challenge 4 — Wrong PUT metric name

Several wrong names were tried before matching NR:

| Attempt | Result |
|---|---|
| `ibmmq_queue_input_mqput1_count` | Not in Mimir |
| `ibmmq_queue_mqput_count` | Not in Mimir |
| **`ibmmq_queue_mqput_mqput1_count`** | Matches NR `average(ibmmq_queue_mqput_mqput1_count)` |

**Fix (v7):** PUT Messages / PUT Count panels use `ibmmq_queue_mqput_mqput1_count` with `avg by (queue)` / `topk`.

---

### Challenge 5 — `rate()` vs NR `latest()` / `average()`

IBM MQ queue counters in NR are read as **snapshot gauges** (`latest()`, `average()`), not Prometheus counter rates.

**Fix (v6):** Replaced `rate(...[5m])` with **`max by (queue)`** or **`avg by (queue)`** on GET/PUT panels to mirror NR behaviour.

---

### Challenge 6 — `time_since_get` / `time_since_put` not in Mimir

NR panels use:

- `ibmmq_queue_time_since_get`
- `ibmmq_queue_time_since_put`

These metrics are **not exported** by the current IBM MQ → Mimir pipeline:

```promql
count(ibmmq_queue_time_since_get)   # → no data
count(ibmmq_queue_time_since_put)   # → no data
```

The YAML cannot create metrics that are not ingested. NR shows data because its IBM MQ integration exports MQI queue statistics attributes that this collector does not (yet).

**Fix (v10 — proxy panels):**

| NR panel | Missing metric | Grafana proxy (from Mimir) |
|---|---|---|
| Time Since Last GET | `ibmmq_queue_time_since_get` | `ibmmq_queue_average_queue_time_seconds` |
| Time Since Last PUT | `ibmmq_queue_time_since_put` | `ibmmq_queue_oldest_message_age` |

Verify proxies in Explore:

```promql
max by (queue) (ibmmq_queue_average_queue_time_seconds)
max by (queue) (ibmmq_queue_oldest_message_age)
```

**Future:** Enable MQI queue statistics on the IBM MQ OTel/Prometheus collector (`ATTR_Q_SINCE_GET`, `ATTR_Q_SINCE_PUT`), then switch panels back to the exact NR metric names.

---

### Challenge 7 — MQ Server Logs: wrong Loki label

The logs panel initially used **`{qmgr=~"$qmgr"}`**. Loki log streams in this cluster have **no `qmgr` label** → empty panel.

NR filters by **host**, not queue manager:

```sql
WHERE host.name = 'muobprodmq01' OR hostname = 'muobprodmq01' OR host = 'muobprodmq01'
```

**Fix (v11 → v12):** Host-based Loki queries + dynamic variables discovered from Loki (no hardcoded NR hostnames).

---

### Challenge 8 — Hardcoded New Relic hostnames

Early versions hardcoded `muobprodmq01`, `muobprodmq02`, `UOB_UQ01`, etc. Those only work on the **NR cluster**, not necessarily yours.

**Fix (v12):** Loki-discovered variables:

| Variable | Loki query |
|---|---|
| Log Host (host_name) | `label_values({host_name=~".+"}, host_name)` |
| Log Host (host) | `label_values({host=~".+"}, host)` |
| Log Host (hostname) | hidden fallback |

Panel queries use `$log_host` / `$log_host_alt` with **`includeAll: true`** and **`allValue: .+`**.

---

### Challenge 9 — Loki line filter parse error (v13/v14)

An IBM-MQ-only line filter caused:

`parse error at line 1, col 22: invalid char escape`

Loki uses **Go RE2** — `(?i)` and `\\.` are invalid.

**Fix (v14):** Simplified regex (still MQ-focused during troubleshooting).

**Production (final):** Line filter removed entirely — panel shows **all host logs** for selected hosts (see last section).

---

## 3. YAML changes summary (v1 → production)

### Dashboard shell (GrafanaDashboard CRD)

- Valid JSON embedded under `spec.json`
- Aligned with other repo dashboards: annotations, timepicker, `graphTooltip: 1`, `refresh: 30s`, folder `LGTM`
- Tags: `d1c`, `ibm-mq`, `prod`, `platform`, `middleware`, `ibmmq`

### Template variables (final)

| Variable | Purpose | Source |
|---|---|---|
| `datasource_mimir` | Metrics | Grafana datasource picker |
| `datasource_loki` | Logs | Grafana datasource picker |
| `qmgr` | Queue manager filter (qmgr/channel metrics) | `label_values({__name__=~"ibmmq_.*"}, qmgr)` |
| `queue` | Queue filter | `label_values({__name__=~"ibmmq_queue_.*"}, queue)` |
| `channel` | Channel filter | `label_values({__name__=~"ibmmq_channel_.*", qmgr=~"$qmgr"}, channel)` |
| `log_host` | Log host filter | Loki `host_name` label (multi + All) |
| `log_host_alt` | Alternate host label | Loki `host` label (hidden fallback) |
| `hostname` / `hostname_alt` | Legacy Mimir host vars | Hidden (unused on queue metrics) |

### Panel query remapping (metrics that work)

| Panel | Metric(s) |
|---|---|
| Queue Managers count | `ibmmq_qmgr_active_listeners` |
| Connections | `ibmmq_qmgr_active_listeners` + `ibmmq_qmgr_active_services` |
| Errors | `ibmmq_qmgr_*` error counters |
| Messages / Channel Messages | `ibmmq_channel_messages` |
| Queues info table | `ibmmq_queue_depth`, attribute usage, put/get/expired |
| GET Messages | `ibmmq_queue_mqget_count` |
| PUT Messages | `ibmmq_queue_mqput_mqput1_count` |
| Time Since GET | `ibmmq_queue_average_queue_time_seconds` (proxy) |
| Time Since PUT | `ibmmq_queue_oldest_message_age` (proxy) |
| Channels section | `ibmmq_channel_messages`, bytes sent/rcvd, cur_inst, time_since_msg |
| MQ Server Logs | `{host_name=~"$log_host"}` (all log lines for selected hosts) |

### Diagnostics row (troubleshooting only — removed for production)

During migration, a **Diagnostics — IBM MQ metrics in Mimir** row was added (metric counts, top metrics tables, time_since availability stats). It helped confirm `ibmmq_*` naming and Explore results. **Removed in production** — see final section below.

---

## 4. Current dashboard layout (33 panels)

| Section | Panels |
|---|---|
| **Queue Managers** | Stats (QM count, Connections, Errors) → Messages / Queues Count → Queues info table → FileSystem Usage → **MQ Server Logs** |
| **Queues** | Depth (bar + timeseries) → GET row → PUT row → Oldest message / queue time → Expired messages |
| **Channels** | Messages + bytes → instances + time since message |

Compact `gridPos` from `y=0` with no gaps left by removed diagnostics.

---

## 5. Verify after apply

1. Open **PROD Platform - IBM MQ** in Grafana (folder **LGTM**).
2. **Queue Manager** dropdown lists your QMGRs (e.g. from `ibmmq_qmgr_active_listeners`).
3. **Queue** dropdown lists queues; depth / GET / PUT panels show data.
4. **Time Since GET/PUT** show proxy metrics (not exact NR `time_since_*` until collector is updated).
5. **MQ Server Logs:** set **Log Host (host_name)** to **All** or pick a specific host.

### Explore queries (PromQL — run in Mimir)

```promql
# Confirm IBM MQ metrics exist
count by (__name__) ({__name__=~"ibmmq_.*"})
topk(20, count by (__name__) ({qmgr=~".+"}))

# Queue panels
topk(30, count by (__name__) ({__name__=~"ibmmq_queue_.*"}))
avg by (queue) (ibmmq_queue_mqput_mqput1_count)
max by (queue) (ibmmq_queue_average_queue_time_seconds)
max by (queue) (ibmmq_queue_oldest_message_age)

# time_since — expect no data until collector exports them
count(ibmmq_queue_time_since_get)
count(ibmmq_queue_time_since_put)
```

### Explore queries (LogQL — run in Loki)

```logql
{host_name=~".+"}
{host=~".+"}
```

If both return empty → logs are not in Loki yet (pipeline issue, not dashboard).

---

## 6. Apply command

```bash
kubectl apply -f "Prod-mou-platform .yaml"
```

---

## 7. Production deployment (final cleanup) — keep this last

When all panels were confirmed working in Explore, the dashboard was prepared for **production deploy** (dashboard JSON **version reset to 1**).

### Removed — diagnostics (10 panels)

- Row **Diagnostics — IBM MQ metrics in Mimir**
- Metric naming text panel
- Stat panels: IBM MQ metric names, time series counts, qmgr counts, PUT/time_since availability
- Tables: Top `ibmmq_*` metrics, Available `ibmmq_queue_*` metrics

These were **troubleshooting aids only** and are not needed in production.

### Production layout

- **33 panels** in a compact sequential grid (`y=0` → `y=90`)
- Row headers: **Queue Managers**, **Queues**, **Channels**
- Clean panel titles (removed “proxy”, “v10”, diagnostic wording)
- Unused Mimir hostname variables hidden

### MQ Server Logs — all host logs

| Before (troubleshooting) | After (production) |
|---|---|
| Title: MQ Server Logs (IBM MQ lines only) | Title: **MQ Server Logs** |
| Query with `\|~ "AMQ..."` line filter | **`{host_name=~"$log_host"}` only** — all log lines |
| Filtered to MQ syslog patterns | Shows **all logs** for selected host(s) |

Production LogQL:

```logql
{host_name=~"$log_host"}
```

Hidden fallback (alternate label):

```logql
{host=~"$log_host_alt"}
```

**Log Host (host_name)** defaults to **All** (`allValue: .+`), so all hosts’ logs appear unless you narrow the dropdown.

### Production dashboard metadata

```yaml
title: PROD Platform - IBM MQ
description: IBM MQ platform monitoring — queue managers, queues, channels, and server logs.
         Metrics: ibmmq_* via Mimir; logs via Loki.
version: 1
refresh: 30s
```

### Optional follow-up (not blocking production)

| Item | Action |
|---|---|
| Exact NR Time Since GET/PUT | Enable `ATTR_Q_SINCE_GET` / `ATTR_Q_SINCE_PUT` on IBM MQ collector → switch panels to `ibmmq_queue_time_since_get/put` |
| MQ-specific log filtering | Re-add a RE2-safe `\|~` filter if syslog is noisy (avoid `(?i)` and `\\.` in Loki) |
| Separate NR cluster hosts | Never hardcode `muobprodmq01` — keep Loki `label_values` discovery |

---

*Last updated: production deploy — diagnostics removed, layout aligned, MQ Server Logs shows all host logs.*
