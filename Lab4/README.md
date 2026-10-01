# Lab 4 Cloud migration discovery and assessment

## 1. Digital Estate Understanding
1. Digital Estate Understanding A digital estate is all the technology resources that an organization uses to run its business. It includes servers, applications, databases, operating systems, storage, networks, and the dependencies between them. Understanding the digital estate is important before planning a cloud migration because the organization needs to know what it has, how the systems are connected, and what each system needs. This helps decide what can move to the cloud and what problems must be solved first. Migrating without enough discovery and assessment can create several risks, like service outages if dependencies are missed, compatibility problems, and incorrect sizing that can increase cloud costs. In this scenario, the organization has five servers. Some information is known, but important details are missing, so more assessment is needed before making final migration decisions.

## 2. Discorevery an assessement approach
I would use Azure Migrate to discover the VMware environment. The Azure Migrate appliance would connect to vCenter and collect information about the virtual machines.
I would collect CPU, memory, storage, operating system, performance and network information. I would investigate WEB-01, APP-01, DB-01, DC-01 and FILE-01.
I would also ask the organization for missing information such as application versions, CPU and memory usage, storage, network dependencies and downtime requirements.
Discovery identifies what exists in the environment. Assessment determines if these servers are suitable for migration to Azure.

## 3. Dependency Analysis
The confirmed dependency is:
- APP-01 → DB-01
Other possible dependencies need to be verified. For example, WEB-01 may communicate with APP-01, and the servers may use DC-01 for authentication. These are assumptions, not confirmed facts.
Azure Migrate dependency analysis can help validate the communication between servers. This reduces the risk of migrating one server while another service it needs is unavailable.
A possible migration group is WEB-01, APP-01 and DB-01 because they may work together.
```mermaid
graph LR
    WEB01[WEB-01] --> APP01[APP-01]
    APP01[APP-01] --> DB01[DB-01]
```

## 4. Azure readiness and suitability
| Server | Readiness Category | Justification and Missing Information | Remediation or Next Step |
|---|---|---|---|
| WEB-01 | Insufficient Information | Windows Server 2019 and IIS are identified, but CPU, memory, storage, utilization, application versions, and detailed dependencies are missing. | Use Azure Migrate assessment to collect performance and dependency information. |
| APP-01 | Insufficient Information | Windows Server 2016 is identified and the dependency on DB-01 is confirmed, but application requirements and performance data are missing. | Assess application compatibility, performance, and the dependency on DB-01. |
| DB-01 | Insufficient Information | The server is mission-critical, but the SQL Server version, configuration, performance, and storage requirements are unknown. | Confirm the SQL Server version and collect database performance and storage information. |
| DC-01 | Insufficient Information | Windows Server 2019 and the identity role are known, but Active Directory configuration and dependencies are not provided. | Review Active Directory configuration and identify services that depend on DC-01. |
| FILE-01 | Conditionally Ready | Windows Server 2012 R2 is a legacy operating system. Migration may be possible, but compatibility and support requirements must be checked. | Assess compatibility and determine whether an operating system upgrade is required. |

## 5. Sizing and cost considerations
- Azure Migrate sizing and cost estimation:
  Azure Migrate collects configuration and performance data from the on-premises VMware environment. It uses this information to recommend suitable Azure VM sizes and estimate the cost of running the workloads in Azure.
- Performance-based sizing vs. As-is sizing:
  Performance-based sizing uses actual resource utilization, such as CPU and memory usage, to recommend an appropriate Azure VM size. As-is sizing recommends a VM based on the current on-premises configuration without considering actual utilization.
- Configuration and utilization data needed:
  To recommend appropriate Azure VM sizes, I would collect the number of CPU cores, memory, disk size and type, storage usage, network usage, and CPU and memory utilization over time.
- Factors affecting Azure operating cost:
  Three factors that could affect the estimated Azure operating cost are the selected VM size, storage requirements, and the Azure region where the resources are deployed.
- Azure VM sizing cost can not be estimated reliably because CPU, memory,storage and utilization details are missing

## 6. Migration prioritization and plan
- Preparation:
Before the migration, we need to check the dependencies, compatibility, and make backups.
- First workload or pilot:
I would first test a non-production workload with low risk. This will help us test the migration without affecting production.
- Migration order:
APP-01 and DB-01 should be migrated together because APP-01 depends on DB-01. DB-01 is mission-critical, so we need to minimize downtime.
- Validation:
After the migration, we need to test the applications, database connections, authentication, and user access.
- Rollback:
If an important service does not work after migration, we should stop and return to the previous on-premises system.
