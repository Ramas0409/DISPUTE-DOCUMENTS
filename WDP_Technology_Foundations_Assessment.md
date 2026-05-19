# WDP (hWP Disputes Platform) — Capabilities Assessment

## Category: Technology Foundations — FINALISED (14/14 sealed)

Competitor: **hGP (heritage Global Payments disputes platform)** — no hGP capability data supplied; "Engineering recommendation" states WDP's evidenced position and flags where a head-to-head is not possible.

Column order matches the assessment sheet: **Category · Capability · Segment · hWP Disputes Platform · Additional data needed · Engineering recommendation · Rationale · Gap to Target**.

Each cell below is written as a single Excel cell value (numbered points are inline within one cell).

---

### TF-1 — Deployment model (on-prem, private cloud, public cloud, SaaS)

| Column | Content |
|---|---|
| Category | Technology Foundations |
| Capability | Deployment model (on-prem, private cloud, public cloud, SaaS) |
| Segment | All — a single deployed platform serves SMB, Enterprise and Integrated merchants through logical multi-tenancy (merchant-scoped data isolation + per-merchant Kafka partitioning). No segment-specific deployment exists. |
| hWP Disputes Platform | Public-cloud, AWS-native, operated as a managed service by Worldpay. All components run as stateless, horizontally scalable microservices on a single shared AWS EKS cluster (multi-AZ), backed by Aurora PostgreSQL (multi-AZ) and AWS MSK Kafka (3-broker / 3-AZ), across 7 environments (local, dev, test, uat, stg, cert, prod). Logical multi-tenancy: row-level isolation by merchant_id plus per-merchant Kafka partitioning — logical multi-tenant isolation is the accepted model for all segments including the largest enterprise merchants. Integrated with card schemes (networks) and issuers for dispute and evidence file transfer via the enterprise file-exchange layer. |
| Additional data needed | (1) hGP deployment model — none supplied. (2) Confirmation of hGP's tenancy/isolation model for a like-for-like comparison. |
| Engineering recommendation | hWP (WDP) — modern AWS-native, elastic, single-platform deployment with logical multi-tenancy accepted across all segments. hGP comparison not possible (no data); convergence onto an unevidenced deployment model is itself a risk. |
| Rationale | 1. Cloud-native, stateless, horizontally scalable on AWS managed services (EKS / Aurora / MSK) — elastic and operationally consistent. 2. One shared platform = a single operational surface and consistent behaviour across all segments and acquiring platforms (lower TCO, no per-tenant fork). 3. Logical multi-tenant isolation (merchant_id + per-merchant partitioning) is proven and accepted for the largest enterprise merchants. |
| Gap to Target | No material capability gap on this line — deployment posture is target-state-aligned. Region topology is confirmed and intentional (see TF-2). |

---

### TF-2 — Hosting footprint (data center locations or cloud regions)

| Column | Content |
|---|---|
| Category | Technology Foundations |
| Capability | Hosting footprint (data center locations or cloud regions) |
| Segment | All — hosting footprint is foundational and applies uniformly across SMB / Enterprise / Integrated. |
| hWP Disputes Platform | AWS cloud, two regional deployments — US and UK. Application layer: stateless microservices, single-region, multi-AZ, highly available across 3 Availability Zones per region — US in us-east-2 (3 AZs), UK in London / eu-west-2 (3 AZs). Database layer: global with regional failover — US: primary us-east-2, secondary us-east-1; UK: primary London (eu-west-2), secondary Ireland (eu-west-1). Architecture pattern: a global application layer connects to the region-specific database based on the merchant's platform, so dispute data is pinned to its required geography while the application tier remains common. |
| Additional data needed | (1) Confirmation of the regional database failover mechanism and tested RPO/RTO per region pair (feeds TF-4 DR). (2) India region timeline and target merchant/platform mapping for the in-region data-residency migration. (3) hGP hosting footprint and data-residency model — none supplied. |
| Engineering recommendation | hWP (WDP) — two-region (US + UK) AWS footprint with multi-AZ application HA, cross-region database failover per geography, and a residency-ready data/application separation pattern. hGP footprint unknown; cannot compare. |
| Rationale | 1. Data-residency-ready by design: the global-application / region-pinned-database pattern lets WDP satisfy data-residency requirements across geographies (e.g., a future India migration — keep India data in an India region while reusing the same global application layer, no data leaving the jurisdiction). 2. Two independent regional deployments (US, UK) already prove the pattern, with cross-region database failover within each geography. 3. Application tier highly available across 3 AZs per region — strong in-region fault tolerance with low operational complexity. |
| Gap to Target | (1) New-geography support requires data provisioned in-region plus an application deployment in that region — low effort by design: data is region-pinned and the application layer is portable, so onboarding a region is an additive deployment. (2) Application tier is single-region multi-AZ today — moving a region to multi-region active-active is minimal change (data already region-resident; only requires deploying the application layer in the additional region) if/when a geography needs it. |

