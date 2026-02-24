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

| # | Question | Why It Matters | Status |
|---|----------|----------------|--------|
| 1 | Does the client have ExpressRoute or VPN Gateway to Azure? | Determines feasibility of private on-premises connectivity to Azure; affects OPDG and VNet gateway routing | Open |
| 2 | What level of network isolation is required? | Drives the entire architecture — tenant PL vs. workspace PL vs. outbound controls only vs. nothing | Open — being assessed |
| 3 | How is Amazon S3 connected? | VPN assumed but not built; OneLake shortcut over public internet may be the simplest viable path | Open — see Discovery Notes |
| 4 | Are Genesys PureCloud APIs public-facing? | Assumed yes (SaaS); if outbound protection enabled, endpoints need allowlisting | Assumed yes |
| 5 | ZScaler — does it bypass Fabric FQDNs or intercept? | Bypass config likely required; may be straightforward to add | Open — needs network team |
| 6 | Will the Azure SQL control DB be network-restricted? | Determines connectivity approach for Pipelines (VNet gateway if restricted) | Open — not yet provisioned |
| 7 | Separate Fabric capacity per environment? | **Confirmed: yes** — DEV, UAT, PROD each have their own F SKU | Confirmed |
| 8 | How many workspaces total (across all environments)? | 10–20 per environment × 3 envs = potentially 30–60+ workspace PL setups | Open — see Discovery Notes |
| 9 | What is Microsoft Purview being used for — catalog, sensitivity labels, or both? | Sensitivity labels incompatible with Private Link; OneLake Catalog Govern tab also unavailable | Open — see Discovery Notes |

---

## Discovery Notes

Responses and analysis on specific points raised during discovery.

### 1. Separate F SKU per environment confirmed
DEV, UAT, and PROD each have their own F SKU capacity. This is the recommended model — it enables independent capacity management, Private Link configuration per workspace, and separate Capacity Metrics App instances per environment. SKU requirement for all security features is met.

### 2. Tenant-level vs. workspace-level Private Link
Workspace-level Private Link is the right direction given the OPDG dependency (Sybase ASE). Tenant-level with Block Public Internet Access would break OPDG registration entirely. Workspace-level keeps the public internet accessible for OPDG while isolating specific workspaces. This aligns with Microsoft's own recommendation for environments where OPDG must coexist with Private Link.

### 3. ZScaler — likely solvable but must be confirmed
Adding Fabric FQDNs to ZScaler's bypass list is a standard configuration. The concern is not whether it's technically possible but whether the client's Cyber Security Operations team will approve it and how long the change process takes. Key FQDNs to bypass: `*.fabric.microsoft.com`, `*.analysis.windows.net`, `*.pbidedicated.windows.net`, `*.powerquery.microsoft.com`. Also need to confirm ZScaler egress IPs if workspace IP Firewall is being considered.

### 4. Amazon S3 — depends on whether "no public internet" is a hard requirement

OneLake supports native shortcuts to Amazon S3. The shortcut reads data in-place from S3 **over the public internet** — no VPN or Azure-to-AWS private connectivity is used.

**If "no public internet" is not a hard requirement (or S3 is treated as an exception):**
- S3 shortcut is the simplest path — no pipeline, no copying, no AWS-side infrastructure
- S3 bucket policy provides access control on the AWS side
- If Outbound Access Protection is enabled, Amazon S3 must be allowlisted as an endpoint-level rule
- If CMK is in scope: disable shortcut caching — cached data lands in OneLake without CMK coverage

**If "no public internet" is a hard requirement applying to all data paths:**
- OneLake S3 shortcut is not compliant — it uses the public internet
- Viable alternatives:
  1. **Stage S3 to Azure Blob first** via an external process (e.g., AWS DataSync, AWS Lambda + SDK, or a third-party tool), then access Azure Blob from Fabric via private connectivity. This adds complexity and infrastructure outside Fabric.
  2. **Accept S3 as a public exception** — argue that the connection is outbound from Fabric to a known endpoint (S3 bucket with policy), not inbound public access to Fabric. Depends on how the policy is written.
  3. **Do not use S3** — if this source is not yet built, it may be worth questioning whether S3 is the right storage choice given the connectivity constraints.
- There is no native Azure-to-AWS PrivateLink option supported by Fabric. Any private path would require complex Transit Gateway / VPN architecture that is disproportionate for a data ingestion use case.

