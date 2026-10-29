# CloudRecover — RDS Backup and Recovery Learning

**Date:** 2026-10-29  
**Project:** CloudRecover  
**Focus:** Cross-region disaster recovery, PostgreSQL backup/restore, Terraform module reuse, and recovery-region readiness  
**Phase:** Phase 7 — Cross-Region Backup and Recovery Preparation

## 1. Session overview

Today I strengthened my understanding of AWS disaster recovery strategies and moved from high-level recovery architecture to practical planning around PostgreSQL RDS snapshots. I also reviewed how to reuse my database and load balancer Terraform modules in the recovery region without requiring services that are only deployed in the primary region.

I am building CloudRecover around a **backup-and-restore** approach to avoid paying continuously for a full duplicate production environment. My initial recovery objectives are **RPO: 24 hours** and **RTO: 4 hours**; these are targets to validate with a recovery drill, not yet demonstrated performance.

## 2. Disaster recovery strategies — concepts clarified

| Strategy | Recovery-region readiness | What happens after a disaster |
|---|---|---|
| Backup and restore | Recoverable backups are retained; most application infrastructure can be absent | Provision infrastructure, restore data, configure and deploy the application, then redirect access |
| Pilot light | Minimum core services/data are maintained in the recovery region | Activate or provision the rest of the application and scale up |
| Warm standby | A working, reduced-capacity environment is already running | Increase capacity and route production traffic to it |
| Multi-site active-active | Multiple regions actively serve users | Surviving regions continue serving traffic, subject to their capacity and data consistency design |

Key distinctions:
- **Multi-AZ and multi-Region are different.** Warm standby is not defined by running in only one Availability Zone; it can span multiple AZs.
- **Load Balancers distribute requests** to healthy targets; **Auto Scaling Groups manage instance counts** based on desired capacity and scaling policies. An ALB does not itself provision EC2 instances.
- Active-active architectures require careful coordination of writes, replication, and data consistency.
- CloudRecover's current choice remains **backup and restore**, primarily for cost efficiency.

## 3. What must be protected in CloudRecover?

Most CloudRecover infrastructure is reproducible through Terraform and application deployment automation:

| Component | Planned recovery approach |
|---|---|
| VPC, subnets, route tables, security groups | Recreate with Terraform in the recovery region |
| Application EC2 | Reprovision with Terraform; bootstrap software and deploy through GitHub Actions/SSM |
| NestJS application | Redeploy from source/build artifacts |
| PostgreSQL RDS data | Restore from a cross-region RDS snapshot |
| Application credentials and secrets | Retrieve from a secure store accessible during recovery |
| Terraform state | Preserve in a durable, separately managed backend |

The **RDS snapshot is the main data backup** for this project's current design. S3, EBS, and EFS are not interchangeable backup mechanisms: S3 can hold backup objects, EBS supports volume snapshots, and EFS can be protected with appropriate backup mechanisms. For RDS, the natural recovery mechanism is an **RDS snapshot**.

A regional disaster requires the backup to be available **outside the affected region**. Therefore, creating a snapshot in the primary region alone is insufficient for CloudRecover's cross-region objective.

## 4. Terraform RDS manual snapshot — behavior

The Terraform resource `aws_db_snapshot` requests one manual snapshot of an existing RDS instance.

```hcl
resource "aws_db_snapshot" "cloudrecover_database_snapshot" {
  db_instance_identifier = var.db_instance_identifier
  db_snapshot_identifier = var.db_snapshot_identifier

  tags = {
    Environment = var.environment
    ManagedBy   = "Terraform"
  }
}
```

What I clarified:
1. `db_instance_identifier` refers to the **AWS RDS DB instance identifier**, not inherently to the PostgreSQL database name.
2. In my existing database resource, `identifier = var.db_name` and `db_name = var.db_name`, so they currently have the same configured value, but they remain different concepts.
3. The first `terraform apply` can create a snapshot representing a **point in time**.
4. A second `terraform apply` does **not** refresh that snapshot when new rows have been inserted; Terraform sees the existing snapshot resource as already managed.
5. A newer recovery point requires creating another snapshot (or using a scheduled backup service).
6. A Terraform apply executes the **entire pending plan**, so inspect `terraform plan` to avoid accidental infrastructure changes.

