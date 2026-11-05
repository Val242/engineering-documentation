# CloudRecover Recovery Pipeline Progress

**Date:** 2026-11-05  
**Project:** CloudRecover  
**Primary region:** `eu-west-1`  
**Recovery region:** `eu-west-2`  
**Repository:** https://github.com/Val242/CloudRecover

## 1. Today's objective

Continue implementing the Phase 8 automated recovery workflow and verify that the recovery environment can restore the database from a snapshot in the recovery region.

The focus today was getting the GitHub Actions recovery workflow to execute successfully and validating the recovered database and application data.

## 2. Work completed

### 2.1 GitHub Actions recovery workflow

- [x] Created and configured a controlled GitHub Actions recovery workflow.
- [x] Used `workflow_dispatch` to expose the recovery inputs:
  - Source RDS snapshot identifier.
  - Terraform operation (`plan` or `apply`).
  - Recovery confirmation.
- [x] Configured the workflow to authenticate to AWS through GitHub Actions OIDC.
- [x] Configured Terraform to use the S3 remote backend and state locking.
- [x] Implemented snapshot handling that checks for an available recovery-region copy and can copy the source snapshot when a recovery copy is not present.
- [x] Successfully ran the recovery pipeline.

### 2.2 AWS and pipeline configuration troubleshooting

The workflow initially failed because the AWS resources and configuration it depended on were not all correctly prepared.

The troubleshooting work reinforced that a pipeline depends on more than valid YAML. Its external prerequisites must also be configured correctly, including:

- The GitHub OIDC identity provider and IAM role trust relationship.
- IAM permissions for the AWS operations performed by the workflow.
- The S3 backend bucket, backend region, state key, and state locking configuration.
- Recovery-region resources and Terraform inputs.
- Files referenced by the workflow, such as the application server's SSH public key, being available in the checked-out repository.

The workflow's AWS authentication and permissions, remote state configuration, resource dependencies, and input files all need to agree with the workflow's actual operations.

### 2.3 Database recovery

- [x] Selected the primary-region snapshot for recovery.
- [x] Used the recovery-region snapshot copy to restore the database.
- [x] Confirmed that the restored database was available.
- [x] Verified that the recovered database contained the expected product/inventory records.

The snapshot identifiers used during this work were:

- Source snapshot: `cloudrecover-primary-snapshot`
- Recovery-region copy: `cloudrecover-recovery-snapshot`

The recovery-region copy was in `eu-west-2`, with its source pointing to the primary-region snapshot in `eu-west-1`.

**Recovery validation result:** The restored database contained the expected data. No data loss was observed in the records checked during this test. This is an observation from this recovery test, not yet a measured RPO of zero.

### 2.4 Manual application recovery validation

The application server and application configuration were still handled manually during this test:

1. Provisioned the recovery web server manually.
2. Configured the application to use the endpoint of the restored recovery database.
3. Started the application.
4. Checked the application's health and product endpoint.
5. Confirmed that the expected product data was returned.

The following local requests were used to validate the running application:

```bash
curl -i http://localhost:3000/health/live
curl -i http://localhost:3000/health/ready
curl -i http://localhost:3000/products
```

This confirms that the recovered database was usable by the application after manual configuration. It does **not** mean that application-server provisioning and deployment are automated yet.

## 3. Current recovery workflow status

| Capability | Status | Notes |
|---|---|---|
| Controlled GitHub Actions workflow | Complete | Workflow can be dispatched from GitHub Actions. |
| Snapshot selection/check | Complete | Workflow checks for an available recovery snapshot. |
| Recovery infrastructure provisioning | Complete for the current tested infrastructure scope | The workflow ran successfully; application-server provisioning and deployment remain separate manual work. |
| Database restoration | Complete | Recovery database restored from the selected snapshot. |
| Application-server provisioning | Manual | Server was provisioned manually for this test. |
| Application deployment | Manual | Application deployment has not yet been automated end to end. |
| Database endpoint configuration | Manual | Restored RDS endpoint was supplied to the application manually. |
| Application/data validation | Manually verified | Health and product data were checked after configuration. |
| Clear workflow success/failure reporting | Pending | Needs explicit recovery-stage results and final validation reporting. |
| Partial-failure handling and safe retries | Pending | Needs deliberate retry and cleanup behaviour. |

## 4. Important design decision: remove manual database URL entry

The application should not require someone to copy the restored RDS endpoint into its configuration during every recovery.

The planned approach is:

1. Terraform restores the RDS instance from the selected snapshot.
2. Terraform exposes the restored database endpoint and other non-sensitive connection details as outputs.
3. GitHub Actions captures those outputs after Terraform completes.
4. The deployment process supplies the endpoint to the application configuration.
5. Sensitive credentials are retrieved securely, preferably from AWS Secrets Manager, rather than printed in logs or exposed as Terraform outputs.
6. The application is restarted and validated against the restored database.

This separates infrastructure discovery from secret management:

- **Terraform outputs:** endpoint, port, and other non-sensitive infrastructure details.
- **AWS Secrets Manager:** database credentials and other sensitive values.
- **Deployment configuration:** assembles and injects the values the application needs without committing them to source control.

This is the planned design; automatic endpoint and credential injection has not yet been implemented.

## 5. Next implementation tasks

The next work should focus on removing the remaining manual application-recovery steps.

1. Automate recovery EC2 application-server provisioning as part of the workflow.
2. Automate installation of the application runtime and required dependencies.
3. Automate deployment of a compatible application release.
4. Retrieve the restored RDS endpoint from Terraform outputs.
5. Retrieve database credentials securely and inject the complete database configuration into the application.
6. Start or restart the application automatically.
7. Run health checks and verify that expected inventory data is available.
8. Make the workflow report clear success or failure, including which recovery stage failed.
9. Design safe behaviour for partial failures and repeated recovery attempts.

After the end-to-end recovery process is automated, introduce an external trigger if needed. A fine-grained GitHub Personal Access Token (PAT) can authorize an external client to call the GitHub Actions workflow-dispatch API. The PAT should only authorize workflow dispatch; GitHub Actions should continue using OIDC for AWS authentication. The PAT-based trigger has not been implemented and is not required for the current manual GitHub Actions trigger.

## 6. Phase 8–10 progress checklist

### Phase 8 — Automated Recovery Workflow

- [x] Create a controlled GitHub Actions recovery workflow.
- [x] Select an available recovery snapshot.
- [x] Provision the required recovery infrastructure for the current tested scope.
- [x] Restore the database from the selected snapshot.
- [ ] Automate application-server provisioning.
- [ ] Deploy the compatible application release automatically.
- [ ] Configure the application to use the restored database automatically.
- [x] Manually validate application health and restored inventory data.
- [ ] Return clear recovery success or failure results.
- [ ] Handle partial failures and safe retries.

### Phase 9 — Traffic Switching and Disaster Simulation

- [ ] Define the traffic-switching strategy.
- [ ] Simulate a primary-region outage.
- [ ] Run the recovery workflow as part of the disaster simulation.
- [ ] Verify the restored database and application.
- [ ] Switch DNS or the appropriate traffic-routing mechanism only after validation.
- [ ] Measure recovery time and potential data loss.
- [ ] Document the recovery procedure and rollback considerations.

### Phase 10 — Security, Reliability, and Cost Validation

- [ ] Review IAM permissions for least privilege.
- [ ] Validate security-group and subnet isolation.
- [ ] Test application and database failure scenarios.
- [ ] Test recovery workflow failure handling.
- [ ] Verify snapshot encryption and access controls.
- [ ] Review AWS resource costs.
- [ ] Ensure recovery resources can be safely deprovisioned when no longer needed.
- [ ] Verify that protected primary resources cannot be accidentally destroyed by recovery automation.

## 7. PostgreSQL client reference

When connecting to the restored RDS database from an Ubuntu machine, install the PostgreSQL client:

```bash
sudo apt update
sudo apt install postgresql-client -y
psql --version
```

Connect using the restored database endpoint:

```bash
psql -h <RDS_ENDPOINT> -p 5432 -U <DB_USERNAME> -d <DB_NAME>
```

Useful commands inside `psql`:

```sql
\l      -- list databases
\dt     -- list tables in the current database
\q      -- quit
```

The machine making the connection must be able to reach the RDS endpoint, and the database security group must allow inbound TCP port `5432` from the appropriate source, such as the application server's security group or an authorized administration host.

## 8. Key lesson from today

A successful recovery pipeline depends on the correctness and availability of every external dependency it uses: AWS identity and permissions, remote Terraform state, regional resources, snapshot availability, configuration inputs, and deployment artefacts.

Today established that the recovery-region database can be restored from the selected snapshot and that the application can read the recovered data once manually provisioned and configured. The next engineering milestone is to automate those remaining application-recovery steps so that the workflow can complete and validate recovery without manual server setup or database URL entry.
