
# Azure Arc Server Insights & Dependency Mapping

## Overview

Azure Arc Server Insights & Dependency Mapping provides hybrid server observability across on-premises, Azure, and Arc-managed SQL estates. Dependency maps de-risk patching, migration, and decommissioning by exposing undocumented server and application connections.

**Owner:** Digital & ICT
**Version:** v1.0

---

## Business Case

Modern hybrid estates contain hundreds of servers with undocumented dependencies between applications, databases, and services. Without visibility into these connections, routine operations like patching, migration, and decommissioning carry significant risk.

This solution provides:

- **Performance monitoring** — CPU, memory, disk, and network utilisation across all servers
- **Dependency mapping** — Automatic discovery of inbound/outbound connections, TCP ports, process-to-process communication, and SQL connectivity
- **Operational confidence** — Evidence-based decision making for change, migration, and retirement activities

---

## Use Cases

- **De-risk Patching** — Understand application dependencies before maintenance windows
- **Migration Planning** — Map dependencies for lift-and-shift or modernisation
- **Decommissioning** — Safely retire servers by confirming no active dependencies
- **Incident Response** — Quickly identify impacted services during outages
- **Capacity Planning** — Historical performance trending for rightsizing
- **Security** — Discover unexpected connections and undocumented communication paths

---

## Architecture

### Data Flow

```
On-Prem Server
  ├─ Connected Machine Agent ──┐
  ├─ Azure Monitor Agent ───────┤
  └─ Dependency Agent ──────────┤
                                │
                                ▼
                    Data Collection Rule
                                │
                                ▼
                  Log Analytics Workspace
                                │
                                ▼
                          VM Insights
                                │
                                ▼
                       Dependency Maps
```

### Network Flow

```
On-Prem Server ──► Corporate Firewall/Proxy ──► Internet ──► Microsoft Entra ID
                                                                     │
                                                          Azure Resource Manager
                                                                     │
                                                            Azure Arc Service
                                                                     │
                                                             Azure Monitor
                                                                     │
                                                      Log Analytics Workspace
                                                                     │
                                                    VM Insights / Dependency Maps
```

All communication is **outbound HTTPS 443 only** — no inbound firewall rules are required.

---

## VM Insights Capabilities

### Performance Monitoring
- CPU, memory, disk, and network utilisation
- Process and service inventory
- Performance history and trending
- Real-time and historical views

### Dependency Mapping
- Inbound and outbound connections
- TCP port discovery
- Process-to-process communication flows
- SQL Server connectivity discovery
- External service dependencies

### Example Dependency Discovery

```
WEB01 ─┬─ 443 ──► APP01
       ├─ 1433 ─► SQL01
       └─ 636 ──► DC01 (LDAPS)

APP01 ──── 1433 ─► SQL01

SQL01 ──── 5022 ─► SQL02 (SQL Mirroring/AG)
```

---

## Core Azure Resources

| Resource | Configuration |
|----------|---------------|
| **Azure Arc** | Hybrid server registration and management |
| **Log Analytics Workspace** | UK South, 90-day retention |
| **Data Collection Rules** | Associated to every server, 60s collection interval |
| **Azure Monitor** | Telemetry ingestion and alerting |
| **VM Insights** | Performance dashboards and dependency maps |

---

## RBAC Roles Required

| Role | Purpose |
|------|---------|
| Azure Connected Machine Resource Administrator | Manage Arc-enabled server resources |
| Monitoring Contributor | Configure monitoring and insights |
| Log Analytics Contributor | Manage workspace and query data |

---

## Risks & Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| Proxy/web-filter misconfiguration | Most common onboarding failure | Pre-validate proxy settings; test connectivity to Azure endpoints before agent deployment |
| Incomplete dependency agent coverage | Gaps in dependency maps | Phased validation; automated deployment checks |
| Log Analytics cost overrun | Dependency data generates significantly more ingestion | Implement daily ingestion cap; monitor workspace costs; right-size retention |
| Legacy OS agent deployment | Agent install failures on EOL systems | OS inventory before deployment; document exceptions; plan upgrades |
| Unreliable outbound connectivity | Agents cannot report telemetry | Pre-flight connectivity tests; document offline servers |

---

## Key Assumptions

1. Servers have reliable outbound internet or proxy connectivity
2. Agents can be deployed via existing management tooling (SCCM, Ansible, GPO)
3. A single UK South workspace is sufficient for the estate
4. Deployment follows five phases to minimise risk
5. Server owners are engaged before agent deployment
6. Workspace cost and daily ingestion cap require ongoing operational ownership

---

## Related Documents

- [Prerequisites Guide](PREREQUISITES.md)
- [Deployment Guide](DEPLOYMENT.md)
