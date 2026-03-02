# Meeting Prep — Network Security Discussion
**Date:** 2026-03-02
**Attendees:** Head of Architecture, Head of Cyber Security Policy, Head of Cyber Security Operations
**Intel:** Head of Data expects the cyber team will request private network for everything

---

## Expected Position from Cyber Team

"All production data must traverse private networks only — no public internet."

This is a reasonable instinct for a regulated financial institution, but it collides with how Fabric works as a SaaS platform. The goal of this meeting is to establish what they are actually trying to achieve (the risk outcome), so we can design controls that deliver that outcome without triggering the Fabric limitations that come with a blanket Private Link mandate.

---

## Key Talking Points

### 1. No compliance framework requires Private Link

Lead with this. It reframes the conversation from "you must have it" to "what risk are you mitigating?"

- **APRA CPS 234** is fully outcome-based. It requires controls commensurate with the threat — it does not prescribe network architecture. The risk assessment determines what is adequate, not the standard itself.
- **PCI-DSS** is the closest thing to a network mandate, but Fabric is very likely outside PCI-DSS scope. PAN is masked to the first-6/last-4 standard at source before reaching Fabric. No CVV, no PIN. Expiry date alone doesn't create scope. QSA confirmation recommended but the position is strong.
- **GDPR and Australian Privacy Act** are both outcome-based. Neither references network isolation.

**The question to ask:** *"Which specific policy or regulatory obligation are you satisfying with private networking?"* If the answer is internal cyber policy rather than a regulatory mandate, that's a different conversation — one about risk appetite and what compensating controls are acceptable.

---

### 2. Tenant-level Private Link is not viable for this architecture

If the cyber team asks for "everything private," they will likely mean tenant-level Private Link. This is where you need to be direct about what breaks.

| What breaks under tenant-level PL + Block Public Internet Access | Client impact | Confirmed for workspace-level PL? |
|------------------------------------------------------------------|--------------|----------------------------------|
| OPDG cannot register with Fabric | **Sybase ASE ingestion fails entirely** — no workaround | ✅ Does NOT apply — workspace-level PL keeps public endpoints available for OPDG |
| Sensitivity labels greyed out in Power BI Desktop | Purview MIP labelling cannot be applied — likely a compliance requirement itself | ⚠️ **Unknown** — not documented either way for workspace-level PL |
| Capacity Metrics App stops working | No capacity monitoring for PROD — operational blind spot | ✅ Does NOT apply — confirmed functional under workspace-level PL |
| OneLake Catalog Govern tab unavailable | Reduced Purview governance capability | ⚠️ **Unknown** — documented for tenant-level PL; workspace-level PL behaviour not confirmed |
| Cannot enable both inbound PL and outbound protection via portal | Requires REST API — adds operational complexity | ⚠️ **Unknown** — likely applies; not confirmed |

The OPDG dependency alone is a hard blocker for tenant-level PL. OPDG registers at the tenant level — there is no workaround that preserves both tenant-level Block Public Internet Access and a functioning OPDG. This is documented Microsoft behaviour, not a configuration gap.

**Critical caveat on workspace-level PL:** Microsoft's documentation on workspace-level Private Link limitations is incomplete. Several limitations that are confirmed for tenant-level PL — sensitivity labels, OneLake Catalog, outbound protection — are not explicitly documented as either present or absent for workspace-level PL. It is possible workspace-level PL inherits some of the same restrictions. **We cannot guarantee workspace-level PL is fully functional without testing against this client's specific workload configuration.**

**If they push back:** Microsoft's own guidance is that workspace-level Private Link is the recommended path when OPDG must coexist with private network isolation — but that recommendation is about the OPDG constraint specifically, not a blanket assurance that all features work.

---

### 3. Workspace-level Private Link may be viable — but comes with uncertainty and overhead

Workspace-level PL avoids the OPDG blocker and is confirmed to preserve the Capacity Metrics App. However, **Microsoft's documentation on what else breaks at workspace-level is incomplete** — several limitations that are confirmed for tenant-level PL (sensitivity labels, OneLake Catalog, outbound access protection) are not documented either way for workspace-level PL. It is possible workspace-level PL inherits some of the same restrictions. We would need to test before committing to this architecture.

The honest overhead beyond the uncertainty:
- Each workspace gets its own private link service + private endpoint in the client VNet + DNS entry
- **10–20 workspaces × 3 environments = 30–60+ provisioning operations minimum**
- Must be configured **before** creating any Lakehouse, Warehouse, or Mirrored Database in each workspace — cannot be retrofitted (default semantic model compatibility issue)
- ZScaler in the traffic path will need Fabric FQDNs added to bypass rules — needs Cyber Ops approval and change process time
- Developers using Power BI Desktop or VS Code locally need DNS resolution for each workspace-specific FQDN

