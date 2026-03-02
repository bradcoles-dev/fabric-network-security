# Regulatory Compliance — Microsoft Fabric

> **Sources:**
> - [Microsoft APRA compliance overview](https://learn.microsoft.com/en-us/compliance/regulatory/offering-apra-australia/)
> - [Microsoft cloud services: compliance with APRA CPS 234](https://query.prod.cms.rt.microsoft.com/cms/api/am/binary/RE2OsZg)
> - [Microsoft cloud services: compliance checklist for Australian financial institutions](https://www.microsoft.com/cms/api/am/binary/RE3ez0C)
> - [Microsoft response to APRA Information Paper on Cloud](https://aka.ms/navigatecloudaustralia)
> - [Microsoft Fabric HIPAA compliance announcement](https://powerbi.microsoft.com/en-us/blog/microsoft-fabric-is-now-hipaa-compliant/) (Jan 2024)
> Last reviewed: 2026-03-02

---

## 1. Fabric's Own Compliance Certifications

Microsoft Fabric holds the following certifications (as of January 2024):

| Certification | Scope |
|---------------|-------|
| ISO/IEC 27001 | Information security management system |
| ISO/IEC 27017 | Cloud-specific security controls |
| ISO/IEC 27018 | Protection of personal data in the cloud |
| ISO/IEC 27701 | Privacy information management (extends 27001 for GDPR/privacy) |
| HIPAA BAA | Fabric covered under Microsoft's Business Associate Agreement |

Fabric also inherits Azure's broader certification portfolio (SOC 1 Type II, SOC 2 Type II, PCI-DSS attestation, FedRAMP) as it runs on Azure infrastructure — but those certifications are issued at the **Azure** level, not Fabric-specifically.

Audit reports and certificates are available from the [Microsoft Service Trust Portal](https://servicetrust.microsoft.com/).

---

## 2. Does Compliance Require Private Link?

**No framework in scope mandates Private Link specifically.** The table below shows what each framework actually requires at the network layer:

| Framework | Network mandate? | What it actually requires |
|-----------|-----------------|--------------------------|
| APRA CPS 234 | None | Outcome-based. Controls must be "commensurate with the size and extent of threats." The risk assessment determines what is adequate — not the standard itself. |
| PCI-DSS | Closest to one, but not Private Link | Req 1 requires network security controls and segmentation of the cardholder data environment. Technology-agnostic — IP firewall + Conditional Access + encrypted transit can satisfy this. **Only applies if cardholder data is in scope.** |
| GDPR | None | "Appropriate technical and organisational measures." Encryption, access controls, and audit logging satisfy this. Network isolation is not referenced. |
| Australian Privacy Act | None | Same outcome-based approach as GDPR. No network architecture prescribed. |

**What actually drives Private Link decisions** is not the regulations themselves, but:

- **Internal cyber policy** — CISO-level rules such as "no production data over the public internet"
- **Risk appetite** — choosing stronger controls than compliance strictly requires
- **APRA heightened-risk scrutiny** — regulators may expect stronger controls for sensitive financial data even when not prescribed, making Private Link a practical governance choice
- **PCI-DSS scoping** — if cardholder data is in scope, the segmentation requirement is the closest thing to a mandate, but it remains technology-agnostic

The implication: if the client's cyber policy does not have an explicit "no public internet" rule and cardholder data does not flow through Fabric, there is no compliance-driven requirement for Private Link. The decision becomes a risk and policy question, not a regulatory one.

---

## 3. APRA CPS 234 — Information Security

### 3.1 The Gap: No Fabric-Specific APRA Coverage

Microsoft's APRA compliance documentation lists **Azure, Dynamics 365, and Office 365** as in-scope services. Microsoft Fabric is not explicitly named.

**This does not mean Fabric is non-compliant.** It means:
- Microsoft's APRA assurances apply at the Azure infrastructure layer, which Fabric runs on.
- Fabric inherits Azure's CPS 234 posture for the controls Microsoft manages.
- The customer (and their implementation partner) is responsible for configuring Fabric correctly to satisfy CPS 234 at the application and workload layer.

Microsoft has published three documents to support APRA-regulated entities using Azure (applicable to Fabric by extension):

1. **CPS 234 mapping document** — maps each CPS 234 obligation to specific Microsoft cloud controls and capabilities.
2. **CPS 231 compliance checklist** — maps Azure against CPS 231 (outsourcing) and related standards.
3. **APRA Information Paper response** — addresses APRA's cloud risk categories (low / heightened / extreme) and mitigation approaches.

### 3.2 CPS 234's Four Core Obligations

CPS 234 requires APRA-regulated entities to:

| Obligation | What It Requires | How Fabric Supports It |
|------------|-----------------|----------------------|
| **Roles and responsibilities** | Clearly define information-security roles across the organisation, third parties, and cloud providers | Microsoft's shared responsibility model is documented in the APRA CPS 234 mapping. Implement RBAC via Entra ID groups; define data owner, workspace admin, and capacity admin roles in Fabric. |
| **Security capability** | Maintain capability commensurate with size and threat level | Azure and Fabric provide baseline security (encryption, identity, DDoS, logging). What additional controls are required (e.g. Conditional Access, network isolation, MDCA monitoring) is determined by the entity's own risk assessment — CPS 234 does not prescribe specific technologies. |
| **Controls + testing** | Implement controls to protect information assets; regularly test and provide assurance of effectiveness | Enable Fabric audit logging → Log Analytics; configure Microsoft Purview sensitivity labels; run annual penetration testing (client responsibility); use Compliance Manager for ongoing posture tracking. |
| **Incident notification** | Notify APRA of material information security incidents within 72 hours | Microsoft notifies customers of Azure security incidents per contractual SLA. Client must maintain an incident response plan covering the 72-hour APRA notification window, including Fabric-sourced incidents. |

### 3.3 APRA Risk Classification for Cloud Outsourcing

APRA categorises cloud usage by inherent risk: **low**, **heightened**, or **extreme**. Hosting customer financial data on a shared SaaS analytics platform (Fabric) is likely to be assessed as **heightened inherent risk**, which requires:

- Formal risk assessment documented and approved by the Board or delegated authority
- Contractual rights for APRA to access information about the arrangement
- Due diligence on Microsoft's security controls (Service Trust Portal, third-party audit reports)
- Ongoing assurance processes (quarterly or annual review)

APRA does not prohibit heightened-risk arrangements — it requires a proportionate level of diligence.

### 3.4 Notification Requirements (CPS 231)

Under CPS 231 (outsourcing), financial institutions must:
- Notify APRA **after** entering material outsourcing agreements within Australia
- **Consult** APRA **before** outsourcing outside Australia (Microsoft's Australian datacenters are in scope for data residency commitments)

Microsoft contractually commits to storing customer data at rest in the Australian geography when Australian datacenters are selected. This supports CPS 231 cross-border data requirements.

---

## 4. PCI-DSS

### 4.1 Scope Is the Critical Question

PCI-DSS prescriptive network controls (Requirement 1: network security controls; Requirement 1.3: restrict inbound/outbound traffic) **only apply to systems that store, process, or transmit cardholder data** (PANs, CVVs, expiry dates).

**If no cardholder data flows through Fabric, PCI-DSS network requirements do not apply to the Fabric environment.**

This is the single most important compliance question to resolve before designing the network architecture. Possible outcomes:

| Cardholder data in Fabric? | PCI-DSS implication |
|---------------------------|---------------------|
| No | PCI-DSS network requirements do not apply; scope limited to other systems |
| Tokenised data only | Depending on token format, may be out of scope — confirm with QSA |
| Yes | Fabric workspaces handling CHD must meet PCI-DSS network controls (inbound/outbound restriction, segmentation) |

### 4.2 Azure PCI-DSS Attestation

Azure holds a PCI-DSS Attestation of Compliance (AoC). Fabric, as an Azure-based service, can operate within a PCI-DSS compliant Azure environment. The customer is still responsible for configuring Fabric in accordance with PCI-DSS requirements if cardholder data is in scope.

### 4.3 Relevant Controls If In-Scope

If Fabric does process cardholder data, the following Fabric controls are directly relevant:

> **Note on Req 1 and Private Link:** PCI-DSS Req 1 mandates network security controls and segmentation of the cardholder data environment — it does not mandate Private Link specifically. IP Firewall + Conditional Access + TLS-encrypted transit can satisfy the segmentation requirement if accepted by a QSA. Private Link is one way to achieve stronger segmentation, not the only way.

| PCI-DSS Requirement | Fabric Control |
|---------------------|---------------|
| Req 1 — Network security controls | IP Firewall + Conditional Access (minimum); Workspace Private Link for stronger segmentation |
| Req 2 — Secure configurations | Capacity and workspace admin controls; Fabric Admin portal settings |
| Req 3 — Protect stored data | Encryption at rest (CMK recommended); column-level security |
| Req 4 — Encrypt transmission | TLS 1.2+ enforced by default for all Fabric traffic |
| Req 7 — Restrict access | RBAC via Entra groups; workspace roles; RLS/CLS/OLS |
| Req 8 — Identity management | Entra ID; MFA via Conditional Access |
| Req 10 — Log and monitor | Fabric audit logs → Log Analytics → Sentinel |
| Req 11 — Test security | Annual pen test; Compliance Manager assessments |
| Req 12 — Security policy | Customer responsibility |

---

## 5. GDPR

### 5.1 Fabric's Position

ISO/IEC 27701 (held by Fabric) is the international standard for privacy information management and is aligned with GDPR. ISO 27018 covers cloud-provider handling of personal data.

GDPR requires **appropriate technical and organisational measures** for personal data — it does not mandate network isolation. The relevant obligations map to Fabric as follows:

| GDPR Principle | Fabric Control |
|----------------|---------------|
| Lawfulness / purpose limitation | Data classification via Purview sensitivity labels; item-level access controls |
| Data minimisation | Column-level security; RLS to restrict what users see |
| Integrity and confidentiality | Encryption at rest (MMK default, CMK optional); TLS in transit; Entra ID authentication |
| Accountability | Audit logs; Purview lineage; sensitivity label inheritance across exports |
| Data subject rights | Purview data map for discovery; deletion must be coordinated with data owners |
| Breach notification (72 hours) | Microsoft incident SLA + client incident response process |

### 5.2 Data Residency

Fabric data at rest is stored in the home region or a multi-geo capacity region of the customer's choice. For GDPR, EU data can be kept within EU regions. For Australian data subject to GDPR (e.g., processing EU citizens' data), ensure Fabric capacity is provisioned in an appropriate region.

---

## 6. Australian Privacy Act (Privacy Act 1988)

The Australian Privacy Act governs the handling of personal information by Australian Government agencies and private sector organisations with annual turnover above AUD 3M. It incorporates the **Australian Privacy Principles (APPs)**.

Like GDPR, the Privacy Act is **outcome-based** — it does not prescribe network architecture. Compliance is achieved through:

| APP Principle | Fabric Control |
|---------------|---------------|
| APP 1 — Open and transparent management | Privacy policy + documented data handling procedures |
| APP 6 — Use/disclosure of personal information | Access controls; RLS; workspace role boundaries |
| APP 11 — Security of personal information | Encryption at rest and in transit; RBAC; audit logging; Conditional Access + MFA |
| APP 12 — Access to personal information | Purview data discovery; documented data subject access request process |

### 6.1 Cross-Border Disclosure (APP 8)

APP 8 requires that entities transferring personal information overseas take reasonable steps to ensure the recipient complies with the APPs. Microsoft's Australian Privacy Act compliance commitments (via the Microsoft Product Terms and Data Protection Addendum) are designed to satisfy APP 8 obligations when using Microsoft cloud services.

---

## 7. Shared Responsibility Summary

| Control Domain | Applicable Regulations | Microsoft Responsibility | Customer / Partner Responsibility |
|----------------|----------------------|-------------------------|----------------------------------|
| Physical security | All | ✅ Microsoft manages | — |
| Infrastructure encryption | CPS 234 (Req: protect information assets); PCI-DSS Req 3; GDPR Art 32; Privacy Act APP 11 | ✅ Default MMK at rest; TLS in transit | Enable CMK if required |
| Platform authentication | CPS 234; PCI-DSS Req 8; GDPR Art 32; Privacy Act APP 11 | ✅ Entra ID enforced | Configure Conditional Access policies |
| Network isolation | PCI-DSS Req 1 (if CHD in scope); CPS 234 (risk-assessment driven, not prescribed) | ✅ Infrastructure | Configure controls per risk assessment and cyber policy (IP Firewall, Conditional Access minimum; Private Link if policy or PCI-DSS segmentation requires it) |
| Access control | CPS 234; PCI-DSS Req 7; GDPR Art 5/32; Privacy Act APP 6/11 | ✅ RBAC framework provided | Implement Entra groups; assign workspace roles; define RLS/CLS/OLS |
| Audit logging | CPS 234 (testing & assurance); PCI-DSS Req 10; GDPR (accountability) | ✅ Logs generated | Configure export to Log Analytics; set retention policy |
| Incident notification | CPS 234 (72-hr APRA notification); GDPR Art 33 (72-hr supervisory authority notification); Privacy Act (eligible data breach — OAIC notification) | ✅ Notify customer per SLA | Maintain notification processes and incident response plan covering all applicable regulators |
| Compliance assessment | CPS 234 (annual testing & assurance); PCI-DSS Req 11 | ✅ Third-party audits; Service Trust Portal | Commission annual pen test; maintain Compliance Manager assessments |
| Data classification | CPS 234 (information asset classification); GDPR (data minimisation, purpose limitation); Privacy Act APP 6 | ✅ Purview tooling provided | Apply sensitivity labels; define classification taxonomy |
| Data residency | CPS 231 (cross-border outsourcing); GDPR (data transfers); Privacy Act APP 8 | ✅ Region selection honoured | Select Australian region capacity for data sovereignty |

---

## 8. Microsoft Purview Compliance Manager

Microsoft Purview Compliance Manager includes a **premium assessment template for APRA CPS 234**. This provides:

- Pre-built control mapping of CPS 234 obligations to Microsoft 365/Azure controls
- Improvement actions assigned to the customer with implementation guidance
- Ongoing compliance score tracking

Compliance Manager is accessible via the [Microsoft Purview portal](https://compliance.microsoft.com/). Financial institutions should use this as the primary tool for tracking their APRA compliance posture, supplemented by the Microsoft CPS 234 mapping document.

---

## 9. Key Actions for the Implementation Team

| Action | Owner | Priority |
|--------|-------|----------|
| Confirm whether cardholder data flows through Fabric (PCI-DSS scoping) | Client — Risk/Compliance | Critical |
| Commission formal APRA risk assessment for cloud outsourcing arrangement | Client — Risk | Critical |
| Enable Fabric audit log export to Log Analytics + configure retention | Partner | Must Have |
| Configure Conditional Access targeting all five Fabric-dependent services | Partner | Must Have |
| Define Entra group structure mapping to workspace roles and data sensitivity tiers | Partner | Must Have |
| Apply Purview sensitivity labels to classified information assets in Fabric | Partner | Must Have |
| Configure Purview Compliance Manager APRA CPS 234 assessment | Partner | Recommended |
| Obtain Microsoft CPS 234 mapping document and include in APRA risk file | Client — Risk | Recommended |
| Define incident response process including 72-hour APRA notification workflow | Client — Risk/IT | Must Have |
| Commission annual penetration test covering Fabric-accessible surfaces | Client | Must Have |
