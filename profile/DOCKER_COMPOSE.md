# OpsOrch Docker Compose Configurations

This repository provides several Docker Compose configurations to run the complete OpsOrch stack in different environments.

## Quick Start (Demo Environment)

The fastest way to try OpsOrch is with the demo configuration using mock adapters:

```bash
# Download the demo configuration
curl -O https://raw.githubusercontent.com/OpsOrch/.github/main/profile/docker-compose.yml

# Start the complete stack with demo data
docker compose up -d

# Access the services
open http://localhost:3000  # Console UI
curl http://localhost:8080/health  # Core API health check
curl http://localhost:7070/mcp -H 'Content-Type: application/json' -d '{"jsonrpc":"2.0","id":1,"method":"tools/list"}'  # MCP tools
```

This starts:
- **OpsOrch Mock Adapters** (includes Core + all mock providers with demo data) on port 8080
- **OpsOrch MCP Server** (AI tools layer) on port 7070  
- **OpsOrch Console OSS** (web UI) on port 3000 — the OSS console talks directly to Core; Copilot is optional and only needed for Enterprise chat features.

## Available Configurations

### 1. `docker-compose.yml` - Demo/Evaluation
**Best for:** First-time users, demos, evaluation

- Uses mock adapters with realistic demo data
- Single OSS console instance
- Minimal configuration required
- Perfect for understanding OpsOrch capabilities

```bash
curl -O https://raw.githubusercontent.com/OpsOrch/.github/main/profile/docker-compose.yml
docker compose up -d
```

### 2. `docker-compose.dev.yml` - Development
**Best for:** Developers, testing both editions, local development

- Mock adapters with debug logging
- Both OSS and Enterprise console editions
- Placeholder for Copilot integration
- Development-friendly configuration

```bash
curl -O https://raw.githubusercontent.com/OpsOrch/.github/main/profile/docker-compose.dev.yml
docker compose -f docker-compose.dev.yml up -d
```

Access:
- Console OSS: http://localhost:3000
- Console Enterprise: http://localhost:3001
- Core API: http://localhost:8080
- MCP Server: http://localhost:7070
- Copilot API: http://localhost:6060 (only if you enable the private Copilot image)

> The Enterprise console block has `NEXT_PUBLIC_COPILOT_URL` commented out. Uncomment (and enable the `opsorch-copilot` service) once you have access to the private Copilot repo; otherwise enterprise chat features remain disabled while the rest of the UI continues to talk directly to Core.

### 3. `docker-compose.prod.yml` - Production Template
**Best for:** Production deployments with real providers

- Base Core image (requires custom build with adapters)
- Production-ready configuration
- Environment variable driven
- Requires real provider credentials

```bash
curl -O https://raw.githubusercontent.com/OpsOrch/.github/main/profile/docker-compose.prod.yml
# 1. Build custom image with your adapters (see example below)
# 2. Configure environment variables
# 3. Start the stack
docker compose -f docker-compose.prod.yml up -d
```

## Production Setup

### Step 1: Build Custom Core Image

Create a `Dockerfile` with your required adapters:

```dockerfile
FROM ghcr.io/opsorch/opsorch-core:latest
WORKDIR /opt/opsorch

# Download specific adapter versions you need
ADD https://github.com/OpsOrch/opsorch-jira-adapter/releases/download/v0.2.1/ticketplugin-linux-amd64 ./plugins/ticketplugin
ADD https://github.com/OpsOrch/opsorch-pagerduty-adapter/releases/download/v0.1.5/incidentplugin-linux-amd64 ./plugins/incidentplugin
ADD https://github.com/OpsOrch/opsorch-prometheus-adapter/releases/download/v0.3.0/metricplugin-linux-amd64 ./plugins/metricplugin
ADD https://github.com/OpsOrch/opsorch-slack-adapter/releases/download/v0.2.0/messagingplugin-linux-amd64 ./plugins/messagingplugin

RUN chmod +x ./plugins/*
```

Build your custom image:
```bash
docker build -t my-opsorch-core:latest .
```

### Step 2: Configure Environment Variables

Create a `.env` file:

```bash
# Security
OPSORCH_BEARER_TOKEN=your-secure-random-token-here

# PagerDuty Configuration
PAGERDUTY_CONFIG={"apiToken":"your-pd-token","serviceID":"your-service-id"}

# Jira Configuration  
JIRA_CONFIG={"apiToken":"your-jira-token","email":"your-email@company.com","baseURL":"https://your-domain.atlassian.net"}

# Prometheus Configuration
PROMETHEUS_CONFIG={"url":"http://your-prometheus:9090"}

# Slack Configuration
SLACK_CONFIG={"token":"xoxb-your-slack-bot-token"}

# Console Configuration
CORE_URL=http://localhost:8080
CONSOLE_URL=http://localhost:3000
# COPILOT_URL=http://localhost:6060  # Enterprise only, when Copilot is deployed
```