---

### TF-3 — Availability SLA vs. trailing-12-month actual uptime

| Column | Content |
|---|---|
| Category | Technology Foundations |
| Capability | Availability SLA vs. trailing-12-month actual uptime |
| Segment | All — availability is a foundational platform property; applies uniformly across SMB / Enterprise / Integrated. |
| hWP Disputes Platform | SLA basis: WDP runs on AWS managed services, so availability SLAs are underpinned by the AWS service SLAs for each tier — Aurora PostgreSQL 99.99% (multi-AZ, auto-failover), EKS 99.95% (multi-AZ, 3 zones per region), MSK Kafka 99.9% (3-broker / 3-AZ), S3 / managed services 99.99%; overall portal + backend design target 99.9%. Kafka durability backing availability: acks=all, min in-sync replicas = 2, unclean leader election disabled, idempotent producer. Trailing-12-month actual uptime is measured and available, evidenced by AWS CloudWatch metrics plus application-layer metrics published via OpenTelemetry — availability is observed end-to-end (infrastructure + application). Liveness and readiness probes are configured on every deployment, and every pod publishes custom application metrics and OpenTelemetry telemetry through a centralized metric library — so hung pods are auto-detected and recycled and availability is measured uniformly across the platform, not sampled. |
| Additional data needed | (1) The specific trailing-12-month measured uptime figure(s) per tier from SRE/observability to populate the headline metric (mechanism confirmed: CloudWatch + OpenTelemetry; the number itself to be inserted). (2) Whether the 99.9% overall is a contractual enterprise commitment or internal design target. (3) hGP availability SLA and actuals — none supplied. |
| Engineering recommendation | hWP (WDP) — availability SLAs inherited from AWS managed-service SLAs (Aurora/S3 99.99%, EKS 99.95%, MSK 99.9%), with actual uptime independently measured via CloudWatch + OpenTelemetry. hGP availability posture unknown; cannot compare. |
| Rationale | 1. SLAs are not self-asserted — they are backed by AWS managed-service SLAs per tier, the strongest possible third-party underpinning. 2. Actuals are independently evidenced end-to-end via CloudWatch (infrastructure) and OpenTelemetry application metrics — claimed availability is verifiable, not theoretical. 3. Architecture reinforces the SLAs: multi-AZ across 3 zones per region, auto-failover database, durable Kafka (acks=all, min-ISR=2), liveness/readiness probes on every deployment, and a centralized metric library publishing custom + OpenTelemetry metrics per pod for uniform, automatic fault detection and recovery. |
| Gap to Target | Insert the trailing-12-month measured uptime figure(s) into the panel pack (data exists via CloudWatch + OpenTelemetry; only the published number needs to be pulled in). No architecture or capability gap on this line — availability posture is target-state-aligned and evidenced. |

---

### TF-4 — Disaster recovery posture (RPO/RTO, DR site, last test)

| Column | Content |
|---|---|
| Category | Technology Foundations |
| Capability | Disaster recovery posture (RPO/RTO, DR site, last test) |
| Segment | All — DR posture is foundational; applies uniformly across SMB / Enterprise / Integrated. |
| hWP Disputes Platform | DR is built on the two-region, region-pinned-database topology (TF-2). Database DR: Amazon Aurora Global Database providing managed cross-region replication and failover within each geography — US: us-east-2 → us-east-1; UK: London (eu-west-2) → Ireland (eu-west-1) — giving low-RPO continuous cross-region replication, complemented by automated DB snapshots every 5 minutes as the point-in-time recovery safety net (effective backup RPO ≈ 5 minutes). Application tier: stateless microservices, multi-AZ across 3 zones per region — an AZ failure is absorbed in-region with no failover event; full regional application loss is recovered by redeploying the stateless app layer into the paired region (data already resident via Aurora Global Database). RPO/RTO are realised through these AWS managed-service recovery characteristics (Aurora Global Database + 5-minute snapshots + multi-AZ auto-failover + S3 durability + MSK 3-AZ replication). DR tests are run on a defined cadence with results documented; the most recent was October 2025, with test date, outcome and achieved RTO/RPO recorded in the DR test documentation. |
| Additional data needed | (1) Pull the documented Oct 2025 DR test result and the defined cadence into the panel pack (both exist and are documented; only the figures/dates need surfacing). (2) hGP DR posture — none supplied. |
| Engineering recommendation | hWP (WDP) — DR built on Aurora Global Database managed cross-region replication per geography, 5-minute snapshot cadence, and a stateless redeployable application tier, with a documented, recurring DR test programme (most recent Oct 2025). hGP DR posture unknown; cannot compare. |
| Rationale | 1. Aurora Global Database delivers managed, low-RPO cross-region replication and fast failover within each geography (US↔, UK↔) — data-loss protection without breaching data-residency boundaries. 2. 5-minute automated snapshots provide an independent point-in-time recovery path on top of live replication — defence in depth. 3. Stateless application tier + region-pinned data makes regional application recovery an additive redeploy (low RTO complexity), and DR is exercised on a documented, recurring cadence (most recent Oct 2025) — a maintained DR programme, not a one-off test. |
| Gap to Target | Surface the documented Oct 2025 test result and DR cadence into the panel pack. No architecture or capability gap — DR posture is target-state-aligned and evidenced by a documented, recurring DR test programme. |

