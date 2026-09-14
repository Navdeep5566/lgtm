# D1C L3 Production Dashboard — Import & Migration Notes

| Field | Value |
|---|---|
| **Dashboard title** | D1C L3 Production Dashboard |
| **K8s resource** | `GrafanaDashboard` / `d1c-l3-production` |
| **Namespace** | `observability` |
| **Grafana UID** | `d1c-l3-production` |
| **Folder** | `LGTM` |
| **Current version** | **7** |
| **Datasources** | `${datasource_mimir}` (Prometheus/Mimir), `${datasource_loki}` (Loki) |
| **Default time range** | Last 3 hours (`now-3h`) — matches NR log widgets |
| **Refresh** | 30s |
| **NR pages → Grafana rows** | POD1, POD2, EWB, SBNA (4 collapsed rows) |

---

## 1. Dashboard overview

This dashboard is the Grafana equivalent of the New Relic **D1C L3 Production Dashboard**, used for daily L3 production operations across four platform pods (Postgres-backed application stacks). It combines:

- **Infrastructure metrics** — per-node CPU, memory, and HTTP response time from Mimir (OpenTelemetry / host metrics).
- **Application log analysis** — critical errors, database exceptions, slow SQL, and JVM/batch exceptions from Loki.
- **Operational placeholders** — NR panels that depended on CloudWatch RDS, synthetic monitoring, or NR-only session metrics are retained as text/stat proxies or notes where Mimir/Loki data is unavailable.

The dashboard is deployed as a **GrafanaDashboard CRD** (`L3-production-Dashboard.yaml`) and targets the LGTM Grafana instance via `instanceSelector: dashboards: grafana-lgtm`.

### Layout (production v7)

Each NR page is a **collapsed row**. Inside each row:

1. **Header** — full-width markdown title (`w=24`).
2. **Node grid** — 3 columns × N rows: CPU | Heap memory | Response time (`w=8` each).
3. **Memory summary** (POD1 only) — High Memory + Overall Memory side by side.
4. **Log guidance banner** — “Log panels — how to use” (how to fix empty log panels).
5. **Log panels** — full-width stacked log tables (`w=24`).
6. **Footer** — session/DB/synthetic placeholder panels in a 3-column grid.

---

## 2. Dashboard variables

| Variable | Source | Purpose |
|---|---|---|
| **Mimir Datasource** | Grafana datasource picker | Metrics (Prometheus-compatible Mimir) |
| **Loki Datasource** | Grafana datasource picker | Logs |
| **Log Namespace** | Loki `label_values({}, k8s_namespace_name)` | Scope logs to Kubernetes namespace(s). Uses `allValue: .+` (Loki-safe). |
| **Log Service** | Loki `label_values({k8s_namespace_name=~"$log_namespace"}, service_name)` | Scope logs to service(s). Uses `allValue: .+`. |
| **Platform (NR page)** | Custom: POD1 / POD2 / EWB / SBNA | Organizational — expand the matching row |
| **Namespace** | Mimir `label_values(target_info, k8s_namespace_name)` | K8s namespace for metrics (optional) |
| **Node / Host** | Mimir `label_values(system_cpu_time, host_name)` | Dynamic host filter — **replaces hardcoded NR hostnames** (`nuobprodapp*`, `muobprodapp*`, etc.) |
| **App / Job** | Mimir `label_values(http_server_request_duration_bucket, job)` | HTTP / Tomcat job filter for response-time panels |
| **Log Pod (optional)** | Loki `k8s_pod_name` (hidden) | Fine-grained pod filter when needed |

**Important for log panels:** If a log panel shows no data, set **Log Namespace** and **Log Service** at the top of the dashboard and/or widen the time range. An empty panel usually means no matching log lines in the interval — not a broken query.

---

## 3. NR → Grafana metric & log mapping

| NR source | Grafana panel type | Grafana query |
|---|---|---|
| `SystemSample` `cpuPercent` | Timeseries | `system_cpu_time{state="idle"}` → CPU % via `host_name=~"$node"` |
| `SystemSample` `memoryUsedPercent` | Timeseries | `system_memory_usage{state="used"}` / total → memory % |
| `apm.service.memory.heap.*` | Timeseries | **`system_memory_usage`** (node memory GB) — JVM heap metrics not in Mimir |
| `ProcessSample` `memoryResidentSizeBytes` | Timeseries | `system_memory_usage{state="used"}` (POD1 High Memory) |
| `apm.service.transaction.duration` | Timeseries | `histogram_quantile` on `http_server_request_duration_bucket` |
| `apm.service.cpu.usertime.utilization` | Timeseries | Same CPU formula as SystemSample |
| `logger.log-table-widget` / `FROM Log` | Logs | LogQL: `{k8s_namespace_name=~"$log_namespace", service_name=~"$log_service"} \|~ "<NR message filter>"` |
| `DatabaseSessionSample` | Stat / timeseries proxy | `db_client_connections_usage` — **not equivalent to NR session counts** |
| `aws.rds.CPUUtilization` | Text panel | Not in Mimir — needs CloudWatch/RDS integration |
| `SyntheticCheck` / PLPM | Text panel | Not in Mimir — needs synthetic monitoring |
| `viz.markdown` | Text | Markdown preserved |
| `viz.billboard` | Stat | Session billboard proxied via DB pool metric |

