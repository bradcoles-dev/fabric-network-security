# Client Project

## Architecture Overview

### Data Sources
| Source | Type | Connectivity Notes |
|--------|------|-------------------|
| MS SQL Server | On-premises | Standard; OPDG or VNet gateway |
| Sybase ASE | On-premises | ODBC driver required; OPDG only |
| Azure Blob Storage | Cloud (Azure) | TWA or MPE |
| Amazon S3 | Cloud (AWS) | VPN assumed; not yet built |
| Genesys PureCloud | SaaS/Public API | Public endpoints |

### Fabric Components
| Component | Layer |
|-----------|-------|
| Data Pipelines + Copy Activity | Ingestion / Orchestration |
| Lakehouse (Landed, Bronze, Silver, Gold) | Storage |
| Spark Notebooks | Transformation |
| Power BI Semantic Model | Reporting |
| Power BI Report / App | Reporting |

### Supporting Infrastructure
| Component | Notes |
|-----------|-------|
| Azure SQL (control DB) | Outside Fabric capacity; metadata-driven ELT + audit; not yet provisioned |
| Fabric Capacity Metrics App | Capacity monitoring |
| Entra Groups | Access control |

### Environments
Three workspaces: **DEV**, **UAT**, **PROD**

### Access Patterns
| User Group | Access Method | Network |
|------------|--------------|---------|
| Central data team | Fabric web UI, Power BI Desktop, VS Code | Corp VPN + ZScaler |
| Business units | Silver Lakehouse, Gold, Semantic Model, Reports | Corp VPN + ZScaler |

---

## Open Questions

| # | Question | Why It Matters |
|---|----------|----------------|
| 1 | Does the client have ExpressRoute or VPN Gateway to Azure? | Determines feasibility of private on-premises connectivity to Azure; affects OPDG and VNet gateway routing |
| 2 | What level of network isolation is required? | Drives the entire architecture — tenant PL vs. workspace PL vs. outbound controls only vs. nothing |
| 3 | How is Amazon S3 connected? (VPN assumed) | If via VPN to Azure, needs a gateway with VPN access; if public, outbound allowlisting only |
| 4 | Are Genesys PureCloud APIs public-facing? | Assumed yes (SaaS); if outbound protection enabled, these endpoints need allowlisting |
| 5 | ZScaler configuration — does it bypass Fabric FQDNs, or does it intercept? | See risks below |
| 6 | Will the Azure SQL control DB be network-restricted? | Determines connectivity approach from Pipelines and Notebooks |
| 7 | Is there a separate Fabric capacity per environment, or one shared capacity? | Affects Private Link isolation options |

---

## Key Risks and Tensions

### 1. OPDG is Required for Sybase ASE — Conflicts with Tenant-Level Private Link

Sybase ASE requires OPDG with an ODBC driver installed on the gateway machine. There is no Fabric-native private endpoint path for Sybase.

- **Tenant-level PL + Block Public Internet Access**: OPDG registration fails. Hard blocker.
- **Tenant-level PL (without Block Public Internet Access)**: OPDG *may* work per MS employee guidance (Oct 2025), but is officially unsupported.
- **Workspace-level PL**: OPDG is not affected — public internet access remains available for non-restricted workspaces. This is Microsoft's stated recommendation when OPDG must coexist with Private Link.

**Implication**: If any form of Private Link is required, workspace-level is strongly preferred over tenant-level to preserve OPDG functionality.

### 2. ZScaler in the Traffic Path

ZScaler is a cloud proxy/ZTNA solution sitting between users and the internet. Potential issues:

- **IP firewall rules**: If workspace IP firewall is used, the source IP seen by Fabric may be a ZScaler egress IP, not the user's corporate IP. Rules based on corporate IP ranges will not work as expected.
- **DNS interception**: ZScaler may intercept DNS resolution. Private Link relies on DNS resolving Fabric FQDNs to private IPs — ZScaler must be configured to bypass or forward Fabric DNS queries correctly, or private endpoint resolution will break.
- **SSL inspection**: ZScaler SSL inspection can interfere with certificate pinning and mutual TLS.
- **Bypass rules needed**: Fabric FQDNs (`*.fabric.microsoft.com`, `*.analysis.windows.net`, `*.pbidedicated.windows.net`, etc.) will likely need to be added to ZScaler bypass/split-tunnel rules.

**This needs to be resolved early** — ZScaler misconfiguration can silently break Private Link without obvious error messages.

### 3. Amazon S3 — No Native Private Connectivity from Fabric

If S3 is accessed via a VPN (e.g., Azure-to-AWS VPN or on-premises-to-AWS VPN), Fabric has no native path to reach it through that VPN. Options:
- Treat S3 as a public endpoint + S3 bucket policy for access control (simplest)
- Route via OPDG or a gateway machine that has access to the VPN path
- Use a landing zone pattern: copy S3 data to Azure Blob first (via a separate process), then ingest into Fabric

### 4. Fabric Capacity Metrics App Does Not Support Private Link

If tenant-level Private Link with Block Public Internet Access is enabled, the Capacity Metrics App stops working. If the client requires this app for capacity management (likely for PROD), it is a known gap that has no workaround other than not enabling Block Public Internet Access.

### 5. Azure SQL Control DB — Connection from Both Pipelines and Notebooks

The control DB will be accessed by both Fabric Data Pipelines (orchestration) and potentially Spark Notebooks (transformation logic). These workloads use different connectivity mechanisms:
- **Pipelines**: VNet Data Gateway (if DB is network-restricted) — GA Oct 2025
- **Notebooks**: Managed Private Endpoint to Azure SQL

If the DB is network-restricted, both mechanisms need to be in place. If provisioned as public (with firewall rules), this is simpler but reduces isolation.

### 6. DEV/UAT/PROD as Workspaces — Workspace-Level PL Operational Overhead

With workspace-level Private Link, each workspace gets its own private link service and workspace-specific FQDN. For three environments this means:
- 3 private link services
- 3+ private endpoints (one per environment per VNet that needs access)
- 3 sets of DNS entries
- Developers using Power BI Desktop or VS Code locally need DNS resolution for each workspace FQDN

Manageable, but needs to be planned into the Azure networking design upfront.

---

## Preliminary Architecture Notes

> Network security level not yet confirmed. Notes below represent considerations, not decisions.

- **Workspace-level Private Link** is a better fit than tenant-level given OPDG dependency (Sybase ASE) and the need to keep Capacity Metrics App functional.
- **Outbound Access Protection** (when it reaches GA and the workspace artifact restrictions are confirmed) is relevant for the Gold/reporting layer workspaces where data exfiltration risk is highest.
- **Managed Private Endpoints** are the right approach for Spark Notebook connections to Azure SQL (control DB) and Azure Blob Storage.
- **OPDG** is required for Sybase ASE and MS SQL Server on-premises sources. Must be installed on a server with ODBC drivers and network access to both sources.
- **Genesys PureCloud**: Treat as a public endpoint; no private connectivity needed. If outbound protection is enabled, allowlist the API endpoints.
- **ZScaler bypass rules** for Fabric FQDNs must be confirmed before any Private Link testing.
