# OpenTelemetry Demo with Kopai

This directory contains the [Kopai](https://github.com/kopai-app/kopai-mono) vendor
fork configuration for the
[OpenTelemetry Astronomy Shop Demo](https://github.com/open-telemetry/opentelemetry-demo).

Kopai is a local-first observability backend. This setup routes OpenTelemetry
signals (traces, metrics, logs) from the demo's collector to a Kopai instance
running on the host machine.

Kopai takes the place of the demo's bundled observability stack, so Jaeger,
Prometheus, OpenSearch, Grafana, and the OpAMP server are not started.

## Prerequisites

- [Docker](https://docs.docker.com/get-docker/) and
  [Docker Compose](https://docs.docker.com/compose/install/)
- [Node.js](https://nodejs.org/) (v22.13.0+). Kopai stores telemetry through
  the built-in `node:sqlite` module, which does not exist before v22.5.0 and
  requires `--experimental-sqlite` until v22.13.0 (v23.4.0 on the
  odd-numbered line). The published `@kopai/app` package understates this as
  `engines: ">=20"`, and npm does not enforce that field by default, so an
  older Node install succeeds and then fails at startup.

## Quick start

1. Start Kopai on the host (listens on port 4318 for OTLP/HTTP):

   ```shell
   npx @kopai/app start
   ```

   > **Linux:** Kopai binds to `localhost` by default. For Docker containers to
   > reach it, bind to all interfaces:
   >
   > ```shell
   > HOST=0.0.0.0 npx @kopai/app start
   > ```

2. In another terminal, start the demo with Kopai:

   ```shell
   make start-kopai
   ```

   Other run modes:

   | Target | Services | Approx. memory |
   | --- | --- | --- |
   | `make start-kopai` | Core demo + Kafka, accounting, fraud-detection | ~4.1 GiB |
   | `make start-kopai-minimal` | Core demo only, no Kafka group | ~3.1 GiB |
   | `make start-kopai-agentic` | `start-kopai` + agent, chatbot, and MCP services | ~5.6 GiB |

   All three stop with `make stop-kopai`.

3. Browse the demo at <http://localhost:8080> and generate some traffic.

4. Query your telemetry with the Kopai CLI:

   ```shell
   npx @kopai/cli traces search --limit 5
   npx @kopai/cli logs search --service cart --fields Timestamp,Body --sort ASC
   npx @kopai/cli metrics discover
   ```

5. Inspect telemetry in the Kopai dashboard at <http://localhost:8000>.

6. Use your coding agent to get insights into the demo services:

   ```text
   > Use `@kopai/cli` to find errors in my services
   ```

7. Stop the demo:

   ```shell
   make stop-kopai
   ```

## How it works

```text
+----------------------------------------------------+
|  Docker Compose Network                            |
|                                                    |
|  +-------------------------------+                 |
|  |  Astronomy Shop Demo          |                 |
|  |  (10+ microservices)          |                 |
|  |                               |                 |
|  |  All services instrumented    |                 |
|  |  with OpenTelemetry           |                 |
|  +-------------------------------+                 |
|                  |                                 |
|                  | OTLP (traces, metrics, logs)    |
|                  v                                 |
|  +-------------------------------+                 |
|  |  OTel Collector               |                 |
|  |  (in Docker)                  |                 |
|  +-------------------------------+                 |
|                  |                                 |
|                  | OTLP/HTTP                       |
|                  v                                 |
|       host.docker.internal:4318                    |
+----------------------------------------------------+
                  |
                  v
+------------------------------------+
|  @kopai/app (on host machine)      |
|                                    |
|  OTel collector: localhost:4318    |
|  Dashboard:      localhost:8000    |
+------------------------------------+
```

Upstream splits its Compose setup into layers (`compose.yaml`,
`compose.full.yaml`, `compose.observability.yaml`) and loads a customization
file, `otelcol-config-extras.yml`, last in the collector's config chain. Kopai
plugs into those two seams:

- `compose.kopai.yaml` mounts `kopai/otelcol-config-kopai.yml` over the
  collector's extras config and adds a `host.docker.internal` mapping so the
  container can reach the host.
- `make start-kopai` layers `compose.yaml` + `compose.full.yaml` +
  `compose.kopai.yaml`, deliberately leaving out `compose.observability.yaml`.

The collector merges config files but **replaces** arrays rather than appending
to them, so each pipeline in `otelcol-config-kopai.yml` repeats the exporters
defined by the core config alongside `otlp_http/kopai`.

`Makefile` is the only upstream file this fork changes, and only to add the
`start-kopai*` and `stop-kopai` targets. Everything else is new files, which is
what keeps merges from upstream cheap.

### GenAI services

`make start-kopai-agentic` adds upstream's `agent`, `chatbot`, and `mcp`
services on top of `start-kopai`, so Kopai also receives GenAI telemetry: LLM
spans, token usage, and MCP tool calls. Chat with the demo at
<http://localhost:8080/chatbot/>.

These are a separate target rather than part of `start-kopai` for two reasons:
they add roughly 1.5 GiB of memory limits, and they depend on an LLM.

**LLM calls are replayed, not live, by default.** `.env` ships with
`USE_VCR=True`, `LLM_BASE_URL=https://local-llm.com`, and an empty `API_KEY`;
requests are served from recorded cassettes in
`src/agent/fixtures/vcr_cassettes/`. Cassettes exist only for the `LLM_MODEL`
values they were recorded against (`azure/gpt-5.5` and `claude-opus-4-7`), and
matching is fuzzy via `VCR_MATCH_THRESHOLD`. Point `LLM_MODEL` at anything else
while `USE_VCR=True` and the agent has nothing to replay.

For real LLM traffic, set these in `.env.override`, the file upstream reserves
for local overrides:

```shell
USE_VCR=False
LLM_BASE_URL=https://api.openai.com/v1
LLM_MODEL=gpt-4o-mini
API_KEY=<your key>
```

The agent passes `API_KEY` through as `OPENAI_API_KEY`, so any OpenAI-compatible
endpoint works. Both `.env` and `.env.override` are tracked in git (the latter
carries a "do not push" banner), so take care not to commit a real key.

### Running Kopai alongside the bundled stack

To send data to Kopai *and* keep Jaeger, Prometheus, OpenSearch, and Grafana:

```shell
docker compose --env-file .env --env-file .env.override \
  -f compose.yaml -f compose.full.yaml -f compose.observability.yaml -f compose.kopai.yaml up
```

The observability layer sets its own pipeline exporters, and the Kopai layer
loads after it, so those exporters must be repeated or they are dropped. Use
these pipelines in `otelcol-config-kopai.yml` for that combination:

```yaml
service:
  pipelines:
    traces:
      exporters: [debug, span_metrics, otlp_grpc/jaeger, otlp_http/kopai]
    metrics:
      exporters: [debug, otlp_http/prometheus, otlp_http/kopai]
    logs:
      exporters: [debug, opensearch, otlp_http/kopai]
```

## Troubleshooting

**Nothing arrives in Kopai.** Check what the collector is doing:

```shell
docker logs otel-collector 2>&1 | grep kopai
```

An `Exporting failed` line naming `otlp_http/kopai` means the collector is
configured correctly but cannot deliver. If you see no `otlp_http/kopai` lines
at all, the extras config was not picked up; confirm the mount with
`docker compose ... config` and look for `otelcol-config-kopai.yml`.

**Something else is already on port 4318.** Both Kopai and the demo's own
collector speak OTLP, and other local tooling (Kubernetes clusters, vendor
agents) often claims 4318 too. A `404` or `401` in the collector's export
errors means *something* answered, but not Kopai. Find the owner:

```shell
lsof -nP -iTCP:4318 -sTCP:LISTEN
```

Free the port, or start Kopai elsewhere and update the `endpoint` in
`otelcol-config-kopai.yml` to match.

**Connection refused on Linux.** Kopai binds to `localhost` by default, which
containers cannot reach. Start it with `HOST=0.0.0.0 npx @kopai/app start`.

## Files

| File | Description |
| --- | --- |
| `otelcol-config-kopai.yml` | Collector extras layer that exports traces, metrics, and logs to Kopai via OTLP/HTTP |
| `../compose.kopai.yaml` | Compose layer that mounts the config above and routes the collector to the host |
| `../Makefile` | Adds the `start-kopai`, `start-kopai-minimal`, `start-kopai-agentic`, and `stop-kopai` targets |
