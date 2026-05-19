# WDP (hWP Disputes Platform) — Capabilities Assessment

This document contains two sealed categories: Technology Foundations (14 capabilities) and Capability (5 capabilities sealed so far). Each cell below is one Excel cell value. Column order matches the assessment sheet: Category, Capability, Segment, hWP Disputes Platform, Additional data needed, Engineering recommendation, Rationale, Gap to Target.

## Category: Technology Foundations (14 capabilities)

Competitor is hGP (heritage Global Payments disputes platform). No hGP data was supplied. The Engineering recommendation column states the WDP position and notes where a direct comparison is not possible.

Column order matches the assessment sheet: Category, Capability, Segment, hWP Disputes Platform, Additional data needed, Engineering recommendation, Rationale, Gap to Target. Each cell below is one Excel cell value.

---

### TF-1 — Deployment model (on-prem, private cloud, public cloud, SaaS)

| Column | Content |
|---|---|
| Category | Technology Foundations |
| Capability | Deployment model (on-prem, private cloud, public cloud, SaaS) |
| Segment | All. One platform serves SMB, Enterprise and Integrated merchants through logical multi-tenancy. There is no segment-specific deployment. |
| hWP Disputes Platform | WDP runs on public cloud. It is AWS-native and operated as a managed service by Worldpay. All components run as stateless microservices on a shared AWS EKS cluster across multiple AZs. The data tier is Aurora PostgreSQL multi-AZ. The event backbone is AWS MSK Kafka with 3 brokers across 3 AZs. There are 7 environments: local, dev, test, uat, stg, cert and prod. Tenancy is logical. Data is isolated per merchant by merchant id, with per-merchant Kafka partitioning. This isolation model is accepted for the largest enterprise merchants. WDP integrates with card schemes and issuers for dispute and evidence file transfer. |
| Additional data needed | hGP deployment model is not supplied. hGP tenancy and isolation model is needed for a like-for-like comparison. |
| Engineering recommendation | WDP. It is a single AWS-native platform that scales horizontally and uses logical multi-tenancy accepted across all segments. hGP cannot be compared with no data supplied. |
| Rationale | The platform is cloud-native and stateless and scales horizontally on AWS managed services. One shared platform gives a single operational surface and consistent behaviour across all segments and acquiring platforms. Logical multi-tenant isolation is proven and accepted for the largest enterprise merchants. |
| Gap to Target | No material capability gap on this line. The deployment posture is aligned to target state. Region topology is confirmed and intentional. See TF-2. |

---

### TF-2 — Hosting footprint (data center locations or cloud regions)

| Column | Content |
|---|---|
| Category | Technology Foundations |
| Capability | Hosting footprint (data center locations or cloud regions) |
| Segment | All. Hosting footprint is foundational and applies to SMB, Enterprise and Integrated. |
| hWP Disputes Platform | WDP runs on AWS in two regional deployments: US and UK. The application layer runs stateless microservices, single region, across 3 AZs per region. US runs in us-east-2 across 3 AZs. UK runs in London (eu-west-2) across 3 AZs. The database layer is global with regional failover. US database is primary in us-east-2 and secondary in us-east-1. UK database is primary in London (eu-west-2) and secondary in Ireland (eu-west-1). A global application layer connects to the region-specific database based on the merchant platform. Dispute data stays in its required region while the application tier stays common. |
| Additional data needed | The regional database failover mechanism and tested RPO and RTO per region pair are needed for TF-4. India region timeline and target merchant or platform mapping are needed for a future residency migration. hGP hosting footprint and residency model are not supplied. |
| Engineering recommendation | WDP. It has a two-region AWS footprint with multi-AZ application HA and cross-region database failover per region. The data and application separation supports residency. hGP cannot be compared with no data supplied. |
| Rationale | Data residency is supported by the design. Data is pinned to its region and the application layer is common, so a region can be added without leaving the jurisdiction. Two regional deployments already prove the pattern. The application tier is highly available across 3 AZs per region. |
| Gap to Target | A new geography needs data provisioned in that region and the application deployed there. This is low effort because data is region-pinned and the application layer is portable. Moving a region to multi-region active-active is also low effort because the data is already in region and only the application needs to be deployed there. |

---

### TF-3 — Availability SLA vs. trailing-12-month actual uptime

| Column | Content |
|---|---|
| Category | Technology Foundations |
| Capability | Availability SLA vs. trailing-12-month actual uptime |
| Segment | All. Availability is foundational and applies to SMB, Enterprise and Integrated. |
| hWP Disputes Platform | WDP runs on AWS managed services, so the availability SLAs follow the AWS service SLAs per tier. Aurora PostgreSQL is 99.99% with multi-AZ auto-failover. EKS is 99.95% across 3 AZs per region. MSK Kafka is 99.9% with 3 brokers across 3 AZs. S3 and managed services are 99.99%. The overall portal and backend design target is 99.9%. Trailing-12-month actual uptime is measured. It is evidenced by AWS CloudWatch metrics and by application metrics published through OpenTelemetry, so availability is measured end to end. Liveness and readiness probes are configured on every deployment. Every pod publishes custom and OpenTelemetry metrics through a centralized metric library. Hung pods are detected and recycled automatically, and availability is measured the same way across the platform. |
| Additional data needed | The trailing-12-month measured uptime figures per tier are needed from SRE for the headline metric. The mechanism is confirmed; only the number needs to be added. Confirm whether the 99.9% overall is a contractual enterprise commitment or an internal target. hGP availability SLA and actuals are not supplied. |
| Engineering recommendation | WDP. Availability SLAs follow the AWS managed-service SLAs per tier and actual uptime is measured by CloudWatch and OpenTelemetry. hGP cannot be compared with no data supplied. |
| Rationale | The SLAs are backed by AWS managed-service SLAs per tier, not self-asserted. Actual uptime is measured end to end and is verifiable. The architecture supports the SLAs with multi-AZ across 3 zones per region, database auto-failover, durable Kafka, probes on every deployment, and a centralized metric library. |
| Gap to Target | Add the trailing-12-month measured uptime figures to the pack. The data exists in CloudWatch and OpenTelemetry; only the number needs to be pulled in. There is no architecture or capability gap on this line. |