This is manageable and can be partially automated via REST API, but it needs to be a conscious decision with eyes open to the operational cost — both for initial build and for every new workspace provisioned going forward.

---

### 4. Propose the tiered approach

Rather than a binary "Private Link everywhere or nowhere," propose applying controls proportionate to data sensitivity. This is explicitly aligned with how CPS 234 works — commensurate controls.

| Tier | Workspaces | Control | Rationale |
|------|-----------|---------|-----------|
| 1 | Gold (PROD), reporting | Workspace Private Link + public access restricted | Highest sensitivity; user-facing; most scrutiny |
| 2 | Silver, UAT | Workspace IP Firewall (corporate IP ranges + ZScaler egress IPs) | Controlled access without full PL overhead |
| 3 | DEV, Bronze, Landed | Entra ID + Conditional Access + MFA | Lower sensitivity; developer access; rapid iteration needed |

The security baseline that applies to **all tiers** regardless:
- Entra ID authentication on every request — no anonymous access possible
- Conditional Access with MFA, targeting all five Fabric-dependent services
- TLS 1.2+ enforced on all traffic in and out of Fabric — no unencrypted transit
- Encryption at rest (Microsoft-managed keys, CMK available if required)
- Audit logging exported to Log Analytics → Sentinel
- MDCA for anomaly detection and session controls
- Purview sensitivity labels for data classification

**The pitch:** Tier 1 gives the cyber team the private network outcome for production data where it matters. Tiers 2 and 3 avoid engineering overhead on environments that don't warrant it, while maintaining a strong identity-and-encryption security posture throughout.

---

### 5. Outbound private connectivity is already in the design

The cyber team may also raise concerns about Fabric reaching out to external data sources over public internet. Cover this proactively:

- **On-premises sources (SQL Server, Sybase ASE):** Traffic goes through the OPDG on-premises — never exposes the source to the public internet
- **Azure Blob Storage:** Trusted Workspace Access — private connectivity without a gateway
- **Amazon S3:** OPDG-backed shortcut — if the OPDG machine has a route to S3 (e.g., via existing AWS connectivity), traffic stays private through the gateway
- **Azure SQL control DB:** VNet Data Gateway — private connectivity to the DB if it's network-restricted

---

## Questions to Ask the Cyber Team

These are diagnostic — the answers determine whether workspace PL is required or whether a strong identity + encryption + monitoring posture is acceptable.

1. **"Is the private network requirement driven by a specific internal policy, or a regulatory obligation? Can you point us to the policy document?"**
   *(Separates a documented policy from a general instinct — both are valid but have different flexibility)*

2. **"Does that policy apply to SaaS platforms, or to IaaS/PaaS resources your team provisions?"**
   *(Fabric is a SaaS service — many "private network" policies are written with Azure VNet resources in mind, not SaaS)*

3. **"What is the specific risk you are mitigating — data interception in transit, unauthorised access, or lateral movement from a compromised endpoint?"**
   *(Each of these has targeted controls; private networking is not the only answer to any of them)*

4. **"Are sensitivity labels / Purview information protection in scope for this project?"**
   *(If yes, tenant-level Private Link is immediately off the table — they are mutually exclusive. Workspace-level PL behaviour is unconfirmed and would need to be tested before committing)*

5. **"What is your ZScaler bypass change process — and what's the typical lead time?"**
   *(This will gate any Private Link implementation regardless of what's decided today)*

---

## What a Good Outcome Looks Like

The best outcome is agreement on a tiered model where:
- Tenant-level PL is explicitly off the table, documented with reasons
- Workspace-level PL is agreed as the direction for Tier 1, subject to a proof-of-concept to confirm which features remain functional (sensitivity labels, OneLake Catalog, outbound protection)
- IP Firewall + Conditional Access covers Silver/UAT
- DEV operates on Entra + CA baseline
- The cyber team understands both the OPDG constraint and the documentation gaps, and accepts that a testing phase is required before committing

A workable second outcome is agreement that the cyber team will review the specific limitations and come back with a revised position — better than committing to an architecture in the meeting before the full picture is known.

The outcome to avoid is a blanket directive — whether tenant-level or workspace-level PL for everything — agreed without the cyber team understanding what may break and without a testing phase built into the plan.

---

## One-Line Compliance Summary to Have Ready

*"None of the four frameworks — APRA CPS 234, PCI-DSS, GDPR, or the Australian Privacy Act — mandate Private Link. PCI-DSS network controls are very likely out of scope given the masked PAN position. APRA CPS 234 requires controls commensurate with risk, which the tiered model satisfies. The private network requirement, if maintained, is a policy decision not a regulatory one — and that distinction matters for how we scope and prioritise it."*
