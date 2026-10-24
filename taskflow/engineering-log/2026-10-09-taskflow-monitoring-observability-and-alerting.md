# TaskFlow — Daily Engineering Progress Report

**Focus:** Day 5 — Monitoring, Observability, and Alerting
**Status:** Completed

## 1. Overview

Today, I completed the monitoring and alerting tasks for Day 5 of the TaskFlow DevOps project. The objective was to establish visibility into application health and infrastructure performance using Prometheus, Grafana, Node Exporter, and Alertmanager.

I also gained practical troubleshooting experience by identifying and resolving a network connectivity issue between Grafana and Prometheus.

## 2. Work Completed

### Prometheus and Metrics Collection

* Configured Prometheus to scrape application metrics from the NestJS API.
* Configured Node Exporter to expose host-level metrics from the application EC2 instance.
* Verified Prometheus target availability and metric collection.
* Practiced querying and analyzing collected metrics using PromQL.

### Grafana Monitoring

* Configured Grafana to visualize application and infrastructure metrics collected by Prometheus.
* Created two monitoring dashboards for application observability and EC2 infrastructure monitoring.
* Accessed graph details to inspect metric values and better understand system behavior over time.

### AWS Networking Troubleshooting

* Encountered a connectivity issue preventing Grafana from accessing Prometheus.
* Investigated the issue and identified that the required inbound security group rule was missing.
* Added the appropriate inbound rule to allow Grafana to communicate with Prometheus.
* Resolved the connectivity issue and enabled Grafana to retrieve metrics from the Prometheus data source.

**Key lesson:** Applications running on separate EC2 instances require appropriate network access rules to communicate. Security group rules must allow the necessary traffic while keeping monitoring services private and restricting access to authorized sources.

### Infrastructure Testing

* Generated CPU load on the application EC2 instance to observe changes in utilization through Grafana.
* Reviewed how stopping the TaskFlow API service affects application availability monitoring.
* Learned to distinguish application availability from host-level monitoring availability.

### Alerting and Notifications

* Worked on Prometheus alerting with Alertmanager.
* Configured Slack notifications for monitoring alerts.
* Prepared alert messaging for CPU utilization exceeding the 75% threshold, including a summary and description to guide investigation.

## 3. Key Learnings

* Prometheus collects and stores time-series metrics from configured targets.
* Grafana visualizes and helps analyze metrics collected by Prometheus.
* Node Exporter provides infrastructure metrics, while application instrumentation exposes application-level metrics.
* AWS security groups control network traffic between EC2 instances.
* Troubleshooting missing inbound rules helped reinforce the relationship between network connectivity and monitoring data availability.
* Simulating failures and increased workloads helps validate monitoring behavior.
* PromQL supports filtering, aggregation, and analysis of collected metrics.

## 4. Areas for Improvement

* Improve my ability to write and modify PromQL queries independently.
* Verify that application errors are correctly recorded in the metrics.
* Test alert delivery, recovery notifications, and alert resolution.
* Continue validating monitoring behavior under different failure scenarios.

## 5. Project Status

**Day 5 — Monitoring and Alerting: Completed.**

TaskFlow now has a monitoring layer that provides visibility into application and infrastructure behavior. The Grafana–Prometheus connectivity issue was also resolved by correcting the missing AWS security group inbound rule.

## 6. Next Steps — Day 6: CI/CD and Security

The next phase will focus on automating application delivery through GitHub Actions.

Planned activities:

1. Configure automated linting, tests, and build verification.
2. Integrate Gitleaks for secret detection.
3. Add dependency vulnerability scanning.
4. Configure application deployment to EC2 through AWS Systems Manager (SSM).
5. Handle Prisma migrations safely when required.
6. Restart the application and verify readiness after deployment.

The existing AWS infrastructure and monitoring services will remain in place during routine application deployments.

## Conclusion

Today's work completed the planned monitoring milestone. In addition to configuring monitoring and alerting, I practiced inspecting metrics, simulating infrastructure load, and troubleshooting AWS network access between monitoring services.

These activities strengthened my understanding of observability, EC2 security groups, and practical infrastructure troubleshooting. The next milestone is to implement a reliable CI/CD pipeline that automates quality checks, security scanning, application deployment, and post-deployment verification.
