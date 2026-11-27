# Week 11 – Application Migration Lab

**Scenario:** Tailwind Traders – migration of a three-tier application from on-premises infrastructure to Microsoft Azure  
**Course:** Cloud Migration (CST8913)  
**Deliverable:** Application discovery, assessment, dependency analysis, validation, migration runbook, and migration waves

> **Basis and assumptions:** This plan follows the instructor's **Week 11 – Application Migration** slides (especially slides 5–14) and the Tailwind Traders scenario. The scenario does not supply actual Azure Migrate results, server CPU/RAM/IOPS, customer geography, or confirmed business-owner answers. Proposed VM sizes, network ports, Azure region, cost assumptions, and unconfirmed business requirements are labeled as examples and require validation. The one-hour outage limit is a firm scenario requirement, not a demonstrated migration result.

## Task 1 – Set Up Tooling for Discovery

### Discovery method and appliances

Use **agentless-first discovery with a mixed approach only when needed**. This follows the course decision-flow diagram (slide 7), which starts with VMware vCenter access and separately covers physical servers and the optional need for per-server dependency agents:

- **Agentless discovery for WEB01, WEB02, and APP01**, hosted on VMware. Use an Azure Migrate discovery and assessment appliance connected to VMware vCenter. It can gather configuration, installed software (where supported and authorized), and performance information without installing a migration agent on each VM.
- **Physical-server discovery for SQL01** using an Azure Migrate physical-server discovery appliance. Discovery is agentless where supported; **deploy per-server agents only if detailed dependency/latency evidence is required or agentless data is insufficient**, after approval and compatibility checks. SQL Server database discovery and migration assessment may also require dedicated SQL discovery functionality or Azure Database Migration tools.
- **LB01, firewalls, backup jobs, DNS, and network paths** must also be documented manually or through appropriate networking tools; they are not automatically represented as migratable VMs.

**Plan for two Azure Migrate discovery appliances:** one for the VMware estate and one for the physical-server discovery workflow. This separates discovery of VMware VMs from SQL01, which is a physical server. Confirm appliance requirements against the current Azure Migrate deployment guidance and network design. A separate migration/replication appliance may be necessary later, depending on the chosen migration method; it is not counted as a discovery appliance here. **Count: two discovery appliances assuming one accessible VMware vCenter and one physical-server scope; add more if separate vCenters, isolation, or appliance capacity limits require them.**

### Credentials and access to prepare

| Discovery function | Credentials or access | Purpose |
|---|---|---|
| VMware infrastructure | vCenter read-only account with adequate inventory and performance permissions | Discover WEB01, WEB02, and APP01 and their configuration |
| Software inventory | Guest OS discovery credentials, such as a least-privilege Windows domain/local account with required remote-management permissions | Identify installed applications, roles, and services |
| SQL discovery | Authorized SQL Server login with the required discovery permissions; Windows account if using integrated authentication; host access as needed | Identify SQL instance, databases, features, compatibility, and configuration |
| Agentless dependency mapping | Authorized Windows server credentials with the required remote discovery permissions (typically remote WMI/WinRM as applicable), plus network access | Collect process and connection dependencies using Azure Migrate appliance |
| Agent-based dependency mapping, if selected | Permission to install/configure supported dependency agents and any required monitoring workspace/connection | Gather detailed dependency/latency observations if agentless mapping is insufficient |
| Azure setup | Azure subscription and appropriate RBAC permissions to create/register Azure Migrate resources | Set up the project and appliances without granting unnecessary access |

Store credentials securely and follow least-privilege practices. Do not put passwords in assessment reports or GitHub.

### Three discovery best practices

