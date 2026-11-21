# DevOps Engineering Progress Report
**Date:** 21 November 2026  
**Project:** Java CI/CD Automation with Jenkins, SonarQube, Nexus and AWS

## 1. Work Accomplished Today

Today, I made significant progress on the infrastructure setup for my Java CI/CD project.

### Infrastructure Provisioning

Successfully provisioned the following infrastructure components:

- **SonarQube Server:** Provisioned the server and installed the required dependencies.
- **Nexus Repository Server:** Provisioned the server and installed the necessary dependencies.
- **Jenkins Server:** Worked on the Jenkins installation and repository configuration.

### Jenkins Installation and Configuration

Worked through the Jenkins installation process, including:

- Updating the system package index.
- Installing Java and required system dependencies.
- Configuring the Jenkins APT repository.
- Troubleshooting the Jenkins GPG signing key issue.
- Understanding Jenkins Java runtime requirements and compatibility with my Java 17 application.

An important distinction established during this process is that Jenkins can run on Java 21 while my application continues targeting Java 17.

### Infrastructure Troubleshooting

Encountered an issue accessing my deployed SonarQube server.

I investigated the problem and identified that the AWS Security Group did not have an appropriate inbound rule allowing access to the SonarQube service.

SonarQube uses port **9000** by default.

This reinforced my understanding of:
- AWS Security Groups.
- Inbound traffic rules.
- Application accessibility and network configuration.
- Troubleshooting infrastructure connectivity issues.

### Jenkins Pipeline Development

Started working on the Jenkins pipeline implementation.

The objective is to automate the application's delivery lifecycle, integrating source control, build automation, code quality analysis, artifact management and deployment.

## 2. Planned CI/CD Architecture

The intended pipeline will integrate:

1. **GitHub:** Source code management and pipeline triggering.
2. **Jenkins:** CI/CD orchestration and automation.
3. **Maven:** Application compilation, dependency management and testing.
4. **SonarQube:** Static code analysis and code quality inspection.
5. **Nexus:** Artifact repository management.
6. **Docker:** Containerization of the Java application.
7. **Amazon ECR:** Storage and management of Docker images.
8. **Amazon EC2:** Hosting and running the deployed application.
9. **Terraform:** Infrastructure provisioning and management.
10. **Ansible:** Configuration management and deployment automation where appropriate.

## 3. Next Technical Tasks

When I resume working on the project, my priorities will be:

- Revise Jenkins pipeline fundamentals.
- Configure Jenkins to connect to GitHub.
- Implement the complete Jenkinsfile.
- Automate application compilation and testing.
- Integrate SonarQube into the pipeline.
- Configure Nexus for artifact publishing.
- Build and tag Docker images.
- Authenticate with Amazon ECR and push images.
- Provision a separate EC2 instance for application deployment.
- Configure the deployment process to pull images from ECR and run containers on EC2.
- Test the complete CI/CD workflow.

I also intend to explore how ECR and EC2 operate independently before integrating them into the complete deployment workflow.

## 4. Infrastructure Cost Management

Since I have other commitments next week, including a hackathon, I decided not to leave temporary infrastructure running unnecessarily.

My plan is to:

- Preserve my Terraform configurations and provisioning scripts.
- Preserve any important application and service configurations.
- Review the infrastructure resources and associated data.
- Destroy temporary infrastructure that I will not need next week.
- Reprovision the environments when I resume development.

This will help minimize unnecessary AWS expenditure while keeping the project reproducible.

## 5. Key Learning Outcomes

Today's work reinforced several important engineering concepts:

- Infrastructure provisioning is only one part of deploying an application; network accessibility and security configuration are equally important.
- Jenkins runtime requirements are independent of the Java version targeted by an application.
- AWS Security Groups control access to services running on cloud infrastructure.
- SonarQube, Nexus and Jenkins serve different purposes within a CI/CD ecosystem.
- Infrastructure automation makes it possible to destroy and recreate environments instead of maintaining unnecessary running resources.
- Troubleshooting should involve understanding the underlying infrastructure rather than blindly applying fixes.

## 6. Current Progress and Next Milestone

**Current status:** Infrastructure provisioning and initial Jenkins setup completed or in progress; pipeline implementation has begun.

**Next milestone:** Build a complete automated CI/CD pipeline that takes application code from GitHub, performs quality checks, builds and publishes artifacts, creates a Docker image, pushes it to Amazon ECR and deploys it to an EC2 instance.

---

**Personal Reflection**

Today was productive. More importantly, I am increasingly becoming comfortable reasoning about infrastructure problems independently.

I am moving beyond learning individual DevOps tools and towards understanding how they integrate into a complete software delivery system.

My next focus is implementation, integration and automation rather than simply installing more tools.