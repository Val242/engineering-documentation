# TaskFlow — Application Metrics

## Overview

TaskFlow exposes application-level metrics through a Prometheus-compatible endpoint.

```text
GET /metrics
```

The implementation uses `prom-client`.

## Node.js Runtime Metrics

Default Node.js metrics are collected automatically.

Examples include:

```text
nodejs_heap_space_size_available_bytes
nodejs_version_info
nodejs_gc_duration_seconds
```

These provide visibility into:

* Heap usage
* Runtime version
* Garbage collection behavior

## HTTP Request Metrics

TaskFlow defines a request counter:

```text
http_requests_total
```

The counter uses the following labels:

```text
method
route
status_code
```

Example:

```text
http_requests_total{
  method="GET",
  route="/health/live",
  status_code="200"
} 2
```

## Request Duration

Request latency is recorded using:

```text
http_request_duration_seconds
```

This is a Prometheus histogram.

The histogram uses multiple latency buckets, allowing Prometheus to calculate latency percentiles.

The eventual monitoring system can therefore calculate values such as:

* p50 latency
* p95 latency
* p99 latency

## Global Instrumentation

HTTP instrumentation is implemented as a NestJS global interceptor.

The interceptor measures:

```text
Request start
    ↓
Request execution
    ↓
Request duration
    ↓
HTTP status
    ↓
Prometheus metrics
```

This avoids requiring individual controllers to implement their own metrics logic.

## Verification

The endpoint can be verified locally using:

```bash
curl http://localhost:3000/metrics
```

Application traffic can then be generated and the counter inspected.

Example:

```bash
for i in {1..10}; do
  curl -s http://localhost:3000/health/live > /dev/null
done
```

The request counter should increase accordingly.

## Current Limitation

The current interceptor primarily records the successful request completion path.

Error-path instrumentation still needs to be strengthened so that failed requests, particularly HTTP `5xx` responses, are reliably represented in the custom application metrics.

This is required for accurate:

* Error-rate dashboards
* 5xx percentage calculations
* Server-error alerting
* Recovery detection