---

### TF-5 — High availability architecture (active-active vs. active-passive)

| Column | Content |
|---|---|
| Category | Technology Foundations |
| Capability | High availability architecture (active-active vs. active-passive) |
| Segment | All — HA architecture is foundational; applies uniformly across SMB / Enterprise / Integrated. |
| hWP Disputes Platform | In-region: active-active. Stateless microservices run across 3 Availability Zones per region (US: us-east-2; UK: London/eu-west-2) on multi-AZ EKS — traffic is served from all zones simultaneously; an AZ loss is absorbed with no failover event. Data tier: Aurora multi-AZ with automatic failover; MSK Kafka 3-broker / 3-AZ with acks=all, min in-sync replicas = 2, unclean leader election disabled — broker/AZ loss does not lose acknowledged data. Cross-region: active-passive by design — Aurora Global Database replicates to the paired secondary region per geography (US↔, UK↔); the stateless application tier is redeployed into the paired region on full regional loss (data already resident). Batch and scheduled jobs are deployed with multiple replicas, and execution is monitored via custom metrics with SRE alerting when a scheduled job does not run — batch processing is not a single-instance availability risk. HA is realised through AWS managed-service HA primitives with liveness/readiness probes on every deployment. |
| Additional data needed | (1) Confirmation that cross-region active-passive is accepted target state for all merchant tiers. (2) hGP HA architecture — none supplied. |
| Engineering recommendation | hWP (WDP) — active-active in-region across 3 AZs per region on AWS managed HA primitives, with controlled active-passive cross-region failover per geography and monitored multi-replica batch processing. hGP HA architecture unknown; cannot compare. |
| Rationale | 1. In-region active-active across 3 AZs with a stateless application tier — zone failure is transparent, no failover event, no recovery window for the common failure case. 2. Data/streaming HA is hardened: Aurora multi-AZ auto-failover and MSK 3-AZ with acks=all / min-ISR=2 / no unclean leader election — acknowledged data survives broker/AZ loss. 3. HA is built on AWS managed-service primitives with full probe-based health management for services, plus multi-replica batch jobs with custom-metric execution monitoring and SRE alerting — proven, supported, self-healing rather than bespoke single-instance. |
| Gap to Target | Cross-region is active-passive by design — a full regional outage is a low-complexity, data-resident redeploy/failover event, not transparent. This is accepted target state (active-active cross-region is not required); state it explicitly against the enterprise RTO so the failover window is understood. |

---

### TF-6 — Coupling with acquiring platform (data, settlement, identity dependencies)

| Column | Content |
|---|---|
| Category | Technology Foundations |
| Capability | Coupling with acquiring platform (data, settlement, identity dependencies) |
| Segment | All |
| hWP Disputes Platform | WDP is architected to decouple disputes from the acquiring platform: dispute events from every source converge on a common processing path and a single auditable case ledger, regardless of originating acquiring platform — disputes are not re-implemented per acquiring platform. LATAM and VAP are in active development on the common path. The strategic decoupling layer is EDIA streaming, work in progress, targeted for completion by end of PI4. Identity: the WDP Portal supports multiple IdP firms, so enterprise/integrated merchants on different identity providers are served by one platform — identity is a multi-tenant capability, not a single-provider coupling. |
| Additional data needed | (1) Target convergence end-state: acceptable level of residual platform-specific paths post-EDIA. (2) Whether settlement/funding reconciliation is independently audited per acquiring platform. (3) hGP coupling model — none supplied. |
| Engineering recommendation | hWP (WDP) — disputes run as a decoupled, common-path capability across acquiring platforms on one ledger, multi-IdP-firm aware, with a dated roadmap (EDIA, end PI4) to extend the common decoupling layer. hGP coupling model unknown; cannot compare. |
| Rationale | 1. Single dispute ledger + common processing path across acquiring platforms — disputes are an independent capability, not a per-platform re-build (the core convergence argument). 2. The common decoupling layer is on a dated roadmap (EDIA, end PI4; LATAM/VAP in development) — managed and time-bound. 3. Multi-IdP-firm support in the portal decouples identity from any single provider — one platform serves enterprise merchants across different identity providers. |
| Gap to Target | (1) Convergence is progressing — EDIA streaming decoupling layer targeted for completion end of PI4; LATAM/VAP common-path delivery in development. (2) Confirm post-EDIA target end-state so "fully decoupled" has a defined finish line. |