---

### TF-4 — Disaster recovery posture (RPO/RTO, DR site, last test)

| Column | Content |
|---|---|
| Category | Technology Foundations |
| Capability | Disaster recovery posture (RPO/RTO, DR site, last test) |
| Segment | All. DR posture is foundational and applies to SMB, Enterprise and Integrated. |
| hWP Disputes Platform | DR is built on the two-region, region-pinned database topology in TF-2. The database uses Amazon Aurora Global Database for cross-region replication and failover within each region. US fails over us-east-2 to us-east-1. UK fails over London (eu-west-2) to Ireland (eu-west-1). This gives low-RPO continuous replication. Automated database snapshots run every 5 minutes as the point-in-time recovery safety net, so the backup RPO is about 5 minutes. The application tier is stateless and runs across 3 AZs per region. An AZ failure is absorbed in region with no failover event. A full regional application loss is recovered by redeploying the stateless application into the paired region, where the data already exists. RPO and RTO come from these AWS managed-service recovery characteristics. DR tests run on a defined cadence and results are documented. The most recent test was October 2025, with date, outcome and achieved RTO and RPO recorded. |
| Additional data needed | Pull the documented October 2025 DR test result and the defined cadence into the pack. Both exist and are documented; only the figures and dates need surfacing. hGP DR posture is not supplied. |
| Engineering recommendation | WDP. DR uses Aurora Global Database cross-region replication per region, 5-minute snapshots, and a stateless redeployable application tier, with a documented and recurring DR test programme. hGP cannot be compared with no data supplied. |
| Rationale | Aurora Global Database gives managed low-RPO cross-region replication and fast failover within each region, without breaching residency. The 5-minute snapshots add a separate point-in-time recovery path on top of replication. The application tier is stateless and the data is region-pinned, so regional recovery is a redeploy, and DR is tested on a documented recurring cadence. |
| Gap to Target | Surface the documented October 2025 test result and the DR cadence into the pack. There is no architecture or capability gap on this line. |

---

### TF-5 — High availability architecture (active-active vs. active-passive)

| Column | Content |
|---|---|
| Category | Technology Foundations |
| Capability | High availability architecture (active-active vs. active-passive) |
| Segment | All. HA architecture is foundational and applies to SMB, Enterprise and Integrated. |
| hWP Disputes Platform | In region the architecture is active-active. Stateless microservices run across 3 AZs per region. US runs in us-east-2 and UK runs in London (eu-west-2). Traffic is served from all zones at the same time. An AZ loss is absorbed with no failover event. The data tier is Aurora multi-AZ with automatic failover. MSK Kafka runs 3 brokers across 3 AZs and does not lose acknowledged data on a broker or AZ loss. Across regions the architecture is active-passive by design. Aurora Global Database replicates to the paired secondary region per region. The stateless application is redeployed into the paired region on a full regional loss, where the data already exists. Batch and scheduled jobs run with multiple replicas. Job execution is monitored with custom metrics and SRE is alerted when a scheduled job does not run, so batch is not a single-instance risk. |
| Additional data needed | Confirm cross-region active-passive is accepted target state for all merchant tiers. hGP HA architecture is not supplied. |
| Engineering recommendation | WDP. It is active-active in region across 3 AZs per region, with controlled active-passive cross-region failover and monitored multi-replica batch. hGP cannot be compared with no data supplied. |
| Rationale | In region is active-active across 3 AZs with a stateless application tier, so a zone failure is transparent with no recovery window. The data and streaming tiers are hardened so acknowledged data survives a broker or AZ loss. HA uses AWS managed-service primitives with probes on every deployment, and batch runs multi-replica with execution monitoring and SRE alerting. |
| Gap to Target | Cross-region is active-passive by design. A full regional outage is a low-complexity, data-resident redeploy and failover, not transparent. This is accepted target state. Active-active cross-region is not required. State the failover window against the enterprise RTO so it is understood. |

---

### TF-6 — Coupling with acquiring platform (data, settlement, identity dependencies)