1. **Collect a full representative month and check discovery health regularly:** capture weekday, weekend, peak, and nightly-backup behavior; verify the appliances are sending data to the Azure Migrate project (slides 8–10).
2. **Coordinate access with infrastructure and SQL DBAs:** validate vCenter permissions, guest-OS credentials, SQL instance discovery, supported versions, and required firewall connectivity; investigate gaps (slide 8).
3. **Validate and clean the collected data:** compare the inventory and dependencies with the application owner; filter only confirmed noise, preserve meaningful external traffic, and review discovery data at regular intervals (slides 8, 10–11).

## Task 2 – Perform Assessment Planning

| Assessment setting | Proposed choice | Justification |
|---|---|---|
| Assessment type | **Production** workload assessment | This is a production three-tier workload with a strict one-hour outage limit. Also create a **non-production rehearsal assessment** to validate test resources: the lecture discusses both production and nonproduction assessments (slide 9). |
| Azure target region | **Organization's primary approved Azure region (example: Canada Central)** | Slide 9 says to use the company's *primary Azure region*, not a specified geographic region. The scenario does not locate Tailwind; Canada Central is only an illustration if approved for data residency, proximity, latency, capacity, and pricing. |
| Performance history | **One month** of representative performance data, as stated in slide 9 | The instructor explicitly specifies one month; include peaks and nightly backup cycles. If fewer days have been collected, extend discovery rather than treating a short sample as a full month. |
| Sizing method | **Performance-based** | Selects Azure capacity from observed CPU, memory, disk IOPS/throughput, and network demand rather than assuming current VMware allocations are efficient. |
| Comfort factor | **1.2 (20% headroom), proposed** | Allows a modest safety margin for workload variation; adjust after reviewing peaks, growth, and the actual assessment recommendations. |
| Pricing model | **Pay-as-you-go for initial migration**, compare reserved VM pricing after stabilization | Reduces commitment risk during the migration and rightsizing period. Model one- or three-year commitments later for stable workloads. |
| Licensing | **Azure Hybrid Benefit for eligible Windows/SQL licenses**; otherwise license-included | Reuse qualifying Windows Server and SQL Server licenses only after checking Software Assurance/subscription rights and license inventory. |

Assess all application tiers together. Include storage performance, outbound data, backups, monitoring, load balancing, high availability, and licensing in the estimated total cost. Capture unsupported OS/SQL features and application compatibility issues as separate risks.

## Task 3 – Dependency Analysis

### Application components

- **WEB01 and WEB02:** two VMware-hosted web/frontend servers.
- **APP01:** VMware-hosted API and application server.
- **SQL01:** physical server running SQL Server 2017.
- **LB01:** internal load balancer distributing requests to the frontend.
- **Supporting services:** corporate DNS, Active Directory/identity (if used), firewalls, backup system, certificate/PKI services, monitoring, and any external integrations.

### Dependency register

The following are **candidate dependencies, not verified findings**. The team must confirm actual ports and protocols using discovery results, configuration files, and firewall logs.

| # | Source → destination | Candidate protocol/port | Dependency and migration action |
|---|---|---|---|
| 1 | Internal clients → LB01 | HTTPS/TCP 443 | Frontend entry point; maintain a stable DNS name and valid TLS certificate. |
| 2 | LB01 → WEB01 | HTTP/TCP 80 or HTTPS/TCP 443 | Load balancing and health probes for the first web node; confirm backend port. |
| 3 | LB01 → WEB02 | HTTP/TCP 80 or HTTPS/TCP 443 | Load balancing and health probes for the second web node. |
| 4 | WEB01 → APP01 | HTTPS/TCP 443 or configured API port | Web/API communication and authentication headers. |
| 5 | WEB02 → APP01 | HTTPS/TCP 443 or configured API port | Same API dependency for the second web node. |
| 6 | APP01 → SQL01 | SQL Server/TCP 1433 or a configured instance port | Application database queries and transactions; verify named-instance/dynamic-port use. |
| 7 | All Windows servers → DNS | DNS UDP/TCP 53 | Name resolution for application and infrastructure endpoints. |
| 8 | Servers → identity/AD services, if applicable | Kerberos TCP/UDP 88, LDAP/LDAPS 389/636, and related AD ports | Domain authentication, service accounts, and group policy; confirm specific requirements. |
| 9 | Servers → backup infrastructure | Product-specific ports and schedules | Nightly backups, backup consistency, retention, and restore tests. |
| 10 | Servers → monitoring/logging | Agent-specific HTTPS/TCP 443 or configured ports | Health alerts, event collection, audit and operations visibility. |
| 11 | Servers → patch repositories | HTTPS/TCP 443 or configured WSUS ports | Operating-system and application patch distribution. |
| 12 | APP01 or SQL01 → external vendor/SMTP/API endpoints | HTTPS/TCP 443 or SMTP/TCP 587, if applicable | Business integrations, notifications, licensing checks, and allowlists. |