> **Action**: Confirm with Cyber Security Policy whether "no public internet" applies to outbound Fabric connections (not just inbound), and whether S3 as a public endpoint with bucket policy controls is acceptable.

### 5. Capacity Metrics App — confirmed significant gap
If workspace-level Private Link is used (without tenant-level Block Public Internet Access), the Capacity Metrics App remains functional. This is another reason workspace-level PL is preferable — it preserves the app. The app only breaks under tenant-level PL with Block Public Internet Access enabled, which is not the recommended path here.

### 6. Azure SQL control DB — Pipelines only, not Notebooks
Connectivity is simpler than initially assessed. Only VNet Data Gateway is needed (for Pipelines). No Managed Private Endpoint required. If the DB is provisioned as publicly accessible with firewall rules allowing the VNet Data Gateway's outbound IPs, even the VNet gateway may not be needed. Decision can be deferred until the DB is provisioned and its network configuration is known.

### 7. Workspace-level Private Link at scale (10–20 workspaces)
The setup overhead per workspace is real: private link service (auto-created), private endpoint in customer VNet, DNS entry. For 10–20 workspaces × 3 environments = 30–60 setups minimum.

**Tiered approach worth considering:**
- **Tier 1 (highest sensitivity — e.g., Gold, PROD reporting)**: Workspace-level Private Link + public access restricted
- **Tier 2 (medium sensitivity — e.g., Silver, UAT)**: Workspace IP Firewall only
- **Tier 3 (lower sensitivity — e.g., DEV, Bronze, Landed)**: No network isolation beyond Entra/Conditional Access

This avoids applying the most expensive isolation model universally. The REST API can automate workspace PL setup where it is applied, reducing manual overhead. **Recommend raising the tiered approach with Head of Cyber Security Policy** — isolation requirements may not be uniform across all workspaces.

### 8. Semantic model creation issue (reported in community)
The community report about not being able to create semantic models with Private Link most likely refers to the **default semantic model** problem: when a Lakehouse, Warehouse, or Mirrored Database is created in a workspace, Fabric automatically creates a default semantic model. This default semantic model does not support workspace-level Private Link, which blocks the ability to restrict public access for that workspace.

**Workaround**: Configure workspace public access restriction *before* creating any Lakehouse, Warehouse, or Mirrored Database. Items created after the restriction is in place generate compatible default semantic models. This must be built into the workspace provisioning runbook — it cannot be retrofitted.

### 9. Tenant-level PL for all workspaces + workspace-level PL just for Sybase — does not work
OPDG registers at the **tenant level**, not the workspace level. Whether OPDG can register with Fabric is determined by the tenant-level Private Link settings, specifically whether Block Public Internet Access is enabled. Having workspace-level PL on the ingestion workspace does not help — if tenant-level PL with Block Public Internet Access is enabled, the OPDG server (on-premises) cannot reach Fabric's public registration endpoints regardless of individual workspace settings.

**The viable model is workspace-level PL only (no tenant-level PL).** This keeps Fabric's public endpoints accessible for OPDG registration while allowing individual workspaces to be privately isolated.

### 10. Microsoft Purview in scope — significant implications
Purview usage needs to be clarified. Two distinct use cases with different impact:

**Data catalog / governance (lineage, classification, scanning):**
- Purview scanning Fabric data requires access to Fabric's APIs. If workspace-level PL with public access restricted is enabled, Purview's scanning service needs a path to the workspace — either via a private endpoint or via the Managed Private Endpoint mechanism. This is not well-documented and needs testing.
- OneLake Catalog Govern tab is unavailable when Private Link is activated.

**Sensitivity labels (Microsoft Information Protection / MIP):**
- Sensitivity labels do **not** work when tenant-level Private Link is enabled. The Sensitivity button is greyed out in Power BI Desktop.
- Workspace-level PL does not have this limitation documented — but it needs to be confirmed.
- If sensitivity labelling is mandatory, tenant-level Private Link is not viable.

> **Action**: Confirm with Head of Data and Cyber Security Policy which Purview capabilities are required. Sensitivity labelling + tenant-level Private Link are mutually exclusive.

### 11. DEV/UAT/PROD each have own F SKU — confirmed
Noted above. Enables independent security configuration per environment and meets the SKU requirement for all security features.

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

