# CloudRecover: Cross-Region Snapshot Validation and GitHub Actions Setup

**Date:** 2026-11-03\
**Project:** CloudRecover\
**Focus:** GitHub Actions OIDC authentication, IAM permissions, and
cross-region RDS snapshot handling

## 1. Objective for today

Continue preparing the CloudRecover disaster-recovery workflow so that a
manually triggered GitHub Actions workflow can coordinate recovery in
AWS.

The intended recovery path is:

1.  Identify and validate the primary RDS snapshot in `eu-west-1`.
2.  Copy the snapshot into the recovery region, `eu-west-2`, when
    necessary.
3.  Wait for the recovery snapshot to become available.
4.  Pass the recovery snapshot identifier to Terraform.
5.  Restore the recovery database and continue application recovery.

## 2. GitHub Actions working-directory issue

The workflow initially failed before running its shell command:

``` text
An error occurred trying to start process '/usr/bin/bash'
with working directory
'/home/runner/work/CloudRecover/CloudRecover/infrastructure/secondary'.
No such file or directory
```

The workflow had this setting:

``` yaml
defaults:
  run:
    working-directory: infrastructure/secondary
```

This showed that GitHub Actions could not find the configured directory
in the checked-out repository.

**Lesson:** `working-directory` is resolved relative to the repository
root on the GitHub-hosted runner. The path must match the committed
repository structure exactly. This failure happens before the shell
script starts, so it is not a Terraform error.

The workflow also exposed these configured values:

  Variable            Value
  ------------------- --------------------------------------------
  `DR_REGION`         `eu-west-2`
  `TF_STATE_REGION`   `eu-west-1`
  `TF_STATE_BUCKET`   `tinoscriptdevops`
  `TF_STATE_KEY`      `cloudrecover/secondary/terraform.tfstate`
  `TF_VERSION`        `1.10.5`
  `TF_WORKING_DIR`    `infrastructure/secondary`

The confirmation input was `false`. The workflow is designed to stop
unless the user explicitly confirms the recovery operation. The
working-directory problem must be corrected separately; enabling
confirmation does not fix a missing directory.

## 3. GitHub Actions OIDC trust relationship

Reviewed the trust relationship for the IAM role:

`CloudRecover-GitHubActions-DR-Role`

The workflow uses the GitHub environment:

``` yaml
environment: recovery
```

The trust policy should restrict role assumption to this repository and
environment:

``` text
repo:Val242/CloudRecover:environment:recovery
```

The OIDC provider uses:

-   Provider URL: `https://token.actions.githubusercontent.com`
-   Audience: `sts.amazonaws.com`

The trust relationship answers **who can assume the role**. It does not
grant permissions to perform AWS operations. Those permissions must be
configured in the IAM role's identity-based permissions policies.

## 4. IAM permission issue

The workflow initially received:

``` text
AccessDenied
not authorized to perform: rds:DescribeDBSnapshots
```

The error identified the assumed role as:

``` text
arn:aws:sts::568740659610:assumed-role/CloudRecover-GitHubActions-DR-Role/GitHubActions
```

This established that GitHub Actions had successfully assumed the role
through OIDC. The failure was in the role's permissions, not the trust
relationship.

The error also showed that the workflow was trying to describe the
primary snapshot in `eu-west-1`, while the earlier workflow validation
step was intended to inspect the recovery region, `eu-west-2`.

**Lesson:** Authentication and authorization are separate:

-   OIDC trust controls whether GitHub Actions can assume the IAM role.
-   IAM permissions control which AWS API operations the assumed role
    can perform.

The recovery workflow will need appropriate permissions for snapshot
discovery and, if it performs the copy itself, cross-region snapshot
copying. Those permissions should be restricted as tightly as practical.

## 5. Verified the primary snapshot

Ran the following command against the primary region:

``` bash
aws rds describe-db-snapshots \
  --region eu-west-1 \
  --snapshot-type manual \
  --query 'DBSnapshots[*].[DBSnapshotIdentifier,Status]' \
  --output table
```

Result:

  Snapshot identifier               Status
  --------------------------------- -------------
  `cloudrecover-primary-snapshot`   `available`

This confirmed that the source snapshot exists and is available in
`eu-west-1`.