### Noise to filter from dependency data

Slide 10 calls for deduplicating agentless dependency observations across a **one-month** scan, using the documented PowerShell process, and filtering known system processes and non-RFC1918 Internet endpoints. **Do not automatically delete a business-critical public endpoint**: first validate it with the application owner and preserve it as an integration if needed. For agent-based mapping, the slides specify KQL for equivalent analysis and note that its longest available view is one hour. Exclude **confirmed non-business traffic** from the application dependency diagram while retaining relevant infrastructure requirements in a separate register. Examples include routine vulnerability-scanner probes, temporary administrator RDP/SSH sessions, discovery-appliance probes, unrelated broadcast/multicast, and one-off troubleshooting connections. **Do not automatically discard DNS, authentication, backup, monitoring, or patch traffic**: these may be essential operational dependencies even if they are not part of a business transaction.

### Business requirements to confirm with the application owner

| Requirement | Proposed record / validation question |
|---|---|
| Business criticality | Treat as business-critical pending owner confirmation; identify revenue/operations impact and peak periods. |
| Availability and downtime | **Maximum one hour of production downtime** is stated. Obtain approved maintenance window, expected availability/SLA, RTO, and RPO separately. |
| Data classification | Unknown. Confirm whether SQL data contains personal, financial, regulated, or confidential records, along with retention and residency rules. |
| Licensing dependencies | Inventory Windows Server, SQL Server 2017 edition/CALs, Software Assurance/subscriptions, application/vendor licenses, and license servers. |
| Patching requirements | Confirm maintenance windows, patch ownership, patch baselines, reboot restrictions, and test/rollback process. |
| Firewall and IP requirements | Preserve approved client access; map source/destination IPs, ports, subnets, NAT, private routing, allowlists, TLS endpoints, and static IP assumptions. |
| Backup/recovery | Confirm nightly backup timing, retention, encryption, restore testing, log backup frequency, and allowable data loss. |
| Application ownership and users | Obtain named business approver, technical owners, testers, final go/no-go authority, and number/type of internal and external users (slide 11). |
| Monitoring | Confirm alerting, observability tool, escalation process, and monitoring agent dependencies (slide 11). |
| IP/MAC-bound licensing | Explicitly check whether software licensing or vendor controls bind to physical MAC/IP addresses; validate reactivation procedures (slide 11). |

## Task 4 – Validate Assessment Results with the Application Owner

### Preliminary Azure VM size recommendations

These are **illustrative starting points only**, not validated Azure Migrate assessment results. Final selection depends on measured CPU, memory, storage IOPS/throughput, disk capacity, SQL licensing, and availability design.

| Server group | Proposed starting SKU | Reason / evidence still required |
|---|---|---|
| WEB01 and WEB02 | **Standard_D2s_v5**, one VM per frontend | General-purpose starting size for stateless web workloads; verify peak CPU/RAM and availability-zone support. |
| APP01 | **Standard_D4s_v5** | More capacity for API/business logic; review request rates, memory use, and whether a second app instance is required for availability. |
| SQL01 | **Standard_E4ds_v5** | Memory-oriented starting point for SQL Server; measure database RAM pressure, IOPS/latency, log throughput, capacity, and correct licensing. |

