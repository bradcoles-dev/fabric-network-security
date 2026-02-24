# Microsoft Fabric Network Security Reference Guide

A deep-dive reference guide for Microsoft Fabric network security, targeted at enterprise security teams evaluating or deploying Fabric in environments with strict network controls.

## Purpose

This guide goes beyond summarizing Microsoft documentation. It aims to document:

- What Fabric **actually supports** vs. what the docs imply
- Known gaps, limitations, and real-world gotchas
- Workarounds and compensating controls where native capabilities fall short
- Honest assessments of feature maturity (GA vs. Preview vs. not supported)

## Sources

Content is built from multiple layers:

1. **Microsoft official documentation** — canonical reference for capabilities and configuration
2. **Microsoft official blogs** — often contain implementation details not in the main docs
3. **Community findings** — real-world limitations and undocumented behaviors

Each document tracks its sources.

## Structure

```
docs/
├── 00-overview/          # What Fabric security is and how it works
├── 01-inbound/           # Controlling traffic INTO Fabric
├── 02-outbound/          # Controlling traffic FROM Fabric to data sources
├── 03-identity/          # Workspace identity, cross-tenant, guest access
├── 04-data-security/     # OneLake, RLS, OLS, encryption
├── 05-workload-security/ # Per-workload network security specifics
├── 06-governance/        # Purview, DLP, sensitivity labels
├── 07-monitoring/        # Audit logs, activity monitoring
└── 08-scenarios/         # End-to-end reference architectures
```

## Quick Reference: Inbound Options

| Option | Scope | Blocks Public Internet | Requires Azure VNet | Maturity |
|--------|-------|----------------------|---------------------|---------|
| Conditional Access | Tenant | No (identity-based) | No | GA |
| Tenant Private Link | Tenant | Yes (optional) | Yes | GA |
| Workspace Private Link | Workspace | Yes (optional) | Yes | GA |
| Workspace IP Firewall | Workspace | No (allowlist only) | No | GA |

## Quick Reference: Outbound Options

| Option | Workloads | Target | Maturity |
|--------|-----------|--------|---------|
| Managed Private Endpoints | Spark, Eventstream | Azure PaaS resources | GA |
| Trusted Workspace Access | Shortcuts, Pipelines, Semantic Models | ADLS Gen2 | GA |
| On-Premises Data Gateway | Data Factory, Power BI | On-premises / any | GA |
| VNet Data Gateway | Data Factory, Power BI | Azure VNet resources | GA |
| Service Tags | NSG / Firewall rules | Azure networking | GA |
| Outbound Access Protection | Data Engineering, Data Factory | All outbound | GA |

## Feature Availability at a Glance

See [feature-availability.md](docs/00-overview/feature-availability.md) for a full matrix of which security features are supported per Fabric item type (Workspace Private Links, Customer Managed Keys, Outbound Access Protection).
