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

## Biggest Call-Outs

These are the items most likely to derail the project or require early decisions. They should be surfaced in Discovery, not after design has started.

### 1. ZScaler Must Be Understood Before Any Network Design Is Finalised

ZScaler sits between all users and Fabric. If it intercepts DNS queries or performs SSL inspection on Fabric traffic, Private Link will silently fail — not with an obvious error, but with connections that appear to work on the public internet but break on the private path. The network team must confirm:
- Whether Fabric FQDNs are bypassed in ZScaler
- Whether SSL inspection is disabled for Fabric traffic
- What the egress IP(s) are — these are what Fabric sees as the source IP, not corporate IPs

This affects every network security option: Private Link (DNS), IP Firewall (source IP), and outbound controls. **It is the first thing to resolve.**

### 2. Sybase ASE Locks In OPDG — Which Constrains the Private Link Choice

There is no native Fabric private endpoint path for Sybase ASE. OPDG with an ODBC driver is the only supported ingestion mechanism. OPDG is:
- Incompatible with tenant-level Private Link + Block Public Internet Access (hard failure)
- Officially unsupported with tenant-level Private Link even without Block Public Internet Access
- Fully compatible with workspace-level Private Link

This single data source effectively decides the Private Link architecture. If Sybase ASE is in scope, tenant-level Private Link with full public internet blocking is off the table. **The network isolation decision cannot be made without knowing whether Sybase is staying.**

### 3. The Network Isolation Level Has Not Been Decided — But It Drives Everything

Every connectivity choice downstream depends on this. The options are meaningfully different:

| Option | OPDG Works | Capacity Metrics App | Dev Experience | Operational Overhead |
|--------|-----------|----------------------|----------------|----------------------|
| No isolation | ✓ | ✓ | Simple | Low |
| Workspace IP Firewall only | ✓ (if ZScaler handled) | ✓ | Simple | Low |
| Workspace-level Private Link | ✓ | ✓ | DNS config required | Medium |
| Tenant-level PL (no block) | Possibly | ✓ | DNS config required | Medium |
| Tenant-level PL + Block Public | ✗ | ✗ | Fully isolated | High |

This decision needs to come from Cyber Security Policy, not be inferred from general security posture.

### 4. Fabric Capacity Metrics App Is Broken Under Full Private Link Isolation

This is a Microsoft-documented limitation with no workaround. If the client enables tenant-level Private Link with Block Public Internet Access — the strictest isolation option — the Capacity Metrics App stops working entirely. Operations teams depend on this for capacity management in PROD. The client needs to consciously accept this trade-off or choose a less restrictive isolation model.

### 5. Microsoft Purview Information Protection Does Not Support Private Link

If the client has a data sensitivity labelling requirement (common in environments with a Head of Cyber Security Policy), they should know that MIP/sensitivity labels do not work in a Private Link-enabled tenant. The Sensitivity button is greyed out in Power BI Desktop on a private-linked tenant. If sensitivity labels are mandatory, this is a blocker for full Private Link isolation.

### 6. Amazon S3 Has No Supported Private Connectivity Path from Fabric

Fabric cannot route through a VPN to reach S3. This source needs its connectivity model decided before design begins — either it stays as a public endpoint (controlled via S3 bucket policy), or the ingestion is pre-staged outside Fabric (e.g., copy to Azure Blob first). Leaving this unresolved means a gap in the security architecture for one of the data sources.

### 7. Default Semantic Models Block Workspace Private Link Restriction — Deployment Sequencing Matters

If workspace-level Private Link is used and public access needs to be restricted, the workspace must be configured with that restriction **before** any Lakehouse, Warehouse, or Mirrored Database is created. Creating these items first generates default semantic models that are incompatible with the restriction, blocking it from being enabled. This is a deployment sequencing requirement that must be built into the provisioning runbook.

### 8. SKU Selection — Most Security Features Require F SKU

Workspace-level Private Link, Trusted Workspace Access, Customer-Managed Keys, and Outbound Access Protection all require F SKU. Trial capacities and P SKU do not support these features. The client must be on F SKU for the security architecture to work. Confirm this with procurement / licensing before design.

---

