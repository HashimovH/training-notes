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