---

### TF-7 — Scheme connectivity architecture (direct vs. gateway, real-time vs. batch)

| Column | Content |
|---|---|
| Category | Technology Foundations |
| Capability | Scheme connectivity architecture (direct vs. gateway, real-time vs. batch) |
| Segment | All — scheme connectivity is foundational; applies uniformly across SMB / Enterprise / Integrated. |
| hWP Disputes Platform | WDP integrates directly with the card schemes for API connectivity. Visa dispute events are retrieved from Visa RTSI queues on a near-real-time basis; Mastercard via MCM queue retrieval; both via direct scheme API integration. Interlink PIN network is connected via direct API as well. Amex / Discover / other PIN networks integrate via scheme/issuer file exchange (dispute and evidence files in/out). NAP is real-time REST API push. WDP's design principle is to pursue direct API connectivity wherever the scheme supports it, falling back to file exchange only where direct API is not available. Outbound to schemes supports representment/accept/contest submission, questionnaire submission, evidence/document submission, and retrieval of issuer documents — the full bidirectional dispute exchange. |
| Additional data needed | (1) Whether true streaming real-time (vs near-real-time retrieval) is a target-state requirement for any merchant tier. (2) Roadmap for moving remaining file-based schemes (Amex/Discover/other PIN) to direct API where the scheme supports it. (3) hGP scheme connectivity model — none supplied. |
| Engineering recommendation | hWP (WDP) — direct, near-real-time API connectivity to the card schemes (Visa, Mastercard, Interlink PIN), broad scheme coverage (incl. Amex, Discover, other PIN via file exchange), and full bidirectional dispute exchange including questionnaire and issuer-document handling, with a stated design principle to favour direct API wherever possible. hGP scheme connectivity unknown; cannot compare. |
| Rationale | 1. Direct scheme API integration for Visa, Mastercard and Interlink PIN — near-real-time, low-latency dispute exchange. 2. Breadth + depth: all four global schemes plus PIN networks, with full bidirectional exchange — dispute events in, and representment/accept/contest, questionnaire submission, document submission and issuer-document retrieval out. 3. Deliberate connectivity strategy: WDP actively pursues direct API wherever a scheme supports it, using file exchange only as a fallback — a forward-leaning, modernising integration posture. |
| Gap to Target | (1) Amex / Discover / other PIN networks remain file-based where direct API is not yet available — confirm the roadmap to move these to direct API as scheme support allows, vs file exchange being the accepted end-state. (2) Confirm whether near-real-time retrieval meets the target SLA or true streaming real-time is required for any tier. |

---

### TF-8 — Post-dispute integration to acquiring back-end (settlement adjustment, funding, reporting)