### Components to replace or optimize

- **LB01:** replace or redesign using **Azure Load Balancer** for layer-4 internal balancing or **Azure Application Gateway** for HTTP(S) layer-7 needs such as TLS termination, path routing, or WAF. Select based on real application requirements.
- **WEB tier:** spread nodes across availability zones where supported, with health probes and autoscaling/VM Scale Sets considered after compatibility tests.
- **APP tier:** APP01 is currently a single point of failure; evaluate a second instance behind an internal endpoint or load balancer, provided the app supports horizontal scaling/session management.
- **SQL tier:** compare SQL Server on an Azure VM with Azure SQL Managed Instance and Azure SQL Database. Explore availability groups, failover, backup, and disaster recovery based on the chosen platform and required RPO/RTO.
- **Backups and monitoring:** evaluate Azure Backup/SQL-native backup, Azure Monitor, Log Analytics, and tested recovery procedures.

### Owner validation and acceptance gates

Review the discovered dependency register with the application owner, database administrator, network/security team, and application developers. Confirm every required flow, database dependency, scheduled job, integration, certificate, and external allowlist. Resolve unknowns before go-live.

**Review projected costs as well as size, storage and SQL options with the owner (slide 12). Do not claim the SLA or one-hour downtime requirement is met based solely on an assessment.** Prove it with an end-to-end rehearsal, timed database synchronization/failover, DNS/client reconnection tests, and an agreed rollback decision point. Set explicit target RTO/RPO and obtain written business sign-off.

### SQL migration options

| Option | Benefits | Limitations / validation |
|---|---|---|
| **Rehost SQL Server on an Azure VM** | Highest control over SQL instance and OS; often the fewest application changes. | Customer manages SQL patching, HA, backups, and VM operations; needs an online replication/low-downtime cutover approach. |
| **Azure SQL Managed Instance** | Managed database service with broad SQL Server instance compatibility and reduced OS maintenance. | Assess feature compatibility, networking, SQL Agent/jobs, linked servers, authentication, and migration method. |
| **Azure SQL Database** | Fully managed database with platform scaling and minimal infrastructure administration. | More significant compatibility and application changes may be required for instance-level or cross-database functionality. |

**Working recommendation:** assess SQL Managed Instance for long-term modernization, but keep SQL Server on Azure VM as the lower-change fallback if the one-hour window or feature compatibility rules out a platform migration. SQL Server 2017 lifecycle and upgrade needs should also be reviewed.

## Task 5 – Migration Runbook

**Strategy (slides 12–13):** Use application-owner interviews and validated assessment results to document exact load-balancer settings, DNS changes, SQL connection strings, licenses, and application/vendor actions. Prepare and test destination resources before downtime. Replicate SQL data continuously using a supported method and perform only the **final synchronized cutover** inside the one-hour maintenance window. If the selected migration tooling cannot support this, the plan does not satisfy the stated constraint and must be redesigned.

### A. Pre-migration (days/weeks before cutover)

1. Confirm owner approval, scope, RTO/RPO, business-critical flows, change ticket, on-call contacts, and rollback authority.
2. Finish Azure Migrate discovery and assessment; resolve unsupported workloads and unknown dependencies.
3. Build the Azure landing-zone prerequisites: subscription, resource groups, VNet/subnets, private connectivity (VPN/ExpressRoute as appropriate), NSGs/firewall rules, DNS, identity, RBAC, and monitoring.
4. Select final VM/SQL targets from measured usage; reserve required regional quota, capacity, storage, licensing, and network IPs.
5. Configure Azure load balancing and health probes; stage TLS certificates and secrets securely.
6. Configure backups and test restoration of the on-premises SQL database and relevant VM/application configuration.
7. Stage WEB01/WEB02 and APP01 in Azure without redirecting production users. Test connectivity across hybrid links to the existing SQL01 where necessary.
8. Establish and monitor supported SQL migration/replication to the Azure destination; perform initial copy outside the outage window and verify synchronization lag.
9. Test scheduled jobs, integrations, authentication, service accounts, application configuration, firewall rules, and observability.
10. Lower the relevant DNS TTL ahead of the change, following the DNS team's policy; record current values and a detailed rollback plan.
11. Perform a complete dress rehearsal with timing, load tests, database failover tests, and a business-owner acceptance checklist. Do not proceed until projected downtime is under one hour with contingency margin.

