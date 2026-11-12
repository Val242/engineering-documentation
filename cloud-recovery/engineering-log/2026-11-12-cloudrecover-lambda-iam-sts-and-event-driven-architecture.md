# 2026-11-12 — CloudRecover: Lambda Deployment, IAM, STS and Event-Driven Architecture

## 1. What I worked on

Today, I continued implementing the event-triggering mechanism for CloudRecover's multi-region disaster recovery architecture.

The objective is to connect infrastructure health monitoring to an automated GitHub Actions workflow that can initiate disaster recovery.

The architecture is:

**Route 53 → CloudWatch Alarm → SNS → AWS Lambda → GitHub Actions → Recovery Infrastructure**

## 2. Understanding Amazon SNS

SNS stands for **Simple Notification Service**. It is a managed publish/subscribe messaging service.

I learned that SNS operates around two important concepts:

- **Publisher:** The component that publishes a message to an SNS topic.
- **Subscriber:** The component that receives messages published to that topic.

In my CloudRecover architecture:

- CloudWatch Alarm acts as the publisher.
- The SNS topic receives and distributes the notification.
- AWS Lambda acts as the subscriber.

### How the communication happens

1. CloudWatch detects a change in the monitored health check.
2. The alarm transitions into the `ALARM` state.
3. CloudWatch publishes a notification to the SNS topic.
4. SNS delivers the notification to the subscribed Lambda function.
5. AWS invokes the Lambda handler and passes it an SNS event.

An important distinction is that SNS does not execute my Lambda code. SNS delivers the message, and the Lambda service invokes the function.

## 3. Understanding AWS Lambda's role

AWS Lambda acts as the bridge between AWS monitoring and GitHub Actions.

Its responsibilities include:

- Receiving notifications from the SNS topic.
- Inspecting the CloudWatch alarm notification.
- Confirming that the notification concerns the expected alarm.
- Checking whether the alarm has entered the `ALARM` state.
- Retrieving the GitHub personal access token from AWS Secrets Manager.
- Sending an authenticated request to the GitHub Actions workflow dispatch API.

Lambda does not directly provision the recovery infrastructure. Instead, it triggers the GitHub Actions workflow responsible for orchestrating the recovery process.

## 4. Understanding IAM execution roles

I learned that an AWS Lambda function needs an IAM execution role to interact with other AWS services.

In my Terraform configuration, the Lambda resource references the execution role ARN through its `role` attribute.

The role determines what the Lambda function is permitted to do when executing.

My IAM configuration has three important components.

### A. Trust policy

The trust policy defines who is allowed to assume an IAM role.

My Lambda execution role trusts the AWS Lambda service:

- Principal: `lambda.amazonaws.com`
- Action: `sts:AssumeRole`

This allows the Lambda service to assume the execution role.

### B. Permission policy

The permission policy defines what the assumed role is allowed to access.

For CloudRecover, I grant:

`secretsmanager:GetSecretValue`

This allows the Lambda function to retrieve the GitHub token from my specific Secrets Manager secret.

### C. CloudWatch logging policy

I also attach the AWS-managed `AWSLambdaBasicExecutionRole` policy.

This allows the Lambda function to create log streams and write execution logs to CloudWatch Logs.

An important distinction I learned is that trust policies and permission policies serve different purposes:

- Trust policy: Who can assume the role?
- Permission policy: What can the assumed role do?

## 5. Understanding AWS STS

STS stands for **AWS Security Token Service**.

It provides temporary security credentials that authorized identities and services can use to access AWS resources.

I learned how STS fits into the Lambda execution process.

### How Lambda obtains temporary credentials

1. I assign an IAM execution role to my Lambda function.
2. The role's trust policy allows the Lambda service to assume it.
3. The Lambda service obtains temporary credentials through AWS STS.
4. Those credentials are made available to the Lambda execution environment.
5. Lambda uses the credentials when accessing AWS services permitted by the role's policies.

The temporary credentials generally contain:

- Access key ID.
- Secret access key.
- Session token.

I do not need to manually generate or configure these credentials inside my Lambda handler. AWS manages their availability and rotation for the execution environment.

**Important distinction:** STS does not generate my GitHub token. It provides temporary AWS credentials for AWS resource access.

## 6. Understanding Secrets Manager and GitHub authentication

I learned that my Lambda function communicates with two different systems, and each requires a different authentication mechanism.

### A. Lambda accessing AWS Secrets Manager

Lambda uses its temporary AWS credentials obtained through its execution role.

Secrets Manager checks whether the execution role has permission to retrieve the requested secret.

If authorized, Secrets Manager returns the GitHub personal access token.

### B. Lambda accessing GitHub

After retrieving the token, Lambda uses it to authenticate its HTTP request to GitHub's workflow dispatch API.

GitHub validates the token and checks whether it has sufficient repository permissions.

If authorized, GitHub accepts the workflow dispatch request.

The distinction is:

| Communication | Authentication |
|---|---|
| Lambda → AWS Secrets Manager | Temporary AWS credentials through IAM execution role |
| Lambda → GitHub API | GitHub Personal Access Token |
| GitHub Actions → AWS | GitHub OIDC federation with an AWS IAM role |

These are separate authentication relationships.

### Security consideration

The GitHub token should not be hardcoded into my Lambda source code or Terraform configuration.

Instead:

- Store the token in AWS Secrets Manager.
- Pass only the secret ARN through Terraform.
- Give Lambda permission to retrieve that specific secret.
- Retrieve the actual token at runtime.

