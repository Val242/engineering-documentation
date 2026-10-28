# CloudRecover Learning Log — Cross-Region Disaster Recovery Planning

**Date:** 2026-10-28  
**Project:** CloudRecover  
**Phase:** Phase 7 — Cross-Region Backup and Recovery Preparation  
**Session focus:** Resuming primary infrastructure, understanding recovery objectives, and designing a low-cost cross-region recovery process.

## 1. Session overview

Today I moved from learning disaster recovery terminology to making concrete architecture decisions for CloudRecover. I reviewed how to bring my paused primary environment back online, clarified the relationship between RPO, RTO, backup frequency, and recovery architecture, and began outlining what my secondary AWS Region needs in order to restore the NestJS API and its PostgreSQL data.

My goal is to **learn and implement the recovery process myself**, using guidance to test design assumptions and understand trade-offs—not to have the whole implementation completed for me.

## 2. Resuming the existing AWS infrastructure safely

Previously, I made NAT Gateways optional in Terraform and stopped EC2 instances to reduce idle costs without destroying their configurations. Today I inspected the plan for reactivating the environment with `enable_nat_gateway=true`.

The Terraform plan showed:

- **4 resources to add:** two NAT Gateways and their two Elastic IP addresses.
- **10 in-place changes:** starting the application, Prometheus, and Grafana EC2 instances; restoring default routes in the two private route tables; and updating five administrator-IP security group ingress rules.
- **0 resources to destroy.**

The bastion host was present in the infrastructure but had **no state change** in this plan. The admin-access CIDR was updated from `143.105.152.34/32` to `143.105.152.79/32` in the relevant security group rules.

**What I learned:** Terraform's plan makes it possible to distinguish safe in-place lifecycle changes from destructive replacement. Restoring NAT-based outbound connectivity does not require recreating application servers. The private route-table `0.0.0.0/0` routes can be reintroduced when NAT is enabled.

**Verification boundary:** I reviewed the plan, but did not confirm a successful `terraform apply` in this session.

## 3. Understanding RPO and RTO

I watched disaster recovery videos and developed a clearer understanding of the two principal recovery objectives:

| Objective | Meaning | Question it answers |
|---|---|---|
| **RPO — Recovery Point Objective** | Maximum tolerable data loss, measured in time | How far back could the recovered data be? |
| **RTO — Recovery Time Objective** | Maximum tolerable service downtime | How long can the application remain unavailable? |

Illustrative case: if the most recent usable backup represents database state at **1:45 PM**, the primary region fails at **2:00 PM**, and service returns at **2:40 PM**, the *actual* recovery point is 15 minutes before failure and the *actual* recovery time is 40 minutes. These are observations from the example, not guaranteed targets.

### Initial CloudRecover design targets

- **Target RPO: 24 hours.** Design the backup and cross-region copying process so that the latest usable recovery point is within the allowed data-loss window.
- **Target RTO: 4 hours.** Restore infrastructure, data, application configuration, and a functioning API within four hours of the disaster.

These are **proposed initial learning-project targets**, not measured results or guarantees. A daily snapshot alone does **not** prove a 24-hour RPO: snapshot scheduling, failures, and cross-region copy completion must be taken into account. Similarly, infrastructure provisioning time is only one component of RTO.

## 4. Disaster recovery strategy: backup and restore

I decided to lean toward **backup and restore**, rather than keeping a second fully running deployment available at all times.

The reasoning is cost-aware:

- The primary region serves normal production traffic.
- Recoverable database snapshots are copied to a second AWS Region.
- The recovery application and networking infrastructure can be provisioned **on demand** using Terraform.
- A recovery operation restores an RDS database instance from an available snapshot, deploys the NestJS application, validates the service, and provides a secondary endpoint for users.

**Trade-off:** Lower standby infrastructure costs come with longer recovery time. Provisioning, database restoration, application deployment, testing, and client traffic redirection must all fit within the target RTO.

**Core lesson:** A backup is not a disaster recovery system by itself. Recovery is proven only when the database and application can be restored and made functional in another region.

## 5. Recovery-region networking and Terraform reuse

Because AWS VPCs are regional resources, I can reuse the same Terraform networking module in the recovery region with different region-specific variables. I plan to use a distinct `infrastructure/recovery/` root configuration and a **separate Terraform state** from the primary root.

### CIDR choice

The recovery region's default VPC has `172.31.0.0/16`. We discussed selecting **`172.20.0.0/16`** for the custom recovery VPC so that it does not overlap with that default VPC or the existing CloudRecover primary VPC (`172.40.0.0/16`).