---

## 4. Panel inventory (current Grafana dashboard — v7)

Panels removed from NR export for production: **Other Dashboards** and NR-only application log panels (RUD/Mobile, SCS, RT, MSJ, MQ, Reports on POD1). See [Section 6](#6-modifications-during-migration-v1--v7) for details.

### POD1 Dashboard — 6.19 (Postgres)

| Panel | Type | NR origin | Grafana behaviour |
|---|---|---|---|
| Markdown | Text | Page title | `# POD1 Dashboard - 6.19 (Postgres)` |
| Node 01–04 CPU | Timeseries | `SystemSample.cpuPercent` | CPU % from `system_cpu_time`, filter `$node` |
| Node 01–04 Heap Memory Usage | Timeseries | `apm.service.memory.heap` | **Node system memory (GB)** from `system_memory_usage` — not JVM heap |
| Node 01–04 Response Time | Timeseries | `apm.service.transaction.duration` | HTTP p50/p95 ms from `http_server_request_duration_bucket` |
| High Memory Usage | Timeseries | `ProcessSample.memoryResidentSizeBytes` | Used system memory (GB) per host |
| Overall Memory Usage | Timeseries | `SystemSample.memoryUsedPercent` | Memory used % per host |
| Log panels — how to use | Text | Added in v7 | Guidance when log panels are empty |
| Critical Application Errors (App & Batch JVM) | Logs | NR Log | OOM, hung, MQ errors; excludes FFDC noise |
| Major Database Exception | Logs | NR Log | SQLException, PSQLException, deadlock, timeout, BadSqlGrammar, etc. |
| Imperforment DB Queries | Logs | NR Log | `RSlow[.]SQL[.]Warn`, queries >40s or >10k records |
| All Exception in Application JVM since last 3 hours | Logs | NR Log | `exception\|error`, excludes `was_ffdc` |
| All Exception in the BATCH JVM since last 3 hours | Logs | NR Log | Batch JVM exceptions/errors |
| All Exception in the Report Delivery Batch since last 3 hours | Logs | NR Log | Report delivery batch errors |
| Exception in Config Server Logs since last 3 hours | Logs | NR Log | Config server exceptions |
| DB CPU utilization | Text | `aws.rds.CPUUtilization` | Placeholder — needs CloudWatch |
| Per-host CPU usage | Timeseries | `apm.service.cpu.usertime.utilization` | CPU % proxy |
| Per Tenant User Session | Timeseries | `DatabaseSessionSample` | `db_client_connections_usage` proxy |
| Login Page Load time | Text | `SyntheticCheck` | Placeholder — needs synthetic monitoring |

### POD2 Dashboard — 6.18 (Postgres)

| Panel | Type | NR origin | Grafana behaviour |
|---|---|---|---|
| Markdown | Text | Page title | POD2 header |
| Node 01–04 CPU / Heap / Response Time | Timeseries | Same as POD1 | 4 nodes × 3 metrics |
| Log panels — how to use | Text | Added v7 | Log guidance banner |
| Major Exception (OOM, heap, rollback, hung, thread pool) | Logs | NR Log | Critical JVM/application errors |
| DB Exception (timeout, BadSql, connection, SQLException) | Logs | NR Log | Database exception patterns |
| All Exception in APP JVM since last 3 hours | Logs | NR Log | Application JVM errors |
| All Exception in BATCH JVM since last 3 hours | Logs | NR Log | Batch JVM errors |
| All Exception in RT JVM since last 3 hours | Logs | NR Log | Real-time JVM errors |
| All Exception in MSJ JVM since last 3 hours | Logs | NR Log | MSJ JVM errors |
| All Exception in MQ since last 3 hours | Logs | NR Log | MQ errors |
| Per Tenant DB Session | Timeseries | `DatabaseSessionSample` | DB pool proxy |
| Login Page Load time | Text | `SyntheticCheck` | Placeholder |
| User Sessions | Stat | `DatabaseSessionSample` billboard | DB pool proxy |
| Per Tenant UOB Session | Timeseries | `DatabaseSessionSample` | DB pool proxy |

### EWB Dashboard — 6.17 (Postgres)

| Panel | Type | NR origin | Grafana behaviour |
|---|---|---|---|
| Markdown | Text | Page title | EWB header |
| Node 01–03 CPU / Heap / Response Time | Timeseries | Same pattern | 3 nodes × 3 metrics |
| Log panels — how to use | Text | Added v7 | Log guidance banner |
| Major Exception | Logs | NR Log | OOM, hung, rollback, etc. |
| DB Exception | Logs | NR Log | Database errors |
| All Exception in Application JVM since last 3 hours | Logs | NR Log | App JVM errors |
| All Exception in the Payment Aggregator MS | Logs | NR Log | Payment aggregator errors |
| All Exception in the HK MS | Logs | NR Log | HK microservice errors |
| All Exception in the BATCH JVM since last 3 hours | Logs | NR Log | Batch errors |
| All Exception on Msi Server since last 3 hours | Logs | NR Log | MSI server errors |
| All Exception on RT Server since last 3 hours | Logs | NR Log | RT server errors |
| Per Tenant UOB Session | Timeseries | `DatabaseSessionSample` | Proxy |
| Login Page Load time | Text | `SyntheticCheck` | Placeholder |
| User Sessions | Stat | Billboard | Proxy |

### SBNA Dashboard — 6.17 (Postgres)

| Panel | Type | NR origin | Grafana behaviour |
|---|---|---|---|
| Markdown | Text | Page title | SBNA header |
| Node 01–03 CPU / Heap / Response Time | Timeseries | Same pattern | 3 nodes × 3 metrics |
| Log panels — how to use | Text | Added v7 | Log guidance banner |
| Critical Application Errors (App & Batch JVM) | Logs | NR Log | OOM, MQ, hung, etc. |
| Major Database Exception | Logs | NR Log | DB exception patterns |
| Imperforment DB Queries | Logs | NR Log | Slow SQL warnings |
| All Exception in Application JVM | Logs | NR Log | App JVM errors |
| All Exception in the BATCH JVM since last 3 hours | Logs | NR Log | Batch errors |
| All Exception in the Web since last 3 hours | Logs | NR Log | Web tier errors |
| User Sessions | Stat | Billboard | DB pool proxy |

---

## 5. Log panel message filters (NR logic preserved)

Each log panel applies **NR message filters** on top of Loki stream selectors. Examples:

| Panel | Key LogQL line filters |
|---|---|
| Critical Application Errors | `OutOfMemoryError`, `Java heap space`, `RollbackException`, `hung`, MQ errors (`NAMQ*`, `AMQ*`) |
| Major Database Exception | `SQLException`, `PSQLException`, `Deadlock`, `Transaction timed out`, `BadSqlGrammarException`, `DSRA0010E`, `Validation failure` |
| Imperforment DB Queries | `RSlow[.]SQL[.]Warn`, `more than 40 seconds`, `10000 records`, `Total Execution Time` |
| Application JVM exceptions | `(?i)(exception\|error)` excluding `was_ffdc` |
| Batch JVM exceptions | `(?i)(exception\|error)` excluding `bat_ffdc` |

Stream selector (all log panels):

```logql
{k8s_namespace_name=~"$log_namespace", service_name=~"$log_service"} |~ "<panel-specific filter>"
```

---

## 6. Modifications during migration (v1 → v7)

| Version | Change |
|---|---|
| **v1** | Initial conversion from broken NR JSON to GrafanaDashboard CRD; 4 collapsed rows; regex-based widget recovery |
| **v2** | Replaced hardcoded NR host patterns (`nuobprodapp*`, etc.) with dynamic **`node`** variable from `system_cpu_time` |
| **v3** | Added per-panel LogQL message filters; separate log panels no longer show identical data |
| **v4** | Removed hardcoded **`log_tag_prefix`**; added Loki **`log_namespace`** / **`log_service`** variables |
| **v5** | Fixed Loki `.*` matcher error (`allValue: .+`); removed NR-only log panels; relayout; single query per log panel |
| **v6** | Removed invalid `\| limit 500` from LogQL (caused `unexpected NUMBER` parse error); heap panels use `system_memory_usage` |
| **v7** | Fixed Imperformant DB **`invalid char escape`** (`RSlow\.SQL\.Warn` → `RSlow[.]SQL[.]Warn`); removed **Other Dashboards**; production grid layout; log empty-state **`noDataMessage`** + guidance banner |

---

## 7. Panels requiring external data (not in current Mimir/Loki)

| Panel | NR metric | What's needed |
|---|---|---|
| DB CPU utilization | `aws.rds.CPUUtilization` | CloudWatch / RDS metrics in Mimir |
| Login Page Load time | `SyntheticCheck` / PLPM | Synthetic monitoring datasource |
| User Sessions / Per Tenant * Session | `DatabaseSessionSample` | NR-equivalent session metrics or accept `db_client_connections_usage` proxy |
| Node Heap (label) | `apm.service.memory.heap` | True JVM heap requires `process_runtime_jvm_memory_usage` or similar in Mimir |

---

## 8. Deploy & validate

```bash
kubectl apply -f L3-production-Dashboard.yaml
```

Validate embedded JSON:

```bash
python3 - <<'PY'
import json, pathlib
text = pathlib.Path("L3-production-Dashboard.yaml").read_text()
block = text.split("json: |\n", 1)[1]
json.loads(block)
print("OK: Grafana dashboard JSON parses")
PY
```

Regenerate from NR source (if `L3-production-Dashboard.nr-source.json` is updated):

```bash
python3 scripts/build_l3_production_dashboard.py
python3 scripts/patch_l3_dashboard_v7.py
```

Explore checks:

```logql
# Loki — confirm namespaces exist
{k8s_namespace_name=~".+"} 
```

```promql
# Mimir — confirm host metrics
count(system_cpu_time{host_name=~".+"})
count(system_memory_usage{host_name=~".+"})
```

---

## 9. Challenges encountered during NR → Grafana migration

The following issues were hit while importing the New Relic JSON export and adapting it to the LGTM stack (Mimir + Loki) on a **different cluster** from the original NR environment.

### Source export quality

- The NR JSON export was **structurally invalid** — premature `]` / `}` closed widget arrays on every page (POD1 after Node 04 CPU, POD2 after Node 03 CPU, EWB after markdown, SBNA after Node 03 Heap). Widgets after those points were orphaned fragments.
- Several widgets had **missing titles** (markdown links, session billboards).
- NR Log NRQL used inconsistent quoting (`'message'` vs `` `message` ``) and typos (`Imperforment`, `Rjava.sql`).
- Widgets were recovered using regex heuristics on `title`, `layout`, `visualization.id`, and `query` fields — not a clean 1:1 JSON parse.

### Host and label mismatch

- NR filtered by hardcoded hostname patterns (`nuobprodapp*`, `muobprodapp08`, etc.) and **`tags.Name`** (`nuob-app-*`, `muob-app-*`).
- The Grafana cluster uses **OpenTelemetry / Kubernetes labels** (`k8s_namespace_name`, `service_name`, `host_name` from Mimir) — not NR's `tags_Name` or `logtype` labels.
- Initial hardcoded **`log_tag_prefix`** (nuob/muob) failed because the target cluster uses different naming (e.g. `uq01` namespace).

### Missing metrics in Mimir

- **`process_runtime_jvm_memory_usage`** and **`process_memory_usage`** — not present; heap panels showed no data until switched to **`system_memory_usage`** (node memory, not JVM heap).
- **`DatabaseSessionSample`**, **`aws.rds.CPUUtilization`**, **`SyntheticCheck`** — NR-only; proxied or replaced with text placeholders.

### LogQL / Loki-specific errors

| Error | Cause | Fix |
|---|---|---|
| `queries require at least one regexp... app=~".*"` | Loki rejects empty-compatible `.*` matchers when **All** is selected | Changed log variable `allValue` from `.*` to `.+` |
| `unexpected NUMBER` at end of query | Invalid `\| limit 500` pipe in log queries | Removed from LogQL; use Grafana `maxLines: 500` instead |
| `invalid char escape` (Imperforment DB) | `RSlow\.SQL\.Warn` — backslash escapes invalid in Loki strings | Changed to `RSlow[.]SQL[.]Warn` |
| Log panels spinning / no output | Multiple hidden fallback queries per panel (3× Loki load) + invalid queries | Single primary query per panel |
| All log panels showing same data (v3) | Every panel used only `{host_name=~"$log_host"}` | Per-panel NR message filters + namespace/service selectors |

### Layout and production readiness

- NR 12-column layout converted to Grafana **24-column** grid.
- Removed non-production panels: **Other Dashboards** (NR cross-links) and NR application-specific log panels not relevant to the deployed cluster.
- Added **log guidance banner** and **`noDataMessage`** because Grafana logs panels cannot fully customize the built-in empty-state text in all versions.

### Semantic gaps (expectations vs reality)

- **Heap Memory Usage** panels display **node system memory**, not JVM heap — label kept for NR parity but description documents the proxy.
- **Session / User Session** panels show **DB connection pool usage**, not NR tenant session counts.
- Empty log panels during a quiet interval are **expected** — users must select the correct **Log Namespace** and **Log Service**; this differs from NR where `tags.Name` and hostname were implicit in the export environment.

---

*Last updated for dashboard **version 7** — `L3-production-Dashboard.yaml`*