| Column | Content |
|---|---|
| Category | Technology Foundations |
| Capability | Coupling with acquiring platform (data, settlement, identity dependencies) |
| Segment | All. |
| hWP Disputes Platform | WDP is built to decouple disputes from the acquiring platform. Dispute events from every source converge on a common processing path and a single audited case store, regardless of the originating acquiring platform. Disputes are not re-implemented per acquiring platform. LATAM and VAP are in active development on the common path. The strategic decoupling layer is EDIA streaming. EDIA is work in progress and is targeted for completion by end of PI4. The WDP Portal supports multiple IdP firms, so merchants on different identity providers are served by one platform. Identity is a multi-tenant capability, not a single-provider coupling. |
| Additional data needed | Confirm the acceptable level of residual platform-specific paths after EDIA. Confirm whether settlement and funding reconciliation is independently audited per acquiring platform. hGP coupling model is not supplied. |
| Engineering recommendation | WDP. Disputes run as a decoupled common-path capability across acquiring platforms on one case store, with multi-IdP support and a dated roadmap to extend the common decoupling layer by end of PI4. hGP cannot be compared with no data supplied. |
| Rationale | One case store and a common processing path mean disputes are an independent capability, not a per-platform rebuild. The common decoupling layer is on a dated roadmap with EDIA targeted for end of PI4 and LATAM and VAP in development. Multi-IdP support means one platform serves merchants across different identity providers. |
| Gap to Target | Convergence is progressing. The EDIA decoupling layer is targeted for completion by end of PI4 and LATAM and VAP common-path delivery is in development. Confirm the post-EDIA target end state so fully decoupled has a defined finish line. |

---

### TF-7 — Scheme connectivity architecture (direct vs. gateway, real-time vs. batch)

| Column | Content |
|---|---|
| Category | Technology Foundations |
| Capability | Scheme connectivity architecture (direct vs. gateway, real-time vs. batch) |
| Segment | All. Scheme connectivity is foundational and applies to SMB, Enterprise and Integrated. |
| hWP Disputes Platform | WDP integrates directly with the card schemes for API connectivity. Visa dispute events are retrieved from Visa RTSI queues in near real time. Mastercard is retrieved from MCM queues. Both are direct scheme API integrations. The Interlink PIN network is connected by direct API. Amex, Discover and other PIN networks integrate by scheme and issuer file exchange for dispute and evidence files. NAP is real-time REST API push. WDP pursues direct API connectivity wherever the scheme supports it and uses file exchange only where direct API is not available. Outbound to schemes supports representment, accept and contest submission, questionnaire submission, document submission, and retrieval of issuer documents. This is full two-way dispute exchange. |
| Additional data needed | Confirm whether true streaming real time is required for any merchant tier. Confirm the roadmap to move remaining file-based schemes to direct API where the scheme supports it. hGP scheme connectivity model is not supplied. |
| Engineering recommendation | WDP. It has direct near-real-time API connectivity to Visa, Mastercard and Interlink PIN, broad scheme coverage including Amex, Discover and other PIN by file exchange, and full two-way dispute exchange. hGP cannot be compared with no data supplied. |
| Rationale | Visa, Mastercard and Interlink PIN use direct scheme API integration with near-real-time exchange. Coverage is broad and the exchange is two-way, including questionnaire and document submission and issuer-document retrieval. WDP pursues direct API wherever a scheme supports it and uses file exchange only as a fallback. |
| Gap to Target | Amex, Discover and other PIN networks remain file-based where direct API is not available. Confirm whether moving these to direct API is a target goal or whether file exchange is the accepted end state. Confirm whether near-real-time retrieval meets the target SLA or true streaming real time is required for any tier. |

---

### TF-8 — Post-dispute integration to acquiring back-end (settlement adjustment, funding, reporting)

| Column | Content |
|---|---|
| Category | Technology Foundations |
| Capability | Post-dispute integration to acquiring back-end (settlement adjustment, funding, reporting) |
| Segment | All. Post-dispute money movement is foundational and applies to SMB, Enterprise and Integrated. |
| hWP Disputes Platform | When a dispute resolves, WDP produces the outcome from the single case store and shares it with the relevant acquiring platforms so they can perform settlement adjustment and funding. WDP is responsible for the authoritative delivery of the dispute outcome to each acquiring platform. The acquiring back-end performs the actual money movement. Outcomes are delivered to NAP and CORE acquiring back-ends. LATAM and VAP are in active development on the common path. Outbound delivery uses per-channel outbox tables for idempotency and retry. Automated recovery detects and re-drives orphaned or undelivered rows, so outcomes are delivered exactly once per channel and recover on failure. Dispute outcomes are also reported to merchants across multiple notification channels: Comms Hub, BEN and Dialogue. The strategic universal outbound route is EDIA streaming, work in progress, targeted for completion by end of PI4. |
| Additional data needed | Confirm the EDIA cutover plan for the current acquiring-platform outbound paths. Confirm whether merchants receive a single consolidated outcome notification or per-channel by preference. hGP post-dispute integration model is not supplied. |
| Engineering recommendation | WDP. It delivers post-dispute outcomes to acquiring platforms with idempotent, self-recovering per-channel delivery, plus multi-channel merchant notification, on a dated convergence roadmap to end of PI4. hGP cannot be compared with no data supplied. |
| Rationale | WDP owns the authoritative dispute-outcome handoff to acquiring platforms for money movement, so there is one source of dispute truth feeding settlement. Delivery is idempotent and self-recovering per channel, which suits funding-triggering events. Outcomes are also pushed to merchants across Comms Hub, BEN and Dialogue, so the result reaches both the acquiring back-end and the merchant. |
| Gap to Target | EDIA convergence is in progress and targeted for end of PI4, so the current outbound is platform-specific until then. Define the post-EDIA end state so uniform outbound has a finish line. There is no capability gap on delivery or merchant notification. |

