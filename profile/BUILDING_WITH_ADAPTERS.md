# Building Custom Docker Images for OpsOrch Core

This guide explains how to build custom Docker images for OpsOrch Core with your specific adapter stack. Use this approach when you want to bundle plugin binaries directly into your image for production deployments.

## Table of Contents

- [When to Use Custom Images](#when-to-use-custom-images)
- [Prerequisites](#prerequisites)
- [Quick Start](#quick-start)
- [Understanding the Plugin Architecture](#understanding-the-plugin-architecture)
- [Finding Adapters](#finding-adapters)
- [Building Custom Images](#building-custom-images)
- [Adapter Configuration Reference](#adapter-configuration-reference)
- [Complete Stack Examples](#complete-stack-examples)
- [Troubleshooting](#troubleshooting)
- [Production Best Practices](#production-best-practices)

## When to Use Custom Images

Build a custom Docker image when you:

- Want to bundle specific adapter plugins into your image
- Need a reproducible, immutable image for production
- Deploy to environments where pulling plugins at runtime isn't ideal
- Want to version control your exact adapter stack

**Alternatives:**
- **In-process providers**: Import adapters as Go packages and compile them into the core binary (requires rebuilding core)
- **External plugin mounting**: Mount plugin binaries via volumes at runtime (flexible but requires volume management)

## Prerequisites

- Docker installed (v20.10+)
- Access to adapter release pages on GitHub
- Understanding of which capabilities your stack needs

## Quick Start

Here's a minimal example that adds Jira ticket support to OpsOrch Core:

**Dockerfile:**
```dockerfile
FROM ghcr.io/opsorch/opsorch-core:v0.2.0
WORKDIR /opt/opsorch

# Download Jira ticket plugin
ADD https://github.com/OpsOrch/opsorch-jira-adapter/releases/download/v0.1.0/ticketplugin-linux-amd64 ./plugins/ticketplugin
RUN chmod +x ./plugins/ticketplugin

# Configure plugin path
ENV OPSORCH_TICKET_PLUGIN=/opt/opsorch/plugins/ticketplugin
```

**Build and run:**
```bash
docker build -t my-opsorch:latest .
docker run -p 8080:8080 \
  -e OPSORCH_TICKET_CONFIG='{"apiToken":"your-token","email":"you@example.com","apiURL":"https://your-domain.atlassian.net","projectKey":"OPS"}' \
  my-opsorch:latest
```

## Understanding the Plugin Architecture

OpsOrch Core supports three capability loading methods:

### 1. In-Process Providers (Go Import)
```go
import _ "github.com/opsorch/opsorch-jira-adapter/ticket"
```
- **Pros**: Fastest, no subprocess overhead
- **Cons**: Requires rebuilding core, static linking

### 2. Plugin Binaries (JSON-RPC)
```bash
OPSORCH_TICKET_PLUGIN=/opt/opsorch/plugins/ticketplugin
```
- **Pros**: No core rebuild, language agnostic, hot-swappable
- **Cons**: Slight RPC overhead

### 3. Volume-Mounted Plugins
```yaml
volumes:
  - ./plugins:/opt/opsorch/plugins:ro
```
- **Pros**: Most flexible, easy to update
- **Cons**: Requires volume management

This guide focuses on **method #2** (bundled plugin binaries) as it balances flexibility and reproducibility.

## Finding Adapters

All OpsOrch adapters are published as open-source repositories under the OpsOrch GitHub organization:

### Production Adapters

| Adapter | Capabilities | Release Page |
|---------|--------------|--------------|
| [opsorch-pagerduty-adapter](https://github.com/OpsOrch/opsorch-pagerduty-adapter) | Incident, Service | [Releases](https://github.com/OpsOrch/opsorch-pagerduty-adapter/releases) |
| [opsorch-jira-adapter](https://github.com/OpsOrch/opsorch-jira-adapter) | Ticket | [Releases](https://github.com/OpsOrch/opsorch-jira-adapter/releases) |
| [opsorch-prometheus-adapter](https://github.com/OpsOrch/opsorch-prometheus-adapter) | Metric, Alert | [Releases](https://github.com/OpsOrch/opsorch-prometheus-adapter/releases) |
| [opsorch-elastic-adapter](https://github.com/OpsOrch/opsorch-elastic-adapter) | Log | [Releases](https://github.com/OpsOrch/opsorch-elastic-adapter/releases) |
| [opsorch-slack-adapter](https://github.com/OpsOrch/opsorch-slack-adapter) | Messaging | [Releases](https://github.com/OpsOrch/opsorch-slack-adapter/releases) |
| [opsorch-github-adapter](https://github.com/OpsOrch/opsorch-github-adapter) | Ticket, Deployment, Team | [Releases](https://github.com/OpsOrch/opsorch-github-adapter/releases) |
| [opsorch-datadog-adapter](https://github.com/OpsOrch/opsorch-datadog-adapter) | Metric, Log, Alert, Incident, Service | [Releases](https://github.com/OpsOrch/opsorch-datadog-adapter/releases) |

### Finding Plugin Binaries

Each adapter repository publishes pre-built plugin binaries for multiple architectures:

1. Go to the adapter's GitHub releases page
2. Find the latest release (e.g., `v0.1.0`)
3. Download the appropriate binary for your platform:
   - `{capability}plugin-linux-amd64` - Linux x86_64
   - `{capability}plugin-linux-arm64` - Linux ARM64
   - `{capability}plugin-darwin-amd64` - macOS Intel
   - `{capability}plugin-darwin-arm64` - macOS Apple Silicon

**Example URLs:**
```
https://github.com/OpsOrch/opsorch-jira-adapter/releases/download/v0.1.0/ticketplugin-linux-amd64
https://github.com/OpsOrch/opsorch-prometheus-adapter/releases/download/v0.1.0/metricplugin-linux-amd64
https://github.com/OpsOrch/opsorch-pagerduty-adapter/releases/download/v0.1.5/incidentplugin-linux-amd64
```

## Building Custom Images

### Single-Adapter Example

**Dockerfile: Prometheus Metrics**
```dockerfile
FROM ghcr.io/opsorch/opsorch-core:v0.2.0
WORKDIR /opt/opsorch

# Download Prometheus metric plugin
ADD https://github.com/OpsOrch/opsorch-prometheus-adapter/releases/download/v0.1.0/metricplugin-linux-amd64 ./plugins/metricplugin
RUN chmod +x ./plugins/metricplugin

# Set plugin path
ENV OPSORCH_METRIC_PLUGIN=/opt/opsorch/plugins/metricplugin \
    OPSORCH_BEARER_TOKEN=changeme
```

### Multi-Adapter Production Image

**Dockerfile: Full Observability Stack**
```dockerfile
FROM ghcr.io/opsorch/opsorch-core:v0.2.0
WORKDIR /opt/opsorch

# Incident Management (PagerDuty)
ADD https://github.com/OpsOrch/opsorch-pagerduty-adapter/releases/download/v0.1.5/incidentplugin-linux-amd64 ./plugins/incidentplugin
ADD https://github.com/OpsOrch/opsorch-pagerduty-adapter/releases/download/v0.1.5/serviceplugin-linux-amd64 ./plugins/serviceplugin

# Ticketing (Jira)
ADD https://github.com/OpsOrch/opsorch-jira-adapter/releases/download/v0.1.0/ticketplugin-linux-amd64 ./plugins/ticketplugin

# Metrics (Prometheus)
ADD https://github.com/OpsOrch/opsorch-prometheus-adapter/releases/download/v0.1.0/metricplugin-linux-amd64 ./plugins/metricplugin
ADD https://github.com/OpsOrch/opsorch-prometheus-adapter/releases/download/v0.1.0/alertplugin-linux-amd64 ./plugins/alertplugin

# Logs (Elasticsearch)
ADD https://github.com/OpsOrch/opsorch-elastic-adapter/releases/download/v0.1.0/logplugin-linux-amd64 ./plugins/logplugin

# Messaging (Slack)
ADD https://github.com/OpsOrch/opsorch-slack-adapter/releases/download/v0.1.0/messagingplugin-linux-amd64 ./plugins/messagingplugin

# Deployments & Teams (GitHub)
ADD https://github.com/OpsOrch/opsorch-github-adapter/releases/download/v0.1.0/deploymentplugin-linux-amd64 ./plugins/deploymentplugin
ADD https://github.com/OpsOrch/opsorch-github-adapter/releases/download/v0.1.0/teamplugin-linux-amd64 ./plugins/teamplugin

# Make all plugins executable
RUN chmod +x ./plugins/*

# Configure plugin paths
ENV OPSORCH_INCIDENT_PLUGIN=/opt/opsorch/plugins/incidentplugin \
    OPSORCH_SERVICE_PLUGIN=/opt/opsorch/plugins/serviceplugin \
    OPSORCH_TICKET_PLUGIN=/opt/opsorch/plugins/ticketplugin \
    OPSORCH_METRIC_PLUGIN=/opt/opsorch/plugins/metricplugin \
    OPSORCH_ALERT_PLUGIN=/opt/opsorch/plugins/alertplugin \
    OPSORCH_LOG_PLUGIN=/opt/opsorch/plugins/logplugin \
    OPSORCH_MESSAGING_PLUGIN=/opt/opsorch/plugins/messagingplugin \
    OPSORCH_DEPLOYMENT_PLUGIN=/opt/opsorch/plugins/deploymentplugin \
    OPSORCH_TEAM_PLUGIN=/opt/opsorch/plugins/teamplugin \
    OPSORCH_BEARER_TOKEN=changeme
```

**Build:**
```bash
docker build -t my-opsorch:latest .
```

**Run with configuration:**
```bash
docker run -p 8080:8080 --env-file .env my-opsorch:latest
```

### Best Practices for Image Building

#### 1. Pin Versions
Always use specific versions instead of `latest`:
```dockerfile
# Good
FROM ghcr.io/opsorch/opsorch-core:v0.2.0
ADD https://github.com/OpsOrch/opsorch-jira-adapter/releases/download/v0.1.0/ticketplugin-linux-amd64

# Avoid
FROM ghcr.io/opsorch/opsorch-core:latest
```

#### 2. Optimize Layer Caching
Group related downloads together:
```dockerfile
# Download all plugins in one layer
ADD https://github.com/.../plugin1-linux-amd64 ./plugins/plugin1 \
    https://github.com/.../plugin2-linux-amd64 ./plugins/plugin2

# Make them executable in one layer
RUN chmod +x ./plugins/*
```

#### 3. Multi-Architecture Builds
Support both amd64 and arm64:
```bash
docker buildx build --platform linux/amd64,linux/arm64 -t my-opsorch:latest .
```

For multi-arch, use conditional downloads:
```dockerfile
ARG TARGETARCH
ADD https://github.com/OpsOrch/opsorch-jira-adapter/releases/download/v0.1.0/ticketplugin-linux-${TARGETARCH} ./plugins/ticketplugin
```

## Adapter Configuration Reference

Each adapter requires specific configuration passed via `OPSORCH_{CAPABILITY}_CONFIG` environment variables. Below are configuration templates for each adapter.

### PagerDuty (Incident & Service)

**Required fields:**
```json
{
  "apiToken": "your-pagerduty-api-token",
  "serviceID": "PXXXXXX",
  "fromEmail": "oncall@example.com",
  "apiURL": "https://api.pagerduty.com"
}
```

**Environment variable:**
```bash
OPSORCH_INCIDENT_CONFIG='{"apiToken":"...","serviceID":"PXXXXXX","fromEmail":"oncall@example.com","apiURL":"https://api.pagerduty.com"}'
OPSORCH_SERVICE_CONFIG='{"apiToken":"...","apiURL":"https://api.pagerduty.com"}'
```

**Setup:**
1. Get API token from PagerDuty: **Integrations** → **API Access Keys** → **Create New API Key**
2. Find Service ID from service URL: `https://yourorg.pagerduty.com/services/PXXXXXX`
3. Use a valid PagerDuty user email for `fromEmail`

### Jira (Ticket)

**Required fields:**
```json
{
  "apiToken": "your-jira-api-token",
  "email": "bot@example.com",
  "apiURL": "https://your-domain.atlassian.net",
  "projectKey": "OPS"
}
```

**Environment variable:**
```bash
OPSORCH_TICKET_CONFIG='{"apiToken":"...","email":"bot@example.com","apiURL":"https://your-domain.atlassian.net","projectKey":"OPS"}'
```

**Setup:**
1. Create API token at: https://id.atlassian.com/manage-profile/security/api-tokens
2. Use your Atlassian account email
3. Find project key from Jira project settings

### Prometheus (Metric & Alert)

**Required fields:**
```json
{
  "url": "http://prometheus:9090"
}
```

For alerts (Alertmanager):
```json
{
  "alertmanagerURL": "http://alertmanager:9093"
}
```

**Environment variables:**
```bash
OPSORCH_METRIC_CONFIG='{"url":"http://prometheus:9090"}'
OPSORCH_ALERT_CONFIG='{"alertmanagerURL":"http://alertmanager:9093"}'
```

### Elasticsearch (Log)

**Required fields:**
```json
{
  "addresses": ["http://elasticsearch:9200"],
  "username": "elastic",
  "password": "your-password",
  "indexPattern": "logs-*"
}
```

**Environment variable:**
```bash
OPSORCH_LOG_CONFIG='{"addresses":["http://elasticsearch:9200"],"username":"elastic","password":"changeme","indexPattern":"logs-*"}'
```

### Slack (Messaging)

**Required fields:**
```json
{
  "token": "xoxb-your-slack-bot-token"
}
```

**Environment variable:**
```bash
OPSORCH_MESSAGING_CONFIG='{"token":"xoxb-..."}'
```

**Setup:**
1. Create Slack app at: https://api.slack.com/apps
2. Add **Bot Token Scopes**: `chat:write`, `channels:read`, `groups:read`, `im:read`, `mpim:read`
3. Install app to workspace
4. Copy **Bot User OAuth Token**

### GitHub (Ticket, Deployment, Team)

**Required fields:**
```json
{
  "token": "ghp_your-github-token",
  "owner": "your-org",
  "repo": "your-repo",
  "organization": "your-org"
}
```

**Environment variables:**
```bash
OPSORCH_TICKET_CONFIG='{"token":"ghp_...","owner":"your-org","repo":"your-repo"}'
OPSORCH_DEPLOYMENT_CONFIG='{"token":"ghp_...","owner":"your-org","repo":"your-repo"}'
OPSORCH_TEAM_CONFIG='{"token":"ghp_...","organization":"your-org"}'
```

**Setup:**
1. Create Personal Access Token: **Settings** → **Developer settings** → **Personal access tokens** → **Tokens (classic)**
2. Required scopes: `repo`, `read:org`, `read:user`

### Datadog (Metric, Log, Alert, Incident, Service)

**Required fields:**
```json
{
  "apiKey": "your-datadog-api-key",
  "appKey": "your-datadog-app-key",
  "site": "datadoghq.com"
}
```

**Environment variables:**
```bash
OPSORCH_METRIC_CONFIG='{"apiKey":"...","appKey":"...","site":"datadoghq.com"}'
OPSORCH_LOG_CONFIG='{"apiKey":"...","appKey":"...","site":"datadoghq.com"}'
OPSORCH_ALERT_CONFIG='{"apiKey":"...","appKey":"...","site":"datadoghq.com"}'
```

**Setup:**
1. Get API key: **Organization Settings** → **API Keys**
2. Get App key: **Organization Settings** → **Application Keys**
3. Set `site` based on your region (e.g., `datadoghq.eu`, `us3.datadoghq.com`)

## Complete Stack Examples

### Example 1: Observability Stack

**Use case:** Monitor infrastructure with Prometheus metrics, Elasticsearch logs, and PagerDuty incidents.

**Dockerfile:**
```dockerfile
FROM ghcr.io/opsorch/opsorch-core:v0.2.0
WORKDIR /opt/opsorch

# Metrics & Alerts
ADD https://github.com/OpsOrch/opsorch-prometheus-adapter/releases/download/v0.1.0/metricplugin-linux-amd64 ./plugins/metricplugin
ADD https://github.com/OpsOrch/opsorch-prometheus-adapter/releases/download/v0.1.0/alertplugin-linux-amd64 ./plugins/alertplugin

# Logs
ADD https://github.com/OpsOrch/opsorch-elastic-adapter/releases/download/v0.1.0/logplugin-linux-amd64 ./plugins/logplugin

# Incidents
ADD https://github.com/OpsOrch/opsorch-pagerduty-adapter/releases/download/v0.1.5/incidentplugin-linux-amd64 ./plugins/incidentplugin

RUN chmod +x ./plugins/*

ENV OPSORCH_METRIC_PLUGIN=/opt/opsorch/plugins/metricplugin \
    OPSORCH_ALERT_PLUGIN=/opt/opsorch/plugins/alertplugin \
    OPSORCH_LOG_PLUGIN=/opt/opsorch/plugins/logplugin \
    OPSORCH_INCIDENT_PLUGIN=/opt/opsorch/plugins/incidentplugin
```

**.env file:**
```bash
OPSORCH_BEARER_TOKEN=your-secure-token

# Prometheus
OPSORCH_METRIC_CONFIG={"url":"http://prometheus:9090"}
OPSORCH_ALERT_CONFIG={"alertmanagerURL":"http://alertmanager:9093"}

# Elasticsearch
OPSORCH_LOG_CONFIG={"addresses":["http://elasticsearch:9200"],"username":"elastic","password":"changeme","indexPattern":"logs-*"}

# PagerDuty
OPSORCH_INCIDENT_CONFIG={"apiToken":"pd_token","serviceID":"PXXXXXX","fromEmail":"oncall@example.com","apiURL":"https://api.pagerduty.com"}
```

### Example 2: Development Platform Stack

**Use case:** Support development teams with GitHub issues, Jira tickets, and Slack notifications.

**Dockerfile:**
```dockerfile
FROM ghcr.io/opsorch/opsorch-core:v0.2.0
WORKDIR /opt/opsorch

# GitHub (Issues, Deployments, Teams)
ADD https://github.com/OpsOrch/opsorch-github-adapter/releases/download/v0.1.0/ticketplugin-linux-amd64 ./plugins/github-ticketplugin
ADD https://github.com/OpsOrch/opsorch-github-adapter/releases/download/v0.1.0/deploymentplugin-linux-amd64 ./plugins/deploymentplugin
ADD https://github.com/OpsOrch/opsorch-github-adapter/releases/download/v0.1.0/teamplugin-linux-amd64 ./plugins/teamplugin

# Jira (Alternative ticketing)
ADD https://github.com/OpsOrch/opsorch-jira-adapter/releases/download/v0.1.0/ticketplugin-linux-amd64 ./plugins/jira-ticketplugin

# Slack
ADD https://github.com/OpsOrch/opsorch-slack-adapter/releases/download/v0.1.0/messagingplugin-linux-amd64 ./plugins/messagingplugin

RUN chmod +x ./plugins/*

# Note: We can't have two ticket providers active simultaneously
# Choose either GitHub or Jira by setting the respective _CONFIG
ENV OPSORCH_DEPLOYMENT_PLUGIN=/opt/opsorch/plugins/deploymentplugin \
    OPSORCH_TEAM_PLUGIN=/opt/opsorch/plugins/teamplugin \
    OPSORCH_MESSAGING_PLUGIN=/opt/opsorch/plugins/messagingplugin
```

**.env file (using Jira for tickets):**
```bash
OPSORCH_BEARER_TOKEN=your-secure-token

# Jira
OPSORCH_TICKET_CONFIG={"apiToken":"jira_token","email":"bot@example.com","apiURL":"https://your-domain.atlassian.net","projectKey":"DEV"}

# GitHub
OPSORCH_DEPLOYMENT_CONFIG={"token":"ghp_token","owner":"your-org","repo":"your-repo"}
OPSORCH_TEAM_CONFIG={"token":"ghp_token","organization":"your-org"}

# Slack
OPSORCH_MESSAGING_CONFIG={"token":"xoxb-slack-token"}
```

### Example 3: Datadog-Centric Stack

**Use case:** Use Datadog as the primary observability platform.

**Dockerfile:**
```dockerfile
FROM ghcr.io/opsorch/opsorch-core:v0.2.0
WORKDIR /opt/opsorch

# Datadog (all capabilities)
ADD https://github.com/OpsOrch/opsorch-datadog-adapter/releases/download/v0.1.0/metricplugin-linux-amd64 ./plugins/metricplugin
ADD https://github.com/OpsOrch/opsorch-datadog-adapter/releases/download/v0.1.0/logplugin-linux-amd64 ./plugins/logplugin
ADD https://github.com/OpsOrch/opsorch-datadog-adapter/releases/download/v0.1.0/alertplugin-linux-amd64 ./plugins/alertplugin
ADD https://github.com/OpsOrch/opsorch-datadog-adapter/releases/download/v0.1.0/incidentplugin-linux-amd64 ./plugins/incidentplugin

# PagerDuty for on-call
ADD https://github.com/OpsOrch/opsorch-pagerduty-adapter/releases/download/v0.1.5/serviceplugin-linux-amd64 ./plugins/serviceplugin

RUN chmod +x ./plugins/*

ENV OPSORCH_METRIC_PLUGIN=/opt/opsorch/plugins/metricplugin \
    OPSORCH_LOG_PLUGIN=/opt/opsorch/plugins/logplugin \
    OPSORCH_ALERT_PLUGIN=/opt/opsorch/plugins/alertplugin \
    OPSORCH_INCIDENT_PLUGIN=/opt/opsorch/plugins/incidentplugin \
    OPSORCH_SERVICE_PLUGIN=/opt/opsorch/plugins/serviceplugin
```

**.env file:**
```bash
OPSORCH_BEARER_TOKEN=your-secure-token

# Datadog (shared config for all capabilities)
OPSORCH_METRIC_CONFIG={"apiKey":"dd_api_key","appKey":"dd_app_key","site":"datadoghq.com"}
OPSORCH_LOG_CONFIG={"apiKey":"dd_api_key","appKey":"dd_app_key","site":"datadoghq.com"}
OPSORCH_ALERT_CONFIG={"apiKey":"dd_api_key","appKey":"dd_app_key","site":"datadoghq.com"}
OPSORCH_INCIDENT_CONFIG={"apiKey":"dd_api_key","appKey":"dd_app_key","site":"datadoghq.com"}

# PagerDuty Services
OPSORCH_SERVICE_CONFIG={"apiToken":"pd_token","apiURL":"https://api.pagerduty.com"}
```

## Troubleshooting

### Plugin Not Loading

**Symptom:** API returns `501 Not Implemented` with error `{capability}_provider_missing`

**Causes:**
1. Plugin binary not executable
2. Plugin path incorrect
3. Config not provided

**Solutions:**
```bash
# Check file permissions
docker exec -it <container> ls -la /opt/opsorch/plugins/

# Verify plugin path
docker exec -it <container> env | grep OPSORCH_

# Test plugin directly
docker exec -it <container> /opt/opsorch/plugins/ticketplugin
# Should start and wait for RPC input

# Check logs
docker logs <container>
```

### Configuration Validation Errors

**Symptom:** Plugin loads but API calls fail with authentication errors

**Causes:**
1. Missing required configuration fields
2. Invalid credentials
3. Malformed JSON

**Solutions:**
```bash
# Validate JSON syntax
echo $OPSORCH_TICKET_CONFIG | jq .

# Check required fields (refer to adapter documentation)
# Example for Jira:
echo $OPSORCH_TICKET_CONFIG | jq 'has("apiToken") and has("email") and has("apiURL") and has("projectKey")'

# Test credentials manually
# Example for Jira:
curl -u 'email:apiToken' -H "Accept: application/json" "https://your-domain.atlassian.net/rest/api/3/myself"
```

### Permission Issues

**Symptom:** Plugin binary exists but won't execute

**Solutions:**
```dockerfile
# Ensure chmod in Dockerfile
RUN chmod +x ./plugins/*

# Or use specific permissions
RUN chmod 755 /opt/opsorch/plugins/ticketplugin

# If using non-root user
USER opsorch
RUN chown opsorch:opsorch /opt/opsorch/plugins/*
```

### Network Connectivity

**Symptom:** Plugin loads but API calls timeout

**Causes:**
1. Incorrect URLs
2. Network policies blocking traffic
3. DNS resolution issues

**Solutions:**
```bash
# Test connectivity from container
docker exec -it <container> curl -v http://prometheus:9090
docker exec -it <container> ping elasticsearch

# Check DNS resolution
docker exec -it <container> nslookup prometheus

# Use IP addresses as fallback
OPSORCH_METRIC_CONFIG='{"url":"http://10.0.1.50:9090"}'
```

### Common Error Messages

| Error | Meaning | Solution |
|-------|---------|----------|
| `provider_missing` | No plugin configured for capability | Set `OPSORCH_{CAP}_PLUGIN` and `OPSORCH_{CAP}_CONFIG` |
| `plugin_not_found` | Plugin binary doesn't exist at path | Check file path and permissions |
| `plugin_start_failed` | Plugin crashed on startup | Check plugin logs, verify architecture matches |
| `rpc_timeout` | Plugin not responding | Increase timeout or check plugin health |
| `auth_failed` | Invalid credentials | Verify API tokens/keys in config |
| `invalid_config` | Missing required fields | Check adapter README for required fields |

## Production Best Practices

### Security

**1. Never commit secrets to images:**
```dockerfile
# BAD - Don't do this
ENV OPSORCH_TICKET_CONFIG='{"apiToken":"hardcoded-secret"}'

# GOOD - Inject at runtime
docker run -e OPSORCH_TICKET_CONFIG="${SECRET_FROM_VAULT}" ...
```

**2. Use secrets management:**
```yaml
# Kubernetes Secret
apiVersion: v1
kind: Secret
metadata:
  name: opsorch-config
stringData:
  ticket-config: |
    {"apiToken":"...","email":"...","apiURL":"...","projectKey":"OPS"}
```

**3. Run as non-root:**
```dockerfile
USER nobody:nogroup
```

**4. Scan images for vulnerabilities:**
```bash
docker scan my-opsorch:latest
trivy image my-opsorch:latest
```

### Health Checks

Add health checks to your deployment:

**Dockerfile:**
```dockerfile
HEALTHCHECK --interval=30s --timeout=10s --start-period=30s --retries=3 \
  CMD wget --no-verbose --tries=1 --spider http://localhost:8080/health || exit 1
```

**Docker Compose:**
```yaml
healthcheck:
  test: ["CMD", "wget", "--no-verbose", "--tries=1", "--spider", "http://localhost:8080/health"]
  interval: 30s
  timeout: 10s
  retries: 3
  start_period: 30s
```

### Resource Limits

Configure appropriate limits:

```yaml
deploy:
  resources:
    limits:
      memory: 512M
      cpus: '0.5'
    reservations:
      memory: 256M
      cpus: '0.25'
```

### Logging

Configure structured logging:

```bash
ENV OPSORCH_LOG_LEVEL=info \
    OPSORCH_LOG_FORMAT=json
```

Ship logs to your aggregation platform:
```yaml
logging:
  driver: "json-file"
  options:
    max-size: "10m"
    max-file: "3"
```

### Image Versioning

**Tag strategy:**
```bash
# Semantic versioning
my-opsorch:1.2.3

# Git commit SHA
my-opsorch:a1b2c3d

# Build date
my-opsorch:2024-01-15

# Combined
my-opsorch:1.2.3-a1b2c3d
```

**Example CI/CD:**
```bash
VERSION=$(cat VERSION)
GIT_SHA=$(git rev-parse --short HEAD)
IMAGE_TAG="${VERSION}-${GIT_SHA}"

docker build -t my-opsorch:${IMAGE_TAG} .
docker tag my-opsorch:${IMAGE_TAG} my-opsorch:latest
docker push my-opsorch:${IMAGE_TAG}
docker push my-opsorch:latest
```

### Backup and Disaster Recovery

While OpsOrch Core is stateless, you should backup:

1. **Configuration secrets** - Store in vault/secrets manager
2. **Image registry** - Ensure images are pushed to a registry with backup
3. **Environment files** - Version control your `.env` templates (without secrets)

**Example backup script:**
```bash
#!/bin/bash
# Backup configuration (without secrets)
cat <<EOF > opsorch-config-manifest.yaml
image: my-opsorch:${VERSION}
adapters:
  - incident: pagerduty
  - ticket: jira
  - metric: prometheus
  - log: elasticsearch
  - messaging: slack
build_date: $(date -u +"%Y-%m-%dT%H:%M:%SZ")
git_ref: $(git rev-parse HEAD)
EOF
```

---

## Next Steps

- Review [adapter documentation](https://github.com/orgs/OpsOrch/repositories?q=adapter) for detailed configuration options
- Check the [production Docker Compose example](docker-compose.prod.yml) for deployment patterns
- Visit [opsorch.com/docs](https://opsorch.com/docs) for additional guides
- Join [GitHub Discussions](https://github.com/orgs/OpsOrch/discussions) for community support

## Contributing

Found an issue or want to improve this guide?
- Open an issue: https://github.com/OpsOrch/.github/issues
- Submit a PR: https://github.com/OpsOrch/.github/pulls