### B. Production cutover (target: within 60 minutes)

| Relative time (illustrative) | Action | Go/no-go check |
|---|---|---|
| T-10 to T0 | Announce maintenance; confirm staffing, backup, replication health, current DNS, and final approval. | Stop if replication lag or critical checks fail. |
| T0–T10 | Enable maintenance mode or stop new writes; drain existing requests and scheduled jobs that could change SQL data. | No in-flight writes; rollback still straightforward. |
| T10–T25 | Synchronize remaining SQL changes; perform supported database failover/cutover; confirm integrity and access. | DB consistency, no unacceptable data loss, connections work. |
| T25–T35 | Update APP01 connection strings/secrets to the new SQL endpoint; restart/recycle as required; run API and transaction tests. | App connects and completes read/write tests. |
| T35–T45 | Enable Azure WEB01/WEB02 and APP01 behind the selected Azure frontend/load balancer; verify health probes and firewall flows. | Both frontend nodes healthy. |
| T45–T50 | Change internal DNS record to the Azure private endpoint/VIP as appropriate; check resolution from representative clients. | Clients reach the correct target. |
| T50–T55 | Execute smoke tests: login, page load, API, create/read/update transaction, data integrity, logs, alerts, and nightly-backup scheduling. | Business owner validates critical path. |
| T55–T60 | Final go-live confirmation **only if acceptance criteria are met**; do not begin a rollback at T55. | Record timestamps, outcome, and sign-off. |

The timeline is a **planning target, not a promise**. Set a rehearsed **go/no-go gate by approximately T30–T35**, leaving time for a proven safe rollback; if failback cannot be performed safely in the remaining time, the cutover design must change before production.

### C. Per-server-group execution

- **WEB01/WEB02:** migrate/stage as independent stateless nodes; validate IIS/web runtime, certificates, API endpoint settings, session state, load-balancer probes, and access from clients. Route production traffic only at coordinated cutover.
- **APP01:** migrate/stage service components; validate identity, dependencies, API endpoints, secrets, scheduled tasks, connectivity to SQL, and configuration changes. During cutover, update the SQL connection string via approved secret/configuration management rather than hardcoding credentials.
- **SQL01:** verify SQL version/edition and target compatibility, baseline/full and log backups, database consistency, jobs/logins, and low-downtime replication. During the maintenance window, stop writes, apply final changes, switch the database endpoint, and validate transaction consistency.
- **LB01:** choose and configure Azure Load Balancer or Application Gateway, frontend VIP, backend pool, probes, routing/TLS/WAF requirements, and source-IP restrictions. Decide whether on-premises LB01 is retained temporarily during transition.

### D. Post-migration checklist

- [ ] WEB01 and WEB02 healthy and reachable through the intended load balancer.
- [ ] APP01 API endpoints respond correctly, with successful authentication and application workflows.
- [ ] SQL read/write transactions, row counts, data consistency, logins, and application connection strings validated.
- [ ] Required dependencies, firewall rules, certificates, DNS, and external endpoints tested.
- [ ] Backups configured, successful, monitored, and restorable; nightly schedules verified.
- [ ] CPU, RAM, IOPS, latency, app errors, and alerts within approved thresholds.
- [ ] Failover and recovery procedures documented and tested to meet agreed RTO/RPO.
- [ ] Business owner signs off; help desk and operations receive a handover.
- [ ] On-premises environment retained for the agreed rollback/observation period before decommissioning.