| Column | Content |
|---|---|
| Category | Technology Foundations |
| Capability | Post-dispute integration to acquiring back-end (settlement adjustment, funding, reporting) |
| Segment | All — post-dispute money movement is foundational; applies uniformly across SMB / Enterprise / Integrated. |
| hWP Disputes Platform | When a dispute resolves, WDP produces the outcome from the single dispute ledger and shares the dispute outcome to the relevant acquiring platforms so they can perform settlement adjustment and funding. WDP's responsibility is the authoritative delivery of the dispute outcome to each acquiring platform; the acquiring back-end performs the actual money movement. Outcomes are delivered to NAP (chargeback outcome / representment and department-notice flows) and CORE acquiring back-ends; LATAM and VAP are in active development on the common path. Delivery integrity: outbound delivery uses per-channel outbox tables for idempotency and retry, with automated recovery in place to detect and re-drive orphaned/undelivered rows — outcomes are delivered exactly once per channel and are self-recovering and auditable on failure. Merchant-facing post-dispute reporting: dispute outcomes are also communicated to merchants through multiple channels — Comms Hub communications, BEN notifications and Dialogue notifications. The strategic universal outbound route is EDIA streaming, work in progress, targeted for completion by end of PI4. |
| Additional data needed | (1) EDIA cutover plan for the current acquiring-platform outbound paths. (2) Whether merchants receive a single consolidated outcome notification or per-channel (Comms Hub / BEN / Dialogue) by merchant preference. (3) hGP post-dispute integration model — none supplied. |
| Engineering recommendation | hWP (WDP) — production post-dispute outcome delivery to acquiring platforms with idempotent, self-recovering per-channel delivery, plus multi-channel merchant outcome notification, on a dated convergence roadmap (EDIA, end PI4). hGP post-dispute integration unknown; cannot compare. |
| Rationale | 1. WDP owns the authoritative dispute-outcome-to-acquiring-platform handoff for money movement — a clear, single source of dispute truth feeding settlement across acquiring platforms. 2. Idempotent, self-recovering delivery — per-channel outbox with retry and automated orphaned-row recovery, appropriate for funding-triggering events (no manual runbook dependency for the common failure case). 3. Closed merchant communication loop — outcomes pushed to merchants across Comms Hub, BEN and Dialogue, so the dispute result reaches both the acquiring back-end and the merchant. |
| Gap to Target | (1) EDIA convergence in progress (target end PI4) — current acquiring-platform outbound is platform-specific until then; the post-EDIA end-state should be defined so "uniform outbound" has a finish line. (2) No capability gap on delivery or merchant notification — outcome delivery, automated recovery and multi-channel merchant comms are in place and target-state-aligned. |

---

### TF-9 — Modularity (can disputes run as a standalone capability across acquiring platforms?)

| Column | Content |
|---|---|
| Category | Technology Foundations |
| Capability | Modularity — can disputes run as a standalone capability across acquiring platforms? |
| Segment | All — the single most decisive line for the convergence decision; modularity across acquiring platforms is the core question this assessment exists to answer. |
| hWP Disputes Platform | WDP is purpose-built as a standalone, centralised dispute platform that operates independently of any single acquiring platform. Disputes from every source converge on a common processing path and a single auditable case ledger, processed uniformly regardless of originating acquiring platform — disputes are not re-implemented per acquiring platform. WDP exposes defined common inbound and outbound integration interfaces — API, Kafka, and file — so a new acquiring platform can integrate via whichever interface suits it, with minimal onboarding effort and no dispute-logic changes. For enrichment, WDP either uses the acquiring platform's own API interface to enrich disputes with additional transaction/merchant data, or — where the platform provides no such interface — integrates directly with the enterprise Cloud Data Platform (CDP) / data-lake to source the additional transaction and merchant data needed to enrich the chargeback. WDP also replicates dispute data into CDP for analytics and reporting. Serves NAP and CORE in production; LATAM and VAP in active development on the common path. Universal decoupling/outbound layer: EDIA streaming, work in progress, targeted for completion by end of PI4. |
| Additional data needed | (1) Post-EDIA target end-state confirmation (100% common-path vs accepted residual platform-specific integration). (2) A reference onboarding timeline for the most recent acquiring platform, to quantify "minimal effort" with a concrete figure. (3) hGP modularity model — none supplied. |
| Engineering recommendation | hWP (WDP) — disputes run as a genuinely standalone, interface-driven capability across acquiring platforms on one ledger and one processing path, with defined common API/Kafka/file integration contracts and a flexible enrichment model (platform API or direct CDP/data-lake). This directly and affirmatively answers the central convergence question. hGP modularity unknown; converging onto an unevidenced platform whose cross-acquirer modularity is unproven is itself the principal risk this assessment exists to weigh. |
| Rationale | 1. Standalone by design, interface-driven onboarding — defined common inbound/outbound interfaces (API, Kafka, file) mean a new acquiring platform integrates via a contract, not a per-platform dispute re-build; onboarding is minimal-effort and additive. 2. Enrichment is not coupled to any one platform — WDP uses the platform's API where available and falls back to direct CDP/data-lake integration where it is not, so dispute enrichment works regardless of an acquiring platform's data-exposure maturity. 3. Dated convergence roadmap — EDIA universal layer targeted end PI4; LATAM/VAP demonstrate the additive-onboarding model in flight today. |
| Gap to Target | (1) Modularity is proven for production platforms (NAP, CORE), interface-defined, and being demonstrated for LATAM/VAP; full universality completes with the EDIA layer (target end PI4) and LATAM/VAP delivery. (2) Define the post-EDIA target end-state so "fully modular" has a stated finish line. (3) Publishing a reference onboarding effort/timeline would convert the (already strong) modularity claim from architectural assertion to fully evidenced. |

---

