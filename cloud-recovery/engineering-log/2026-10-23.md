# CloudRecover Learning Log — October 23, 2026

## Focus

Today, I focused on improving CloudRecover's deployment automation by debugging GitHub Actions authentication with AWS, resolving remote deployment failures, and aligning EC2 provisioning with the release-based deployment workflow.

The main objective was to make the deployment process more consistent, repeatable, and easier to troubleshoot.

## 1. GitHub Actions Authentication with AWS OIDC

### What I worked on

- Investigated why GitHub Actions could not initially assume the AWS deployment role.
- Debugged the IAM OIDC trust policy and determined the expected GitHub OIDC `sub` claim.
- Corrected the trust configuration.
- Investigated an `ssm:SendCommand` access-denied error after addressing role assumption.

### What I learned

**An IAM trust policy determines who can assume a role, while a permissions policy determines what the assumed role can do.**

GitHub OIDC allows GitHub Actions to authenticate with AWS using temporary credentials rather than storing long-lived AWS access keys in GitHub secrets.

My trust-policy debugging forced me to investigate the identity context GitHub Actions presents to AWS, particularly the `sub` (subject) claim.

Because my deployment workflow uses the GitHub Actions `production` environment, the expected subject was:

`repo:Val242/CloudRecover:environment:production`

This helped me understand that the OIDC subject is determined by the workflow's context. The IAM trust policy must match the repository and environment identity represented by the claim.

I also learned to distinguish two separate failure types:

- **Role-assumption failure:** GitHub Actions cannot assume the IAM role because its OIDC claims do not satisfy the trust policy.
- **AWS authorization failure:** The role has been assumed, but the requested AWS action is not permitted by its permissions policy.

Resolving the trust-policy mismatch did not automatically resolve the subsequent `ssm:SendCommand` denial. That required separate investigation of the role's AWS permissions and resource ARNs.

### Understanding the deployment identities

The GitHub Actions role and EC2 instance profile have different responsibilities.

- **GitHub Actions IAM role:** Uploads the deployment artifact to S3 and initiates and checks Systems Manager commands.
- **EC2 instance profile:** Provides the instance with permissions to retrieve deployment artifacts from S3 and operate as an SSM managed node.

The resource ARNs in the permissions policy must match the actual AWS account, Region, EC2 instance, and SSM document.

### Key lesson

When debugging GitHub-to-AWS authentication, first establish whether the workflow can assume the role. Only after authentication succeeds should I investigate authorization failures for individual AWS actions.

## 2. Debugging the Remote Deployment Failure

### Failure encountered

After resolving the IAM issues, the SSM command reached the EC2 instance but failed with exit status `127`.

### Root cause

The deployment script attempted to execute:

`aws s3 cp`

However, AWS CLI v2 was not installed on the EC2 instance.

The workflow could authenticate with AWS and invoke SSM, but the remote script could not download the deployment artifact because the required executable was missing from the target machine.

### Resolution

I installed AWS CLI v2 on the EC2 instance, after which the deployment succeeded.

I then incorporated AWS CLI installation into the EC2 provisioning script so that a newly provisioned instance would have the tool required by the deployment workflow.

The installation logic detects the machine architecture and downloads the corresponding AWS CLI package.

### What I learned

A successful remote command invocation does not guarantee that the commands inside the script can execute successfully.

The target machine has its own runtime environment and dependencies. Provisioning must install or validate the tools that deployment automation requires rather than assuming they are already available.

This also reinforced the importance of debugging failures at the correct layer: an IAM issue, a missing executable, and an application startup failure require different investigations.

## 3. Aligning EC2 Provisioning with Release-Based Deployment

### Issue discovered

While reviewing the provisioning script, I identified a mismatch between the application directory used by systemd and the directory structure used by GitHub Actions.

The original systemd configuration ran the application directly from:

`/opt/cloudrecover-api`

However, the deployment workflow created versioned releases and updated:

`/opt/cloudrecover-api/current`

This meant the deployment workflow could switch the `current` symlink while systemd continued running the application from the original directory.

The problem was not simply an incorrect systemd path. The provisioning and deployment processes used different assumptions about how an application release becomes active.

### Intended directory structure

- `/opt/cloudrecover-api/.env` — persistent environment configuration stored outside versioned releases.
- `/opt/cloudrecover-api/releases/<release-id>/` — individual application releases.
- `/opt/cloudrecover-api/current` — symlink pointing to the active release.

This structure separates persistent configuration from application versions and provides a stable path for the running service.

### Correct systemd configuration

The service should use the `current` symlink:

`WorkingDirectory=/opt/cloudrecover-api/current`

`ExecStart=/usr/bin/node /opt/cloudrecover-api/current/dist/src/main.js`

