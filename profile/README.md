# OpsOrch

OpsOrch is an open, modular operations orchestrator that keeps your operational data and execution in the tools you already use, while providing a single control plane to investigate, coordinate, and act. If observability tells you what happened, OpsOrch helps you answer: what do we do next?

## Platform at a Glance

### Runtime + Experience Repos
| Repo | What it does | Highlights |
| --- | --- | --- |
| [`opsorch-core`](https://github.com/OpsOrch/opsorch-core) | Stateless Go orchestration service | REST APIs for every capability, registry + schema boundaries, secret backends, JSON-RPC plugin loader |
| [`opsorch-mcp`](https://github.com/OpsOrch/opsorch-mcp) | MCP server that wraps Core as typed tools/resources | HTTP + stdio transports, resources for docs, auth via bearer tokens, built with TypeScript |
| _`opsorch-copilot` (private)_ | AI runtime that plans + executes MCP tool calls | Capability-aware handler registry, multi-step planning, OpenAI/Mock backends, HTTP chat API |
| [`opsorch-console`](https://github.com/OpsOrch/opsorch-console) | Open-source Next.js operator UI | Incidents/logs/metrics/tickets/services views, alert surface, Copilot chat, OSS + Enterprise editions |
| _`opsorch-com` (private)_ | Marketing/docs site | Next.js 14 app powering opsorch.com with docs + launch content |

### Adapter + Integration Repos
| Repo | Capabilities | Notes |
| --- | --- | --- |
| [`opsorch-adapter`](https://github.com/OpsOrch/opsorch-adapter) | Template | Starter repo showing provider + plugin patterns |
| [`opsorch-mock-adapters`](https://github.com/OpsOrch/opsorch-mock-adapters) | All (demo) | Ships seeded data + demo binaries used in Docker examples |
| [`opsorch-github-adapter`](https://github.com/OpsOrch/opsorch-github-adapter) | Ticket, Deployment, Team | GitHub Issues + Actions workflow runs + Teams with comprehensive filtering |
| [`opsorch-datadog-adapter`](https://github.com/OpsOrch/opsorch-datadog-adapter) | Metric, Log, Alert, Incident, Service | Uses official Datadog Go SDKs |
| [`opsorch-pagerduty-adapter`](https://github.com/OpsOrch/opsorch-pagerduty-adapter) | Incident, Service | PagerDuty incidents/timelines/services |
| [`opsorch-jira-adapter`](https://github.com/OpsOrch/opsorch-jira-adapter) | Ticket | Jira CRUD + advanced field handling |
| [`opsorch-prometheus-adapter`](https://github.com/OpsOrch/opsorch-prometheus-adapter) | Metric | PromQL query + discovery |
| [`opsorch-elastic-adapter`](https://github.com/OpsOrch/opsorch-elastic-adapter) | Log | Elasticsearch query integration |
| [`opsorch-slack-adapter`](https://github.com/OpsOrch/opsorch-slack-adapter) | Messaging | Slack channel + DM messaging w/ Block Kit |

### Shared References
- `.github/profile/docker-compose*.yml`: reproducible local stacks (quick demo, dev overrides, prod baseline)
- [DOCKER_COMPOSE.md](DOCKER_COMPOSE.md): detailed documentation for Docker Compose configurations

## Architecture
```mermaid
graph TB
    subgraph "Experience"
        Console["OpsOrch Console\n(Next.js UI)"]
        Copilot["OpsOrch Copilot\n(LLM planning runtime)"]
    end
    
    subgraph "Tools Layer"
        MCP["opsorch-mcp\n(MCP server)\nTyped tools/resources"]
    end
    
    subgraph "Orchestration"
        Core["OpsOrch Core\n(Go service)\nRouting • Schemas • Secrets"]
    end
    
    subgraph "Adapters"
        InProc["In-process providers\n(Go registry)"]
        Plugins["JSON-RPC plugins\n(stdio)"]
    end
    
    subgraph "External Systems"
        PD[PagerDuty]
        Jira[Jira]
        GitHub[GitHub]
        DD[Datadog]
        Prom[Prometheus]
        ES[Elasticsearch]
        Slack[Slack]
        Secret[Secret Store]
        Mock[Mock Data]
    end
    
    Console -.Enterprise optional.-> Copilot
    Console -->|HTTP all editions| Core
    Copilot -->|MCP Protocol| MCP
    MCP -->|HTTP REST| Core
    Core -->|Registry lookup| InProc
    Core -->|JSON-RPC stdin/stdout| Plugins
    InProc --> Mock
    InProc --> Secret
    Plugins --> PD
    Plugins --> Jira
    Plugins --> GitHub
    Plugins --> DD
    Plugins --> Prom
    Plugins --> ES
    Plugins --> Slack
```

### Design Principles
- **Single API surface:** incidents, alerts, timelines, logs, metrics, tickets, messaging, services, teams, secrets
- **Config-driven routing:** `OPSORCH_<CAP>_PROVIDER`, `OPSORCH_<CAP>_PLUGIN`, and `OPSORCH_<CAP>_CONFIG` select providers per capability
- **No operational data storage:** Core keeps encrypted configs + optional audit logs; data stays in source tools
- **Security-first:** pluggable secret backends (Vault/KMS/local AES-GCM), env-scoped queries, audit hooks
- **AI-native:** MCP provides typed tools; Copilot plans tool calls, tracks evidence, and surfaces deep links back to Console. OSS Console connects only to Core; Enterprise wiring adds optional Copilot chat.

### Provider Configuration Model
`opsorch-core` reads provider settings with
`loadProviderConfig` (`opsorch-core/api/provider_config_handler.go:199`). Every
capability looks for the trio `OPSORCH_<CAP>_PROVIDER`,
`OPSORCH_<CAP>_PLUGIN`, and `OPSORCH_<CAP>_CONFIG`:
- `OPSORCH_<CAP>_PROVIDER` – registered provider name (e.g., `jira`, `pagerduty`)
- `OPSORCH_<CAP>_PLUGIN` – optional JSON-RPC plugin path that overrides the registry
- `OPSORCH_<CAP>_CONFIG` – JSON payload forwarded verbatim to the adapter constructor

The production Compose templates ship with the following defaults; reuse the
same JSON keys when building custom stacks:

| Capability | Env Var | Example Config |
| --- | --- | --- |
| Incident (PagerDuty) | `OPSORCH_INCIDENT_CONFIG` | `{"apiToken":"pd_token","serviceID":"PXXXXXX","fromEmail":"pd-user@example.com","apiURL":"https://api.pagerduty.com"}` |
| Ticket (Jira) | `OPSORCH_TICKET_CONFIG` | `{"apiToken":"jira_token","email":"user@example.com","apiURL":"https://your-domain.atlassian.net","projectKey":"OPS"}` |
| Metric (Prometheus) | `OPSORCH_METRIC_CONFIG` | `{"url":"http://prometheus:9090"}` |
| Log (Elasticsearch) | `OPSORCH_LOG_CONFIG` | `{"addresses":["http://elasticsearch:9200"],"username":"elastic","password":"changeme","indexPattern":"logs-*"}` |
| Messaging (Slack) | `OPSORCH_MESSAGING_CONFIG` | `{"token":"xoxb-your-slack-bot-token"}` |
| Service (PagerDuty) | `OPSORCH_SERVICE_CONFIG` | `{"apiToken":"pd_token","apiURL":"https://api.pagerduty.com"}` |
| Team (GitHub) | `OPSORCH_TEAM_CONFIG` | `{"token":"ghp_token","organization":"your-org"}` |

Adapters such as GitHub Issues, Datadog, Prometheus alerts, etc. expose the
fields they expect inside their own README files; the Core runtime simply passes
whatever JSON you provide straight through.

### Provider Deep Links
Normalized resources carry optional `url` fields for deep linking back to upstream systems. For individual resources (incidents, alerts, tickets, etc.), the URL links to that specific resource. For collections like log entries and metric series, the URL links to the query results or filtered view in the source system (e.g., Datadog logs dashboard, Grafana metric chart). Adapters should populate these URLs whenever the provider exposes canonical UI links so OpsOrch clients can jump directly to the source system. The field is passthrough only—OpsOrch does not generate, log, or modify these URLs—so adapters remain responsible for ensuring they do not leak secrets.

## Run the Stack Locally

### Docker Compose (fastest path)
A ready-to-run stack lives at `.github/profile/docker-compose.yml` and includes the mock adapters (Core + seeded data), MCP, and the Console.

```bash
curl -O https://raw.githubusercontent.com/OpsOrch/.github/main/profile/docker-compose.yml
DOCKER_DEFAULT_PLATFORM=linux/amd64 docker compose up -d

open http://localhost:3000     # Console
curl http://localhost:8080/health
curl http://localhost:7070/mcp \
  -H 'Content-Type: application/json' \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/list"}'
```

Variants:
- `docker-compose.dev.yml` – mounts local source for live dev against the containers
- `docker-compose.prod.yml` – opinionated production wiring (TLS, secrets, logging)

### Manual Dev Workflow
1. **Core + mock plugins**
   ```bash
   cd opsorch-core
   OPSORCH_INCIDENT_PLUGIN=./plugins/incidentmock \
   OPSORCH_LOG_PLUGIN=./plugins/logmock \
   OPSORCH_METRIC_PLUGIN=./plugins/metricmock \
   OPSORCH_SECRET_PLUGIN=./plugins/secretmock \
   go run ./cmd/opsorch
   ```
2. **MCP tools layer**
   ```bash
   cd ../opsorch-mcp
   npm install
   OPSORCH_CORE_URL=http://localhost:8080 \
   OPSORCH_CORE_TOKEN=demo \
   npm run dev
   ```
3. **Copilot runtime**
   ```bash
   cd ../opsorch-copilot
   npm install
   MCP_URL=http://localhost:7070/mcp \
   LLM_PROVIDER=mock \
   npm run dev
   ```
4. **Console UI**
   ```bash
   cd ../opsorch-console
   npm install
   NEXT_PUBLIC_OPSORCH_CORE_URL=http://localhost:8080 \
   npm run dev:oss
   ```
   Open [http://localhost:3000](http://localhost:3000) and confirm the Incident, Logs, Metrics, Services, Tickets, and Chat tabs load.
   > Enterprise-only Copilot features require `NEXT_PUBLIC_COPILOT_URL` pointing at a Copilot deployment; the OSS edition talks directly to Core only.

Health checks:
- `curl http://localhost:8080/health` (Core)
- `curl http://localhost:7070/mcp -H 'Content-Type: application/json' -d '{"jsonrpc":"2.0","id":1,"method":"tools/list"}'` (MCP)
- `curl http://localhost:6060/health` (Copilot)
- Visit `/settings` in the Console to verify endpoints.

## Build New Adapters
1. **Fork the template**: start from [`opsorch-adapter`](https://github.com/OpsOrch/opsorch-adapter) and rename it to `opsorch-<provider>-adapter`.
2. **Pick capabilities**: implement `incident`, `alert`, `log`, `metric`, `ticket`, `messaging`, `service`, `team`, or `secret` interfaces from `opsorch-core`.
3. **Register**: export `New(config map[string]any)` and call `<cap>.RegisterProvider("<name>", New)` in an `init()`.
4. **Normalize**: map upstream payloads into the schemas in `opsorch-core/schema`; stash provider-specific fields under `metadata`.
5. **Plugin optional**: add a `cmd/<cap>plugin` and run `make plugin` to ship JSON-RPC binaries.
6. **Test**: unit + integration tests should prove schema fidelity and error handling; the `opsorch-mock-adapters` repo is a good reference.

## Building OpsOrch with Your Adapter Stack

Every organization uses different tools. OpsOrch Core is a minimal base image—you add the adapters you need as plugin binaries. This is the standard deployment approach.

Use `ghcr.io/opsorch/opsorch-core:<version>` as the base and add only the plugins for your stack:

**Example: Incident Management + Ticketing + Monitoring Stack**

```dockerfile
FROM ghcr.io/opsorch/opsorch-core:v0.2.0
WORKDIR /opt/opsorch

# Incidents (PagerDuty)
ADD https://github.com/OpsOrch/opsorch-pagerduty-adapter/releases/download/v0.1.0/incidentplugin-linux-amd64 ./plugins/incidentplugin

# Tickets (Jira)  
ADD https://github.com/OpsOrch/opsorch-jira-adapter/releases/download/v0.1.0/ticketplugin-linux-amd64 ./plugins/ticketplugin

# Metrics (Prometheus)
ADD https://github.com/OpsOrch/opsorch-prometheus-adapter/releases/download/v0.1.0/metricplugin-linux-amd64 ./plugins/metricplugin

# Logs (Elasticsearch)
ADD https://github.com/OpsOrch/opsorch-elastic-adapter/releases/download/v0.1.0/logplugin-linux-amd64 ./plugins/logplugin

# Messaging (Slack)
ADD https://github.com/OpsOrch/opsorch-slack-adapter/releases/download/v0.1.0/messagingplugin-linux-amd64 ./plugins/messagingplugin

RUN chmod +x ./plugins/*

ENV OPSORCH_INCIDENT_PLUGIN=/opt/opsorch/plugins/incidentplugin \
    OPSORCH_TICKET_PLUGIN=/opt/opsorch/plugins/ticketplugin \
    OPSORCH_METRIC_PLUGIN=/opt/opsorch/plugins/metricplugin \
    OPSORCH_LOG_PLUGIN=/opt/opsorch/plugins/logplugin \
    OPSORCH_MESSAGING_PLUGIN=/opt/opsorch/plugins/messagingplugin
```

Then configure each adapter at runtime:

```bash
docker build -t my-opsorch:latest .
docker run -p 8080:8080 --env-file .env my-opsorch:latest
```

**.env file:**
```bash
OPSORCH_BEARER_TOKEN=your-token

# PagerDuty
OPSORCH_INCIDENT_CONFIG={"apiToken":"pd_token","serviceID":"PXXXXXX","fromEmail":"oncall@example.com"}

# Jira
OPSORCH_TICKET_CONFIG={"apiToken":"jira_token","email":"bot@example.com","apiURL":"https://your-domain.atlassian.net","projectKey":"OPS"}

# Prometheus
OPSORCH_METRIC_CONFIG={"url":"http://prometheus:9090"}

# Elasticsearch
OPSORCH_LOG_CONFIG={"addresses":["http://elasticsearch:9200"],"username":"elastic","password":"changeme","indexPattern":"logs-*"}

# Slack
OPSORCH_MESSAGING_CONFIG={"token":"xoxb-slack-token"}
```

**Comprehensive Guide:** See [BUILDING_WITH_ADAPTERS.md](BUILDING_WITH_ADAPTERS.md) for complete examples, configuration reference for all adapters, troubleshooting, and production best practices. Also available on [opsorch.com/docs/building-with-adapters](https://opsorch.com/docs/building-with-adapters).

## Safety & Operations
- Scope every query with `service` / `team` / `environment` to keep provider-side searches lean. Team scoping enables organizational filtering across all capabilities.
- Keep mutation workflows (paging, escalations, ticket creation, messaging) behind approvals in Console or Copilot.
- Secrets are always encrypted at rest; never log decrypted configs or tokens.
- Use the audit log APIs (`opsorch-core/api/audit_log*.go`) when you need tamper-evident trails.

## Contributing & Support
- Issues/PRs welcome across repos—follow `opsorch-core/CONTRIBUTING.md` for coding standards.
- See the Architecture section above for the system diagram.
- Need an adapter we do not ship yet? File an issue in this org or reach out at [opsorch.com](https://www.opsorch.com/).