## 6. Investigated the cross-region copy

Attempted to copy the primary snapshot into `eu-west-2`:

``` bash
aws rds copy-db-snapshot \
  --region eu-west-2 \
  --source-db-snapshot-identifier arn:aws:rds:eu-west-1:568740659610:snapshot:cloudrecover-primary-snapshot \
  --target-db-snapshot-identifier cloudrecover-recovery-snapshot
```

AWS returned:

``` text
DBSnapshotAlreadyExists
Cannot create the snapshot because a snapshot with the identifier
cloudrecover-recovery-snapshot already exists.
```

Rather than deleting or replacing it, inspected the existing destination
snapshot:

``` bash
aws rds describe-db-snapshots \
  --region eu-west-2 \
  --db-snapshot-identifier cloudrecover-recovery-snapshot \
  --query 'DBSnapshots[0].[DBSnapshotIdentifier,Status,SnapshotCreateTime,SourceDBSnapshotIdentifier]' \
  --output table
```

Result:

  -----------------------------------------------------------------------------------------------------------------
  Property                            Value
  ----------------------------------- -----------------------------------------------------------------------------
  Snapshot identifier                 `cloudrecover-recovery-snapshot`

  Region                              `eu-west-2`

  Status                              `available`

  Creation time                       `2026-10-30T05:12:21.695000+00:00`

  Source snapshot ARN                 `arn:aws:rds:eu-west-1:568740659610:snapshot:cloudrecover-primary-snapshot`
  -----------------------------------------------------------------------------------------------------------------

This verified that the destination snapshot already exists, is
available, and was copied from the intended primary snapshot.

## 7. Important recovery and automation concepts

### Regional snapshot availability

An RDS snapshot in `eu-west-1` is not automatically available in
`eu-west-2`. A cross-region copy is required before it can be used to
restore an RDS instance in the recovery region.

### Idempotent snapshot handling

The `DBSnapshotAlreadyExists` response highlighted a requirement for the
workflow: it must handle repeated executions safely.

The intended logic is:

1.  Check whether the destination snapshot exists in `eu-west-2`.
2.  If it exists and is `available`, reuse it.
3.  If it exists and is still `creating`, wait for it to become
    available.
4.  If it does not exist, initiate the cross-region copy.
5.  If the copy fails, report the failure and handle a retry safely.

This avoids failing merely because an earlier execution already created
the recovery snapshot.

### Recovery-point freshness

The existing recovery snapshot was created on **October 30, 2026**.
Reusing it restores the database to that recovery point. It does not
represent the latest primary database state automatically.

The workflow will eventually need a clear snapshot-selection policy so
the selected recovery point meets the project's RPO objective.

## 8. Current state at the end of today's work

**Verified:** - GitHub Actions successfully assumed
`CloudRecover-GitHubActions-DR-Role` through OIDC. - The primary
snapshot `cloudrecover-primary-snapshot` exists and is `available` in
`eu-west-1`. - The recovery snapshot `cloudrecover-recovery-snapshot`
exists and is `available` in `eu-west-2`. - The recovery snapshot's
source ARN points to the expected primary snapshot.

**Not yet completed:** - Correcting the GitHub Actions working-directory
path. - Completing and validating the IAM permissions required by the
workflow. - Automating source snapshot validation, cross-region copying,
and destination availability checks in GitHub Actions. - Passing the
destination snapshot identifier into Terraform and completing the
automated database and application recovery sequence.

The cross-region snapshot itself was already present and available.
Today's work verified its source and destination relationship; it did
not yet implement automatic snapshot copying in the workflow.

## 9. Next work session

1.  Confirm the actual committed repository path for the secondary
    Terraform configuration and correct `working-directory`.
2.  Review the IAM policy for the recovery role, separating snapshot
    discovery/copy permissions from recovery infrastructure permissions.
3.  Update the workflow to check the destination snapshot before
    attempting a copy.
4.  If the destination is absent, copy the source snapshot from
    `eu-west-1` to `eu-west-2`.
5.  Wait for the destination snapshot to become `available`.
6.  Pass the destination snapshot identifier to the secondary Terraform
    configuration.
7.  Continue testing the recovery process while ensuring the primary
    Terraform state and infrastructure remain isolated from recovery
    operations.