---

### TF-9 — Modularity (can disputes run as a standalone capability across acquiring platforms?)

| Column | Content |
|---|---|
| Category | Technology Foundations |
| Capability | Modularity. Can disputes run as a standalone capability across acquiring platforms? |
| Segment | All. This is the most decisive line for the convergence decision. Modularity across acquiring platforms is the core question this assessment answers. |
| hWP Disputes Platform | WDP is built as a standalone, centralised dispute platform that runs independently of any single acquiring platform. Disputes from every source converge on a common processing path and a single audited case store and are processed the same way regardless of the originating acquiring platform. Disputes are not re-implemented per acquiring platform. WDP exposes defined common inbound and outbound interfaces: API, Kafka and file. A new acquiring platform can integrate through whichever interface suits it, with minimal onboarding effort and no dispute-logic changes. For enrichment WDP uses the acquiring platform API where it exists, and where it does not, WDP integrates directly with the enterprise Cloud Data Platform or data-lake to source the extra transaction and merchant data needed to enrich the dispute. WDP also replicates dispute data into CDP for analytics and reporting. NAP and CORE are in production. LATAM and VAP are in active development on the common path. The universal decoupling and outbound layer is EDIA streaming, work in progress, targeted for completion by end of PI4. |
| Additional data needed | Confirm the post-EDIA target end state, whether 100% common-path or some accepted residual platform-specific integration. A reference onboarding timeline for the most recent acquiring platform would quantify minimal effort. hGP modularity model is not supplied. |
| Engineering recommendation | WDP. Disputes run as a standalone, interface-driven capability across acquiring platforms on one case store and one processing path, with defined common API, Kafka and file interfaces and a flexible enrichment model. This directly answers the central convergence question. Converging onto hGP, whose cross-acquirer modularity is unproven and unevidenced, is itself the main risk this assessment weighs. |
| Rationale | WDP is standalone by design and onboarding is interface-driven, so a new acquiring platform integrates through a contract, not a per-platform rebuild. Enrichment is not tied to one platform because WDP uses the platform API where available and falls back to direct CDP or data-lake access where it is not. The convergence roadmap is dated, with EDIA targeted for end of PI4 and LATAM and VAP in development. |
| Gap to Target | Modularity is proven for NAP and CORE, interface-defined, and being demonstrated for LATAM and VAP. Full universality completes with EDIA, targeted for end of PI4, and LATAM and VAP delivery. Define the post-EDIA target end state so fully modular has a finish line. A published reference onboarding effort would make the modularity claim fully evidenced. |

---

### TF-10 — Scalability and peak processing capacity

| Column | Content |
|---|---|
| Category | Technology Foundations |
| Capability | Scalability and peak processing capacity |
| Segment | All. Scalability is foundational and applies to SMB, Enterprise and Integrated. |
| hWP Disputes Platform | WDP scales horizontally on AWS managed services. Stateless microservices on multi-AZ EKS auto-scale on load. The event backbone is AWS MSK Kafka. The data tier is Aurora PostgreSQL multi-AZ with auto-scaling read replicas. Batch and scheduled jobs run multi-replica. Capacity is load-test evidenced. The platform has been load-tested and supports 500 TPS, validated by CloudWatch and Instana metrics, with substantial spare infrastructure headroom beyond the tested figure. Scaling is demand-led and auto-scaled. As new acquiring platforms are integrated and traffic grows, the target TPS is raised, re-tested, and the AWS infrastructure auto-scales to the new level, so capacity tracks demand and is not fixed. Document lookup on DynamoDB uses partition and key design. Relational scaling on Aurora and RDS uses multi-AZ and auto-scaling read replicas. |
| Additional data needed | Confirm the next target TPS tier as new acquiring platforms onboard and the re-test trigger or cadence, so the demand-led model has a defined governance point. hGP scalability and peak capacity are not supplied. |
| Engineering recommendation | WDP. It scales horizontally on AWS managed services, is load-test evidenced at 500 TPS with spare headroom, and re-tests as acquiring platforms onboard. hGP cannot be compared with no data supplied. |
| Rationale | Current capacity is load-test proven at 500 TPS and is independently observable in CloudWatch and Instana, not a paper target. There is substantial spare infrastructure today and AWS auto-scaling means capacity grows with demand without re-architecture. TPS targets are raised and re-tested as new acquiring platforms integrate, so capacity is matched to the convergence roadmap. |
| Gap to Target | Current evidenced capacity of 500 TPS is sized to current demand. As acquiring platforms converge onto WDP the target TPS must be re-set and re-tested under the demand-led model. Define the re-test trigger so scale-up is governed. There is no architecture gap. |

---

### TF-11 — Data residency and sovereignty controls

