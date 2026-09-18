
# Deployment Guide — Azure Arc Server Insights & Dependency Mapping

This guide covers the five-phase deployment of Azure Arc Server Insights and Dependency Mapping. Ensure all [prerequisites](PREREQUISITES.md) are met before proceeding.

---

## Deployment Phases Overview

| Phase | Name | Description |
|-------|------|-------------|
| 1 | Core Resources | Provision Azure infrastructure |
| 2 | Arc Onboarding | Register servers with Azure Arc |
| 3 | Monitoring | Deploy AMA and configure data collection |
| 4 | Dependency Mapping | Deploy Dependency Agent and enable service maps |
| 5 | Validation | Confirm end-to-end data flow and operational handoff |

---

## Phase 1: Core Resources

### Objective
Create the foundational Azure resources required for monitoring and dependency mapping.

### Steps

1. **Create the Log Analytics Workspace**

```bash
az monitor log-analytics workspace create \
  --resource-group <resource-group> \
  --workspace-name <workspace-name> \
  --location uksouth \
  --retention-time 90
```

2. **Register Resource Providers**

```bash
az provider register --namespace Microsoft.HybridCompute
az provider register --namespace Microsoft.GuestConfiguration
az provider register --namespace Microsoft.Insights
az provider register --namespace Microsoft.Monitor
az provider register --namespace Microsoft.OperationalInsights
```

3. **Assign RBAC Roles**

Assign the following roles to the deploying identity on the target resource group:

- Azure Connected Machine Resource Administrator
- Monitoring Contributor
- Log Analytics Contributor

4. **Create Data Collection Rules**

Configure DCRs with the following performance counters at **60-second intervals**:

- CPU utilisation
- Memory utilisation
- Disk I/O and space
- Network traffic

---

## Phase 2: Arc Onboarding

### Objective
Install the Connected Machine Agent and register all target servers with Azure Arc.

### Steps

1. **Generate the onboarding script** from the Azure Portal (Azure Arc > Servers > Add)

2. **Deploy the Connected Machine Agent** to all target servers using your preferred method:
   - Azure Portal (single server)
   - Service principal script (bulk)
   - SCCM / Ansible / GPO (enterprise scale)

3. **Verify registration** — All servers should show as **"Connected"** in the Azure Arc blade

4. **Enable system-assigned managed identity** on each Arc-enabled server (enabled by default on onboarding)

### Validation

```kql
Heartbeat
| where TimeGenerated > ago(5m)
| summarize LastHeartbeat = max(TimeGenerated) by Computer
| order by Computer asc
```

All onboarded servers should return a recent heartbeat.

---

## Phase 3: Monitoring

### Objective
Deploy the Azure Monitor Agent and associate Data Collection Rules to begin performance data ingestion.

### Steps

1. **Deploy Azure Monitor Agent** as an extension on all Arc-enabled servers:

```bash
# Windows
az connectedmachine extension create \
  --machine-name <server-name> \
  --resource-group <resource-group> \
  --name AzureMonitorWindowsAgent \
  --type AzureMonitorWindowsAgent \
  --publisher Microsoft.Azure.Monitor

# Linux
az connectedmachine extension create \
  --machine-name <server-name> \
  --resource-group <resource-group> \
  --name AzureMonitorLinuxAgent \
  --type AzureMonitorLinuxAgent \
  --publisher Microsoft.Azure.Monitor
```

2. **Associate Data Collection Rules** to every monitored server

3. **Configure proxy settings** on the AMA if required (must match Connected Machine Agent proxy config)

### Validation

Confirm performance data is flowing:

```kql
Perf
| where TimeGenerated > ago(15m)
| summarize count() by Computer, ObjectName
| order by Computer asc
```

All servers should show CPU, memory, disk, and network counters.

---

## Phase 4: Dependency Mapping

### Objective
Deploy the Dependency Agent to enable process-level dependency discovery and service maps.

### Steps

1. **Deploy the Dependency Agent** as an extension on all Arc-enabled servers:

```bash
# Windows
az connectedmachine extension create \
  --machine-name <server-name> \
  --resource-group <resource-group> \
  --name DependencyAgentWindows \
  --type DependencyAgentWindows \
  --publisher Microsoft.Azure.Monitoring.DependencyAgent

# Linux
az connectedmachine extension create \
  --machine-name <server-name> \
  --resource-group <resource-group> \
  --name DependencyAgentLinux \
  --type DependencyAgentLinux \
  --publisher Microsoft.Azure.Monitoring.DependencyAgent
```

2. **Enable VM Insights** on the Arc-enabled servers via the Azure Portal or CLI

3. **Verify agent provisioning state** — All Dependency Agents must show **"Succeeded"**

### Validation

Confirm dependency data is being collected:

```kql
VMConnection
| where TimeGenerated > ago(1h)
| summarize ConnectionCount = count() by Computer, Direction
| order by Computer asc
```

> **Important:** Dependency maps will be incomplete if the Dependency Agent is missing on **any** server in a communication flow. Ensure full coverage.

---

## Phase 5: Validation

### Objective
Confirm end-to-end data flow, validate dashboards, and complete operational handoff.

### Validation Checklist

- [ ] All target servers show **"Connected"** in Azure Arc
- [ ] AMA provisioning state = **Succeeded** on all servers
- [ ] Dependency Agent provisioning state = **Succeeded** on all servers
- [ ] DCR associated to every monitored server
- [ ] Performance counters collecting at 60-second intervals
- [ ] Heartbeat data confirmed via KQL
- [ ] VMConnection data confirmed via KQL
- [ ] VM Insights health dashboards visible and populated
- [ ] Performance metrics visible to operations team
- [ ] Dependency maps rendering correctly with expected topology
- [ ] RBAC assignments validated for operations team

### KQL Validation Queries

**Heartbeat Confirmation:**

```kql
Heartbeat
| where TimeGenerated > ago(5m)
| summarize LastHeartbeat = max(TimeGenerated) by Computer
```

**VMConnection Data:**

```kql
VMConnection
| where TimeGenerated > ago(1h)
| summarize ConnectionCount = count() by Computer, Direction
```

**Agent Extension Status:**

```kql
arg("").resources
| where type == "microsoft.hybridcompute/machines/extensions"
| extend extensionName = name, provisioningState = properties.provisioningState
| project extensionName, provisioningState
```

### Operational Handoff

Once all validation checks pass:

1. **Document** the deployed estate and any exceptions (e.g., servers on unsupported OS)
2. **Hand off** to the operations team with access to VM Insights dashboards
3. **Establish** ongoing cost monitoring for Log Analytics ingestion
4. **Set** daily ingestion cap alerts to prevent cost overruns
5. **Schedule** regular reviews of dependency maps for accuracy

---

## Troubleshooting

### Common Issues

| Issue | Likely Cause | Resolution |
|-------|-------------|------------|
| Server not appearing in Azure Arc | Connected Machine Agent not installed or connectivity failure | Verify agent installation; test outbound HTTPS 443 connectivity |
| Agent provisioning state "Failed" | Proxy misconfiguration or unsupported OS | Check proxy settings on both CMA and AMA; verify OS is supported |
| No performance data in VM Insights | DCR not associated or AMA not reporting | Verify DCR association; check AMA extension status |
| Incomplete dependency maps | Dependency Agent missing on one or more servers | Deploy Dependency Agent to all servers in the communication flow |
| High Log Analytics costs | Dependency mapping generates significant data volume | Implement daily ingestion cap; review retention policy; consider sampling |
| Heartbeat missing for a server | Agent connectivity lost | Check network/proxy; restart the Connected Machine Agent service |

### Useful Commands

**Check agent status (Windows):**

```powershell
& "$env:ProgramFiles\AzureConnectedMachineAgent\azcmagent.exe" show
```

**Check agent status (Linux):**

```bash
azcmagent show
```

**Restart Connected Machine Agent (Windows):**

```powershell
Restart-Service himds
```

**Restart Connected Machine Agent (Linux):**

```bash
sudo systemctl restart himdsd
```

---

## Related Documents

- [README](README.md)
- [Prerequisites](PREREQUISITES.md)