### TF-10 — Scalability and peak processing capacity

| Column | Content |
|---|---|
| Category | Technology Foundations |
| Capability | Scalability and peak processing capacity |
| Segment | All — scalability is a foundational platform property; applies uniformly across SMB / Enterprise / Integrated. |
| hWP Disputes Platform | WDP scales horizontally on AWS managed services. Stateless microservices on multi-AZ EKS auto-scale on load; the event backbone is AWS MSK Kafka; the data tier is Aurora PostgreSQL (multi-AZ) with auto-scaling read replicas; batch/scheduled jobs run multi-replica. Capacity is load-test evidenced, not just designed: the platform has been load-tested and supports 500 TPS, validated by CloudWatch and Instana metrics, with substantial spare infrastructure headroom beyond the tested figure. Scaling model is demand-led and auto-scaled: as new acquiring platforms are integrated and traffic grows, the target TPS is raised, re-tested, and AWS infrastructure auto-scales to the new level — capacity tracks demand rather than being fixed. Document-lookup scale on DynamoDB uses partition/key design; relational (Aurora/RDS) scaling is via multi-AZ and auto-scaling read replicas. |
| Additional data needed | (1) The next target TPS tier as new acquiring platforms onboard, and the re-test trigger/cadence (so the demand-led scaling model has a defined governance point). (2) hGP scalability / peak capacity — none supplied. |
| Engineering recommendation | hWP (WDP) — horizontally scalable on AWS managed services, load-test-evidenced at 500 TPS with spare headroom, and a demand-led auto-scaling model that re-tests as acquiring platforms onboard. hGP scalability unknown; cannot compare. |
| Rationale | 1. Evidenced, not asserted — current capacity is load-test-proven (500 TPS) and independently observable via CloudWatch + Instana, not a paper design target. 2. Headroom + auto-scaling — substantial spare infrastructure today, and AWS-managed auto-scaling means capacity grows by demand without re-architecture. 3. Demand-led scaling discipline — TPS targets are raised and re-tested as new acquiring platforms integrate, so capacity is deliberately matched to the convergence roadmap rather than guessed. |
| Gap to Target | Current evidenced capacity (500 TPS) is sized to current demand by design — as acquiring platforms converge onto WDP, the target TPS must be re-set and re-tested per the demand-led model; define the re-test trigger so scale-up is governed, not reactive. No architecture gap — the scaling model and headroom are target-state-aligned. |

---

### TF-11 — Data residency and sovereignty controls

| Column | Content |
|---|---|
| Category | Technology Foundations |
| Capability | Data residency and sovereignty controls |
| Segment | All — data residency/sovereignty is a foundational platform capability; applies across SMB / Enterprise / Integrated. |
| hWP Disputes Platform | Data residency is architecturally supported by design through WDP's region-pinned-data / global-application pattern (per TF-2). Dispute data is pinned to its required jurisdiction's region, while a common global application layer connects to the region-specific database based on the merchant's platform — so data does not leave its jurisdiction, yet a single application/codebase serves all regions. This is proven in production today across two jurisdictions: US (us-east-2 primary / us-east-1 secondary) and UK (London primary / Ireland secondary) — US dispute data stays in US regions, UK dispute data stays in UK/EU regions, with no cross-jurisdiction data movement. The pattern is designed to extend to additional jurisdictions as residency requirements arise, by provisioning data in the required region and deploying the portable application layer there. |
| Additional data needed | (1) Confirmation of any specific contractual/regulatory residency commitments by jurisdiction so the control can be mapped to obligations. (2) Whether the eu-west-2 onboarding-bucket usage is fully within the UK/EU residency boundary (housekeeping consistency, not a capability gap). (3) hGP data-residency model — none supplied. |
| Engineering recommendation | hWP (WDP) — data residency is a proven, production capability (US and UK jurisdictionally separated today) on a single global application layer, designed to extend to further jurisdictions as required. This is a strong convergence differentiator. hGP residency model unknown; cannot compare, and converging onto a platform with an unevidenced residency model is itself a regulatory risk. |
| Rationale | 1. Proven in production across two jurisdictions — US and UK dispute data are already region-pinned and separated; demonstrated, not designed-only. 2. Residency without platform fragmentation — region-pinned data plus a single global application layer means jurisdictional separation is achieved without forking the platform or multiplying the operational surface. 3. Extensible to regulated markets — the same proven pattern extends to further jurisdictions as residency requirements arise, de-risking future regulated-market expansion. |
| Gap to Target | (1) Additional jurisdictions are supported by extending the proven region-pinned pattern (provision in-region data + deploy the portable application layer); this is the established path as new residency requirements arise. (2) Map the residency control to specific per-jurisdiction regulatory obligations so "supports data residency" is evidenced against named requirements rather than stated generically. |

