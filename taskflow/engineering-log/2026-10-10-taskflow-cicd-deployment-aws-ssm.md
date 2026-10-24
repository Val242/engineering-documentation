# TaskFlow API — CI/CD Deployment on AWS

## Overview

Today, I worked on automating the deployment of the TaskFlow API to an AWS EC2 instance using GitHub Actions and AWS Systems Manager.

The goal was to create a CI/CD pipeline that automatically builds, tests, packages, and deploys the NestJS application whenever changes are pushed to the `main` branch.

This session involved troubleshooting several real-world deployment issues, including AWS authentication, shell compatibility, missing dependencies, memory exhaustion, and npm working-directory errors.

## Technology Stack

- **Backend:** NestJS, TypeScript, Node.js
- **Database and ORM:** PostgreSQL, Prisma
- **CI/CD:** GitHub Actions
- **Cloud:** AWS EC2, Amazon S3, AWS Systems Manager
- **Authentication:** GitHub Actions OIDC and AWS IAM
- **Process management:** Linux systemd
- **Monitoring endpoint:** Application metrics and health checks

## CI/CD Pipeline

The deployment process consists of the following stages:

1. **Source checkout:** GitHub Actions checks out the repository.
2. **Dependency installation:** Installs project dependencies using `npm ci`.
3. **Prisma generation:** Generates the Prisma client.
4. **Code quality:** Runs ESLint and unit tests.
5. **Build:** Compiles the NestJS application.
6. **Artifact packaging:** Packages the compiled application, dependency manifests, and Prisma configuration.
7. **Security checks:** Runs Gitleaks and an npm dependency audit.
8. **AWS authentication:** Assumes an IAM deployment role through GitHub OIDC.
9. **Artifact upload:** Uploads the release archive to Amazon S3.
10. **Remote deployment:** Uses AWS Systems Manager to execute deployment commands on EC2.
11. **Application activation:** Installs production dependencies, applies database migrations, and switches the active release.
12. **Verification:** Restarts the systemd service and checks the application's readiness endpoint.

## Deployment Architecture

```text
Developer
    |
    v
GitHub Repository
    |
    v
GitHub Actions
    |
    +--> Install Dependencies
    +--> Lint and Test
    +--> Build NestJS Application
    +--> Package Release Artifact
    |
    v
GitHub OIDC --> AWS IAM Role
    |
    v
Amazon S3
    |
    v
AWS Systems Manager
    |
    v
EC2 Instance (web01)
    |
    +--> Extract Release
    +--> Install Production Dependencies
    +--> Generate Prisma Client
    +--> Apply Database Migrations
    +--> Update Active Release
    +--> Restart systemd Service
    |
    v
Health Check
```

## Troubleshooting and Solutions

### 1. GitHub OIDC Authentication

**Problem:** GitHub Actions initially failed to assume the AWS deployment role because the OIDC subject did not match the IAM trust policy.

**Investigation:** Inspected the repository's OIDC subject configuration and identified the subject format used by GitHub.

**Solution:** Updated the IAM role trust policy to match the repository's immutable subject and the `production` environment.

**Lesson learned:** OIDC trust policies must match the actual claims presented by the identity provider.

### 2. Incorrect S3 Bucket Configuration

**Problem:** The artifact bucket variable contained trailing whitespace.

**Solution:** Corrected the `ARTIFACT_BUCKET` GitHub repository variable.

**Lesson learned:** Invisible whitespace in environment variables can cause cloud resource references to fail.

### 3. Shell Compatibility with AWS Systems Manager

**Problem:** The deployment script failed with:

```text
set: Illegal option -o pipefail
```

**Root cause:** AWS Systems Manager was executing the command through `/bin/sh`, which does not support Bash-specific options such as `pipefail`.

**Solution:** Changed the shell configuration to use POSIX-compatible options:

```sh
set -eu
```

Enabled command tracing with:

```sh
set -x
```

**Lesson learned:** Always understand which shell executes a remote script before using shell-specific features.

### 4. Missing AWS CLI on EC2

**Problem:** The deployment script failed because the AWS CLI was not installed on the target instance.

**Solution:** Installed the AWS CLI:

```bash
sudo apt update
sudo apt install -y awscli
```

**Lesson learned:** Deployment scripts must validate that their required tools are available on the target host.

### 5. EC2 Memory Exhaustion During Dependency Installation

**Problem:** The deployment failed while executing:

```bash
npm ci --omit=dev
```

The process was killed without completing the installation.

**Investigation:** Checked memory, disk usage, and kernel logs:

```bash
free -h
df -h
sudo dmesg -T | grep -Ei 'out of memory|oom-kill|killed process' | tail -n 20
```

The instance had approximately 914 MiB of RAM and no swap configured. The kernel logs confirmed that the Linux Out-Of-Memory (OOM) killer terminated both npm and the running application process.

**Solution:** Created and enabled a 2 GB swap file:

```bash
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
```

Verified the configuration:

```bash
free -h
swapon --show
```

Added the following entry to `/etc/fstab` to enable swap after reboot:

```text
/swapfile none swap sw 0 0
```

**Result:** The subsequent manual production dependency installation completed successfully.

**Lesson learned:** Resource exhaustion can terminate both deployment processes and running applications. Always inspect system-level logs when a process is unexpectedly killed. Swap is a mitigation, not a replacement for sufficient RAM.

### 6. Incorrect npm Working Directory

**Problem:** npm displayed its help output and reported that `npm ci` required a valid lockfile.

**Investigation:** Inspected the npm debug log:

```bash
sudo head -n 25 /root/.npm/_logs/<npm-log-filename>
```

The log revealed that npm was running from:

```text
/etc/taskflow-api
```

This directory contained the application's environment configuration, not the application source and lockfile.

**Solution:** Changed into the extracted release directory before running npm commands:

```bash
cd "$RELEASE_DIR"
npm ci --omit=dev
```

Added checks to verify that the release contained the required files:

```bash
test -f "$RELEASE_DIR/package.json"
test -f "$RELEASE_DIR/package-lock.json"
```

**Result:** Running `npm ci --omit=dev` from the correct release directory installed 264 packages successfully.

**Lesson learned:** `npm ci` must run from a directory containing the project's `package-lock.json`. The `cwd` entry in npm's debug log is especially useful for diagnosing this type of error.

### 7. Prisma Installation Warnings

**Problem:** npm warned that Prisma lifecycle scripts were not covered by the current install-script approval policy.

**Investigation:** Identified warnings involving `prisma` and `@prisma/engines`.

**Action:** Kept the warnings separate from the fatal deployment errors and avoided broadly bypassing npm's install-script protections.

**Lesson learned:** A warning is not necessarily the cause of a deployment failure. Investigate the actual failing command before changing security settings.

### 8. Dependency Audit Findings

**Problem:** npm reported high-severity dependency vulnerabilities.

**Action:** Kept the audit step non-blocking temporarily while investigating the findings. Avoided `npm audit fix --force` because it proposed potentially breaking dependency changes.

**Follow-up:** Review the current audit report and apply compatible dependency updates.

**Lesson learned:** A passing deployment does not mean all dependency vulnerabilities have been resolved.

## Application Deployment and Verification

The application is deployed under:

```text
/opt/taskflow-api
```

Release directories are stored under:

```text
/opt/taskflow-api/releases/
```

The active release is referenced through:

```text
/opt/taskflow-api/current
```

The systemd service is:

```text
taskflow-api
```

The application readiness endpoint is:

```text
http://127.0.0.1:3000/health/ready
```

Useful verification commands:

```bash
systemctl status taskflow-api --no-pager -l

curl -i --max-time 5 \
  http://127.0.0.1:3000/health/ready

journalctl -u taskflow-api -n 100 --no-pager
```

The deployment script also attempts to restore the previous release if the service restart or readiness check fails.

## Key Takeaways

- Implemented automated application deployment from GitHub Actions to AWS EC2.
- Configured GitHub OIDC authentication with an AWS IAM role.
- Used Amazon S3 to store versioned release artifacts.
- Used AWS Systems Manager for remote deployment commands.
- Diagnosed and resolved an EC2 out-of-memory failure.
- Configured persistent swap on Ubuntu.
- Diagnosed an npm working-directory error using debug logs.
- Addressed shell compatibility issues in remote execution.
- Integrated Prisma generation and database migrations into deployment.
- Used systemd and HTTP readiness checks to verify application health.

## Final Result

**The TaskFlow API CI/CD deployment is working successfully.**

The pipeline can build and package the application, upload the release to Amazon S3, deploy it to EC2 through AWS Systems Manager, and verify the running application.

This session provided practical experience in CI/CD automation, AWS identity and access management, Linux troubleshooting, deployment reliability, and production deployment verification.

## Future Improvements

- Resolve the outstanding dependency audit findings.
- Reduce memory usage during deployment or consider an EC2 instance with more RAM.
- Evaluate packaging production dependencies in CI to reduce work on the EC2 host.
- Test rollback behavior in a safe environment, especially when database migrations are involved.
- Improve deployment observability and failure reporting.