| Column | Content |
|---|---|
| Category | Technology Foundations |
| Capability | Data residency and sovereignty controls |
| Segment | All. Data residency and sovereignty is foundational and applies to SMB, Enterprise and Integrated. |
| hWP Disputes Platform | Data residency is supported by the design through the region-pinned data and global application pattern in TF-2. Dispute data is pinned to its required region. A common global application layer connects to the region-specific database based on the merchant platform, so data does not leave its jurisdiction while one application and codebase serves all regions. This is proven in production across two regions. US runs in us-east-2 primary and us-east-1 secondary. UK runs in London primary and Ireland secondary. US dispute data stays in US regions and UK dispute data stays in UK and EU regions, with no cross-region data movement. The pattern is designed to extend to more regions as residency requirements arise, by provisioning data in the required region and deploying the portable application layer there. |
| Additional data needed | Confirm any contractual or regulatory residency commitments by region so the control can be mapped to obligations. Confirm whether the eu-west-2 onboarding bucket usage is fully within the UK and EU residency boundary. This is housekeeping, not a capability gap. hGP data-residency model is not supplied. |
| Engineering recommendation | WDP. Data residency is a proven production capability with US and UK separated today, on a single global application layer, designed to extend to more regions. hGP cannot be compared with no data supplied. Converging onto a platform with an unevidenced residency model is itself a regulatory risk. |
| Rationale | Residency is proven in production across two regions, not designed only. Region-pinned data and a single global application layer give regional separation without forking the platform or growing the operational surface. The same proven pattern extends to more regions as residency requirements arise. |
| Gap to Target | More regions are supported by extending the proven region-pinned pattern: provision data in region and deploy the portable application layer. This is the established path as new residency requirements arise. Map the residency control to specific per-region obligations so the claim is evidenced against named requirements. |

---

### TF-12 — Security and compliance posture (PCI DSS, SOC 2, ISO 27001, GDPR/PSD2)

| Column | Content |
|---|---|
| Category | Technology Foundations |
| Capability | Security and compliance posture (PCI DSS, SOC 2, ISO 27001, GDPR/PSD2) |
| Segment | All. Security and compliance are foundational and apply to SMB, Enterprise and Integrated. |
| hWP Disputes Platform | WDP is PCI-DSS certified. Full evidence is provided and the platform is recertified every year. WDP also maintains SOC 2 Type II annually, GDPR and CCPA data-subject controls, and SOX 7-year immutable audit retention. CVV is not retained as part of the dispute or cardholder data record, in line with the PCI-DSS prohibition on storing CVV after authorisation. PAN is protected with AES-256-GCM encryption, an HMAC-SHA256 HPAN and EPAN two-token strategy, a dedicated EncryptionService boundary as the only handler of plaintext PAN, AWS KMS at FIPS 140-2 Level 3 with IAM isolation, and a crypto audit trail. Clear PAN is not stored on the standard enrichment path. Role-based access control is in place. Clear PAN is shown only after decryption and only to users whose role explicitly allows PAN viewing. Data in transit uses TLS. Secrets are held in AWS Secrets Manager. |
| Additional data needed | ISO 27001 status is not confirmed: held, in progress or not pursued. PSD2 and SCA applicability for in-scope flows is not confirmed. hGP compliance posture is not supplied. |
| Engineering recommendation | WDP. It is PCI-DSS certified and recertified annually, with SOC 2 Type II, GDPR, CCPA and SOX, no CVV retention, strong PAN cryptography, and role-gated PAN access. hGP compliance posture is unevidenced, so converging onto an unknown compliance state is itself a regulatory risk. |
| Rationale | The posture is certified and re-validated every year across PCI-DSS, SOC 2 Type II, GDPR, CCPA and SOX, not a point-in-time claim. PAN cryptography is strong and access is least-privilege, with clear PAN shown only after decryption to explicitly authorised roles. The posture is documented and audited and source-verified, while hGP is unknown. |
| Gap to Target | Confirm ISO 27001 status and PSD2 and SCA applicability so framework coverage is complete. There is no capability gap on this line. |

---

### TF-13 — Tech stack and modernization posture (codebase age, roadmap)

| Column | Content |
|---|---|
| Category | Technology Foundations |
| Capability | Tech stack and modernization posture (codebase age, roadmap) |
| Segment | All. Tech stack and modernization are foundational and apply to SMB, Enterprise and Integrated. |
| hWP Disputes Platform | WDP is a modern, cloud-native, event-driven platform. It uses Spring Boot microservices on the latest stable framework and runtime versions, AWS MSK Kafka, Aurora PostgreSQL, AWS EKS multi-AZ, DynamoDB, S3, AWS KMS, and an Angular SPA portal. Observability uses OpenTelemetry, CloudWatch and Instana through a centralized metric library. The codebase is no more than about 2 years old. It is kept current with frequent Snyk scans that drive dependency upgrades, and there are no end-of-life dependencies. The architecture is decomposed, stateless and scales horizontally. Resilience controls are in place for external API calls. The modernization roadmap has dated items: Ethoca pre-dispute alerts by end of PI4, the dispute analytics dashboard by end of PI4, EDIA streaming as the universal acquiring-platform integration layer by end of PI4, LATAM and VAP onboarding to the common path, and a continued direct-API-first scheme connectivity strategy. |
| Additional data needed | The funded roadmap horizon beyond PI4, the post-EDIA target end state, is not yet defined. hGP tech stack and modernization posture are not supplied. |
| Engineering recommendation | WDP. It is a recent cloud-native event-driven codebase, no more than about 2 years old, on the latest stable versions, with Snyk-driven dependency currency, no EOL dependencies, and a dated in-flight modernization roadmap. hGP cannot be compared with no data supplied. |
| Rationale | The codebase is recent and version-current, no more than about 2 years old, on the latest stable versions, with no EOL dependencies and Snyk-driven dependency hygiene. The architecture is cloud-native and scales horizontally, with resilience controls on external API calls. Modernization is continuous with dated items: Ethoca, dashboard, EDIA, LATAM and VAP. |
| Gap to Target | Define the post-EDIA target end state and the roadmap horizon beyond PI4 so the roadmap has a finish line. There is no codebase-age or stack-currency gap. The platform is recent, version-current and actively maintained. |

