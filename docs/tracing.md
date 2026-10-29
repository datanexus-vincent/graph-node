# OpenTelemetry tracing

graph-node can export traces of block processing, database writes and
GraphQL requests to any OpenTelemetry collector or vendor agent that accepts
OTLP. Tracing is off by default. When it is off, no exporter is installed
and the instrumented code costs one atomic load at the start and one at the
end of each section.

Traces complement the [Prometheus metrics](./metrics.md), which say how
long things take on average, by showing where the time went in one specific
block or request.

## Enabling tracing

Set `GRAPH_OTEL_TRACING=true` and point the exporter at a collector:

```sh
export GRAPH_OTEL_TRACING=true
export OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4318
export OTEL_SERVICE_NAME=graph-node
```

graph-node logs one line at startup when tracing is on, `OpenTelemetry
tracing enabled`, naming the endpoint, the service name and the enabled span
groups.

The exporter speaks OTLP over `http/protobuf`, which the OpenTelemetry
Collector and most vendor agents accept, some after enabling an OTLP HTTP
receiver. gRPC is not supported. Everything about
the exporter is configured with the standard `OTEL_*` variables, listed in
[environment-variables.md](./environment-variables.md#opentelemetry-tracing).
graph-node adds only two variables of its own: `GRAPH_OTEL_TRACING` and
`GRAPH_OTEL_SPANS`.

### A local collector

The quickest way to see traces is the OpenTelemetry Collector with its
debug exporter, which prints every span it receives:

```yaml
# otel-collector.yaml
receivers:
  otlp:
    protocols:
      http:
        endpoint: 0.0.0.0:4318
exporters:
  debug:
    verbosity: detailed
service:
  pipelines:
    traces:
      receivers: [otlp]
      exporters: [debug]
```

```sh
docker run --rm -p 4318:4318 \
  -v "$PWD/otel-collector.yaml:/etc/otelcol/config.yaml" \
  otel/opentelemetry-collector:latest --config /etc/otelcol/config.yaml
```

To keep the traces, add an `otlp` or `otlphttp` exporter that forwards them
to a backend such as Jaeger or Tempo, both of which accept OTLP. Vendor
exporters usually need the `otel/opentelemetry-collector-contrib` image and
their own configuration.

### Datadog

The Datadog Agent accepts OTLP directly. Enable its HTTP receiver with
`DD_OTLP_CONFIG_RECEIVER_PROTOCOLS_HTTP_ENDPOINT=0.0.0.0:4318` and point
`OTEL_EXPORTER_OTLP_ENDPOINT` at the agent. The `service.name` resource
attribute becomes the Datadog service, so set `OTEL_SERVICE_NAME` to
whatever the service should be called there. graph-node does not read
`DD_SERVICE`, `DD_ENV` or `DD_VERSION`; the equivalent resource attributes
go in `OTEL_RESOURCE_ATTRIBUTES`, for example
`deployment.environment.name=prod` for the `env` tag. The agent's own
configuration can still add tags on its side.

## Sampling

Sampling is decided once per trace, at its root span, by the sampler
configured with `OTEL_TRACES_SAMPLER` and `OTEL_TRACES_SAMPLER_ARG`. The
default, `parentbased_always_on`, exports everything. On a node that
indexes many subgraphs or serves many queries that is a lot of spans, so
either sample a fraction:

```sh
export OTEL_TRACES_SAMPLER=parentbased_traceidratio
export OTEL_TRACES_SAMPLER_ARG=0.01
```

or sample in the collector with the
[tail sampling processor](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/processor/tailsamplingprocessor),
which can keep only slow or failed traces. Tail sampling needs all the
spans first, so budget for the export volume even if little of it is kept.

An unsampled trace records and exports nothing: once the sampler has said
no at the root, no spans are created under it. What remains is the cost of
building that root and asking the sampler, once per trace.

## Shutdown

Spans are buffered and exported in batches by a background thread. When
graph-node receives SIGINT or SIGTERM, it flushes the buffer before
exiting. The flush is best effort: with the collector unreachable it gives
up after the exporter timeout, once per batch left in the buffer. On
SIGKILL, or if the process panics, the spans of the last few seconds are
lost.

## Overhead

With tracing off, each instrumented section does one atomic load at its
start and one at its end more than before, and nothing else. Once a
deployment is synced, graph-node stops timing its sections, as it always
has, since the `deployment_sync_secs` counter measures the time to sync.

With tracing on, every sampled span allocates its attributes and is handed
to the batch processor. The `block`, `store` and `graphql` groups add a few
spans per block or request and are cheap. `handler` adds one or two spans
per trigger. `host_fn` is the expensive one: a handler that reads and writes
thousands of entities makes thousands of spans, which is why it is off by
default. Turn it on for a specific investigation, together with a low
sampling ratio.