Suggested recovery subnet pattern:

| Subnet | CIDR |
|---|---|
| Recovery VPC | `172.20.0.0/16` |
| Public subnet AZ 1 | `172.20.10.0/24` |
| Public subnet AZ 2 | `172.20.11.0/24` |
| Application subnet AZ 1 | `172.20.20.0/24` |
| Application subnet AZ 2 | `172.20.21.0/24` |
| Monitoring subnet AZ 1 (optional) | `172.20.30.0/24` |
| Monitoring subnet AZ 2 (optional) | `172.20.31.0/24` |
| Database subnet AZ 1 | `172.20.40.0/24` |
| Database subnet AZ 2 | `172.20.41.0/24` |

Important addressing lesson: `172.20.0.0/16` is within the private RFC 1918 range (`172.16.0.0/12`); **`172.40.0.0/16` is not**. The primary VPC already exists, so changing it is a separate decision and not a prerequisite to building the recovery environment.

### Reusable-module review

I drafted a recovery root that references the existing network, security, load balancer, bastion, application, and database modules. During review, I identified necessary adjustments:

- **Provider region:** The recovery root's AWS provider must target the chosen secondary Region; passing a `region` input into a module alone does not choose the provider's region.
- **Availability Zones and AMI:** Use AZs and a valid AMI for the selected secondary Region; AMI IDs are region-specific.
- **Monitoring references:** The load balancer call referenced `prometheus_server` and `grafana_server` modules that were not declared in the recovery root. The load-balancer module should support an app-only recovery deployment if monitoring is excluded.
- **RDS initialization:** The existing database module needs a restoration path rather than always creating a fresh, empty database.
- **State separation:** Primary and recovery environments must not inadvertently manage the same Terraform state or resources.

No recovery-region `terraform apply` was confirmed today.

## 6. What should be recovered?

I identified the resources needed for a first successful recovery drill, rather than duplicating everything in the primary region.

### Core resources

- VPC and non-overlapping subnets.
- Internet Gateway, route tables, and subnet associations.
- Security groups for the ALB, application, and PostgreSQL.
- IAM roles and instance profiles for provisioning and SSM-based deployment.
- NestJS application EC2 instance and its application deployment configuration.
- PostgreSQL RDS instance **restored from a copied snapshot**, plus a DB subnet group.
- Application Load Balancer, application target group, and listener.
- NAT or another required outbound connectivity mechanism, only where necessary.

### Prerequisites that must survive primary-region failure

- Usable cross-region database snapshots and suitable KMS keys/permissions.
- Recovery-region application secrets and credentials.
- Application release artifacts available independently of the failed primary Region.
- Terraform configuration/state and deployment IAM permissions that remain available during recovery.

### Not essential in the first drill

Prometheus, Grafana, and monitoring-specific ALB listeners can be deferred until the API/database recovery works. A separate recovery bastion host may also be unnecessary.

## 7. Bastion host versus Systems Manager

I initially expected to provision a bastion host because I would need to configure the recovered EC2 application server. The distinction I learned is:

- **Terraform provisions EC2** using AWS APIs; it does not require SSH or a bastion to create an instance in a private subnet.
- **SSM can configure and access EC2** without a jump server if the instance has the required agent, IAM instance profile, and connectivity to SSM services.
- **EC2 user data** can perform initial bootstrap tasks.
- **GitHub Actions + OIDC + SSM** can extend the deployment process I already use in the primary Region.

I therefore plan to **omit a bastion from the initial recovery region**, subject to validating SSM connectivity and access. Private instances will need either NAT egress or appropriately configured VPC endpoints to reach the relevant SSM services.

The general principle is to reduce manual server setup during a disaster and make the recovery process reproducible.

## 8. RDS snapshot restoration and the database URL

A key discovery was that I **cannot attach an RDS snapshot to an existing RDS instance** as if it were a disk. Restoring an RDS snapshot produces a **new database instance** with an endpoint that will differ from the primary database endpoint.

This creates a dependency in the recovery workflow:

1. Restore the recovery RDS instance from the copied snapshot.
2. Wait for the restored instance to become available.
3. Obtain the new RDS endpoint from Terraform outputs or AWS APIs.
4. Build or retrieve the recovery application connection configuration.
5. Make that configuration securely available to the recovered NestJS server.
6. Start the application and verify real queries against the restored data.

The new database must also have working credentials, compatible configuration, and the right network/security-group path from the application instance.

### Secrets and configuration handoff