| Option | OPDG Works | Capacity Metrics App | MIP Sensitivity Labels | Purview Scanning | Dev Experience | Operational Overhead |
|--------|-----------|----------------------|------------------------|-----------------|----------------|----------------------|
| No isolation | ✓ | ✓ | ✓ | ✓ | Simple — public internet | Low |
| Workspace IP Firewall only | ✓ | ✓ | ✓ | ✓ | ZScaler egress IPs must be allowed; traffic still over public internet | Low |
| Workspace-level Private Link | ✓ | ✓ (needs confirmation — see note) | ✓ (unconfirmed) | Unconfirmed | Workspace-specific FQDNs (see below); DNS config via VPN required | Medium |
| Tenant-level PL (no block) | Possibly | ✓ | ✓ | ✓ | Tenant-specific FQDN; DNS config via VPN required | Medium |
| Tenant-level PL + Block Public | ✗ | ✗ | ✗ | Likely broken | Fully isolated; no public internet for any user | High |

**What each option means in practice:**

**No isolation**: Fabric is accessed over the public internet. Authentication and authorisation (Entra ID, workspace roles) are the only controls. Appropriate for non-sensitive environments (DEV).

**Workspace IP Firewall only** *(FQDN = Fully Qualified Domain Name — the specific domain name used to address a resource)*: Restricts inbound access to Fabric based on the source IP address of the request. Traffic still travels over the public internet, but only from approved IP ranges. If ZScaler is in the path, ZScaler's egress IPs are what Fabric sees — not the user's corporate IP. Rules must be based on ZScaler egress IPs. Does not provide network-layer isolation (traffic is still public internet). Simple to configure, no DNS changes.

> **Capacity Metrics App note**: The documented incompatibility with the Capacity Metrics App is specific to **tenant-level** Private Link. With workspace-level PL (no tenant-level PL), public Fabric endpoints remain accessible for the rest of the tenant — the Capacity Metrics App should therefore still function. This needs to be confirmed by testing before it is relied upon in architecture decisions.

**Workspace-level Private Link**: Each workspace gets its own private link service. A private endpoint in the customer's Azure VNet connects to that workspace. DNS must resolve the workspace-specific FQDN (e.g., `{workspaceid}.z{xy}.w.api.fabric.microsoft.com`) to a private IP — this must work through VPN + ZScaler. OPDG and Capacity Metrics App are unaffected (public internet still accessible for the rest of the tenant). MIP sensitivity label compatibility is unconfirmed and needs testing. Recommended model for this client.

**Tenant-level PL (no block)**: Routes Fabric traffic through private endpoints but does not block public internet access. OPDG *may* still work (per MS employee guidance, Oct 2025). Less isolation than workspace-level PL in practice, with similar DNS overhead but applied tenant-wide. Less flexible than workspace-level — cannot do per-workspace isolation.

**Tenant-level PL + Block Public Internet Access**: Full isolation — all public Fabric endpoints blocked. OPDG registration fails (hard blocker given Sybase dependency). Capacity Metrics App stops working. MIP sensitivity labels stop working. Genesys PureCloud and other public API connections from Fabric may be affected. Not viable for this client given the Sybase ASE, Capacity Metrics App, and Purview requirements.

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

> Network security level not yet confirmed. Notes below represent the current working direction, not final decisions.

- **Tenant-level Private Link is not recommended** for this client. OPDG dependency (Sybase ASE) and Purview sensitivity label requirements are both potentially incompatible with tenant-level PL. Workspace-level PL is the right model.
- **Workspace-level Private Link** — apply selectively using a tiered approach rather than universally. Apply to PROD Gold/reporting workspaces at minimum; consider whether lower-sensitivity workspaces need it.
- **Tiered isolation model** (to be validated with Cyber Security Policy):
  - PROD Gold / reporting workspaces → Workspace-level Private Link + public access restricted
  - Remaining PROD workspaces → Workspace IP Firewall or workspace-level PL without public restriction
  - UAT → Workspace IP Firewall
  - DEV → Entra/Conditional Access only