Marking a Terraform variable as `sensitive` does not prevent its value from being stored in Terraform state.

Therefore, I should populate the secret outside Terraform rather than placing the actual token in an `aws_secretsmanager_secret_version` resource.

## 7. Understanding AWS regional placement

One important architectural detail I learned concerns the relationship between SNS and Lambda.

My active CloudWatch alarm notification SNS topic is in `us-east-1`, while my recovery infrastructure is in `eu-west-2`.

The Lambda function and its SNS subscription must be deployed in the same region as the SNS topic.

Therefore:

| Component | Region |
|---|---|
| Primary monitoring SNS topic | `us-east-1` |
| Recovery trigger Lambda | `us-east-1` |
| Recovery infrastructure | `eu-west-2` |

This does not mean my recovery infrastructure needs to move to `us-east-1`.

Lambda is simply the event-triggering component. It invokes GitHub Actions, which then orchestrates recovery in `eu-west-2`.

I also learned that IAM roles are account-wide resources rather than resources tied to a particular AWS region.

## 8. Understanding Lambda deployment packages

I encountered a Terraform error when attempting to deploy my Lambda function:

```text
Error in function call

Call to function "filebase64sha256" failed:
open ../../lambda/recovery-trigger/deployment.zip:
The system cannot find the file specified.
```

I learned that Terraform requires an actual deployment ZIP file when using:

```hcl
filename         = var.lambda_deployment_package
source_code_hash = filebase64sha256(var.lambda_deployment_package)
```

The `filename` argument identifies the deployment package that Terraform uploads to AWS.

The `filebase64sha256()` function calculates the package hash, allowing Terraform to detect changes to the Lambda deployment artifact.

Terraform does not automatically compile my TypeScript code or create the deployment ZIP.

### TypeScript compilation and packaging

My intended workflow is:

1. Write the Lambda function in TypeScript.
2. Use esbuild to compile and bundle the code.
3. Generate a JavaScript file.
4. Use Terraform's `archive_file` data source to package the compiled JavaScript.
5. Deploy the resulting ZIP to AWS Lambda.

The expected deployment artifact structure is:

```text
deployment.zip
└── handler.js
```

The JavaScript handler must be at the ZIP root because my Terraform configuration specifies:

```hcl
handler = "handler.handler"
```

This refers to the `handler` export inside `handler.js`.

### Why use esbuild and archive_file?

I considered three approaches:

- Manually creating ZIP files.
- Using Terraform's `archive_file` data source.
- Deploying TypeScript directly.

My preferred approach is to use esbuild for compilation and bundling, then Terraform's `archive_file` for packaging.

This separates application compilation from infrastructure provisioning while reducing repetitive manual work.

An important distinction: `archive_file` packages files; it does not compile TypeScript.

## 9. Architectural decisions and lessons

Today's important takeaways:

- SNS is a managed publish/subscribe messaging service.
- CloudWatch Alarm publishes notifications to an SNS topic.
- SNS delivers notifications to subscribed endpoints.
- AWS Lambda is invoked when it receives the SNS event.
- IAM trust policies define who can assume a role.
- IAM permission policies define what an assumed role can access.
- STS provides temporary AWS security credentials after role assumption.
- Lambda uses its execution role to access AWS services.
- GitHub authentication is separate from AWS IAM authentication.
- Secrets Manager provides a safer way to retrieve credentials at runtime.
- IAM roles are account-wide, but Lambda functions and SNS topics are regional.
- SNS and Lambda must be in the same region for this subscription.
- Terraform deployment packages must exist before Terraform calculates their hashes.
- TypeScript compilation and Lambda deployment packaging are separate operations.
- Bundling dependencies is important because the Lambda runtime does not automatically provide every third-party package used by my application.

## 10. Current progress

| Component | Status |
|---|---|
| Route 53 health check | Implemented |
| CloudWatch health alarm | Implemented |
| SNS notification topic | Implemented |
| Lambda TypeScript handler | Written |
| Lambda IAM execution role | Configuration prepared |
| Secrets Manager integration | Configuration prepared |
| Lambda Terraform module | Configuration prepared |
| SNS-to-Lambda subscription | Configuration prepared |
| TypeScript build and ZIP packaging | Not yet implemented |
| Lambda deployment and integration testing | Pending |
| Duplicate recovery protection | Pending |

## 11. Next session

My next objective is to implement the Lambda build and packaging process.

Planned activities:

1. Configure esbuild in the Lambda project.
2. Compile and bundle the TypeScript handler.
3. Configure Terraform's `archive_file` data source.
4. Ensure the deployment ZIP has the correct structure.
5. Run Terraform plan and deploy the Lambda function.
6. Verify the SNS subscription and Lambda invocation permission.
7. Test the GitHub Actions workflow dispatch integration.

After that, I will work on duplicate-trigger protection and end-to-end disaster recovery testing.

---

## Final reflection

Today's work helped me understand that deploying a Lambda function involves more than writing its handler.

I need to understand its execution permissions, trust relationships, temporary credentials, regional placement, secret retrieval, dependencies, deployment artifact and event source integration.

One of my most important takeaways is understanding the difference between AWS authentication and GitHub authentication. STS provides temporary AWS credentials, while the GitHub PAT authenticates requests to GitHub.

I am gradually connecting individual AWS services into a complete disaster recovery architecture rather than treating them as isolated services.

**The objective remains to build a reliable, automated, multi-region disaster recovery system with a clear recovery process and controlled failover.**