To avoid hardcoding the DB identifier, I can expose it from `modules/database/outputs.tf`:

```hcl
output "db_instance_identifier" {
  description = "Identifier of the CloudRecover RDS instance"
  value       = aws_db_instance.cloudrecover_database.identifier
}
```

The backup module can consume `module.database.db_instance_identifier` in the primary root configuration.

### Backup automation: no SSM required

I initially considered a GitHub Actions workflow with SSM for recurring backups. I learned that **SSM is not required to create RDS snapshots** because snapshots are managed by the AWS RDS API, not by running commands inside EC2.

Possible future automation:
- **AWS Backup**, configured using Terraform, for scheduled backup jobs, retention, and cross-region copies.
- **GitHub Actions + AWS CLI** to explicitly create and copy RDS snapshots on a schedule.

SSM remains useful for **configuration and deployment of the recovery EC2 application**, not for taking managed RDS snapshots.

An RPO target of 24 hours requires monitoring the newest **successfully completed and usable snapshot in the recovery region**. A nominal daily schedule alone does not prove the RPO is being met.

## 5. Module boundaries and state lifecycle

I decided that database backup configuration should be conceptually separate from database provisioning:

```text
infrastructure/
├── modules/
│   ├── network/
│   ├── security/
│   ├── application/
│   ├── database/
│   ├── database-backup/
│   └── load-balancer/
├── primary/
└── recovery/
```

- `database` manages an actual PostgreSQL RDS instance.
- `database-backup` manages backup-related resources and policies.

**Critical insight:** A separate Terraform module is **not** automatically a separate state or lifecycle. If a manual snapshot resource is in the same Terraform root/state as disposable primary infrastructure, `terraform destroy` will also attempt to delete that snapshot. Backups intended to survive teardown need separate lifecycle protection (for example, independent backup management and state, plus appropriate retention controls).

My existing RDS resource currently includes `skip_final_snapshot = true`, so destroying the instance will not automatically generate a final snapshot. I must preserve and verify backups explicitly before any teardown if the data matters.

## 6. How snapshot restoration works

The key clarification was that an **RDS snapshot is not injected into an already running, empty RDS database**. Instead, AWS restores the snapshot as a **new RDS DB instance**.

Recovery sequence:
1. Create a snapshot of the primary RDS instance.
2. Copy the snapshot to the recovery AWS Region and wait for it to be available.
3. Provision recovery networking, DB subnet group, security groups, and permissions.
4. Restore a **new** RDS instance using the copied snapshot.
5. Obtain the new RDS endpoint and configure the recovery NestJS `DATABASE_URL` securely.
6. Deploy the application through the existing GitHub Actions/SSM approach.
7. Test the API and confirm the backed-up inventory records exist.

The Terraform RDS resource supports:

```hcl
snapshot_identifier = var.snapshot_identifier
```

I do **not** need duplicate `aws_db_instance` definitions in a reusable module. One module can support both provisioning modes:

- **Primary:** `snapshot_identifier = null`, configure engine, database name, and credentials for a fresh database.
- **Recovery:** provide the identifier of the already copied snapshot, and avoid supplying fresh-creation-only arguments such as initial database name or username/password where inappropriate.

Module variable:

```hcl
variable "snapshot_identifier" {
  description = "Snapshot identifier used to restore the RDS instance"
  type        = string
  default     = null
}
```

The recovery root must pass the snapshot identifier and use an AWS provider configured for the recovery region. Changes to `snapshot_identifier` on an existing managed instance can trigger replacement; I must **inspect the plan** and should not retrofit snapshot restoration onto the running primary instance.

## 7. Preparing inventory data for a recovery drill

I prepared an expanded SQL seed script based on the CloudRecover inventory tables created through Prisma migrations.

Expected dataset from the expanded script:
- **83 products** (including the original three).
- **326 stock movements** (including the original six).
- **8 products with zero quantity**.
- Product quantities designed to reconcile with their movement histories.

The data is **test seed data**, not a PostgreSQL dump or a true backup. The intended procedure is to populate the primary database, take an RDS snapshot, restore it in the recovery region, and compare the recovered records.

To execute the SQL while connecting to RDS from EC2:

```bash
psql -h <RDS_ENDPOINT> -p 5432 -U <DB_USERNAME> -d <DATABASE_NAME> -v ON_ERROR_STOP=1 -f backup.sql
```

Or, if `DATABASE_URL` is present in the shell:

```bash
psql "$DATABASE_URL" -v ON_ERROR_STOP=1 -f backup.sql
```

The SQL file must be available on the EC2 filesystem, and Prisma migrations must already have created the tables. The seed script is intended for an empty test database; rerunning it unchanged would conflict with existing primary keys.

Verification queries:

```sql
SELECT COUNT(*) FROM "products";
SELECT COUNT(*) FROM "stock_movements";
SELECT COUNT(*) FROM "products" WHERE "quantity" = 0;
```

I also discussed testing the NestJS API via its products endpoint, for example `curl http://localhost:3000/products` **if that matches the actual controller route and API prefix**. The endpoint route and successful test results were not independently verified in this session.

## 8. Reusing the ALB module without monitoring in recovery

In the primary region, I have been using **one Application Load Balancer** with separate target groups/listeners for:
- NestJS application
- Prometheus
- Grafana

In the recovery region, I do not initially plan to deploy Grafana or Prometheus. Mandatory references to those modules in the ALB root configuration cause errors when their modules are absent.

**Decision:** Make monitoring-specific resources optional in the shared `load-balancer` module, while retaining the application target group and listener as mandatory.

Planned approach:

```hcl
variable "enable_monitoring" {
  description = "Create Grafana and Prometheus ALB resources"
  type        = bool
  default     = false
}
```

- **Primary root:** `enable_monitoring = true`.
- **Recovery root:** `enable_monitoring = false`.
- Conditional creation must cover **target groups, listeners, and target-group attachments**, not just target groups.
- Monitoring instance ID inputs can default to empty lists so the recovery root does not need to reference nonexistent Grafana and Prometheus modules.

**Terraform state warning:** Adding `count` changes existing resource addresses, such as `aws_lb_target_group.grafana` becoming `aws_lb_target_group.grafana[0]`. If primary monitoring resources still exist in state, use appropriate `moved` blocks (including corresponding listeners and attachments where applicable) or a carefully planned state migration, rather than inadvertently recreating resources.

Also review any monitoring outputs that assume the resources always exist.

## 9. What is planned versus verified?

**Conceptually understood / design decisions made today:**
- Differences among backup-and-restore, pilot light, warm standby, and active-active DR.
- RDS manual snapshot behavior across repeated Terraform applies.
- Need for cross-region snapshot copies and independent backup retention.
- Restoring a snapshot creates a new RDS instance.
- Optional snapshot-restoration input for the shared database module.
- Optional monitoring target groups/listeners/attachments in the shared ALB module.
- Using AWS Backup or GitHub Actions for scheduled backups rather than SSM commands on EC2.

**Not yet confirmed by outputs in today's discussion:**
- Successful execution of the expanded SQL seed against RDS.
- Creation and availability of a manual RDS snapshot.
- Successful copy of that snapshot to the recovery region.
- Restoration of RDS from the copied snapshot.
- Application response or record-count verification after restoration.
- Successful Terraform plan/apply after making monitoring optional.

## 10. Tomorrow's starting point

1. Update the `load-balancer` module so Grafana/Prometheus target groups, listeners, attachments, and outputs are optional; ensure the recovery root has no references to absent monitoring modules.
2. Review `terraform plan` in the primary and recovery roots, watching for resource address migrations and unintended destruction.
3. Confirm the reusable database module supports `snapshot_identifier = null` for fresh provisioning and a valid copied snapshot identifier for restoration.
4. Create and verify the first manual primary RDS snapshot, then implement and test the cross-region snapshot copy.
5. Restore the snapshot to a new recovery RDS instance and compare the recovered inventory records against the source.
6. Once the manual recovery path works, design scheduled backups, retention, and monitoring to support the 24-hour RPO.

## 11. Key takeaway

> Infrastructure as Code recreates the *environment*; RDS snapshots recover the *data*. A successful disaster recovery process joins both with secure configuration, automated deployment, and validation. Backups must survive the failure or teardown of the primary environment, and recovery objectives must ultimately be measured through a real drill.
