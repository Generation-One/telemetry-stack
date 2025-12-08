# Telemetry Stack

A complete observability stack with OpenTelemetry Collector, Tempo, Prometheus, Grafana, and Loki — secured behind OAuth2-Proxy with OAuth authentication.

## Architecture

```
┌─────────────────┐      ┌──────────────────────┐
│  Applications   │─────►│  OpenTelemetry       │
│  (OTLP)         │      │  Collector           │
└─────────────────┘      │  :4317 (gRPC)        │
                         │  :4318 (HTTP)        │
                         └──────────┬───────────┘
                                    │
              ┌─────────────────────┼─────────────────────┐
              ▼                     ▼                     ▼
      ┌───────────────┐     ┌───────────────┐     ┌───────────────┐
      │    Tempo      │     │  Prometheus   │     │     Loki      │
      │   (Traces)    │     │   (Metrics)   │     │    (Logs)     │
      └───────┬───────┘     └───────┬───────┘     └───────┬───────┘
              │                     │                     │
              └─────────────────────┼─────────────────────┘
                                    ▼
                            ┌───────────────┐
                            │    Grafana    │
                            │ (Dashboards)  │
                            └───────┬───────┘
                                    │
                                    ▼
                         ┌─────────────────────┐
                         │    OAuth2-Proxy     │
                         │   (OAuth Proxy)     │
                         │       :4180         │
                         └─────────────────────┘
                                    │
                                    ▼
                           grafana.localhost
```

## Components

| Service | Purpose | Access URL |
|---------|---------|------------|
| **OAuth2-Proxy** | OAuth reverse proxy | `localhost:4180` |
| **OpenTelemetry Collector** | Telemetry ingestion | `localhost:4317` (gRPC), `localhost:4318` (HTTP) |
| **Tempo** | Distributed tracing | `localhost:3200` (internal) |
| **Prometheus** | Metrics storage | Internal (scraped by Grafana) |
| **Loki** | Log aggregation | `localhost:3100` (internal) |
| **Grafana** | Dashboards & visualization | `https://grafana.localhost` (via OAuth2-Proxy) |

## Prerequisites

- Docker and Docker Compose
- OAuth provider credentials (Google, Azure AD, etc.)

## Setup

### 1. Create `.env` file

```bash
# Identity Provider (google, azure, github, etc.)
OAUTH2_PROXY_PROVIDER=google
OAUTH2_PROXY_CLIENT_ID=your-client-id
OAUTH2_PROXY_CLIENT_SECRET=your-client-secret

# Generate with: head -c32 /dev/urandom | base64
OAUTH2_COOKIE_SECRET=your-base64-cookie-secret

# OAuth callback URL
OAUTH2_PROXY_REDIRECT_URL=https://grafana.localhost/oauth2/callback

# Allowed domains
OAUTH2_PROXY_ALLOWED_DOMAINS=.localhost

# Optional: Custom Grafana domain (default shown)
GRAFANA_DOMAIN=grafana.localhost

# Optional: Custom data paths (defaults shown)
PROMETHEUS_DATA_PATH=./data/prometheus
LOKI_DATA_PATH=./data/loki
TEMPO_DATA_PATH=./data/tempo
```

### 2. Create data directories

```bash
mkdir -p data/prometheus data/loki data/tempo data/grafana
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


### Example: .NET Application

```csharp
builder.Services.AddOpenTelemetry()
    .WithTracing(tracing => tracing
        .AddOtlpExporter(opt => opt.Endpoint = new Uri("http://localhost:4317")))
    .WithMetrics(metrics => metrics
        .AddOtlpExporter(opt => opt.Endpoint = new Uri("http://localhost:4317")));
```

## Data Flow

- **Traces** → OpenTelemetry Collector → Tempo → Grafana
- **Metrics** → OpenTelemetry Collector → Prometheus → Grafana
- **Logs** → OpenTelemetry Collector → Loki → Grafana

## Test Data

Sample OTLP payloads are available in the `test-data/` directory:
- `otel-logs.json`
- `otel-metrics.json`
- `otel-traces.json`

## License

MIT
