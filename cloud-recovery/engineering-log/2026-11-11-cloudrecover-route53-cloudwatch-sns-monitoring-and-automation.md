# CloudRecover – Daily DevOps Progress Report

Date: Wednesday, 11 November 2026\
Project: CloudRecover – Multi-Region AWS Disaster Recovery Platform\
Focus: Route 53 health checks, CloudWatch monitoring, SNS notifications, and automated disaster recovery preparation.

## 1. Studied Route 53 Health Checks

Today, I worked on understanding how Amazon Route 53 health checks operate and how they can be integrated into the CloudRecover disaster recovery architecture.

Topics covered:

- How Route 53 health checks monitor application availability.
- The difference between HTTP health checks and calculated health checks.
- How health checks can monitor an application's readiness endpoint.
- How private hosted zones work.
- Whether primary and recovery regions require separate health checks.
- Why Route 53 health checkers need publicly accessible endpoints when performing HTTP checks.

I established that the primary application's health check should monitor the public-facing Application Load Balancer rather than an internal application endpoint.

### Primary health check configuration

I configured a Route 53 HTTP health check with the following settings:

| Configuration     | Value                                                             |
| ----------------- | ----------------------------------------------------------------- |
| Endpoint          | Primary Application Load Balancer                                 |
| DNS name          | `cloudrecover-primary-alb-1719900429.eu-west-1.elb.amazonaws.com` |
| Protocol          | HTTP                                                              |
| Port              | 80                                                                |
| Path              | `/health/ready`                                                   |
| Request interval  | 10 seconds                                                        |
| Failure threshold | 2                                                                 |

The purpose of `/health/ready` is to verify that the application is actually ready to serve requests, including checking its database dependency.

I also worked on understanding how Route 53 health check status can be used as a CloudWatch metric.

## 2. Designed the Automated Disaster Recovery Trigger

I worked on the architecture that will initiate disaster recovery when the primary environment becomes unhealthy.

The intended architecture is:

Route 53 Health Check → CloudWatch Alarm → SNS → Lambda → GitHub Actions.

I established the responsibility of each component:

1. Route 53: Monitors the availability of the primary application.
2. CloudWatch: Evaluates the health check metric and determines when the primary environment becomes unhealthy.
3. SNS: Publishes notifications when the alarm enters the ALARM state.
4. Lambda: Receives the SNS notification and communicates with GitHub.
5. GitHub Actions: Executes the automated disaster recovery workflow.

I also clarified an important distinction: SNS does not directly execute GitHub Actions. Lambda will be responsible for calling GitHub's workflow dispatch API.

The Lambda function will need an appropriate GitHub credential, such as a fine-grained personal access token with Actions write permission or a GitHub App token.

## 3. Configured Terraform Provider Regions

One of the major challenges today was configuring the correct AWS provider region for CloudWatch alarms.

My infrastructure uses:

- `eu-west-1` for the primary environment.
- `eu-west-2` for the recovery environment.
- `us-east-1` for CloudWatch Route 53 health check metrics.

I initially encountered a CloudWatch error:

`ValidationError: Invalid region eu-west-1 specified. Only us-east-1 is supported.`

I investigated the regional provider configuration and worked with Terraform provider aliases.

The root Terraform configuration includes three AWS provider configurations:

- Default AWS provider: `eu-west-1`.
- Recovery provider alias: `recovery`, configured for `eu-west-2`.
- Virginia provider alias: `virginia`, configured for `us-east-1`.

I also configured the Route 53 Terraform module to accept the Virginia provider alias through `configuration_aliases`.

The CloudWatch alarm resource was configured to use:

`provider = aws.virginia`

This ensured that the CloudWatch alarm itself would be created in `us-east-1`.

## 4. Investigated and Resolved the SNS Regional Configuration Issue

After configuring the provider alias, I still encountered the same regional validation error.

The important issue was that the alarm was referencing an SNS topic located in `eu-west-1`, while the CloudWatch alarm was being created in `us-east-1`.

I addressed this by introducing a new SNS topic in the same region as the CloudWatch alarm.

New SNS topic:

`cloudrecover-route53-health-alerts`

Region:

`us-east-1`

This allows the CloudWatch alarm to publish notifications to an SNS topic in its own region.

The existing primary SNS topic in `eu-west-1` was not destroyed or modified by this successful apply.

## 5. Successfully Created the CloudWatch Alarm and SNS Topic

After correcting the regional configuration, I ran Terraform plan again.

The final plan showed:

- 2 resources to add.
- 0 resources to change.
- 0 resources to destroy.

I reviewed the plan and approved the Terraform apply.

Successfully applied

The following resources were created:

### A. CloudWatch Metric Alarm

Resource:

`module.route53.aws_cloudwatch_metric_alarm.primary_health[0]`

Alarm name:

`cloudrecover-primary-health-alarm`

Configuration:

| Property               | Value               |
| ---------------------- | ------------------- |
| Region                 | `us-east-1`         |
| Namespace              | `AWS/Route53`       |
| Metric                 | `HealthCheckStatus` |
| Statistic              | `Minimum`           |
| Period                 | 60 seconds          |
| Evaluation periods     | 2                   |
| Datapoints to alarm    | 2                   |
| Threshold              | 1                   |
| Comparison operator    | `LessThanThreshold` |
| Missing data treatment | `breaching`         |

The alarm monitors the Route 53 health check using its health check ID:

`15a1c8ed-d542-5102-0876-7a0d72661e78`

Its purpose is to detect when the primary application health check reports an unhealthy status.

### B. SNS Topic

Resource:

`module.route53.aws_sns_topic.route53_health_alerts[0]`

Topic name:

`cloudrecover-route53-health-alerts`

Region:

`us-east-1`

This topic will receive notifications from the CloudWatch alarm.

### Terraform Apply Result

```
Apply complete! Resources: 2 added, 0 changed, 0 destroyed.
```

This confirms that both resources were successfully created without modifying or destroying existing infrastructure.

## 6. Verified Existing Infrastructure Outputs

After the successful Terraform apply, Terraform returned the following important outputs:

| Resource        | Output                                                            |
| --------------- | ----------------------------------------------------------------- |
| Application DNS | `api.cloudrecover.internal`                                       |
| Primary ALB     | `cloudrecover-primary-alb-1719900429.eu-west-1.elb.amazonaws.com` |
| Primary EC2     | `i-0ccd56eacdb39caa3`                                             |
| Hosted zone     | `cloudrecover.internal`                                           |
| Hosted zone ID  | `Z07739042OKR4D75JTKJH`                                           |
| VPC             | `vpc-0c5b0288678b9fb57`                                           |

The existing primary environment remained intact.

## 7. Current Architecture Progress

The monitoring and notification foundation now looks like this:

```
          PRIMARY ENVIRONMENT
               eu-west-1
                  |
                  v
          Application Load Balancer
                  |
                  v
          /health/ready endpoint
                  |
                  v
          Route 53 Health Check
                  |
                  |
                  v
          CLOUDWATCH MONITORING
               us-east-1
                  |
                  v
          CloudWatch Metric Alarm
                  |
                  v
          SNS Notification Topic
                  |
                  v
          Lambda Function
                  |
                  v
          GitHub Workflow Dispatch
                  |
                  v
          GitHub Actions
                  |
                  v
       AUTOMATED DISASTER RECOVERY
               eu-west-2
```

The Lambda, SNS subscription, and GitHub Actions integration are not yet connected. They represent the next implementation stages.

## 8. Remaining Tasks

The following tasks remain before the automated recovery trigger is operational:

1. Implement the Lambda function.
2. Create the Lambda execution IAM role with the required permissions.
3. Configure secure storage and retrieval of the GitHub credential.
4. Create the SNS subscription using the Lambda function as its endpoint.
5. Create the Lambda resource-based permission allowing SNS to invoke it.
6. Configure and test the GitHub Actions workflow dispatch integration.
7. Implement safeguards against duplicate recovery executions.
8. Test the complete monitoring-to-recovery trigger.
9. Ensure recovery infrastructure is provisioned before switching DNS traffic.
10. Validate the recovery application and database before activating recovery DNS.

The recovery workflow must follow this sequence:

```
Detect Primary Failure
        |
        v
Trigger Recovery Workflow
        |
        v
Provision Recovery Infrastructure
        |
        v
Restore RDS Snapshot
        |
        v
Deploy Application
        |
        v
Validate Application Readiness
        |
        v
Activate Recovery DNS
```

DNS should not switch to the recovery environment until the recovery application has passed its readiness checks.

## 9. Today's Key Achievements

Today, I:

- Studied Route 53 HTTP health checks and calculated health checks.
- Improved my understanding of private hosted zones.
- Designed the monitoring-to-disaster-recovery trigger architecture.
- Established the regional requirements for Route 53 health check metrics.
- Configured Terraform AWS provider aliases.
- Investigated CloudWatch regional validation errors.
- Identified and corrected the SNS topic regional mismatch.
- Successfully provisioned a CloudWatch metric alarm.
- Successfully provisioned a new SNS topic in `us-east-1`.
- Completed Terraform apply with zero infrastructure destruction.

Overall progress: I have successfully established the primary health monitoring and notification foundation for CloudRecover. The next major milestone is connecting SNS to Lambda and integrating Lambda with GitHub Actions to initiate automated disaster recovery.

Today's lesson: AWS services have regional constraints that matter when designing cross-region architectures. Understanding those constraints and correctly mapping Terraform providers and resource dependencies is essential for building reliable infrastructure.

Final reflection

Today was not without frustration. I spent considerable time troubleshooting the CloudWatch regional configuration, but I eventually resolved the issue and successfully created both resources.

More importantly, I am not simply provisioning AWS resources. I am progressively understanding how the different services interact to form an automated disaster recovery system.

CloudRecover is moving from a manually tested disaster recovery project towards an automated, orchestrated multi-region recovery platform.