## Stakeholder Conversation Guide

### Head of Architecture

Owns the Azure landing zone, network topology, and cross-system integration design.

**Confirm:**
- Does the client have ExpressRoute or a VPN Gateway connecting on-premises to Azure? This determines whether private on-premises connectivity to Azure is feasible and how OPDG traffic is routed.
- What is the Azure VNet structure? Are there existing VNets, subnets, and NSGs we're designing into, or is this greenfield?
- Who owns the Azure Private DNS zones? Private Link requires DNS entries for Fabric FQDNs — these need to be configured in the DNS infrastructure that VPN/ZScaler clients use.
- How is Amazon S3 connected, or planned to be connected, to Azure? Is there an Azure-to-AWS VPN or Transit Gateway arrangement planned?
- Is the Azure SQL control DB going to be provisioned inside an existing VNet or as a new resource? If network-restricted, what's the intended access path for Fabric?
- Is there a shared capacity or per-environment capacity? Workspace-level Private Link scope is per-workspace; tenant-level is per-tenant — the capacity structure affects isolation granularity.
- What is the DEV/UAT/PROD network separation model in Azure? Separate subscriptions, separate VNets, or just separate subnets?
- ZScaler: what traffic does it intercept vs. bypass? Can Fabric FQDNs be added to the bypass list? Who owns that configuration?

---

### Head of Cyber Security Policy

Owns security requirements, compliance mandates, and acceptable risk decisions.

**Confirm:**
- Is there a mandatory network isolation requirement, or is this risk-based? If risk-based, what is the classification of the data being processed (PII, financial, regulated)?
- Is there a specific compliance framework in scope (e.g., ISO 27001, SOC 2, PCI-DSS, GDPR, industry-specific)?
- Is Private Link mandatory, or is network-layer isolation acceptable via other controls (IP Firewall, ZScaler policies, Entra Conditional Access)?
- Is there a policy requiring all data at rest to be encrypted with customer-managed keys? Note: CMK in Fabric does NOT cover Spark temporary storage (shuffle data, spills). If 100% CMK coverage is a hard requirement, Spark-based transformations cannot meet it without an external storage workaround.
- Is there a sensitivity labelling requirement? Microsoft Information Protection / Purview sensitivity labels do not work when Fabric tenant-level Private Link is enabled. Confirm whether this is a hard requirement.
- What is the policy on public-facing endpoints remaining accessible? The Fabric Capacity Metrics App, OPDG registration, and several other features require some public internet access. A policy of "all traffic must be private" is not achievable with all Fabric features.
- Is there a data residency requirement? Fabric stores data in the capacity region. Multi-geo and home region configuration needs to be confirmed.
- What is the policy on third-party SaaS data sources? Genesys PureCloud is a public API. Is there an acceptable use policy or DLP requirement for that connection?
- Is there a requirement for audit logging of data access? Fabric produces audit logs via Microsoft 365 Unified Audit Log. Is that sufficient, or is SIEM integration required?

---

### Head of Cyber Security Operations

Owns ZScaler configuration, security monitoring, incident response, and operational security controls.

**Confirm:**
- ZScaler bypass configuration: Are `*.fabric.microsoft.com`, `*.analysis.windows.net`, `*.pbidedicated.windows.net`, `*.powerquery.microsoft.com`, and related Fabric FQDNs currently bypassed in ZScaler? If not, can they be added, and what is the change process?
- ZScaler SSL inspection: Is SSL inspection enabled for Microsoft 365 / Azure traffic? Fabric connectivity (especially Private Link) can break under SSL inspection. Are certificate pinning exceptions in place?
- What are the ZScaler egress IP addresses? These are what Fabric will see as source IPs. If workspace IP Firewall is used, rules must be based on ZScaler egress IPs, not corporate network IPs.
- Is there a SIEM in place (e.g., Microsoft Sentinel, Splunk)? Microsoft Fabric audit logs are available via the Microsoft 365 Unified Audit Log and can be forwarded to SIEM.
- Is there a vulnerability management process for gateway infrastructure? OPDG servers require patching and monitoring. Who will own this?
- What is the change management process for Azure Private Link approvals? When Fabric creates a managed private endpoint to an Azure resource, the Azure resource owner must approve it in the Azure Portal. Operations needs a workflow for this.
- Is there a requirement for network traffic monitoring (NSG flow logs, Azure Monitor)? If private endpoints are used, traffic patterns change — flow logs should be reviewed.
- Conditional Access: Are there existing CA policies for workload identities or service principals? A policy that applies to all service principals will break Trusted Workspace Access unless Fabric workspace identities are explicitly excluded.

