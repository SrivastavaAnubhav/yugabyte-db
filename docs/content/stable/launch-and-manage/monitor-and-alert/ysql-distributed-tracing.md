---
title: YSQL Distributed Tracing
linkTitle: YSQL Distributed Tracing
headerTitle: YSQL Distributed Tracing
description: Export OpenTelemetry traces for YSQL query execution.
headcontent: Trace YSQL queries with OpenTelemetry
menu:
  stable:
    parent: monitor-and-alert
    identifier: ysql-distributed-tracing
    weight: 130
type: docs
tags:
  feature: tech-preview
rightNav:
  hideH4: true
---

YSQL queries spend time in many places: parsing, planning, execution, transaction commit, and processing in YB-TServer and YB-Master. YugabyteDB can export timing data for these stages as OpenTelemetry (OTel) traces so you can inspect a waterfall view of query execution in tools such as [Jaeger](/stable/integrations/jaeger/), [Grafana Tempo](https://grafana.com/oss/tempo/), or [Honeycomb](https://www.honeycomb.io).

Distributed Tracing is {{<tags/feature/tp>}}. A traced YSQL query can include spans from the YSQL backend, YB-TServers, and YB-Masters.

## How it works

YSQL Distributed Tracing follows the [W3C Trace Context](https://www.w3.org/TR/trace-context/) standard. Your application supplies a `traceparent` value. YugabyteDB creates spans for the query lifecycle and exports them to an OTel collector over OTLP/HTTP.

YugabyteDB propagates the trace context from the YSQL backend to YB-TServer over shared memory or RPC, and between daemon processes over YB RPC. Each participating process adds spans to the same trace.

Each traced query produces a _trace_ made up of _spans_. Spans are nested to show where time is spent. For example, planning, execution, commit, shared-memory and RPC calls, tablet read and write processing, and master RPCs when the query contacts a master.

When tracing is disabled (the default), there is no measurable performance impact. When tracing is enabled for a query, other queries and other YSQL backends are not affected.

## Prerequisites

- YSQL must be enabled on the cluster.
- The OTLP/HTTP endpoint must be reachable from every YB-Master and YB-TServer node. YSQL backends use the endpoint configured for their YB-TServer.
- Because Distributed Tracing is a preview feature, add `otel_collector_traces_endpoint` to `allowed_preview_flags_csv` on both [YB-Master](../../../reference/configuration/yb-master/#allowed-preview-flags-csv) and [YB-TServer](../../../reference/configuration/yb-tserver/#allowed-preview-flags-csv) before setting the endpoint.

## Configure Distributed Tracing

Set `otel_collector_traces_endpoint` on every YB-Master and YB-TServer. YSQL backends use the tracing settings supplied through YB-TServer. Changing these flags requires restarting the affected processes.

| Flag | Used by | Description | Default |
| :--- | :------ | :---------- | :------ |
| otel_collector_traces_endpoint | All traced processes | OTLP/HTTP URL where spans are exported. For example, `http://<collector-host>:4318/v1/traces`. | Empty |
| otel_batch_max_queue_size | YB-Master and YB-TServer daemons | Maximum spans buffered in each daemon process. Spans beyond this limit are dropped. Must be greater than 0 and at least as large as `otel_batch_max_export_batch_size`. | `16384` |
| otel_ysql_batch_max_queue_size | YSQL backends | Maximum spans buffered by each YSQL backend process. Each backend has its own queue. Must be greater than 0 and at least as large as `otel_batch_max_export_batch_size`. | `2048` |
| otel_batch_schedule_delay_ms | All traced processes | Milliseconds between batch exports. Lower values reduce export latency but increase export frequency. | `500` |
| otel_batch_max_export_batch_size | All traced processes | Maximum spans per export batch. Must be greater than 0 and no larger than the queue used by the process. | `512` |

## Set up tracing

The following example uses [Jaeger](https://www.jaegertracing.io/) as the trace backend. Jaeger accepts OTLP over HTTP on port 4318. Other OTLP-compatible backends follow the same pattern; change the endpoint as appropriate.

### Start Jaeger

Run Jaeger all-in-one with OTLP enabled:

```sh
docker run --rm --name jaeger \
  -e COLLECTOR_OTLP_ENABLED=true \
  -p 16686:16686 \
  -p 4318:4318 \
  jaegertracing/all-in-one:1.61
```

Open the Jaeger UI at [http://localhost:16686](http://localhost:16686).

For a local single-node cluster started with yugabyted:

```sh
./bin/yugabyted start \
  --master_flags "allowed_preview_flags_csv=otel_collector_traces_endpoint,otel_collector_traces_endpoint=http://127.0.0.1:4318/v1/traces" \
  --tserver_flags "allowed_preview_flags_csv=otel_collector_traces_endpoint,otel_collector_traces_endpoint=http://127.0.0.1:4318/v1/traces"
```

If you use YugabyteDB Anywhere, set the endpoint and preview allowlist for both YB-Master and YB-TServer using [Edit configuration flags](../../../yugabyte-platform/manage-deployments/edit-config-flags/#modify-configuration-flags).

{{< note title="Note" >}}

If `otel_collector_traces_endpoint` is not set, attempting to use the [yb_dist_tracecontext](#configuration-parameter-per-session-or-transaction) configuration parameter returns an error indicating that Distributed Tracing is not enabled.

{{</note >}}

### Trace a query

After the cluster is running with tracing configured, enable tracing for individual queries using one of the following methods.

#### SQL comment (per query)

Prepend or append a block comment that contains a W3C `traceparent` value. The comment must be the _first_ block comment at the start of the query, or the _last_ block comment at the end of the query. If the comment is not the first or last, the `traceparent` is not parsed.

```sql
/*traceparent='00-0af7651916cd43dd8448eb211c80319c-b7ad6b7169203331-01'*/
SELECT * FROM orders WHERE customer_id = 500;
```

You can also place the comment at the end of the statement:

```sql
SELECT * FROM orders WHERE customer_id = 500
/*traceparent='00-0af7651916cd43dd8448eb211c80319c-b7ad6b7169203331-01'*/;
```

#### Configuration parameter (per session or transaction)

Set the `yb_dist_tracecontext` YSQL [configuration parameter](../../../reference/configuration/yb-tserver/#postgresql-configuration-parameters).

For example, to trace every query in the current transaction:

```sql
BEGIN;
SET LOCAL yb_dist_tracecontext = 'traceparent=''00-00000000000000000000000000000001-0000000000000005-01''';
SELECT * FROM users WHERE id = 10;
UPDATE accounts SET balance = balance - 100 WHERE id = 10;
COMMIT;
```

To stop tracing for the session:

```sql
RESET yb_dist_tracecontext;
```

{{< note title="Parameter takes precedence" >}}

If you provide a traceparent using both the parameter _and_ SQL comment, the parameter takes priority and a warning is emitted.

{{< /note >}}

### View traces

Run a traced query, wait for the batch export (scheduled every 500 ms by default), and then open the Jaeger UI.

1. Select **Service** `ysql`.
2. Click **Find Traces**.
3. Open a trace to see YSQL query spans and the service boundaries into `TabletServer` and, when contacted, `Master`.

Selecting `ysql` finds traces by their query root. The trace view includes spans exported by all participating services.

## Trace data

### Process attributes

Each participating process exports process metadata with its spans:

| Attribute | Description |
| :-------- | :---------- |
| service.name | `ysql` for a YSQL backend, `TabletServer` for YB-TServer, and `Master` for YB-Master. |
| service.instance.id | UUID of the YB-TServer or YB-Master node. YSQL backends use their YB-TServer node UUID. |
| process.pid | Operating-system PID of the process exporting the span. |

### Span attributes

Every span includes standard OpenTelemetry fields such as operation name, span ID, parent span ID, trace ID, start time, duration, span kind, and status.

Additional attributes depend on the span type:

| Span | Additional attributes |
| :--- | :-------------------- |
| Root (`query`) | `query.text` (truncated to 256 characters), `user.id` (PostgreSQL role OID) |
| RPC or shared-memory boundary | `rpc.system` identifies `yb_rpc` or `yb_shmem`. YSQL client spans can also include `rpc.table_names` for tables accessed by the call. |

Depending on the query and execution path, typical span names include:

- Query lifecycle: `query`, `parse`, `rewrite`, `plan`, `execute`, `commit`, `abort`
- Extended query protocol: `ext.parse`, `ext.bind`, `ext.execute`, `ext.sync`, `ext.describe`, `ext.flush`
- Process boundaries: `shmem yb.tserver.PgClientService.Perform`, `shmem yb.tserver.PgClientService.AcquireObjectLock`, and RPC names such as `rpc yb.tserver.PgClientService.Perform`; master RPC span names start with `rpc yb.master.`
- Tablet read path: `tserver.read`, `tserver.read.resume`, `tserver.read.execute`, `tablet.read`, `docdb.pgsql_read`, `docdb.pgsql_read.scalar`
- Tablet write path: `tablet.write`, `tablet.write.prepare_locks`, `tablet.write.assemble`, `tablet.write.replicate`, `tablet.write.apply`, `tablet.write.storage_apply`

A process-boundary operation has a client span from the caller and a matching server span from the recipient, joined by the propagated trace context.

## Join application traces

Because YugabyteDB accepts W3C `traceparent` values, you can continue a trace started in application code. Pass the same `traceparent` in a SQL comment or `yb_dist_tracecontext` parameter so database spans appear as children of your application's root span in your observability backend.

## Related topics

- [Monitor with Active Session History](../active-session-history-monitor/) - sample-based view of database wait events
- [Query tuning](../query-tuning/) - optimize query performance with EXPLAIN, pg_stat_statements, and related tools
- [Jaeger integration](../../../integrations/jaeger/) - use YCQL as Jaeger trace storage (separate from YSQL query tracing)