- **Amazon S3** — if public internet is acceptable: OneLake shortcut is the simplest path. If "no public internet" is a hard requirement: S3 must be staged to Azure Blob first via an external process. Confirm policy scope with Cyber Security Policy.
- **OPDG** — required for Sybase ASE (ODBC driver, no alternative). For MS SQL Server: VNet Data Gateway is viable if VPN/ExpressRoute connects the Azure VNet to on-premises; otherwise OPDG is also needed for SQL Server. Confirm once on-premises connectivity topology is known.
- **Azure SQL control DB** — VNet Data Gateway for Pipeline connectivity only (Notebooks do not interact with control DB). Network configuration TBD at provisioning time.
- **Managed Private Endpoints** — for Spark Notebook connections to Azure Blob Storage if required. Note: Notebooks are expected to access Fabric Lakehouses only — if that remains true, no MPEs are needed for Notebooks.
- **Genesys PureCloud** — public endpoint; allowlist in Outbound Access Protection if that is enabled.
- **ZScaler bypass rules** for Fabric FQDNs must be in place before any Private Link testing begins.
- **Workspace provisioning runbook must enforce**: restrict public access *before* creating Lakehouse/Warehouse items to avoid default semantic model incompatibility.
- **Purview sensitivity labels** — confirm whether MIP labelling is required before deciding on isolation model. If mandatory, tenant-level PL is ruled out (workspace-level PL impact on MIP is unconfirmed and needs testing).

---

## Stakeholder Steering Strategy

The client will likely default toward maximum isolation ("Tenant-level PL + Block Public") because it sounds most secure. The consultant's interest is in a simpler, more implementable architecture. These interests align when the client is helped to understand what they are actually trading away — and whether the trade is worth making.

The goal is not to under-engineer security. It is to help the client design to their actual requirements rather than to a theoretical maximum.

### The Core Argument: Perimeter Security vs. Zero Trust

Network isolation (Private Link) is a perimeter security control — it assumes that restricting network access makes data safer. Microsoft's own recommended model for modern cloud security is Zero Trust, which does not rely on network perimeter:

- Every access request is authenticated via Entra ID (already in place)
- Conditional Access enforces device compliance, MFA, and location
- Workspace roles and item permissions enforce least-privilege data access
- All data in transit is encrypted (TLS 1.2+), all data at rest is encrypted by Microsoft

The question to put to Cyber Security Policy: **"Which specific control in our compliance framework requires network isolation, and does Entra ID + Conditional Access + encryption not satisfy that control?"** Most clients cannot answer this with a specific citation. When they have to justify it in writing, the requirement often softens.

### Arguments to Use Per Stakeholder

**With Head of Cyber Security Policy:**
- Ask for the specific compliance mandate, not the general aspiration. ISO 27001 A.8.20 (network controls) can be satisfied by Conditional Access and encryption — it does not mandate Private Link specifically.
- Sensitivity labels via Microsoft Purview stop working with tenant-level Private Link. If their Purview investment and data governance programme depend on labelling, full isolation undermines it. This is a policy conflict within their own security requirements.
- "No public internet" as an absolute policy is not achievable with all Fabric features regardless of which Private Link option is chosen. The Capacity Metrics App, OPDG registration, and Genesys PureCloud are all public internet dependencies. Establish what "no public internet" actually means in practice.

**With Head of Cyber Security Operations:**
- Full tenant Private Link with Block Public Internet Access means OPDG registration fails — the Sybase ASE ingestion pipeline does not run on day one. That is an operational incident, not a security win.
- The Capacity Metrics App stops working under full tenant PL. Capacity management and cost control in production depend on it. Ask who owns the risk of having no capacity visibility in PROD.
- ZScaler bypass rules must be in place before Private Link works for any user. The change management process for that config change is a dependency on the entire project timeline — surface it as a project risk early.
- Managed Private Endpoint approval requests appear in the Azure Portal. Operations needs a defined workflow to approve or reject them. Without one, ingestion pipelines stall waiting for approvals. This is an operational overhead that does not exist without Private Link.

**With Head of Architecture:**
- Workspace-level Private Link across 10–20 workspaces × 3 environments means 30–60+ private endpoints, DNS entries, and private link services to provision and maintain. This is ongoing operational complexity that lives in the Azure platform team, not in the Fabric team. Confirm that the Azure platform team has capacity to own this.
- DNS for workspace-specific FQDNs must be resolvable through VPN and ZScaler for every developer and business user. One misconfigured DNS entry silently breaks access to a workspace. Confirm the DNS management ownership and change process.
- If there is no VPN/ExpressRoute from on-premises to Azure, VNet Data Gateway cannot reach on-premises SQL Server — meaning OPDG is required regardless of other choices. Confirm the connectivity topology before any Private Link discussion.

