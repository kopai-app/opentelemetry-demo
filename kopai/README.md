# OpenTelemetry Demo with Kopai

This directory contains the [Kopai](https://github.com/kopai-app/kopai-mono) vendor
fork configuration for the
[OpenTelemetry Astronomy Shop Demo](https://github.com/open-telemetry/opentelemetry-demo).

Kopai is a local-first observability backend. This setup routes OpenTelemetry
signals (traces, metrics, logs) from the demo's collector to a Kopai instance
running on the host machine.

Kopai takes the place of the demo's bundled observability stack, so Jaeger,
Prometheus, OpenSearch, and Grafana are not started.

## Prerequisites

- [Docker](https://docs.docker.com/get-docker/) and
  [Docker Compose](https://docs.docker.com/compose/install/)
- [Node.js](https://nodejs.org/) (v22.5.0+)

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
   |--------|----------|----------------|
   | `make start-kopai` | Core demo + Kafka group (accounting, fraud-detection) | ~4.1 GiB |
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
   ❯ Use `@kopai/cli` to find errors in my services
   ```

7. Stop the demo:

   ```shell
   make stop-kopai
   ```

## How it works

```text
┌───────────────────────────────────────────────────┐
│  Docker (OpenTelemetry Demo)                      │
│                                                   │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐         │
│  │ frontend │  │ cart     │  │ checkout │  ...    │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘         │
│       │             │             │               │
│       └─────────────┴─────────────┘               │
│                     │                             │
│             ┌───────▼────────┐                    │
│             │ otel-collector │                    │
│             └───────┬────────┘                    │
│                     │ OTLP/HTTP                   │
└─────────────────────┼─────────────────────────────┘
                      │ host.docker.internal:4318
              ┌───────▼────────┐
              │     Kopai      │
              │  (host machine)│
              └────────────────┘
```

Upstream splits its Compose setup into layers (`compose.yaml`,
`compose.full.yaml`, `compose.observability.yaml`) and loads a customization
file, `otelcol-config-extras.yml`, last in the collector's config chain. Kopai
plugs into those two seams and modifies no upstream files:

- `compose.kopai.yaml` mounts `kopai/otelcol-config-kopai.yml` over the
  collector's extras config and adds a `host.docker.internal` mapping so the
  container can reach the host.
- `make start-kopai` layers `compose.yaml` + `compose.full.yaml` +
  `compose.kopai.yaml`, deliberately leaving out `compose.observability.yaml`.

The collector merges config files but **replaces** arrays rather than appending
to them, so each pipeline in `otelcol-config-kopai.yml` repeats the exporters
defined by the core config alongside `otlp_http/kopai`.

### GenAI services

`make start-kopai-agentic` adds upstream's `agent`, `chatbot`, and `mcp`
services on top of `start-kopai`, so Kopai also receives GenAI telemetry — LLM
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
endpoint works. Both `.env` and `.env.override` are tracked in git — the latter
carries a "do not push" banner — so take care not to commit a real key.

### Running Kopai alongside the bundled stack

To send data to Kopai *and* keep Jaeger, Prometheus, OpenSearch, and Grafana:

```shell
docker compose --env-file .env --env-file .env.override \
  -f compose.yaml -f compose.full.yaml -f compose.observability.yaml -f compose.kopai.yaml up
```

The observability layer sets its own pipeline exporters, so also add
`otlp_grpc/jaeger`, `otlp_http/prometheus`, and `opensearch` to the matching
pipelines in `otelcol-config-kopai.yml` — otherwise the Kopai layer, which loads
last, drops them.

## Files

| File | Description |
|------|-------------|
| `otelcol-config-kopai.yml` | Collector extras layer that exports traces, metrics, and logs to Kopai via OTLP/HTTP |
| `../compose.kopai.yaml` | Compose layer that mounts the config above and routes the collector to the host |