---

### TF-14 — Observability and ops tooling (monitoring, alerting, runbooks)

| Column | Content |
|---|---|
| Category | Technology Foundations |
| Capability | Observability and ops tooling (monitoring, alerting, runbooks) |
| Segment | All. Observability and ops tooling are foundational and apply to SMB, Enterprise and Integrated. |
| hWP Disputes Platform | WDP has end-to-end observability across infrastructure and application layers. It uses AWS CloudWatch for infrastructure and managed-service metrics, Instana for APM, and OpenTelemetry on every pod publishing custom and standard telemetry through a centralized metric library. Grafana dashboards give operational visibility and Prometheus alerting is in place. Distributed tracing uses a correlation id for end-to-end request traceability across services. Liveness and readiness probes are on every deployment, so unhealthy pods are detected and recycled. Prometheus alerts and SRE alerting cover platform health, including custom metrics that alert when a scheduled or batch job does not run. A dedicated SRO and SRE team provides 24x7 monitoring and on-call support, with an owned runbook inventory for incident response, failover and recovery. |
| Additional data needed | Alerting SLOs and MTTR targets would strengthen the line. This is an operational metric, not a capability gap. hGP observability and ops tooling are not supplied. |
| Engineering recommendation | WDP. It has full-stack observability with CloudWatch, Instana and OpenTelemetry through a centralized metric library, Grafana dashboards, Prometheus alerts, correlation-id tracing, probes on every deployment, and a dedicated SRO and SRE team on 24x7 with owned runbooks. hGP cannot be compared with no data supplied. |
| Rationale | Observability is full-stack across infrastructure, APM and per-pod telemetry, with dashboards, alerting and correlation-id tracing for end-to-end traceability. The platform is self-healing with probes on every deployment and is actively operated with Prometheus and SRE alerting and a dedicated SRO and SRE team on 24x7 and on-call. Operations are procedural with an owned runbook inventory for incident, failover and recovery. |
| Gap to Target | There is no capability gap on this line. Observability, alerting, tracing, runbooks and 24x7 SRO and SRE operations are in place and aligned to target state. Optionally publish alerting SLOs and MTTR targets in the pack. |

---

## Summary

| # | Capability | Position |
|---|---|---|
| TF-1 | Deployment model | Strength |
| TF-2 | Hosting footprint | Strength |
| TF-3 | Availability SLA vs actual uptime | Strength. Add measured uptime figure |
| TF-4 | Disaster recovery posture | Strength |
| TF-5 | High availability architecture | Strength |
| TF-6 | Coupling with acquiring platform | Strength |
| TF-7 | Scheme connectivity architecture | Strength |
| TF-8 | Post-dispute integration to acquiring back-end | Strength |
| TF-9 | Modularity across acquiring platforms | Strength. Decisive line |
| TF-10 | Scalability and peak capacity | Strength |
| TF-11 | Data residency and sovereignty | Strength |
| TF-12 | Security and compliance posture | Strength. Confirm ISO 27001 and PSD2 |
| TF-13 | Tech stack and modernization | Strength |
| TF-14 | Observability and ops tooling | Strength |

Open data items to close before the panel: trailing-12-month uptime figure for TF-3, ISO 27001 and PSD2 status for TF-12, an optional reference onboarding timeline for TF-9, and an optional load-test headroom ceiling for TF-10.

---

## Category: Capability (5 capabilities sealed)

Competitor is hGP (heritage Global Payments disputes platform). No hGP data was supplied. The Engineering recommendation column states the WDP position and notes where a direct comparison is not possible.

---

### CAP-1 — API and integration surface (REST, webhooks, file-based)