### E. Back-out / rollback plan

1. **Trigger:** failed critical business tests, unacceptable SQL inconsistency, security issue, or forecast inability to finish safely within the approved window.
2. **Freeze new Azure writes** and stop production traffic to Azure to avoid divergent databases.
3. **Confirm the authoritative data source.** If Azure has accepted writes, determine whether reverse synchronization is supported and whether those transactions can be recovered. Never simply point users back at stale SQL data.
4. **Restore the old application path:** reactivate the original SQL01 and on-premises APP/WEB servers, revert connection strings, load-balancer routing and DNS, and reapply documented firewall rules, only after data consistency is established.
5. **Validate** application functionality, database integrity, and client connectivity; notify stakeholders and record the incident.
6. **Escalate** if safe restoration would exceed the one-hour window or breach the approved RPO; invoke the incident/DR procedure. A rollback is not guaranteed to fit in one hour without a rehearsal and a bidirectional data-consistency strategy.

## Task 6 – Plan the Migration Waves

**Important distinction:** these are **preparation and testing waves**, not three independent production outages. The strict one-hour downtime limit applies to a **single coordinated final cutover**. Azure-hosted tiers may temporarily communicate with the on-premises database over secure hybrid connectivity while being tested.

| Wave | Servers/components | Reason and validation |
|---|---|---|
| **Wave 1** | **WEB01, WEB02** (staging only) | Relatively stateless; validate Azure networking, web runtime, health probes, TLS, and rollback without directing production users to the new environment. |
| **Wave 2** | **APP01** (staging only), API and application integrations | More business logic and dependencies; verify web-to-app and app-to-on-prem SQL connectivity, authentication, and transaction testing. |
| **Wave 3** | **SQL01** final synchronization/cutover, **LB01/DNS** production switching, and coordinated application release | Highest data-consistency and business risk; requires database-owner approval, tested replication, validated dependencies, and one-hour cutover rehearsal. |

**Wave acceptance criteria:** each wave has successful connectivity, security and functional testing, performance baselines, and explicit owner approval. Production users remain on the on-premises path until Wave 3's coordinated cutover. For SQL, data replication can begin before Wave 1; only final synchronization and traffic switching occur in Wave 3.

## Slide Alignment and Unresolved Inputs

| Course slide(s) | How it is applied in this plan |
|---|---|
| 5–8 | Four-step framework, discovery decision flow, VMware and physical appliance distinction, inventory/SQL/dependency credentials |
| 9 | Production and nonproduction assessments; primary Azure region; performance-based sizing; **one month**; comfort/pricing/licensing settings |
| 10–11 | One-month deduplicated dependency scan, agentless PowerShell versus agent-based KQL and one-hour view, noise filtering, application-owner interview data |
| 12 | Review VM/storage and SQL options, gaps, cost, and SLA with the owner |
| 13 | Document runbook, load balancer, DNS, SQL connection strings, licenses, pre/post actions |
| 14 | Group workloads into waves based on architecture, dependencies, testing, and criticality |

**Inputs still needed before an actual migration:** one-month assessment output; measured server utilization, disk and database metrics; target country/approved Azure region; SQL features and edition; mapped network flows; verified backup/restore times; business-owned RTO/RPO and SLA; and tested cutover/rollback durations. None of these values has been fabricated.

## Final Recommendation

Use Azure Migrate for discovery and assessment, validate findings with the application owner, stage web and application components first, and prepare a supported low-downtime SQL migration path. Adopt conservative production controls and a single rehearsed cutover with an explicit rollback decision. Final VM sizes, region, database platform, costs, SLA, and RTO/RPO require confirmation against real discovery data and business requirements.
