# DevOps Learning Journal — October 9, 2026

## Topic: AWS CI/CD, Elastic Beanstalk, Amazon RDS, and AWS Lambda

### 1. AWS CI/CD Project

Today, I revisited an AWS CI/CD project from my DevOps course. The project involved deploying an application using AWS services, including Elastic Beanstalk and Amazon RDS.

Repeating the project helped me understand the architecture more clearly than when I first worked on it approximately three to four weeks ago.

The project demonstrated how application deployment, database hosting, networking, and security configurations work together in an AWS environment.

### 2. AWS Elastic Beanstalk

I learned more about how AWS Elastic Beanstalk simplifies application deployment.

Elastic Beanstalk allows developers to upload application code or packaged artifacts and deploy them into a managed application environment without manually configuring every underlying infrastructure component.

Depending on the environment configuration, Elastic Beanstalk can provision and manage resources such as:

* Amazon EC2 instances for running the application.
* Application Load Balancers for distributing incoming traffic.
* Target groups for routing traffic to registered application instances.
* Health checks and environment monitoring.
* Auto Scaling resources for adjusting compute capacity.

I observed that Elastic Beanstalk can manage several infrastructure components together, reducing the amount of manual configuration required compared with building an environment directly with EC2.

**Key takeaway:** Elastic Beanstalk is a managed application deployment service that abstracts much of the infrastructure management while still allowing configuration and customization.

### 3. Amazon RDS and Database Connectivity

The project also involved an Amazon RDS instance running a MySQL database.

I learned how an application environment can connect to a separately managed database using its RDS endpoint.

To test connectivity, I installed the MySQL client on an application EC2 instance associated with the Elastic Beanstalk environment. I then used the database endpoint to connect to the RDS instance from the terminal.

Example:

```bash
mysql -h your-rds-endpoint \
      -P 3306 \
      -u your-db-user \
      -p
```

This helped me understand how an application server communicates with a database hosted on AWS.

The connection depends on several factors:

* Network connectivity between the application environment and the database.
* Appropriate security group rules allowing inbound MySQL traffic on TCP port 3306.
* Valid database credentials and sufficient database permissions.

I also reinforced my understanding that Amazon RDS is a managed database service. Unlike a self-managed database installed directly on an EC2 instance, RDS abstracts much of the underlying infrastructure and database administration.

**Key takeaway:** A successful database connection requires network reachability, appropriate security rules, and valid database authentication.

### 4. Security Groups and Network Access

I observed how security groups control communication between AWS resources.

In this project, the RDS security group allowed database traffic from the application environment's security group.

This reinforced the principle of restricting access to only the resources that require it.

For example:

* The application environment receives incoming application traffic according to its security group rules.
* The RDS instance accepts MySQL connections from authorized application resources.
* Database access should not be unnecessarily exposed to the public internet.

**Key takeaway:** Security groups act as virtual firewalls that control inbound and outbound traffic for supported AWS resources.

### 5. Application Load Balancing and Target Groups

While examining the Elastic Beanstalk environment, I encountered load balancing and target groups.

I learned that a load balancer distributes incoming application traffic across registered targets, such as EC2 instances.

A target group identifies the targets that receive traffic and supports health checks used to determine whether those targets are healthy.

Elastic Beanstalk can configure and manage these components automatically in a load-balanced environment.

This contrasts with manually building an infrastructure where I may need to configure the load balancer, listeners, target groups, security groups, and associated networking resources individually.

**Key takeaway:** Managed services can simplify infrastructure operations by coordinating multiple underlying AWS resources.

### 6. Infrastructure as Code (IaC)

After working with the AWS resources, I began thinking about how to provision the same infrastructure using Infrastructure as Code.

I recognized that much of the setup could be defined using Terraform rather than recreated manually through the AWS Console.

Potential resources to manage through Terraform include:

* VPCs and subnets.
* Security groups and networking rules.
* Amazon RDS instances and related configurations.
* Elastic Beanstalk applications and environments.
* IAM roles and policies.
* Supporting infrastructure for CI/CD.

The goal would be to make infrastructure reproducible, version-controlled, and easier to maintain.

Terraform could provision the infrastructure, while a CI/CD pipeline would build and deploy application artifacts to the environment.

**Key takeaway:** Infrastructure as Code makes infrastructure configuration repeatable and manageable through version-controlled definitions.

### 7. AWS Lambda

I also learned the fundamentals of AWS Lambda and completed basic practical examples involving Lambda functions.

AWS Lambda is a serverless compute service that executes code in response to configured triggers or events.

Instead of provisioning and maintaining the underlying servers, I define and deploy a function while AWS manages the compute infrastructure required for execution.

Lambda can be triggered by services and events such as:

* Amazon S3 object events.
* Amazon API Gateway requests.
* Messages arriving in supported queues.
* Scheduled events and other supported integrations.

For example, an S3 upload could trigger a Lambda function to process an image automatically.

I learned that serverless does not mean servers do not exist. It means the cloud provider manages the underlying compute infrastructure.

**Key takeaway:** Lambda is useful for event-driven workloads, automation, and background processing without requiring me to manage application servers directly.

### 8. Comparing AWS Compute Options

| Service               | Primary purpose                                                                                 |
| --------------------- | ----------------------------------------------------------------------------------------------- |
| Amazon EC2            | Provides virtual machines with direct control over the server environment.                      |
| AWS Elastic Beanstalk | Simplifies application deployment and management using an underlying AWS environment.           |
| AWS Lambda            | Executes functions in response to events without requiring me to manage the underlying servers. |

These services address different requirements and can be combined within the same architecture.

For example, a backend API could run in Elastic Beanstalk while Lambda handles asynchronous image processing triggered by S3 events.

## Summary of Today's Learning

Today, I strengthened my understanding of how AWS services work together in a practical application deployment.

My main lessons were:

1. Elastic Beanstalk simplifies application deployment and can manage resources such as EC2 instances, load balancers, and target groups.
2. Amazon RDS provides managed relational database hosting.
3. Security groups control which resources can communicate with one another.
4. Application instances can connect to RDS through its endpoint when networking, security rules, and database authentication are correctly configured.
5. Terraform can be used to define and provision the infrastructure reproducibly.
6. AWS Lambda provides serverless, event-driven compute for functions and background workloads.
7. AWS services can be combined to create architectures that balance operational simplicity, control, scalability, and maintainability.

## Next Steps

For future projects, I intend to apply these concepts by considering Infrastructure as Code from the beginning, rather than relying exclusively on manual configuration through the AWS Console.

I will continue developing my practical understanding of AWS, Terraform, CI/CD, networking, and monitoring as part of my DevOps learning journey.

**Personal reflection:** Repeating the AWS project helped me connect concepts that were initially less clear. I am increasingly understanding not only how to configure individual AWS services, but also how they fit together in a complete application architecture.