| Column | Content |
|---|---|
| Category | Capability |
| Capability | API and integration surface (REST, webhooks, file-based) |
| Segment | All. The integration surface serves SMB, Enterprise and Integrated merchants and partners. |
| hWP Disputes Platform | WDP provides three integration interfaces: REST API, webhooks, and file. REST APIs follow industry standards. Versioning is URL-based. Backward compatibility is in place. APIs are externalised to merchants and partners through the enterprise API gateway (APIGEE and Akamai). The API rate limit is 50 requests per minute per merchant. Most API responses return in 500 to 800 milliseconds. Broad dispute search returns in 1 to 3 seconds. Webhooks are supported for outbound event delivery to merchants and partners. The file interface uses a common file format. Files are processed near real time. S3 events are integrated with SQS queues and an SQS listener processes the files. Kafka is used for internal event-driven integration. Inbound covers dispute events, evidence and documents. Outbound covers dispute outcomes, questionnaire submission, document submission and merchant notifications across Comms Hub, BEN and Dialogue. |
| Additional data needed | Confirm whether a file-processing SLA is required for target state. There is no defined file SLA today, though files are processed near real time. hGP API and integration surface is not supplied. |
| Engineering recommendation | WDP. It has a standards-based REST API with URL-based versioning and backward compatibility, webhook delivery, and a near-real-time file interface, all externalised through the enterprise API gateway. hGP cannot be compared with no data supplied. |
| Rationale | WDP offers REST, webhooks and file so merchants and partners integrate the way that suits them. REST is standards-based with URL-based versioning and backward compatibility, externalised through APIGEE and Akamai, with a 50 request per minute per merchant rate limit and typical responses in 500 to 800 milliseconds. The file interface processes files near real time using S3 events, SQS queues and an SQS listener. |
| Gap to Target | There is no defined file-processing SLA, although files are processed near real time. Confirm whether a formal file SLA is required for target state. Publish the API rate limit, response targets and versioning policy in the integration contract so it is documented for partners. No capability gap on the interface set. REST, webhooks and file are all in place. |

---

### CAP-2 — Localization (UI and language coverage)

| Column | Content |
|---|---|
| Category | Capability |
| Capability | Localization (UI and language coverage) |
| Segment | All. Localization applies to SMB, Enterprise and Integrated merchants in non-English markets. |
| hWP Disputes Platform | The platform currently operates in English. Localization is planned. The UI is built on Angular and Ionic, both of which provide built-in internationalization frameworks, and the large majority of user-facing content is static UI text held in the presentation layer rather than embedded in backend logic or data. This means localization can be delivered as a front-end content and framework exercise rather than a re-architecture. The dispute processing, data and integration layers are language-neutral and are not affected. Localization is in the delivery pipeline. |
| Additional data needed | The target languages and markets are not yet confirmed. Confirm the priority locale set so the localization scope can be sized. Confirm whether any scheme, regulatory or notification content also requires translation, since merchant notifications go out through Comms Hub, BEN and Dialogue. hGP localization and language coverage are not supplied. |
| Engineering recommendation | hGP cannot be compared with no data supplied. The recommendation is conditional. If localization is required for target state, WDP can deliver it with low effort and low architectural risk because the UI framework supports internationalization natively and content is concentrated in the presentation layer. If hGP already has broad live language coverage, hGP leads on current state for this line until WDP delivers the planned work. |
| Rationale | Angular and Ionic both ship native internationalization support, so no new framework or library is needed. Most user-facing content is static UI text in the presentation layer, so translation is a content exercise, not a logic change. The processing, data and integration layers are language-neutral, so localization does not touch the core platform or its risk surface. |
| Gap to Target | Localization is planned and in the delivery pipeline, not yet built. To close it: confirm the priority locale set, adopt the Angular and Ionic i18n resource model, produce translations, and add locale selection. Effort is low and contained to the front end because the framework support exists and content is presentation-layer. Confirm whether translated merchant notification content (Comms Hub, BEN, Dialogue) is also in scope, as that is a separate content workstream from UI localization. |

---

### CAP-3 — Case management workflow and UI

| Column | Content |
|---|---|
| Category | Capability |
| Capability | Case management workflow and UI |
| Segment | All. Case management serves SMB and Enterprise merchants in self-service, and Worldpay ops in managed-service. |
| hWP Disputes Platform | WDP provides a single Angular and Ionic web portal with two modes. Merchant mode is self-service for merchants to view and action their disputes, including bulk accept and bulk contest. Ops mode is for Worldpay operations to work disputes on the merchant's behalf. The portal covers dispute search, dispute detail, and the full set of dispute actions across the lifecycle, including accept, defend or contest, pre-arbitration, arbitration, retrieval response, reversal and issuer accept. It supports three contest modes: merchant self-assistance, Worldpay-assistance, and retrieval response. Ops mode adds skill-based queue routing so disputes are assigned to the right operator, plus operational sections for administration, fax matching and analytics, and card auth, settlement and dispute history. The portal also provides user management and org management. Access is role-based across merchant, ops and PB user types. The UI is a single-page application, responsive, accessible, and built for performance. The dispute portal is exposed directly to merchants as an enterprise application, and is also accessed through the IQ Enterprise, IQ SMB and Worldpay Dashboard portals. These currently open in a separate tab. Work is in progress to expose the dispute UI as a webpack for a better embedded experience. The IQ SMB and Worldpay Dashboard webpack work is targeted for completion by PI4. Worldpay also has a Global Sign-In platform, and work is in progress to expose the dispute UI from Global Sign-In via webpack for enterprise merchants. |
| Additional data needed | Confirm whether a UI benchmark against hGP is needed. hGP case management workflow and UI are not supplied. |
| Engineering recommendation | WDP. It is a single portal with self-service and managed-service modes, full lifecycle actions, three contest modes, skill-based ops queue routing, plus user and org management, on a modern, responsive, accessible Angular and Ionic single-page front end, accessible directly and through the IQ and Worldpay Dashboard portals. hGP cannot be compared with no data supplied. |
| Rationale | One portal covers both self-service and managed-service, so merchants and Worldpay ops work the same case base. The action set covers the full dispute lifecycle with three contest modes, plus bulk accept and contest for merchants and user and org management. The UI is a responsive, accessible single-page application, accessible directly and embedded through the IQ Enterprise, IQ SMB and Worldpay Dashboard portals. |
| Gap to Target | No core capability gap. The portal covers the dispute lifecycle, contest modes, ops queue routing, user and org management today. Embedded access is being improved: the IQ SMB and Worldpay Dashboard webpack integration and the Global Sign-In webpack integration for enterprise merchants are in progress and targeted for completion by PI4. The Dashboard and Automations portal sections are also planned, UI and UX design finalized, with development targeted for completion by PI4. |

