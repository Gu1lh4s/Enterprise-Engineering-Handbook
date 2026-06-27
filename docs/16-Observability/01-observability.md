# Observability — Metrics, Logs, Traces

> **Category:** Observability
> **Version:** 1.0.0
> **Level:** Staff Engineer

---

## Table of Contents

1. [The Three Pillars](#1-the-three-pillars)
2. [Metrics with Prometheus](#2-metrics-with-prometheus)
3. [Structured Logging](#3-structured-logging)
4. [Distributed Tracing](#4-distributed-tracing)
5. [Alerting Strategy](#5-alerting-strategy)
6. [SLOs and Error Budgets](#6-slos-and-error-budgets)
7. [Dashboards (Grafana)](#7-dashboards-grafana)
8. [OpenTelemetry](#8-opentelemetry)
9. [On-Call Runbooks](#9-on-call-runbooks)
10. [Incident Response](#10-incident-response)

---

## 1. The Three Pillars

```
Observability: can you understand what's happening inside your system
               by only looking at its external outputs?

Three outputs:
  Metrics → aggregated numeric measurements (CPU 80%, requests/sec 500, error rate 2%)
  Logs    → discrete events with context (request failed for user X because Y)
  Traces  → distributed request path (service A → B → C: where did 300ms go?)

Four "Golden Signals" (Google SRE Book):
  Latency:   How long does it take to serve a request?
  Traffic:   How much demand is on the system?
  Errors:    What is the rate of failed requests?
  Saturation: How "full" is the system? (CPU, memory, disk, queue depth)

RED Method (per service):
  Rate:   requests per second
  Errors: error rate
  Duration: distribution of response times (p50, p95, p99)

USE Method (per resource):
  Utilization: % time resource is busy
  Saturation:  amount of work queued/waiting
  Errors:      error events (disk errors, network drops)
```

---

## 2. Metrics with Prometheus

### Metric Types

```typescript
import { Counter, Gauge, Histogram, Summary, register } from 'prom-client'

// Counter: monotonically increasing (requests, errors, bytes sent)
const httpRequestsTotal = new Counter({
  name: 'http_requests_total',
  help: 'Total HTTP requests processed',
  labelNames: ['method', 'route', 'status_code', 'service'],
})

// Gauge: can go up and down (current connections, queue depth, memory used)
const activeConnections = new Gauge({
  name: 'active_connections',
  help: 'Number of active WebSocket connections',
})

const queueDepth = new Gauge({
  name: 'job_queue_depth',
  help: 'Number of jobs waiting in queue',
  labelNames: ['queue_name'],
})

// Histogram: distribution of values (request duration, request size)
// Use for: latency (p50, p95, p99), payload sizes
const httpRequestDuration = new Histogram({
  name: 'http_request_duration_seconds',
  help: 'HTTP request duration in seconds',
  labelNames: ['method', 'route', 'status_code'],
  // Buckets: boundaries for histogram; choose based on SLO
  // E.g., if SLO is p99 < 300ms: have a bucket at 0.3
  buckets: [0.001, 0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5, 10],
})

// Summary: percentiles (use Histogram instead — Histograms aggregate across instances)
// Summary calculates percentiles per instance; cannot aggregate across replicas
// Prefer Histogram for anything you need across multiple replicas
```

### Instrumentation Middleware

```typescript
// Express metrics middleware
import { Request, Response, NextFunction } from 'express'
import { Histogram, Counter } from 'prom-client'

function metricsMiddleware(req: Request, res: Response, next: NextFunction) {
  const startTime = Date.now()
  
  res.on('finish', () => {
    const duration = (Date.now() - startTime) / 1000
    const route = req.route?.path ?? req.path  // Normalize: /users/123 → /users/:id
    
    const labels = {
      method: req.method,
      route: normalizeRoute(route),
      status_code: String(res.statusCode),
      service: process.env.SERVICE_NAME ?? 'api',
    }
    
    httpRequestsTotal.inc(labels)
    httpRequestDuration.observe(labels, duration)
  })
  
  next()
}

function normalizeRoute(path: string): string {
  // Prevent high cardinality: /users/123 → /users/:id
  return path
    .replace(/\/[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}/gi, '/:uuid')
    .replace(/\/\d+/g, '/:id')
}

// Metrics endpoint for Prometheus scraping
app.get('/metrics', async (req, res) => {
  res.set('Content-Type', register.contentType)
  res.end(await register.metrics())
})
```

### Prometheus Configuration

```yaml
# prometheus.yml
global:
  scrape_interval: 15s      # How often to scrape targets
  evaluation_interval: 15s  # How often to evaluate rules

scrape_configs:
  - job_name: 'api-service'
    static_configs:
      - targets: ['api-service:3000']
    metrics_path: '/metrics'
    scrape_timeout: 10s
  
  - job_name: 'postgres'
    static_configs:
      - targets: ['postgres-exporter:9187']
  
  - job_name: 'redis'
    static_configs:
      - targets: ['redis-exporter:9121']
  
  - job_name: 'node-exporter'  # Host-level metrics (CPU, disk, network)
    static_configs:
      - targets: ['node-exporter:9100']

# Service discovery (Kubernetes)
  - job_name: 'kubernetes-pods'
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true  # Only scrape pods with prometheus.io/scrape: "true" annotation
```

### PromQL Queries

```promql
# Request rate (per minute, 5-minute window)
rate(http_requests_total[5m]) * 60

# Error rate percentage
sum(rate(http_requests_total{status_code=~"5.."}[5m])) /
sum(rate(http_requests_total[5m])) * 100

# P99 latency
histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket[5m])) by (le, route))

# Apdex score (user satisfaction metric)
# Satisfied: < 0.1s, Tolerating: 0.1s-0.4s, Frustrated: > 0.4s
(
  sum(rate(http_request_duration_seconds_bucket{le="0.1"}[5m])) +
  sum(rate(http_request_duration_seconds_bucket{le="0.4"}[5m])) / 2
) / sum(rate(http_request_duration_seconds_count[5m]))

# Saturation: CPU usage
100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

# Memory usage %
100 - ((node_memory_MemFree_bytes + node_memory_Cached_bytes + node_memory_Buffers_bytes) / node_memory_MemTotal_bytes * 100)
```

---

## 3. Structured Logging

### Log Schema

```typescript
// Every log entry must be structured JSON
interface LogEntry {
  timestamp: string        // ISO 8601: "2025-06-27T10:00:00.000Z"
  level: 'debug' | 'info' | 'warn' | 'error' | 'fatal'
  message: string
  
  // Tracing context (always)
  traceId: string
  spanId: string
  
  // Service context (always)
  service: string          // "booking-service"
  version: string          // "1.2.3"
  environment: string      // "production"
  
  // Request context (for HTTP requests)
  requestId?: string
  method?: string
  path?: string
  statusCode?: number
  duration?: number
  userId?: string
  
  // Error context (for errors)
  error?: {
    message: string
    type: string
    stack?: string         // Stack trace (only in development or for unexpected errors)
  }
  
  // Business context (event-specific)
  [key: string]: unknown
}
```

### Pino Logger (Node.js)

```typescript
import pino from 'pino'

const logger = pino({
  level: process.env.LOG_LEVEL ?? 'info',
  
  // Always include these fields
  base: {
    service: process.env.SERVICE_NAME,
    version: process.env.APP_VERSION,
    environment: process.env.NODE_ENV,
    pid: process.pid,
  },
  
  // Human-readable in development, JSON in production
  transport: process.env.NODE_ENV === 'development'
    ? { target: 'pino-pretty', options: { colorize: true } }
    : undefined,
  
  // Redact sensitive fields
  redact: {
    paths: ['*.password', '*.token', '*.secret', '*.authorization', '*.credit_card'],
    censor: '[REDACTED]',
  },
  
  // Custom serializers
  serializers: {
    req: (req) => ({
      method: req.method,
      url: req.url,
      userAgent: req.headers?.['user-agent'],
    }),
    res: (res) => ({
      statusCode: res.statusCode,
    }),
    err: pino.stdSerializers.err,
  },
})

// Usage
logger.info({ userId: 'user_123', bookingId: 'booking_456' }, 'Booking confirmed')
logger.error({ err: error, bookingId: 'booking_456' }, 'Failed to confirm booking')
logger.warn({ requestId, duration: 5000 }, 'Slow request detected')

// Log levels:
// debug: Detailed diagnostic info (disabled in production)
// info:  Normal operations (user logged in, booking created)
// warn:  Unexpected but handled (slow query, retry succeeded)
// error: Failures requiring attention (payment failed, DB timeout)
// fatal: System is unrecoverable (startup failure, OOM)
```

### Log Sampling (High-Volume Applications)

```typescript
// Log every request at 100 req/s = 8.6M logs/day = expensive
// Sampling: log 1% of successful requests, 100% of errors

const sampler = pino({
  // Always log at warn+ (errors, warnings)
  // Sample 1% of info/debug
  customLevels: {},
  useOnlyCustomLevels: false,
})

app.use((req, res, next) => {
  const start = Date.now()
  
  res.on('finish', () => {
    const shouldLog = 
      res.statusCode >= 400 ||           // Always log errors
      req.path === '/health' ? false :   // Never log health checks
      Math.random() < 0.01              // 1% sample of normal requests
    
    if (shouldLog) {
      logger.info({
        method: req.method,
        path: req.path,
        statusCode: res.statusCode,
        duration: Date.now() - start,
        traceId: req.traceId,
      }, 'HTTP request')
    }
  })
  
  next()
})
```

---

## 4. Distributed Tracing

### OpenTelemetry SDK

```typescript
// Initialize tracing (must be first import!)
// instrumentation.ts — run before any other code
import { NodeSDK } from '@opentelemetry/sdk-node'
import { OTLPTraceExporter } from '@opentelemetry/exporter-trace-otlp-http'
import { Resource } from '@opentelemetry/resources'
import { SEMRESATTRS_SERVICE_NAME, SEMRESATTRS_SERVICE_VERSION } from '@opentelemetry/semantic-conventions'
import { getNodeAutoInstrumentations } from '@opentelemetry/auto-instrumentations-node'

const sdk = new NodeSDK({
  resource: new Resource({
    [SEMRESATTRS_SERVICE_NAME]: 'booking-service',
    [SEMRESATTRS_SERVICE_VERSION]: process.env.APP_VERSION ?? '0.0.0',
    environment: process.env.NODE_ENV ?? 'development',
  }),
  traceExporter: new OTLPTraceExporter({
    url: process.env.OTEL_EXPORTER_OTLP_ENDPOINT ?? 'http://otel-collector:4318/v1/traces',
  }),
  instrumentations: [
    getNodeAutoInstrumentations({
      '@opentelemetry/instrumentation-http': { enabled: true },
      '@opentelemetry/instrumentation-express': { enabled: true },
      '@opentelemetry/instrumentation-pg': { enabled: true },
      '@opentelemetry/instrumentation-redis': { enabled: true },
    }),
  ],
})

sdk.start()
process.on('SIGTERM', () => sdk.shutdown())
```

```typescript
// Manual instrumentation for business logic
import { trace, context, SpanStatusCode } from '@opentelemetry/api'

const tracer = trace.getTracer('booking-service', '1.0.0')

async function createBooking(input: CreateBookingInput, userId: string): Promise<Booking> {
  return tracer.startActiveSpan('booking.create', async (span) => {
    span.setAttribute('user.id', userId)
    span.setAttribute('service.id', input.serviceId)
    
    try {
      // Check availability
      const available = await tracer.startActiveSpan('booking.checkAvailability', async (childSpan) => {
        childSpan.setAttribute('scheduled_at', input.scheduledAt)
        const result = await checkAvailability(input.serviceId, input.scheduledAt)
        childSpan.setAttribute('is_available', result)
        childSpan.end()
        return result
      })
      
      if (!available) {
        span.setStatus({ code: SpanStatusCode.ERROR, message: 'Slot not available' })
        throw new BookingConflictError('Slot not available')
      }
      
      const booking = await db.query('INSERT INTO bookings ...', [input])
      
      span.setAttribute('booking.id', booking.id)
      span.setStatus({ code: SpanStatusCode.OK })
      return booking
      
    } catch (err) {
      span.recordException(err as Error)
      span.setStatus({ code: SpanStatusCode.ERROR })
      throw err
    } finally {
      span.end()
    }
  })
}
```

---

## 5. Alerting Strategy

### Alert Fatigue Prevention

```yaml
# Bad alerts: symptom-based, high noise
# Good alerts: user-impact focused, actionable

# Principles:
# 1. Every alert must be actionable (someone must do something)
# 2. Alert on symptoms, not causes (high error rate, not high CPU)
# 3. Start with paging alerts only (if not paging, don't alert)
# 4. Tune to reduce false positives (< 5% false positive rate)

# Prometheus alerting rules
groups:
  - name: service.alerts
    interval: 30s
    rules:
      # HIGH SEVERITY: user-facing, page immediately
      - alert: HighErrorRate
        expr: |
          (
            sum(rate(http_requests_total{status_code=~"5.."}[5m])) /
            sum(rate(http_requests_total[5m]))
          ) > 0.05
        for: 5m  # Must be true for 5 min (prevent flapping)
        labels:
          severity: critical
          team: platform
        annotations:
          summary: "Error rate > 5% for 5 minutes"
          description: "Current error rate: {{ $value | humanizePercentage }}"
          runbook: "https://wiki.example.com/runbooks/high-error-rate"
          dashboard: "https://grafana.example.com/d/overview"
      
      # HIGH SEVERITY: latency SLO breach
      - alert: HighP99Latency
        expr: |
          histogram_quantile(0.99, 
            sum(rate(http_request_duration_seconds_bucket[5m])) by (le)
          ) > 1
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "P99 latency > 1 second"
          runbook: "https://wiki.example.com/runbooks/high-latency"
      
      # MEDIUM: saturation
      - alert: HighMemoryUsage
        expr: |
          container_memory_usage_bytes{container="api"} / 
          container_spec_memory_limit_bytes{container="api"} > 0.85
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Memory usage > 85%"
      
      # Queue depth alert (capacity planning signal)
      - alert: JobQueueHigh
        expr: job_queue_depth{queue_name="default"} > 1000
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Job queue depth > 1000 ({{ $value }} jobs)"
```

### Alert Routing (AlertManager)

```yaml
# alertmanager.yml
route:
  group_by: ['alertname', 'team']
  group_wait: 30s       # Wait before sending initial notification
  group_interval: 5m    # Wait before sending subsequent notifications
  repeat_interval: 12h  # Repeat notification if alert still firing
  
  receiver: 'default-receiver'
  
  routes:
    # Critical alerts → PagerDuty (wake someone up)
    - match:
        severity: critical
      receiver: pagerduty
      continue: true  # Also send to Slack
    
    # All alerts → Slack
    - match_re:
        severity: "critical|warning"
      receiver: slack-alerts
    
    # Security alerts → security team
    - match:
        team: security
      receiver: security-slack

receivers:
  - name: pagerduty
    pagerduty_configs:
      - service_key: {{ PAGERDUTY_SERVICE_KEY }}
        description: '{{ template "pagerduty.default.description" . }}'
        
  - name: slack-alerts
    slack_configs:
      - api_url: {{ SLACK_WEBHOOK_URL }}
        channel: '#alerts-production'
        title: '{{ .GroupLabels.alertname }}'
        text: '{{ range .Alerts }}{{ .Annotations.description }}{{ end }}'
        color: '{{ if eq .Status "firing" }}danger{{ else }}good{{ end }}'
```

---

## 6. SLOs and Error Budgets

```
SLI (Service Level Indicator): a metric measuring service quality
  → "99.5% of requests succeed" (measured over rolling 30 days)

SLO (Service Level Objective): target for SLI
  → "requests will succeed ≥ 99.9% of the time"

SLA (Service Level Agreement): contractual commitment (penalty if SLO breached)
  → "if uptime < 99.9%, customer receives credit"

Error Budget: (1 - SLO) × time period
  → 99.9% SLO over 30 days = 0.1% allowed errors
  → 30 days × 24h × 60min × 0.1% = 43.2 minutes of downtime allowed
  → Once error budget depleted: no more deploys until budget replenishes

Using error budgets:
  Budget > 50% remaining:  Feature development, risky changes OK
  Budget 25-50% remaining: Careful with changes, more testing
  Budget < 25% remaining:  Reliability work only, no risky changes
  Budget exhausted:        Feature freeze, focus on reliability
```

```promql
# Error budget consumption (30-day window)
# SLO: 99.9% success rate

# Current error rate
1 - (
  sum(rate(http_requests_total{status_code!~"5.."}[30d])) /
  sum(rate(http_requests_total[30d]))
)

# Error budget remaining (as percentage)
(0.001 - (
  sum(rate(http_requests_total{status_code=~"5.."}[30d])) /
  sum(rate(http_requests_total[30d]))
)) / 0.001 * 100

# Burn rate alert (error budget burning too fast)
# If burning 14.4x faster than allowed: alert (at this rate, 1-hour budget exhausted in 1 day)
sum(rate(http_requests_total{status_code=~"5.."}[1h])) /
sum(rate(http_requests_total[1h])) > 14.4 * 0.001
```

---

## 7. Dashboards (Grafana)

### Service Overview Dashboard

```json
// Dashboard JSON structure (key panels)
{
  "title": "Service Overview",
  "refresh": "30s",
  "panels": [
    {
      "title": "Request Rate",
      "type": "stat",
      "targets": [{
        "expr": "sum(rate(http_requests_total[5m])) * 60",
        "legendFormat": "req/min"
      }]
    },
    {
      "title": "Error Rate",
      "type": "stat",
      "targets": [{
        "expr": "sum(rate(http_requests_total{status_code=~\"5..\"}[5m])) / sum(rate(http_requests_total[5m])) * 100",
        "legendFormat": "error %"
      }],
      "thresholds": [
        { "value": 1, "color": "yellow" },
        { "value": 5, "color": "red" }
      ]
    },
    {
      "title": "Request Duration (P50/P95/P99)",
      "type": "timeseries",
      "targets": [
        { "expr": "histogram_quantile(0.5, sum(rate(http_request_duration_seconds_bucket[5m])) by (le))", "legendFormat": "p50" },
        { "expr": "histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[5m])) by (le))", "legendFormat": "p95" },
        { "expr": "histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket[5m])) by (le))", "legendFormat": "p99" }
      ]
    }
  ]
}
```

---

## 8. OpenTelemetry

```yaml
# OpenTelemetry Collector configuration
# Acts as an agent between your services and your backends (Jaeger, Prometheus, Loki)

receivers:
  otlp:
    protocols:
      grpc:
        endpoint: "0.0.0.0:4317"
      http:
        endpoint: "0.0.0.0:4318"
  
  prometheus:
    config:
      scrape_configs:
        - job_name: 'api-service'
          static_configs:
            - targets: ['api:3000']

processors:
  batch:
    timeout: 10s
    send_batch_size: 1000
  
  memory_limiter:
    check_interval: 1s
    limit_mib: 512
  
  resource:
    attributes:
      - key: environment
        value: production
        action: insert

exporters:
  # Traces → Jaeger
  jaeger:
    endpoint: "jaeger:14250"
    tls:
      insecure: true
  
  # Metrics → Prometheus
  prometheus:
    endpoint: "0.0.0.0:8889"
  
  # Logs → Loki
  loki:
    endpoint: "http://loki:3100/loki/api/v1/push"

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [batch, memory_limiter]
      exporters: [jaeger]
    metrics:
      receivers: [otlp, prometheus]
      processors: [batch]
      exporters: [prometheus]
    logs:
      receivers: [otlp]
      processors: [batch]
      exporters: [loki]
```

---

## 9. On-Call Runbooks

```markdown
# Runbook: HighErrorRate Alert

## Alert Details
- **Alert:** HighErrorRate
- **Severity:** Critical
- **SLO Impact:** Yes — error budget burning
- **Team:** Platform

## Immediate Steps (first 5 minutes)

1. **Check dashboards:**
   - Service Overview: [link]
   - Error breakdown by route: [link]
   - Recent deployments: [link]

2. **Identify scope:**
   - Is this affecting all routes or specific endpoints?
   - Is this affecting all regions or one?
   - When did it start?

3. **Check recent changes:**
   - `git log main --since="2 hours ago"`
   - Check deployment history
   - Check database migration log

## Investigation

```bash
# Check logs for error patterns
kubectl logs -l app=api --since=10m | grep '"level":"error"' | jq .error.message | sort | uniq -c | sort -rn

# Check DB connection pool
kubectl exec -it $(kubectl get pod -l app=api -o name | head -1) -- node -e "console.log(require('./db').pool.totalCount)"

# Check downstream services
curl https://api.example.com/health/ready
```

## Escalation
- Not resolved in 15 minutes → escalate to Engineering Lead
- Database down → escalate to DBA on-call
- Payment service → escalate to Payments Team
```

---

## 10. Incident Response

```
Incident Severity:
  P0 (Critical): Total service outage, payment processing down, data loss
     → Immediate response, all hands, 15-min stakeholder updates
  P1 (High): Major feature broken, > 10% error rate, > 5s P99 latency
     → Response within 30 minutes, hourly updates
  P2 (Medium): Degraded performance, minor features broken, < 5% error rate
     → Response within 2 hours, daily updates
  P3 (Low): Cosmetic issues, non-critical features, workaround available
     → Next business day

Incident Response Process:
  1. DETECT:  Automated alert or user report
  2. DECLARE: Create incident in PagerDuty/Slack
  3. ASSIGN:  Incident Commander (IC) takes ownership
  4. DIAGNOSE: Understand what's broken (not why yet)
  5. MITIGATE: Stop the bleeding (rollback, disable feature flag, scale up)
  6. RESOLVE:  Fix root cause or confirm mitigated
  7. POST-MORTEM: Blameless review within 72 hours

Communication template:
  [Incident #123 - ACTIVE]
  Impact: Users cannot complete bookings since 14:30 UTC
  Status: Investigating — identified issue with payment gateway
  Next update: 15:00 UTC
  IC: @engineer-name

Post-mortem (blameless) template:
  ## Summary
  ## Timeline
  ## Root Cause
  ## Contributing Factors
  ## What Went Well
  ## What Went Poorly
  ## Action Items (with owner and due date)
```
