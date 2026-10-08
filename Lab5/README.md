Lab 5 – Rehosting to Microsoft Azure

Course: Cloud Migration (CST8913)
Case Study: Washington State Department of Labor & Industries (L&I)

1. Business Drivers and Migration Strategy

The Washington State Department of Labor & Industries had an old IT environment that had developed over approximately 45 years. The organization had more than 400 servers and over 200 applications and systems.

Two important reasons for moving to Azure were:

First, modernizing the IT infrastructure. The old on-premises environment was becoming difficult to maintain and improve. The organization wanted a more modern and stable infrastructure while continuing to provide important public services.

Second, reducing costs and improving financial planning. Moving to Azure helped the organization reduce operational costs and better predict its future IT expenses. This was important because the agency needed to manage its budget carefully.

Why Rehosting?

Rehosting was a good choice because L&I was already using VMware. With Azure VMware Solution (AVS), the organization could move its existing virtual machines to Azure without making major changes to the applications.

This approach helped reduce migration risks and avoid long interruptions to business operations.

Rehosting vs. Replatforming:

Rehosting: Move an application to the cloud with little or no modification.

Replatforming: Move an application to the cloud while changing some parts of its platform, such as moving a database to a managed Azure database service.

For L&I, rehosting was a practical first step. The organization could migrate quickly and continue modernizing its applications afterward.

2. Architecture Diagram

The following diagram shows a proposed high-level architecture for migrating an on-premises VMware environment to Azure VMware Solution.

flowchart TB
    subgraph ONPREM["SOURCE: On-Premises Data Center"]
        direction TB
        A["VMware Environment"]
        B["Application VMs"]
        C["Database VMs"]
        D["Network and Shared Services"]
        A --> B
        A --> C
        A --> D
    end

    subgraph AZURE["DESTINATION: Microsoft Azure"]
        direction TB
        E["Azure VMware Solution (AVS)"]
        F["Migrated Application VMs"]
        G["Migrated Database VMs"]
        H["Migrated Network and Shared Services"]
        E --> F
        E --> G
        E --> H
        F <-->|"Application and data requests"| G
    end

    D <-->|"Private connectivity - assumed"| E
    B -.->|"Migration Wave 1, 2, 3..."| F
    C -.->|"Migrate with dependent applications"| G

Explanation and Assumptions

The source environment is the organization's existing VMware infrastructure in its on-premises data center.

The destination is Azure VMware Solution, where the migrated virtual machines continue running in a VMware-based environment.

The diagram separates application and database workloads to show their dependencies. Applications may need databases to work correctly, so related systems should be considered together during migration.

Assumptions:

The diagram shows application and database VMs separately for clarity. The case study does not identify the exact application and database architecture.

Private connectivity between the data center and Azure is assumed. The article does not specify whether ExpressRoute, VPN, or another connection was used.

The application-to-database connection represents a possible dependency, not a confirmed design of L&I's systems.

The article confirms that L&I migrated collections of servers, networks, data, and applications in planned segments.

3. Migration and Validation Plan

I would use a phased migration plan to reduce risk and keep business services available.

Step 1: Identify Workloads and Dependencies

Before starting the migration, I would create an inventory of virtual machines, applications, databases, and network services.

I would also identify dependencies between them. For example, if an application needs a specific database, both systems should be included in the same migration group or connected through a tested network path.

This would help avoid situations where an application is moved but cannot communicate with its database.

Step 2: Start with a Small Migration Wave

I would begin with approximately 10 low-risk virtual machines, following the approach described in the case study.

The goal of this first wave would be to confirm that the migration process works correctly before moving more important workloads.

After the first successful wave, I would gradually increase the size of the migration groups. The case study reports that L&I later migrated approximately 75 VMs per wave.

Step 3: Test and Validate

After each migration wave, I would check:

Whether all migrated VMs start correctly.

Whether applications can connect to their databases.

Whether network communication works.

Whether users can access the applications.

Whether application performance and important business functions remain acceptable.

I would compare these results with the tests completed before migration.

Step 4: Proceed or Roll Back

Proceed: If the virtual machines, application connections, and business functions pass validation, I would approve the next migration wave.

Rollback: If a critical application fails, data is inconsistent, or important users cannot access a service, I would stop the migration and return the affected workload to its previous working environment using a tested recovery plan.

I would investigate and correct the problem before trying again.

Step 5: Complete the Migration

After all migration waves are validated, I would confirm that the applications are stable and that the organization can continue its normal operations.

I would keep monitoring performance, security, and costs. Once the migrated environment is stable, the organization could begin replatforming or refactoring selected applications to use more Azure cloud services.

Conclusion

The L&I case study shows that rehosting can be a practical way to move a large VMware environment to the cloud. By migrating related workloads in phases, testing each wave, and preparing a rollback plan, an organization can reduce migration risks while keeping important services running.

Reference

Microsoft. (2024, August 1). Washington State Department of Labor & Industries drives cultural shift and achieves measurable cost savings with Microsoft Azure.

https://www.microsoft.com/en/customers/story/1797135170129986863-lni-azure-vmware-solution-national-government-en-united-states