---

### TF-12 — Security and compliance posture (PCI DSS, SOC 2, ISO 27001, GDPR/PSD2)

| Column | Content |
|---|---|
| Category | Technology Foundations |
| Capability | Security and compliance posture (PCI DSS, SOC 2, ISO 27001, GDPR/PSD2) |
| Segment | All — security and compliance are foundational and apply uniformly across SMB / Enterprise / Integrated. |
| hWP Disputes Platform | WDP is PCI-DSS certified — full evidence provided and recertified annually — and maintains SOC 2 Type II (annual), GDPR and CCPA data-subject controls, and SOX 7-year immutable audit retention. CVV is not retained as part of the dispute / cardholder data record, consistent with the PCI-DSS prohibition on post-authorisation CVV retention. PAN protection: AES-256-GCM encryption, HMAC-SHA256 HPAN / EPAN two-token strategy, a dedicated EncryptionService boundary (sole plaintext-PAN handler), AWS KMS FIPS 140-2 Level 3 key management with IAM isolation, and a crypto audit trail; clear PAN is not persisted on the standard enrichment path. Access control: role-based access control (RBAC) in place; clear PAN is exposed only after decryption and only to users whose role explicitly authorises PAN viewing. TLS in transit; secrets via AWS Secrets Manager. |
| Additional data needed | (1) ISO 27001 certification status (held / in progress / not pursued) — currently unconfirmed. (2) PSD2 / SCA applicability and posture for in-scope flows — currently unconfirmed. (3) hGP compliance posture — none supplied. |
| Engineering recommendation | hWP (WDP) — annually-recertified PCI-DSS with SOC 2 Type II, GDPR, CCPA and SOX controls, no CVV retention, best-practice PAN cryptography, and role-gated least-privilege PAN access. hGP compliance posture is unevidenced; converging onto a platform whose compliance state is unknown is itself a material regulatory risk this assessment exists to weigh. |
| Rationale | 1. Certified and continuously re-validated — PCI-DSS certified with full evidence and annual recertification, plus SOC 2 Type II / GDPR / CCPA / SOX — a sustained, audited posture, not a point-in-time claim. 2. Best-practice PAN cryptography with least-privilege access — AES-256-GCM, HPAN/EPAN two-token, KMS FIPS 140-2 L3, dedicated encryption boundary, clear PAN exposed only post-decryption to explicitly role-authorised users. 3. Evidenced vs unevidenced — WDP's posture is documented, audited and source-verified; hGP's is unknown, making WDP the lower-compliance-risk convergence target on available evidence. |
| Gap to Target | (1) Confirm ISO 27001 position and PSD2/SCA applicability so framework coverage is complete and not partially unknown. (2) No capability gap on this line; security and compliance posture is target-state-aligned. |

---

### TF-13 — Tech stack and modernization posture (codebase age, roadmap)

| Column | Content |
|---|---|
| Category | Technology Foundations |
| Capability | Tech stack and modernization posture (codebase age, roadmap) |
| Segment | All — tech stack and modernization are foundational; apply uniformly across SMB / Enterprise / Integrated. |
| hWP Disputes Platform | WDP is a modern, cloud-native, event-driven platform: Spring Boot microservices on the latest stable framework/runtime versions, AWS MSK Kafka event backbone, Aurora PostgreSQL, AWS EKS (multi-AZ), DynamoDB, S3, AWS KMS, and an Angular SPA portal — observability via OpenTelemetry + CloudWatch + Instana through a centralized metric library. Codebase age: the WDP codebase is no more than ~2 years old, kept current through frequent Snyk scans driving dependency upgrades, with no end-of-life dependencies. Architecture is decomposed, stateless and horizontally scalable rather than monolithic; resilience controls are in place for external API calls. Active modernization roadmap with dated items: Ethoca pre-dispute alerts (end PI4), dispute analytics dashboard (end PI4), EDIA streaming as the universal acquiring-platform integration layer (end PI4), LATAM/VAP onboarding to the common path, and a continued direct-API-first scheme connectivity strategy. |
| Additional data needed | (1) Funded roadmap horizon beyond PI4 (post-EDIA target end-state) — currently none defined. (2) hGP tech stack / modernization posture — none supplied. |
| Engineering recommendation | hWP (WDP) — modern, recent (≤2-year) cloud-native event-driven microservice codebase on latest stable versions with continuous Snyk-driven dependency currency and no EOL dependencies, plus a dated in-flight modernization roadmap. hGP stack/age unknown; cannot compare, and converging onto an unevidenced (potentially legacy) stack is itself a modernization risk this assessment exists to weigh. |
| Rationale | 1. Modern and recent by evidence, not just style — ≤2-year-old codebase, latest stable framework/runtime versions, no EOL dependencies, Snyk-driven dependency hygiene — version-current, not merely modern-architecture. 2. Cloud-native, scalable architecture — microservices + Kafka + Aurora + EKS + Angular on AWS managed services, stateless and horizontally scalable, with resilience controls on external API calls. 3. Continuous, dated modernization — concrete in-flight items (Ethoca, dashboard, EDIA, LATAM/VAP) with PI-level targets; modernization is an operating rhythm, not a deferred re-platform. |
| Gap to Target | Define the post-EDIA target end-state / roadmap horizon beyond PI4 so the modernization roadmap has a stated finish line. No codebase-age or stack-currency gap — the platform is recent, version-current and actively maintained. |

