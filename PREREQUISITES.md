
# Prerequisites — Azure Arc Server Insights & Dependency Mapping

This document outlines everything that must be in place **before** beginning the deployment.

---

## Azure Subscription Requirements

### Resource Providers

The following resource providers must be registered on the target subscription:

- `Microsoft.HybridCompute`
- `Microsoft.GuestConfiguration`
- `Microsoft.Insights`
- `Microsoft.Monitor`
- `Microsoft.OperationalInsights`

To register a provider:

```bash
az provider register --namespace Microsoft.HybridCompute
az provider register --namespace Microsoft.GuestConfiguration
az provider register --namespace Microsoft.Insights
az provider register --namespace Microsoft.Monitor
az provider register --namespace Microsoft.OperationalInsights
```

### RBAC Roles

The deploying identity requires the following roles on the target resource group:

| Role | Purpose |
|------|---------|
| **Azure Connected Machine Resource Administrator** | Register and manage Arc-enabled servers |
| **Monitoring Contributor** | Create and configure monitoring resources, DCRs, and VM Insights |
| **Log Analytics Contributor** | Create and manage the Log Analytics Workspace |

---

## Networking Requirements

### Firewall Rules

| Direction | Protocol | Port | Purpose |
|-----------|----------|------|---------|
| **Outbound only** | HTTPS | 443 | All agent communication to Azure |

> **No inbound firewall rules are required.**

### Required Azure Endpoints

All servers must have outbound HTTPS 443 connectivity to:

- **Azure Arc** service endpoints
- **Azure Monitor** endpoints
- **Log Analytics** workspace endpoints
- **Azure Resource Manager** (`management.azure.com`)
- **Microsoft Entra ID** (`login.microsoftonline.com`)

### Web Filtering / Proxy

- Allow **Azure service tags** on web filtering and proxy appliances
- Tag-based filtering simplifies management versus individual IP allowlisting

### Proxy Configuration

Proxy is supported via:

- WinHTTP (Windows)
- Machine-level proxy settings
- Explicit proxy configuration
- Authenticated proxy

> **Critical:** Proxy settings must be configured on **both** the Connected Machine Agent **and** the Azure Monitor Agent.

### Connectivity Validation

Before deploying agents, validate outbound connectivity from each server:

```powershell
# Windows - Test Azure Arc endpoint
Test-NetConnection -ComputerName "gbl.his.arc.azure.com" -Port 443

# Windows - Test Azure Monitor endpoint
Test-NetConnection -ComputerName "global.handler.control.monitor.azure.com" -Port 443
```

```bash
# Linux - Test Azure Arc endpoint
curl -v https://gbl.his.arc.azure.com:443

# Linux - Test Azure Monitor endpoint
curl -v https://global.handler.control.monitor.azure.com:443
```

---

## Server Requirements

### Supported Operating Systems

**Windows Server:**
- Windows Server 2012 R2
- Windows Server 2016
- Windows Server 2019
- Windows Server 2022
- Windows Server 2025

**Linux Distributions:**
- Red Hat Enterprise Linux (RHEL)
- Ubuntu
- SUSE Linux Enterprise Server
- Oracle Linux
- Debian

### Agents Required

Three agents must be deployed on every monitored server:

| Agent | Extension Name (Windows) | Extension Name (Linux) | Purpose |
|-------|--------------------------|------------------------|---------|
| Connected Machine Agent | — (installed directly) | — (installed directly) | Registers server with Azure Arc |
| Azure Monitor Agent | `AzureMonitorWindowsAgent` | `AzureMonitorLinuxAgent` | Collects performance telemetry |
| Dependency Agent | `DependencyAgentWindows` | `DependencyAgentLinux` | Captures process-level network dependencies |

### Identity Requirements

- **System-assigned managed identity** must be enabled on all Arc-enabled servers
- Used for secure, passwordless authentication to Azure services
- Automatically managed by Azure Arc upon onboarding

---

## Organisational Prerequisites

- **Server owner engagement** — All server owners must be contacted and informed before agent deployment begins
- **Deployment ownership** — Cloud & Infrastructure team owns the phased rollout
- **Cost management ownership** — An operational owner must be assigned for ongoing workspace cost and daily ingestion cap management
- **Management tooling** — Existing deployment tooling (e.g., SCCM, Ansible, GPO) should be available for agent deployment at scale

---

## Pre-Deployment Checklist

- [ ] Azure subscription identified and accessible
- [ ] All five resource providers registered
- [ ] RBAC roles assigned to deploying identity
- [ ] Target resource group created
- [ ] Outbound HTTPS 443 confirmed from all target servers
- [ ] Proxy settings documented (if applicable)
- [ ] Azure service tags allowed on web filters/proxy
- [ ] OS inventory completed — all servers on supported OS versions
- [ ] Server owners notified and engaged
- [ ] Deployment tooling confirmed (SCCM / Ansible / GPO / manual)
- [ ] Cost management owner assigned
- [ ] Daily ingestion cap strategy agreed

---

## Related Documents

- [README](README.md)
- [Deployment Guide](DEPLOYMENT.md)