---

### Head of Data

Owns the data strategy, data governance, business unit access, and data product definitions.

**Confirm:**
- What is the data sensitivity classification across layers? Is all data equally sensitive, or are Gold/reporting layer data products less restricted? This affects whether every workspace needs isolation or just selected ones.
- Who are the business unit consumers, and what is their expected access method? (Browser only? Power BI Desktop? Direct SQL/TDS connections to Warehouse/Lakehouse SQL endpoint?) Each method has different Private Link implications.
- Is Microsoft Purview in scope for data governance and cataloguing? Purview integration with Fabric has its own network considerations.
- Is there a requirement for external data sharing (sharing Fabric data with external organisations or partners)? Outbound Access Protection is incompatible with Fabric external data sharing (cross-tenant). This is a mutually exclusive choice.
- Are Deployment Pipelines planned for CI/CD between DEV/UAT/PROD? Workspaces assigned to a deployment pipeline cannot have public inbound access restricted via workspace-level Private Link. These are mutually exclusive — you cannot use both simultaneously.
- What are the SLAs for data availability? Spark cold start (3–5 minutes after managed VNet provisioning) and Private Link propagation delays affect pipeline schedules.
- Is Fabric licensing confirmed as F SKU? Most security features (Private Link, TWA, CMK, Outbound Protection) require F SKU. Trial and P SKU are not supported.
- Will the Fabric Capacity Metrics App be required for capacity management in production? If yes, full Private Link isolation with Block Public Internet Access is not compatible.

---

### Additional Stakeholders to Involve

**Network Team / Network Engineer**
- VNet design, subnet sizing, NSG rules, route tables
- Private DNS zone ownership and configuration for Fabric FQDNs
- ZScaler bypass rules change process
- ExpressRoute / VPN Gateway topology

**Azure Platform / Cloud Team**
- Private endpoint provisioning for Azure resources (Azure SQL, Azure Blob, Key Vault)
- Managed Private Endpoint approval workflow (approvals happen in Azure Portal)
- Azure Key Vault provisioning (for CMK if required)
- Azure subscription and resource group structure for Fabric support infrastructure

**On-Premises Infrastructure / DBA Team**
- OPDG server provisioning: hardware/VM, network access to Sybase ASE and SQL Server, ODBC driver installation
- Sybase ASE ODBC driver availability and version compatibility
- SQL Server firewall rules to allow OPDG server IP
- Ongoing gateway maintenance, patching, and HA clustering

**Procurement / Licensing**
- Fabric SKU confirmation (F SKU required for security features)
- Power BI Premium Per User (PPU) vs. capacity licensing for business unit consumers
- Azure consumption estimate for Private Endpoints, VNet Data Gateway, Key Vault (if CMK)

---

## Preliminary Architecture Notes

> Network security level not yet confirmed. Notes below represent considerations, not decisions.

- **Workspace-level Private Link** is a better fit than tenant-level given OPDG dependency (Sybase ASE) and the need to keep Capacity Metrics App functional.
- **Outbound Access Protection** (when it reaches GA and the workspace artifact restrictions are confirmed) is relevant for the Gold/reporting layer workspaces where data exfiltration risk is highest.
- **Managed Private Endpoints** are the right approach for Spark Notebook connections to Azure SQL (control DB) and Azure Blob Storage.
- **OPDG** is required for Sybase ASE and MS SQL Server on-premises sources. Must be installed on a server with ODBC drivers and network access to both sources.
- **Genesys PureCloud**: Treat as a public endpoint; no private connectivity needed. If outbound protection is enabled, allowlist the API endpoints.
- **ZScaler bypass rules** for Fabric FQDNs must be confirmed before any Private Link testing.