---

### TF-14 — Observability and ops tooling (monitoring, alerting, runbooks)

| Column | Content |
|---|---|
| Category | Technology Foundations |
| Capability | Observability and ops tooling (monitoring, alerting, runbooks) |
| Segment | All — observability and ops tooling are foundational; apply uniformly across SMB / Enterprise / Integrated. |
| hWP Disputes Platform | WDP has end-to-end observability across infrastructure and application layers: AWS CloudWatch (infrastructure/managed-service metrics), Instana (APM), and OpenTelemetry instrumentation on every pod publishing custom and standard telemetry through a centralized metric library, with Grafana dashboards for operational visibility and Prometheus alerting in place. Distributed tracing via correlation-ID provides end-to-end request traceability across services. Health management: liveness and readiness probes on every deployment — unhealthy pods auto-detected and recycled. Alerting: Prometheus alerts plus SRE alerting on platform health, including custom metrics that alert when scheduled/batch jobs do not run as expected. Operations: a dedicated SRO/SRE team provides 24×7 monitoring and on-call support, with an owned operational runbook inventory (incident response, failover, recovery). |
| Additional data needed | (1) Alerting SLOs / MTTR targets (optional strengthener — operational metric, not a capability gap). (2) hGP observability / ops tooling — none supplied. |
| Engineering recommendation | hWP (WDP) — mature, full-stack observability (CloudWatch + Instana + OpenTelemetry via a centralized metric library, Grafana dashboards, Prometheus alerts), correlation-ID distributed tracing, full probe-based health management, and a dedicated SRO/SRE team providing 24×7 monitoring, on-call support and owned runbooks. hGP observability/ops posture unknown; cannot compare. |
| Rationale | 1. Full-stack, evidenced observability — infrastructure (CloudWatch) + APM (Instana) + per-pod OpenTelemetry through a centralized metric library, Grafana dashboards and Prometheus alerting, with correlation-ID distributed tracing for end-to-end traceability. 2. Self-healing + actively operated — liveness/readiness probes on every deployment, Prometheus + SRE alerting (including missed-batch detection), and a dedicated SRO/SRE team on 24×7 monitoring and on-call. 3. Operationally mature — an owned runbook inventory (incident/failover/recovery) plus measured availability/capacity means operations are evidenced and procedural, not ad hoc. |
| Gap to Target | No capability gap on this line — observability, alerting, distributed tracing, runbooks and 24×7 SRO/SRE operations are in place and target-state-aligned. Optional strengthener: publish alerting SLOs / MTTR targets in the panel pack. |

---

## Summary

| # | Capability | Position |
|---|---|---|
| TF-1 | Deployment model | Strength |
| TF-2 | Hosting footprint | Strength |
| TF-3 | Availability SLA vs actual uptime | Strength (insert measured uptime figure) |
| TF-4 | Disaster recovery posture | Strength |
| TF-5 | High availability architecture | Strength |
| TF-6 | Coupling with acquiring platform | Strength |
| TF-7 | Scheme connectivity architecture | Strength |
| TF-8 | Post-dispute integration to acquiring back-end | Strength |
| TF-9 | Modularity across acquiring platforms | Strength (decisive line) |
| TF-10 | Scalability & peak capacity | Strength |
| TF-11 | Data residency & sovereignty | Strength |
| TF-12 | Security & compliance posture | Strength (confirm ISO 27001 / PSD2) |
| TF-13 | Tech stack & modernization | Strength |
| TF-14 | Observability & ops tooling | Strength |

Open data items to close before the panel: trailing-12-month uptime figure (TF-3); ISO 27001 / PSD2 status (TF-12); optional reference onboarding timeline (TF-9) and load-test headroom ceiling (TF-10).
