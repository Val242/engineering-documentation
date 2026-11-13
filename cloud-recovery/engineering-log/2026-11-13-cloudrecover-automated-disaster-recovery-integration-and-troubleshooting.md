# 2026-11-13 — CloudRecover: Automated Disaster Recovery Integration and Troubleshooting

## 1. Today's learning objectives

Today's focus was on understanding and testing the automated disaster recovery trigger chain:

**CloudWatch → SNS → Lambda → GitHub Actions → Terraform → AWS Recovery Infrastructure**

The goal was to move beyond provisioning individual AWS resources and test how an infrastructure failure can initiate an automated recovery workflow.

---

## 2. Understanding the Lambda-to-GitHub Actions integration

The Lambda function acts as an intermediary between AWS monitoring and GitHub Actions.

When CloudWatch detects a health-check failure:

1. CloudWatch changes the alarm state to `ALARM`.
2. CloudWatch publishes an alarm notification to SNS.
3. SNS invokes the Lambda function.
4. Lambda processes the SNS message.
5. Lambda extracts the alarm information and dispatches a GitHub Actions workflow through the GitHub API.

### Understanding the workflow inputs

The Lambda function sends the following inputs:

```typescript
inputs: {
  trigger_source: "cloudwatch_alarm",
  alarm_name: message.AlarmName,
  source_region: message.Region || "eu-west-1",
  recovery_region: "eu-west-2",
},
```

Understanding where these values originate:

| Input | Origin |
|---|---|
| `trigger_source` | Hardcoded in Lambda |
| `alarm_name` | Extracted from the SNS alarm message |
| `source_region` | Extracted from the alarm message, with a fallback |
| `recovery_region` | Configured in Lambda |

These values are sent through GitHub's `workflow_dispatch` API.

An important distinction is that workflow inputs are not the same as GitHub Secrets.

- Workflow inputs carry information into a particular workflow execution.
- GitHub repository or environment variables hold non-sensitive configuration.
- GitHub Secrets hold sensitive credentials and configuration.

The workflow also has inputs that Lambda does not supply, such as `snapshot_identifier` and `action_type`. These must be handled through workflow defaults or manual inputs.

---

## 3. Manually testing Lambda without waiting for CloudWatch

I learned that I do not need to wait for another CloudWatch alarm transition to test the Lambda function.

I can invoke Lambda directly using AWS CLI:

```bash
aws lambda invoke \
  --function-name cloudrecover-recovery-trigger \
  --invocation-type RequestResponse \
  --payload fileb://lambda-test.json \
  --region us-east-1 \
  lambda-response.json
```

The test payload simulates an SNS event containing the CloudWatch alarm message.

The invocation returned:

```text
StatusCode: 200
ExecutedVersion: "$LATEST"
```

The response file contained:

```json
null
```

### What this taught me

- `StatusCode: 200` indicates that the Lambda invocation completed successfully at the invocation level.
- `null` is not necessarily an error. An SNS handler does not need to return a value.
- A successful invocation alone does not prove that GitHub received the workflow dispatch.
- The GitHub Actions run and Lambda CloudWatch logs provide further evidence.

**Important:** Direct Lambda invocation bypasses CloudWatch and SNS. It tests the Lambda execution and downstream integration, not the entire monitoring chain.

---

## 4. Terraform workflow input and variable management

After the GitHub Actions workflow was dispatched, Terraform reported several missing required variables.

The missing variables included:

- `route53_hosted_zone_id`
- `github_secret_arn`
- `lambda_deployment_package`
- `github_owner`
- `github_repository`
- `github_workflow`
- `lambda_s3_bucket`
- `lambda_s3_key`
- `primary_sns_topic_arn`

### Understanding the issue

Terraform root-module input variables must receive values.

There are different ways to supply them:

1. Explicitly through `-var`.
2. Through a Terraform variable file.
3. Through environment variables prefixed with `TF_VAR_`.
4. Through defaults declared in `variables.tf`, where appropriate.

For example:

```yaml
env:
  TF_VAR_github_owner: Val242
  TF_VAR_github_repository: CloudRecover
  TF_VAR_github_workflow: disaster-recovery.yml
```

Terraform recognizes these as input variables.

However, ordinary environment variables do not automatically become Terraform variables.

This distinction helped me understand why a GitHub Actions workflow can contain configuration values while Terraform still reports missing inputs.

---

## 5. Troubleshooting IAM permissions in GitHub Actions

After resolving the missing Terraform inputs, the workflow progressed further and encountered AWS authorization errors.

The IAM role being assumed by GitHub Actions was:

```text
CloudRecover-GitHubActions-DR-Role
```

