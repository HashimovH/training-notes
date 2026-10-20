PromQL for incident resolution

# 1. ERROR RATE - Is something broken?
sum(rate(http_requests_total{status=~"5.."}[5m])) 
  / sum(rate(http_requests_total[5m])) * 100
# Returns: Error percentage (e.g., 5.2 means 5.2% errors)

# 2. ERROR RATE BY SERVICE - Where is it broken?
sum(rate(http_requests_total{status=~"5.."}[5m])) by (service)
  / sum(rate(http_requests_total[5m])) by (service) * 100

# 3. LATENCY SPIKE - Is something slow?
histogram_quantile(0.99,
  sum(rate(http_request_duration_seconds_bucket[5m])) by (le, service)
)

# 4. DEPENDENCY HEALTH - Is an external service the cause?
sum(rate(http_client_requests_total{status=~"5.."}[5m])) by (target_host)
  / sum(rate(http_client_requests_total[5m])) by (target_host) * 100

# 5. RESOURCE SATURATION - Are we out of capacity?
# CPU
100 - (avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
# Memory
(1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100
# Database connections
pg_stat_activity_count / pg_settings_max_connections * 100

# 6. TRAFFIC ANOMALY - Did traffic spike unexpectedly?
sum(rate(http_requests_total[5m])) 
  / sum(rate(http_requests_total[5m] offset 1d)) 
# Returns: Ratio vs same time yesterday (>1.5 = 50% spike)

Quantifying Impact with PromQL



Before optimizing, quantify the potential impact using span metrics. A span taking 50% of request time is worth optimizing. A span taking 1% probably is not.

# Top 10 slowest operations by P99 latency
topk(10,
  histogram_quantile(0.99,
    sum(rate(traces_spanmetrics_latency_bucket[5m])) by (service_name, span_name, le)
  )
)

# Operations where P99 is 10x worse than P50 (outlier pattern)
histogram_quantile(0.99,
  sum(rate(traces_spanmetrics_latency_bucket[5m])) by (service_name, span_name, le)
)
/
histogram_quantile(0.50,
  sum(rate(traces_spanmetrics_latency_bucket[5m])) by (service_name, span_name, le)
) > 10

# Detect N+1 pattern: operations called far more than their parent
# A span called 50x per request is almost certainly an N+1 problem
sum(rate(traces_spanmetrics_calls_total[5m])) by (service_name, span_name)
/ sum(rate(http_requests_total[5m])) by (service_name) > 20




Finding errors with PromQL

# Error rate by service — find who is failing
sum(rate(traces_spanmetrics_calls_total{status_code="STATUS_CODE_ERROR"}[5m])) by (service_name)
/
sum(rate(traces_spanmetrics_calls_total[5m])) by (service_name)

# Error rate by operation — find which endpoint is failing
sum(rate(traces_spanmetrics_calls_total{status_code="STATUS_CODE_ERROR"}[5m])) by (service_name, span_name)
/
sum(rate(traces_spanmetrics_calls_total[5m])) by (service_name, span_name)

# Alert: error rate > 1% for any service for 5 minutes
sum(rate(traces_spanmetrics_calls_total{status_code="STATUS_CODE_ERROR"}[5m])) by (service_name)
/
sum(rate(traces_spanmetrics_calls_total[5m])) by (service_name)
> 0.01

# Sudden error rate increase (5x worse than 1 hour ago)
sum(rate(traces_spanmetrics_calls_total{status_code="STATUS_CODE_ERROR"}[5m])) by (service_name)
/
sum(rate(traces_spanmetrics_calls_total[5m] offset 1h)) by (service_name)
> 5


PromQL Queries for Trace Metrics

# === SPAN METRICS (from spanmetrics connector) ===

# Request rate by service and operation
sum(rate(traces_spanmetrics_calls_total[5m])) by (service_name, span_name)

# Error rate by service
sum(rate(traces_spanmetrics_calls_total{status_code="STATUS_CODE_ERROR"}[5m])) by (service_name)
/
sum(rate(traces_spanmetrics_calls_total[5m])) by (service_name)

# P99 latency by service (histogram)
histogram_quantile(0.99, 
  sum(rate(traces_spanmetrics_latency_bucket[5m])) by (service_name, le)
)

# P95 latency by operation
histogram_quantile(0.95,
  sum(rate(traces_spanmetrics_latency_bucket[5m])) by (service_name, span_name, le)
)

# Slowest operations (average > 500ms)
topk(10,
  sum(rate(traces_spanmetrics_latency_sum[5m])) by (service_name, span_name)
  /
  sum(rate(traces_spanmetrics_latency_count[5m])) by (service_name, span_name)
  > 0.5
)

# Services with highest error rates
topk(5,
  sum(rate(traces_spanmetrics_calls_total{status_code="STATUS_CODE_ERROR"}[5m])) by (service_name)
  /
  sum(rate(traces_spanmetrics_calls_total[5m])) by (service_name)
)

# Requests per minute by HTTP status
sum(rate(traces_spanmetrics_calls_total[1m])) by (http_status_code) * 60