**With Head of Data:**
- Deployment Pipelines (CI/CD between DEV/UAT/PROD) cannot be used on workspaces with restricted public access. If the team plans to use Deployment Pipelines for release management, workspace-level PL with public access restriction on those workspaces is incompatible. This is a development workflow decision, not just a security one. See deployment strategy note below.
- Business unit users accessing Silver and Gold lakehouses via browser or Power BI Desktop need their DNS to resolve workspace-specific FQDNs correctly through VPN. Any user who is not on the VPN or whose ZScaler config is not correct will be locked out. Confirm the business unit access model before restricting public access on consumer workspaces.
- Spark cold start increases from ~30 seconds to 3–5 minutes once a managed VNet is provisioned (triggered by Private Link). This affects pipeline schedule design and SLA commitments.

### Deployment Strategy: How to Deploy to PROD with Workspace-Level PL

The constraint: workspaces with **public access restricted** cannot be part of a Fabric Deployment Pipeline. This means the standard "Deploy to PROD" button in the Fabric portal does not work for restricted workspaces. Three options:

**Option 1 — Workspace-level PL without restricting public access on PROD**

Configure the private link service on PROD workspaces but do not block public access. Deployment Pipelines continue to work. The private path exists for users who use it; the public path is not closed.

- Deployment Pipelines: ✓ works
- Isolation: partial — private link configured but public internet not blocked
- Security posture: defensible if policy requires "private link in place" rather than "public access blocked"
- Simplest to implement; weakest of the three

**Option 2 — Git-based CI/CD (Azure DevOps or GitHub)**

Each Fabric workspace connects to a branch in a Git repository. Deployment to PROD happens by merging to the PROD branch; Fabric syncs automatically. No Deployment Pipeline needed.

- Deployment Pipelines: not used
- Isolation: full — PROD workspace can have public access restricted
- Additional benefit: full version control, PR review process, audit trail in Git — more mature than Deployment Pipelines for enterprise environments
- Requires Git integration setup per workspace and a branching strategy (e.g., DEV → `dev` branch, UAT → `uat` branch, PROD → `main`)
- **Recommended for PROD**

**Option 3 — Hybrid**

Deployment Pipelines for DEV → UAT (neither environment has public access restricted). Git-based deployment for UAT → PROD (PROD has restricted access, so Deployment Pipelines cannot be used for that stage).

- Gives development iteration convenience (Deployment Pipelines) with production rigour (Git)
- Reasonable middle ground during transition

> **Recommendation**: Option 2 (full Git-based CI/CD) is the right enterprise pattern regardless of Private Link — it provides audit trail, rollback, and code review that Deployment Pipelines do not. The Private Link constraint is a forcing function toward better practice. Raise this with the Head of Data as a CI/CD maturity conversation, not just a security constraint.

### The Recommended Position to Defend

**Workspace-level Private Link on PROD consumer workspaces (Gold, Semantic Model, Reporting) only, with Workspace IP Firewall on all other workspaces.** This is a defensible, professionally responsible recommendation because:

1. It isolates the layer where sensitive data reaches the widest audience — the consumer layer
2. It does not break OPDG, the Capacity Metrics App, Purview sensitivity labels, or Deployment Pipelines
3. It scales manageably — 3–5 consumer workspaces across environments rather than 30–60
4. It satisfies a "network isolation for sensitive data" policy requirement if one exists
5. It can be extended later if requirements change — adding workspace PL to more workspaces is straightforward; removing tenant-level PL after it is in place is disruptive

The client gets meaningful, demonstrable network isolation on the workspaces that matter most. The implementation remains manageable. The operational gaps are avoided.

### What to Confirm Before Finalising the Recommendation

- [ ] Does a specific compliance mandate require network isolation, and which control?
- [ ] Is Purview sensitivity labelling a hard requirement?
- [ ] Is the Capacity Metrics App required in PROD? (Confirms workspace-level PL is viable and tenant-level PL is not)
- [ ] Are Deployment Pipelines planned for DEV → UAT → PROD promotion?
- [ ] What does "no public internet" specifically mean — inbound to Fabric, outbound from Fabric, or both?
- [ ] Confirm the Capacity Metrics App functions under workspace-level PL (requires testing)