The errors identified four missing read permissions.

| AWS service | Missing permission | Purpose |
|---|---|---|
| IAM | `iam:GetRole` | Read the Lambda execution role |
| Route 53 | `route53:ListHostedZonesByVPC` | Read hosted-zone/VPC associations |
| Route 53 | `route53:GetHealthCheck` | Read a Route 53 health check |
| SNS | `sns:GetTopicAttributes` | Read SNS topic attributes |

### Why does Terraform need read permissions?

Terraform does not simply create resources.

During its refresh phase, Terraform compares the resources recorded in its state with the actual resources in AWS.

To do this, it needs permission to read existing infrastructure.

This means that even if an IAM role has permissions to create certain resources, Terraform may still fail if it cannot read those resources.

**Key lesson:** Infrastructure deployment permissions must account for both resource creation/modification and resource discovery/refresh.

The appropriate solution is to update the IAM policy attached to the GitHub Actions role with the necessary least-privilege permissions, rather than granting unrestricted administrator access.

---

## 6. Terraform state inconsistency: RDS security group ingress rule

Another message appeared during Terraform refresh:

```text
module.security_groups.aws_vpc_security_group_ingress_rule.rds_postgresql_from_web

couldn't find resource
Automatically removing from Terraform State instead of returning the error
```

This indicates that Terraform had a record of a security group ingress rule in its state, but AWS could not find the corresponding rule.

This is relevant to my earlier database outage simulation.

I had manually removed the PostgreSQL ingress rule that allowed the application EC2 security group to connect to RDS.

### What I learned

Terraform state represents Terraform's record of managed infrastructure.

When infrastructure is changed manually outside Terraform, configuration, state, and actual AWS resources can become inconsistent.

Terraform may detect this during refresh and remove a stale state entry.

If the resource is still declared in Terraform configuration, a later plan may propose recreating it.

**Important:** I should inspect the Terraform plan before applying changes, especially because this rule was deliberately removed during my outage simulation.

---

## 7. An SNS configuration detail to investigate

The Terraform error referenced this SNS topic:

```text
cloudrecover-primary-health-alerts
```

in:

```text
eu-west-2
```

However, the alarm-triggering SNS topic used by the Lambda integration is:

```text
cloudrecover-route53-health-alerts
```

in:

```text
us-east-1
```

These are different SNS topics.

This may be intentional because the secondary infrastructure has its own SNS configuration, but I need to verify that the distinction is deliberate.

I should not assume that the two topics are interchangeable.

---

## 8. Troubleshooting strategy learned today

Today's debugging reinforced the importance of separating failures by layer.

| Layer | What to investigate |
|---|---|
| CloudWatch | Alarm state and health-check metrics |
| SNS | Topic configuration and subscriptions |
| Lambda | Execution logs, event processing and GitHub API response |
| GitHub Actions | Workflow dispatch, input contract and execution logs |
| Terraform | Required variables, provider configuration and state refresh |
| IAM | Permissions of the GitHub Actions assumed role |
| AWS infrastructure | Actual resource state and Terraform configuration |

A successful step does not automatically mean the entire disaster recovery process is successful.

For example, a successful Lambda invocation does not prove that Terraform can provision the recovery environment.

---

## 9. Next steps when I resume

1. Inspect the IAM policy attached to `CloudRecover-GitHubActions-DR-Role`.
2. Add the missing read permissions using least privilege.
3. Rerun the GitHub Actions workflow.
4. Inspect the Terraform plan before applying anything.
5. Verify the RDS security group ingress rule and reconcile it with the intended configuration.
6. Investigate why the secondary Terraform configuration references `cloudrecover-primary-health-alerts` in `eu-west-2`.
7. Continue testing the recovery workflow after Terraform refresh and planning succeed.

## 10. Key takeaways

Today's work helped me understand several important DevOps concepts:

- How AWS monitoring events can trigger external CI/CD workflows.
- How Lambda can communicate with GitHub Actions through the workflow dispatch API.
- How to test Lambda independently of CloudWatch.
- The distinction between workflow inputs, repository variables and secrets.
- How Terraform obtains root-module input variables.
- Why Terraform needs AWS read permissions during state refresh.
- How manual AWS changes can introduce Terraform state inconsistencies.
- Why infrastructure automation must be validated layer by layer.

**My biggest takeaway:** Building an automated disaster recovery system is not just about provisioning a secondary environment. It also requires reliable event propagation, authentication, authorization, state management and controlled infrastructure changes.

The integration has progressed from the monitoring trigger to Terraform execution. The remaining work is to resolve the IAM permissions and verify the infrastructure reconciliation before proceeding with recovery.