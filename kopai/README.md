# OpenTelemetry Demo with Kopai

This directory contains the [Kopai](https://github.com/kopai-app/kopai) vendor
fork configuration for the
[OpenTelemetry Astronomy Shop Demo](https://github.com/open-telemetry/opentelemetry-demo).

Kopai is a local-first observability backend. This setup routes all OpenTelemetry
signals (traces, metrics, logs) from the demo's collector to a Kopai instance
running on the host machine.

## Prerequisites

- [Docker](https://docs.docker.com/get-docker/) and
  [Docker Compose](https://docs.docker.com/compose/install/)
- [Node.js](https://nodejs.org/) (v18+)

## Quick start

1. Start Kopai on the host (listens on port 4318 for OTLP/HTTP):

   ```shell
   npx @kopai/app start
   ```

2. In another terminal, start the demo with Kopai:

   ```shell
   make start-kopai
   ```

3. Browse the demo at <http://localhost:8080> and generate some traffic.

4. Query your telemetry:

   ```shell
   npx @kopai/cli traces search --limit 5
   npx @kopai/cli logs search --limit 5
   npx @kopai/cli metrics discover
   ```

5. Stop the demo:

   ```shell
   make stop-kopai
   ```

## How it works

```
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

The setup uses Docker Compose's `include:` mechanism with an override file to
swap the collector's extras config. No upstream files are modified.

## Files

| File | Description |
|------|-------------|
| `otelcol-config-kopai.yml` | Collector pipeline config that exports all signals to Kopai via OTLP/HTTP |
| `docker-compose-kopai_include-override.yml` | Override for the otel-collector service (volumes, extra_hosts) |
| `../docker-compose.kopai.yml` | Root-level entry point that includes the base compose + override |
