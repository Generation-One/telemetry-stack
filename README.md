# Telemetry Stack

A complete observability stack with OpenTelemetry Collector, Tempo, Prometheus, Grafana, and Loki — secured behind OAuth2-Proxy with Google OAuth.

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
                         ┌──────────┴──────────┐
                         │      nginx          │
                         │  (reverse proxy)    │
                         ├─────────────────────┤
                         │ /api/* → direct     │
                         │   (Bearer token)    │
                         │ /* → OAuth2-Proxy   │
                         │   (Google sign-in)  │
                         └─────────────────────┘
```

## Components

| Service | Purpose | Access URL |
|---------|---------|------------|
| **OAuth2-Proxy** | Google OAuth for UI access | `localhost:4180` |
| **OpenTelemetry Collector** | Telemetry ingestion | `localhost:4317` (gRPC), `localhost:4318` (HTTP) |
| **Tempo** | Distributed tracing | `localhost:3200` (internal) |
| **Prometheus** | Metrics storage | Internal (scraped by Grafana) |
| **Loki** | Log aggregation | `localhost:3100` (internal) |
| **Grafana** | Dashboards & visualization | `https://grafana.observability.generation.one` |

## Prerequisites

- Docker and Docker Compose
- Google OAuth credentials
- nginx with certbot SSL

## Authentication

### UI Access (humans)
Google OAuth via oauth2-proxy. All routes except `/api/` require sign-in.

### API Access (machines)
Grafana service account tokens (`glsa_...`) are passed directly to Grafana, bypassing OAuth. Create tokens via Grafana UI (Administration → Service Accounts).

The nginx config in `nginx/` separates these paths:
- `/api/*` → proxied direct to Grafana (Bearer token auth)
- `/*` → proxied via oauth2-proxy (Google OAuth)

## Setup

### 1. Create `.env` file

```bash
# Identity Provider
OAUTH2_PROXY_PROVIDER=google
OAUTH2_PROXY_CLIENT_ID=your-client-id
OAUTH2_PROXY_CLIENT_SECRET=your-client-secret

# Generate with: openssl rand -base64 32
OAUTH2_COOKIE_SECRET=your-base64-cookie-secret

# OAuth callback URL
OAUTH2_PROXY_REDIRECT_URL=https://grafana.observability.generation.one/oauth2/callback

# Allowed domains
OAUTH2_PROXY_ALLOWED_DOMAINS=.observability.generation.one

# Grafana domain
GRAFANA_DOMAIN=grafana.observability.generation.one

# Optional: Custom data paths (defaults shown)
PROMETHEUS_DATA_PATH=./data/prometheus
LOKI_DATA_PATH=./data/loki
TEMPO_DATA_PATH=./data/tempo
```

### 2. Create data directories

```bash
mkdir -p data/prometheus data/loki data/tempo data/grafana
```

### 3. Install nginx configs

```bash
sudo cp nginx/*.conf /etc/nginx/sites-available/
sudo ln -sf /etc/nginx/sites-available/*.conf /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx
```

### 4. Start the stack

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

Sample OTLP payloads are available in the `test-data/` directory.

## Nginx Configs

The `nginx/` directory contains the reverse proxy configs for all services:
- `grafana.observability.generation.one.conf` — Grafana (OAuth + API token passthrough)
- `auth.observability.generation.one.conf` — OAuth2-Proxy callback
- `ingest.observability.generation.one.conf` — OTLP ingestion endpoint
- `stands.observability.generation.one.conf` — StandPanel

## License

MIT
