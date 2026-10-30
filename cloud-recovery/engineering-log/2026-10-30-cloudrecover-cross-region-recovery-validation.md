# CloudRecover — First Successful Cross-Region Disaster Recovery Validation

**Date:** 2026-10-30  
**Project:** CloudRecover  
**Phase:** 7 — Cross-Region Backup and Recovery  
**Milestone:** Recovered PostgreSQL RDS data and restored a functioning NestJS application in the recovery AWS Region

## 1. Session outcome

I successfully brought CloudRecover online in the **recovery region (`eu-west-2`, London)**. I restored an Amazon RDS PostgreSQL instance from a recovery snapshot, accessed the recovered inventory data, and confirmed that my NestJS application was functioning in the recovery environment.

**Result:** The core backup-and-restore process was functionally demonstrated: recover database data, deploy application infrastructure in another region, reconnect the application, and verify the application works.

This is a **successful functional recovery**, not yet proof that the system meets its proposed recovery objectives or that the entire process is automated.

## 2. Architecture involved

| Component | Primary region | Recovery region |
|---|---|---|
| AWS Region | `eu-west-1` (Ireland) | `eu-west-2` (London) |
| VPC address space | `172.40.0.0/16` | `172.20.0.0/16` |
| Application | NestJS inventory API on EC2 | NestJS inventory API on recovery EC2 |
| Database | PostgreSQL on Amazon RDS | New RDS instance restored from snapshot |
| Infrastructure | Terraform modules | Reused Terraform modules with recovery-specific settings |
| Traffic entry | Application Load Balancer | Recovery-region Application Load Balancer |
| Monitoring | Grafana and Prometheus used in primary architecture | Omitted from initial recovery setup |

**Recovery strategy:** Backup and restore, to avoid paying for a continuously running warm standby environment.

## 3. What I accomplished

### 3.1 Provisioned the recovery environment

- Configured the recovery AWS region as `eu-west-2`, with subnets in `eu-west-2a` and `eu-west-2b`.
- Used a dedicated non-overlapping VPC CIDR, `172.20.0.0/16`.
- Reused my Terraform infrastructure modules for networking, security, application compute, load balancing, and database provisioning.
- Adjusted the shared load balancer module so Grafana and Prometheus target groups, listeners, and attachments are optional instead of mandatory.
- Provisioned recovery EC2 infrastructure and accessed the application server for configuration.

### 3.2 Restored the PostgreSQL database

- Worked through the distinction between an RDS *instance identifier* and a PostgreSQL *database name*.
- Used RDS snapshot restoration for the recovery database rather than manually importing a database dump.
- Configured the reusable database module to accept an optional `snapshot_identifier`.
- Resolved the snapshot-location issue: a snapshot in `eu-west-1` is not automatically available in `eu-west-2`; the recovery-region copy must exist before restoration.
- **Successfully launched the recovery RDS instance and accessed the restored inventory data.**

**Key concept:** Restoring an RDS snapshot creates a **new DB instance** with the data from the snapshot; it does not inject the snapshot into an existing RDS instance. The new database has an endpoint that the recovered application must use.

### 3.3 Recovered application functionality

- Configured the private application EC2 instance and its access path through the recovery infrastructure.
- Diagnosed SSH public-key authentication issues involving the bastion and application-server key pairs.
- Diagnosed the application server's inability to fetch a setup script from GitHub: DNS resolution worked, but outbound HTTPS timed out while NAT was disabled.
- Investigated the need for NAT or suitable VPC endpoints and outbound routes during bootstrapping and deployment.
- **Confirmed that the NestJS application is fully functional in the recovery region and can access the restored RDS data.**

## 4. Troubleshooting and lessons

| Issue encountered | Technical lesson |
|---|---|
| Terraform requested Prometheus/Grafana inputs in the recovery module | Optional infrastructure needs optional input variables **and** conditional resources. |
| `count` and `for_each` were both used in an attachment | Terraform resource blocks can use **one or the other**, not both. |
| Missing index on a target group with `count` | Counted resources must be addressed by index, e.g. `[0]`; optional outputs should handle zero instances. |
| Prometheus listener referenced an empty target-group tuple | Conditionally creating a target group also requires conditionally creating the dependent listener. |
| RDS reported `DBSnapshotNotFound` | Cross-region restoration requires a snapshot that is available in the **destination** region. |
| SSH returned `Permission denied (publickey)` | Network reachability and SSH authentication are distinct; check the instance key pair, matching private key, and OS username. |
| `wget` resolved GitHub but HTTPS timed out | DNS resolution does not prove internet access; private subnets require a viable egress path, such as NAT, for public downloads. |

## 5. What today's recovery validates

| Check | Result |
|---|---|
| Recovery-region infrastructure available | **Confirmed** |
| PostgreSQL RDS instance restored from snapshot | **Confirmed** |
| Recovered database data accessible | **Confirmed** |
| NestJS application operational in recovery region | **Confirmed** |
| Application able to use restored database | **Confirmed** |
| Proposed 24-hour recovery point objective (RPO) | **Not yet measured** |
| Proposed 4-hour recovery time objective (RTO) | **Not yet measured** |
| Automated end-to-end recovery | **Not yet demonstrated** |
| Transparent failover of existing clients | **Not yet demonstrated** |

## 6. Remaining work and next engineering steps

1. **Record a reproducible DR drill:** capture snapshot creation time, snapshot copy completion time, recovery start time, service-ready time, and any manual steps.
2. **Validate restored data quantitatively:** compare product and stock-movement counts and check several known records against a documented snapshot baseline. The proposed seed dataset contains 83 products and 326 movements, **if that complete seed was loaded**; actual recovered counts were not recorded in this session.
3. **Measure RPO and RTO:** calculate observed data age at the failure scenario and total elapsed recovery time; do not claim the 24-hour / 4-hour targets until measured.
4. **Automate snapshot creation and cross-region copies:** explore AWS Backup schedules, retention, and successful-copy monitoring.
5. **Automate application recovery:** use the established Terraform, GitHub Actions, AWS OIDC, and SSM workflow, including secure handoff of the restored RDS endpoint and credentials.
6. **Validate repeatability and costs:** document recovery prerequisites, regional access to secrets/artifacts, NAT or endpoint requirements, and resource teardown after testing.
7. **Document client traffic cutover:** while no registered/delegated application domain is configured, the recovery ALB can be tested directly using its AWS-provided DNS name; transparent DNS failover remains a separate concern.

## 7. Engineering reflection

Today's key learning is that **disaster recovery is not just creating a snapshot**. A successful recovery depends on bringing back a working system: network connectivity, compute, database contents, application configuration, deployment dependencies, and a reachable application endpoint.

By restoring PostgreSQL data and running the NestJS API against it in a second region, I have achieved the project's first practical backup-and-restore milestone. The next objective is to make the recovery **repeatable, measurable, secure, and increasingly automated**.

---

**Status:** First functional cross-region recovery successful. Recovery-objective validation and full automation still pending.
