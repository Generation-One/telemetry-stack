# Telemetry Stack

A complete observability stack with OpenTelemetry Collector, Jaeger, Prometheus, Grafana, and Seq — secured behind Pomerium reverse proxy with OAuth authentication.

## Architecture

```
┌─────────────────┐      ┌──────────────────────┐
│  Applications   │─────▶│  OpenTelemetry       │
│  (OTLP)         │      │  Collector           │
└─────────────────┘      │  :4317 (gRPC)        │
                         │  :4318 (HTTP)        │
                         └──────────┬───────────┘
                                    │
              ┌─────────────────────┼─────────────────────┐
              ▼                     ▼                     ▼
      ┌───────────────┐     ┌───────────────┐     ┌───────────────┐
      │    Jaeger     │     │  Prometheus   │     │     Seq       │
      │   (Traces)    │     │   (Metrics)   │     │    (Logs)     │
      └───────┬───────┘     └───────┬───────┘     └───────┬───────┘
              │                     │                     │
              │                     ▼                     │
              │             ┌───────────────┐             │
              │             │    Grafana    │             │
              │             │ (Dashboards)  │             │
              │             └───────┬───────┘             │
              │                     │                     │
              └─────────────────────┼─────────────────────┘
                                    ▼
                         ┌─────────────────────┐
                         │     Pomerium        │
                         │   (OAuth Proxy)     │
                         │    :80 / :443       │
                         └─────────────────────┘
                                    │
          ┌─────────────────────────┼─────────────────────────┐
          ▼                         ▼                         ▼
   jaeger.localhost         grafana.localhost          seq.localhost
```

## Components

| Service | Purpose | Access URL |
|---------|---------|------------|
| **Pomerium** | OAuth reverse proxy | Handles authentication |
| **OpenTelemetry Collector** | Telemetry ingestion | `localhost:4317` (gRPC), `localhost:4318` (HTTP) |
| **Jaeger** | Distributed tracing | `https://jaeger.localhost` |
| **Prometheus** | Metrics storage | `localhost:9090` (internal) |
| **Grafana** | Dashboards & visualization | `https://grafana.localhost` |
| **Seq** | Structured log management | `https://seq.localhost` |

## Prerequisites

- Docker and Docker Compose
- OAuth provider credentials (Google, Azure AD, etc.)

## Setup

### 1. Create `.env` file

```bash
# Identity Provider (google, azure, github, okta, etc.)
POMERIUM_IDP_PROVIDER=google
POMERIUM_IDP_CLIENT_ID=your-client-id
POMERIUM_IDP_CLIENT_SECRET=your-client-secret

# Generate with: head -c32 /dev/urandom | base64
POMERIUM_COOKIE_SECRET=your-base64-cookie-secret
POMERIUM_SIGNING_KEY=your-base64-signing-key

# Optional: Custom domains (defaults shown)
GRAFANA_DOMAIN=grafana.localhost
JAEGER_DOMAIN=jaeger.localhost
SEQ_DOMAIN=seq.localhost
```

### 2. Create data directories

```bash
mkdir -p data/prometheus data/jaeger data/seq
```

Or configure custom paths in `.env`:

```bash
PROMETHEUS_DATA_PATH=./data/prometheus
```

### 3. Start the stack

```bash
docker compose up -d
```

## Sending Telemetry Data

Configure your applications to send OTLP data to the collector:

| Protocol | Endpoint |
|----------|----------|
| gRPC | `localhost:4317` |
| HTTP | `localhost:4318` |

### Required Headers

The collector expects these headers for proper data routing:

```
x-user-id: <user identifier>
x-node-id: <node identifier>
```

Records missing these headers will be filtered out.

### Example: .NET Application

```csharp
builder.Services.AddOpenTelemetry()
    .WithTracing(tracing => tracing
        .AddOtlpExporter(opt => opt.Endpoint = new Uri("http://localhost:4317")))
    .WithMetrics(metrics => metrics
        .AddOtlpExporter(opt => opt.Endpoint = new Uri("http://localhost:4317")));
```

## Data Flow

- **Traces** → OpenTelemetry Collector → Jaeger
- **Metrics** → OpenTelemetry Collector → Prometheus → Grafana
- **Logs** → OpenTelemetry Collector → Seq


## Test Data

Sample OTLP payloads are available in the `test-data/` directory:
- `otel-logs.json`
- `otel-metrics.json`
- `otel-traces.json`

## License

MIT