### Step 3: Update docker-compose.prod.yml

Replace the Core service image with your custom image:

```yaml
services:
  opsorch-core:
    image: my-opsorch-core:latest  # Your custom image
    # ... rest of configuration
```

### Step 4: Deploy

```bash
docker compose -f docker-compose.prod.yml up -d
```

## Available Adapters

| Provider | Capability | Plugin Binary | Configuration |
|----------|------------|---------------|---------------|
| **PagerDuty** | Incident, Service | `incidentplugin`, `serviceplugin` | `{"apiToken":"pd_token","serviceID":"service_id"}` |
| **Jira** | Ticket | `ticketplugin` | `{"apiToken":"token","email":"user@domain","baseURL":"https://domain.atlassian.net"}` |
| **Prometheus** | Metric | `metricplugin` | `{"url":"http://prometheus:9090"}` |
| **Elasticsearch** | Log | `logplugin` | `{"url":"http://elasticsearch:9200","username":"user","password":"pass"}` |
| **Datadog** | Metric, Log, Alert, Incident, Service | `metricplugin`, `logplugin`, etc. | `{"apiKey":"dd_api_key","appKey":"dd_app_key","site":"datadoghq.com"}` |
| **Slack** | Messaging | `messagingplugin` | `{"token":"xoxb-slack-bot-token"}` |

Download pre-built binaries from each adapter's [GitHub Releases](https://github.com/OpsOrch).

## Service Architecture

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│  Console UI     │    │   MCP Server    │    │ Mock Adapters   │
│  (Next.js)      │    │  (TypeScript)   │    │ (Core + Mocks)  │
│  Port 3000      │◄──►│  Port 7070      │◄──►│   Port 8080     │
└─────────────────┘    └─────────────────┘    └─────────────────┘
                                                       │
                                                       ▼
                                              ┌─────────────────┐
                                              │   Demo Data     │
                                              │ (All Providers) │
                                              └─────────────────┘
```

## Health Checks

All services include health checks. Monitor service status:

```bash
# Check all services
docker compose ps

# Check individual service logs
docker compose logs opsorch-mock-adapters
docker compose logs opsorch-mcp
docker compose logs opsorch-console

# Manual health checks
curl http://localhost:8080/health
curl http://localhost:7070/mcp -H 'Content-Type: application/json' -d '{"jsonrpc":"2.0","id":1,"method":"tools/list"}'
curl http://localhost:3000
```

## Scaling and High Availability

For production deployments:

1. **Load Balancing**: Put a load balancer (nginx, HAProxy) in front of multiple Core instances
2. **Database**: Core is stateless, but consider external secret storage for production
3. **Monitoring**: Add Prometheus metrics and Grafana dashboards
4. **Logging**: Configure centralized logging (ELK stack, Datadog, etc.)
5. **Security**: Use proper TLS certificates and secure token management

## Troubleshooting

### Common Issues

**Services won't start:**
```bash
# Check logs
docker compose logs

# Verify network connectivity
docker network ls
docker network inspect opsorch-network
```

**Console can't reach Core:**
```bash
# Verify Mock Adapters service is healthy
curl http://localhost:8080/health

# Check network configuration in docker-compose.yml
# Ensure NEXT_PUBLIC_OPSORCH_CORE_URL is correct
```

**MCP tools not working:**
```bash
# Test MCP server directly
curl http://localhost:7070/mcp \
  -H 'Content-Type: application/json' \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/list"}'

# Check Core connectivity from MCP container
docker exec opsorch-mcp curl http://opsorch-mock-adapters:8080/health
```

**Provider authentication errors:**
```bash
# Check Mock Adapters logs for any issues
docker compose logs opsorch-mock-adapters

# Verify provider configuration JSON is valid (for production)
echo $JIRA_CONFIG | jq .
```

### Getting Help

- Check service logs: `docker compose logs [service-name]`
- Verify environment variables: `docker compose config`
- Test individual components: Use curl commands above
- File issues: [GitHub Issues](https://github.com/OpsOrch/opsorch-core/issues)

## Cleanup

```bash
# Stop services
docker compose down

# Remove volumes (careful - this deletes data!)
docker compose down -v

# Remove images
docker compose down --rmi all
```
