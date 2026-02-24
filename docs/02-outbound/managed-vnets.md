# Managed Virtual Networks

> **Source:** [Overview of managed virtual networks in Microsoft Fabric](https://learn.microsoft.com/en-us/fabric/security/security-managed-vnets-fabric-overview)
> Last reviewed: 2026-02-24

## What They Are

Managed VNets are Azure Virtual Networks that Fabric creates and manages automatically — one per Fabric workspace. They provide **network isolation for Fabric Spark workloads**. Spark compute clusters run in a dedicated VNet rather than the shared Fabric VNet used by all tenants.

You do not create or manage these VNets in Azure. They don't appear in the Azure portal. Fabric manages them entirely.

## Value Proposition

| Benefit | Details |
|---------|---------|
| Network isolation | Spark clusters (which run arbitrary user code) run in a dedicated network |
| No subnet sizing | Fabric scales the VNet automatically; you don't pre-size subnets |
| Secure outbound access | Enables managed private endpoints for connections to firewall-protected data sources |
| Private link support | Enables Private Link support for Data Engineering and Data Science items using Spark |

## How a Managed VNet Gets Provisioned

A managed VNet is created for a workspace automatically when either of these occurs:

1. **A managed private endpoint is created** in the workspace (by the workspace admin via workspace settings)
2. **The tenant Private Link setting is enabled AND the first Spark job runs** in the workspace (first notebook execution, Spark job definition run, or Lakehouse operation like Load to Table, Optimize, or Vacuum)

## Spark Cold Start Impact

Once a managed VNet is provisioned, **starter pools (pre-warmed shared clusters) are disabled** for that workspace. Spark jobs then run on custom pools created on-demand within the dedicated managed VNet.

This means:
- **Session startup time increases to ~3–5 minutes** (vs. near-instant with starter pools)
- This applies to every new Spark session, not just the first one
- The trade-off is isolated compute with secure outbound connectivity

> **Enterprise planning note**: If your workloads are latency-sensitive (interactive notebook sessions, real-time debugging workflows), factor in this cold start overhead when evaluating whether to use managed private endpoints or tenant-level private links. This is a significant operational change for data engineering teams.

## Regional Availability

Managed VNets are **not supported** in:
- Switzerland West
- West Central US

In these regions:
- **Outbound**: Managed Private Endpoints cannot be created
- **Inbound**: If workspaces are attached to Fabric capacities in these regions within tenants where Private Link is enabled, Data Engineering jobs (notebooks, Spark job definitions, lakehouse operations) will fail

## Workspace Migration Restriction

Once a managed VNet is allocated to a workspace, **workspace migration across capacities in different regions is not supported**. Plan capacity region assignments before enabling features that trigger managed VNet creation.

## Relationship to Other Features

| Feature | Relationship |
|---------|-------------|
| Managed Private Endpoints | Require a managed VNet; their creation triggers VNet provisioning |
| Tenant Private Link | First Spark job after enabling triggers VNet provisioning |
| Workspace Private Link | Also uses managed VNet for Spark workloads |
| Outbound Access Protection | Requires Fabric Data Engineering (managed VNet) in the region |