---

### CAP-4 — Decisioning rules engine and auto-respond automation

| Column | Content |
|---|---|
| Category | Capability |
| Capability | Decisioning rules engine and auto-respond automation |
| Segment | All. The rules engine and automation serve SMB, Enterprise and Integrated merchants and Worldpay ops. |
| hWP Disputes Platform | WDP has a configurable rules engine. A dedicated business rules processor evaluates configured rules against every dispute event. Rules drive automated decisioning, including auto-accept and auto-contest, as well as automated routing and automated outbound response and notification. Rule branching is supported per dispute stage and per acquiring platform. WDP is also building LLM-based auto-contest. On merchant document upload, the dispute can be handed to the ops team, and the platform analyses the documents together with dispute and transaction data to auto-generate the Visa questionnaire for ops review and submission, with the manual review step planned for removal as the model matures. This is in development and targeted for completion by PI4. |
| Additional data needed | hGP decisioning and automation are not supplied. |
| Engineering recommendation | WDP. It has a configurable rules engine that auto-accepts and auto-contests disputes, drives automated routing and auto-response, supports per-stage and per-platform branching, and has LLM-based auto-contest targeted for PI4. hGP cannot be compared with no data supplied. |
| Rationale | WDP has a dedicated rules engine that evaluates configured rules against every dispute event. Rules drive automated decisioning, including auto-accept and auto-contest, plus automated routing and outbound auto-response, with branching per stage and per acquiring platform. LLM-based auto-contest is being built to auto-generate the Visa questionnaire from documents and dispute and transaction data, targeted for completion by PI4. |
| Gap to Target | No capability gap. The rules engine auto-accepts and auto-contests disputes and drives auto-response today. LLM auto-contest is in development and targeted for completion by PI4. |

---

### CAP-5 — Evidence management and compelling-evidence packaging

| Column | Content |
|---|---|
| Category | Capability |
| Capability | Evidence management and compelling-evidence packaging |
| Segment | All. Evidence management serves SMB, Enterprise and Integrated merchants and Worldpay ops. |
| hWP Disputes Platform | WDP has a dedicated document management capability. Evidence documents are stored in S3 with metadata in DynamoDB. Evidence is accepted from multiple sources: merchant upload through the portal, file-based intake, API, and Worldpay ops. A questionnaire service drives stage-specific compelling-evidence questionnaires for the relevant dispute stages, including representment, pre-compliance, pre-arbitration and allocation. The UI presents the corresponding compelling-evidence template based on the case stage, so the user provides the information that the stage and scheme require. Document size limits are applied per scheme, so limits vary by network. Document format and document-requirement validation is applied per scheme, based on scheme documentation. Submitted evidence and acknowledgements are retained as immutable records for 7 years. WDP is also building LLM-based auto-contest, which uses the uploaded documents together with dispute and transaction data to auto-generate the Visa questionnaire for ops review and submission. This is in development and targeted for completion by PI4. |
| Additional data needed | hGP evidence management and compelling-evidence packaging are not supplied. |
| Engineering recommendation | WDP. It has a dedicated evidence store with multi-source intake, stage-based compelling-evidence templates presented in the UI, per-scheme size and format validation, 7-year immutable retention, and LLM-based questionnaire generation targeted for PI4. hGP cannot be compared with no data supplied. |
| Rationale | Evidence is centralised in a dedicated store with multi-source intake and metadata. The UI presents the compelling-evidence template for the case stage, so collected evidence matches what the stage and scheme require, with per-scheme size and format validation against scheme documentation. Submitted evidence and acknowledgements are immutable for 7 years, and LLM auto-contest will auto-generate the Visa questionnaire from documents and dispute and transaction data. |
| Gap to Target | No capability gap. Multi-source evidence intake, stage-based compelling-evidence templates, per-scheme validation and immutable retention are in place today. LLM-based auto-generation of the Visa questionnaire is in development and targeted for completion by PI4, which extends current capability rather than closing a gap. |

---

## Summary — Capability category

| # | Capability | Position |
|---|---|---|
| CAP-1 | API and integration surface (REST, webhooks, file-based) | Strength |
| CAP-2 | Localization (UI and language coverage) | Gap. Planned, in delivery pipeline, low effort |
| CAP-3 | Case management workflow and UI | Strength |
| CAP-4 | Decisioning rules engine and auto-respond automation | Strength |
| CAP-5 | Evidence management and compelling-evidence packaging | Strength |

Capability category open items: priority locale set and notification-translation scope for CAP-2; optional file-processing SLA decision for CAP-1.

Not yet in this document: the Data & Intelligence (5) and Architecture & Integration (5) lines were drafted as single sentences only, not full sealed cells, and are held separately pending a decision to expand or compile them.