We discussed storing a recovery database connection parameter such as `/cloudrecover/recovery/database-url` in recovery-region **SSM Parameter Store** as a `SecureString`, or using **AWS Secrets Manager**. The deployment role/instance role would need appropriately scoped access to read/decrypt secrets.

My release-based deployment layout already keeps `/opt/cloudrecover-api/.env` outside versioned release directories. The recovery pipeline could securely generate this file from recovery-specific configuration before starting the service.

**Security lesson:** Do not print credentials in GitHub Actions logs or include database passwords in Terraform outputs/state when they can be handled through a dedicated secrets mechanism.

## 9. ALB health checks versus cross-region failover

I clarified the responsibilities of different components:

| Component | Responsibility |
|---|---|
| **Regional ALB** | Performs health checks and routes traffic to registered application targets within its deployment. |
| **Application readiness endpoint** | Can assess critical dependencies such as PostgreSQL connectivity, not merely whether the Node.js process is running. |
| **Route 53** | Can handle DNS-based routing/failover between regional endpoints when I control a usable domain/subdomain. |
| **Terraform and recovery automation** | Actually provisions/restores the recovery infrastructure and database. |

An ALB **does not automatically move requests to an ALB in another AWS Region**. Also, if the recovery environment has not yet been provisioned, merely changing DNS cannot make it available.

Because I currently do not have a domain budget, the practical first drill will use the **AWS-provided ALB DNS names** for primary and recovery endpoints. I can validate that recovery works through the secondary ALB URL and document Route 53 failover as a future improvement. A Route 53 public hosted zone alone does not confer ownership of a domain.

## 10. Proposed recovery sequence

1. Detect or declare a primary-region outage.
2. Confirm that a recent usable RDS snapshot exists in the recovery Region.
3. Use the independent recovery Terraform root to provision the necessary networking, permissions, and application entry point.
4. Restore PostgreSQL RDS from the selected snapshot.
5. Retrieve the restored database endpoint and securely prepare environment/secrets configuration.
6. Bootstrap/deploy the NestJS application through user data, GitHub Actions, and SSM as appropriate.
7. Validate application readiness, connectivity, and retrieval of the restored data.
8. Make the recovery endpoint accessible through the recovery ALB DNS name; add managed DNS failover later if a domain becomes available.
9. Record actual recovery duration and recovered data age, then compare them with RTO/RPO targets.

The precise implementation order may need adjustment as I design Terraform dependencies and the recovery workflow.

## 11. Decisions, assumptions, and open work

| Area | Current status |
|---|---|
| Recovery strategy | **Chosen:** backup and restore |
| Initial target RPO | **Proposed:** 24 hours; not validated |
| Initial target RTO | **Proposed:** 4 hours; not validated |
| Recovery VPC CIDR | **Proposed:** `172.20.0.0/16` |
| Recovery AWS Region | **Still to be finalized/confirmed** |
| Separate recovery Terraform state | **Planned** |
| Cross-region snapshot schedule/copy | **Not yet implemented or verified** |
| KMS and cross-region IAM permissions | **To design and validate** |
| Recovery RDS module | **Needs snapshot-based restore support** |
| Recovery deployment workflow | **To design and implement** |
| Database URL/secret handoff | **Design discussed; not yet implemented** |
| Recovery bastion | **Excluded initially in favor of SSM** |
| Route 53 automatic DNS failover | **Deferred until a suitable domain is available** |
| End-to-end DR drill | **Not yet performed** |

## 12. Next session priorities

1. Finalize the recovery AWS Region and verify regional service/AMI availability.
2. Finish the recovery Terraform root with separate state and the minimal required modules.
3. Review how the RDS module will restore from a cross-region snapshot without creating an empty database.
4. Design snapshot copy frequency, encryption/KMS access, and backup verification against the 24-hour RPO target.
5. Design the recovery deployment/configuration handoff so the application can connect to the restored RDS endpoint without manual SSH.
6. Plan an end-to-end recovery drill and the measurements needed to determine actual RPO/RTO.

## 13. Reflection

Today made the disaster recovery phase clearer. The main challenge is not duplicating EC2 or VPC resources in another Region; it is **recovering consistent data and reconstructing a working application without depending on the failed primary region**. I now understand why RPO/RTO should guide the recovery design, why a copied snapshot has to be restored into a new RDS instance, and why health checks, traffic routing, infrastructure recovery, and secret management are distinct responsibilities.

My next focus is to turn this architecture into a repeatable recovery implementation while making the engineering decisions myself.