With this configuration, systemd can use the active release without requiring its service definition to change every time a new version is deployed.

### Why bootstrap initialization matters

Changing the systemd paths alone is insufficient.

During initial provisioning, the script must create a valid bootstrap release and initialize the `current` symlink before starting the service. Otherwise, systemd may attempt to start an application from a path that does not yet exist.

Provisioning and deployment must therefore agree on the same directory structure from the beginning.

## 4. Understanding the Release Deployment Lifecycle

The release-based deployment model follows a sequence in which a new application version is prepared before it becomes active.

The intended process is:

1. **Prepare the release:** Download and extract the versioned application artifact into its release directory.
2. **Install dependencies:** Install the production dependencies required by that release.
3. **Prepare Prisma:** Generate the Prisma client and apply database migrations according to the deployment strategy.
4. **Activate the release:** Update the `current` symlink to point to the prepared release.
5. **Restart the service:** Restart `cloudrecover-api.service` so the application runs from the active release.
6. **Verify readiness:** Check the application's readiness endpoint.
7. **Recover from failure:** If activation or the health check fails, restore the previous symlink and restart the service.

This approach makes application versions explicit and creates a practical path for rollback.

However, rollback must be implemented and tested rather than assumed to work. Database migrations may also introduce changes that cannot safely be reversed by switching the application symlink alone.

## 5. Updating the CloudRecover Provisioning Script

I prepared a corrected provisioning script to align the EC2 setup with the GitHub Actions release-based deployment workflow.

The intended corrections include:

- Installing AWS CLI v2 during provisioning.
- Initializing the release directory structure.
- Creating the initial bootstrap release and `current` symlink.
- Configuring systemd to run the application through `current`.
- Keeping `.env` outside versioned release directories.
- Preserving the existing Node Exporter, firewall, and application health-check configuration.

The script's Bash syntax was checked using `bash -n`. This confirms that the script parses, but it does not prove that the complete script has been executed successfully on EC2 or that all runtime behavior is correct.

### Remaining validation

Before considering the deployment architecture fully validated, I need to:

- Confirm that bootstrap initialization creates the correct release structure.
- Verify that `.env` remains outside release directories and is accessible to the running application.
- Confirm that dependencies and Prisma commands execute from the expected directories.
- Verify the EC2 instance profile's S3 permissions and SSM connectivity.
- Confirm that the deployment workflow switches `current` to the intended release.
- Verify that systemd runs the newly activated version.
- Check `/health/ready` after deployment.
- Test rollback in a controlled manner.

## 6. Key Takeaways

### 1. OIDC trust policies require exact identity matching

Debugging the trust policy taught me to investigate GitHub Actions' identity context and determine the expected `sub` claim for the repository and environment.

### 2. Authentication and authorization are separate concerns

Being able to assume an IAM role does not mean that every AWS action is permitted. Trust-policy conditions and permissions policies solve different problems.

### 3. Remote deployment depends on the target environment

The missing AWS CLI demonstrated that the deployment workflow's success depends on the dependencies available on EC2, not just on the workflow configuration.

### 4. Provisioning and deployment must share a consistent contract

The provisioning script, deployment workflow, directory structure, environment configuration, and systemd service must agree on how the active release is selected and executed.

### 5. Versioned releases improve deployment control

Keeping application versions in separate release directories and using a `current` symlink provides a stable service path and supports rollback when the workflow implements the required recovery steps.

### 6. Successful commands do not guarantee application readiness

SSM command completion is not equivalent to application health. A deployment must verify that the application starts and responds successfully to its readiness endpoint.

## 7. Next Session

1. Review the corrected provisioning script against the GitHub Actions deployment workflow.
2. Verify bootstrap release initialization, the `current` symlink, and `.env` handling.
3. Confirm the live OIDC trust policy, IAM permissions, EC2 instance profile, S3 access, and SSM connectivity.
4. Deploy a new release and verify that `current` points to the expected version.
5. Check `systemctl status cloudrecover-api` and inspect logs with `journalctl -u cloudrecover-api`.
6. Verify `http://127.0.0.1:3000/health/ready`.
7. Test rollback deliberately and confirm that the previous release can be restored.

## Reflection

Today's work helped me understand CloudRecover's deployment process as an integrated system rather than a collection of independent AWS services and scripts.

I traced an OIDC trust-policy mismatch to the workflow's expected subject claim, separated that problem from AWS permission errors, identified a missing dependency on EC2, and discovered a mismatch between provisioning and release activation.

The central lesson is that reliable deployment automation depends on consistency across identity, permissions, artifacts, remote execution, application releases, and service management. Each component must work with the others for a deployment to be repeatable and recoverable.