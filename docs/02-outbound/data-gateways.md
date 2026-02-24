# Data Gateways in Microsoft Fabric

> **Sources:**
> - [On-premises data gateway](https://learn.microsoft.com/en-us/power-bi/connect-data/service-gateway-onprem)
> - [VNet data gateway](https://learn.microsoft.com/en-us/data-integration/vnet/overview)
> - [Microsoft Fabric end-to-end security scenario](https://learn.microsoft.com/en-us/fabric/security/security-scenario)
> - [Blog: Mission-Critical Data Integration — What's New in Fabric Data Factory](https://blog.fabric.microsoft.com/en-us/blog/mission-critical-data-integration-whats-new-in-fabric-data-factory/) — Oct 2, 2025
> - [Blog: Fabric October 2025 Feature Summary](https://blog.fabric.microsoft.com/en-us/blog/fabric-october-2025feature-summary) — Oct 29, 2025
> Last reviewed: 2026-02-24

## Feature Timeline

| Date | Event |
|------|-------|
| Oct 2025 | **VNet data gateway for Pipelines and Copy Jobs** goes **GA** — previously only supported Dataflows Gen2 and semantic model refresh |
| Oct 2025 | **VNet data gateway lifecycle controls** (start, stop, restart) go **GA** |

> **Impact of VNet gateway GA for Pipelines**: Prior to Oct 2025, Data Factory Pipelines and Copy Jobs could only connect to private data sources via the on-premises data gateway or by using trusted workspace access. The VNet gateway GA added a fully managed, Azure-native option without requiring on-premises infrastructure.

## Overview

Gateways are the bridge between Fabric cloud services and data that is either on-premises or protected within a VNet. There are two types:

| Gateway Type | Use Case | Infrastructure Required |
|-------------|---------|------------------------|
| On-Premises Data Gateway (OPDG) | On-premises data sources or non-Azure firewall-protected sources | Server in your network |
| VNet Data Gateway | Azure data sources within a VNet (private endpoints, firewalls) | Azure VNet (Fabric manages containers) |

## On-Premises Data Gateway (OPDG)

The OPDG is installed on a server within your network. It acts as a bridge: Fabric connects outbound through the gateway, and the gateway connects to your data sources within the network. No inbound ports need to be opened to the server.

**Compatible with:**
- Dataflows Gen2
- Data Factory Pipelines (copy activity, pipeline activities)
- Power BI semantic model refresh

**Not compatible with:**
- Tenant-level Private Link — **OPDG fails to register when tenant Private Link is enabled**. This is a hard limitation. If your tenant uses Private Link, you must use VNet data gateways.

**Clustering:** Gateway servers can be clustered for high availability and load balancing.

## VNet Data Gateway

The VNet data gateway allows Fabric to inject compute containers into your Azure VNet on demand. You provide the VNet and subnet; Fabric manages the gateway containers. This eliminates the need to provision and maintain on-premises gateway infrastructure for Azure-hosted data.

**Compatible with:**
- Dataflows Gen2
- Power BI semantic model refresh
- Data Factory Pipelines and Copy Jobs (GA as of Oct 2025)
- Works alongside tenant-level Private Link (unlike OPDG)

**Common use case:** Connecting to Azure SQL, Azure Synapse, or other Azure PaaS services behind private endpoints or VNet firewalls.

**Supported via workspace private links:** Yes (as shown in the feature availability matrix).

## Gateway Selection Guide

| Scenario | Recommended Gateway |
|---------|---------------------|
| On-premises SQL Server, Oracle, SAP, file shares | On-Premises Data Gateway |
| Azure SQL Database behind private endpoint (no infra to manage) | VNet Data Gateway |
| Azure SQL MI behind private endpoint | VNet Data Gateway |
| Dataflows Gen2 with Azure data sources in VNet | VNet Data Gateway |
| Dataflows Gen2 with on-premises data sources | On-Premises Data Gateway |
| Pipelines / Copy Jobs connecting to Azure data sources in VNet | VNet Data Gateway (GA Oct 2025) |
| Tenant Private Link enabled, need to connect to on-premises | VNet Data Gateway (OPDG not supported) |
| Spark notebooks connecting to Azure data sources | Managed Private Endpoints (not gateway) |

## Limitations and Considerations

- **OPDG + Tenant Private Link**: These are mutually exclusive. OPDG does not work when tenant Private Link is enabled.
- **VNet Data Gateway download diagnostics**: Does not work with Private Links.
- **OPDG for non-Power BI apps (PowerApps, Logic Apps)**: On-premises gateway not supported when Fabric tenant Private Link is enabled; use VNet data gateway.
- **Dataflow Gen1**: Does not support outbound access protection — not an option if strict outbound control is needed.

## Common Misconception: "Pipelines Can't Use Private Endpoints"

> **Community source:** r/MicrosoftFabric (Nov 2025) — *"As far as I'm aware, Pipelines can't use Private Endpoints, so you need a Gateway (vnet or install)."*

This was **accurate before October 2025** but is now outdated. The distinction to understand:

- **Managed Private Endpoints (MPEs)**: Pipelines still cannot use MPEs — that mechanism is exclusive to Data Engineering (Spark) workloads (Notebooks, Spark Job Definitions, Lakehouses) and Eventstream.
- **Connecting to resources behind private endpoints**: Pipelines CAN do this via VNet Data Gateway, which went GA for Pipelines and Copy Jobs in October 2025.

The commenter's pattern — using Notebooks for everything so they can use MPEs — remains valid. But "Pipelines have no private endpoint option" is no longer accurate. Community content predating Oct 2025 making this claim reflects the pre-GA state.
