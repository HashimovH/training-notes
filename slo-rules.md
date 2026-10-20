SLO Recording rules
```yaml
groups:
  - name: slo_recording_rules
    interval: 30s
    rules:
      # ========================================
      # Availability SLO Recording Rules
      # ========================================
      
      # Total requests (denominator)
      - record: slo:http_requests:rate5m
        expr: sum(rate(http_server_request_duration_seconds_count[5m])) by (service_name)
      
      # Successful requests (numerator)
      - record: slo:http_requests_success:rate5m
        expr: |
          sum(rate(http_server_request_duration_seconds_count{
            http_status_code!~"5.."
          }[5m])) by (service_name)
      
      # Availability ratio (SLI value)
      - record: slo:availability:ratio5m
        expr: |
          slo:http_requests_success:rate5m
          /
          slo:http_requests:rate5m
      
      # ========================================
      # Latency SLO Recording Rules
      # ========================================
      
      # P50 latency
      - record: slo:http_latency:p50_5m
        expr: |
          histogram_quantile(0.50,
            sum(rate(http_server_request_duration_seconds_bucket[5m])) 
            by (service_name, le)
          )
      
      # P99 latency
      - record: slo:http_latency:p99_5m
        expr: |
          histogram_quantile(0.99,
            sum(rate(http_server_request_duration_seconds_bucket[5m])) 
            by (service_name, le)
          )
      
      # Requests within latency threshold (500ms)
      - record: slo:http_latency_good:rate5m
        expr: |
          sum(rate(http_server_request_duration_seconds_bucket{le="0.5"}[5m])) 
          by (service_name)
      
      # Latency SLI ratio
      - record: slo:latency:ratio5m
        expr: |
          slo:http_latency_good:rate5m
          /
          slo:http_requests:rate5m
```

Burn Rate Alert
```
groups:
  - name: slo_burn_rate_alerts
    rules:
      # ============================================
      # Fast burn alerts (catch severe incidents quickly)
      # ============================================
      
      # 14.4x burn rate over 1 hour = budget gone in ~2 days
      # 14.4x burn rate over 5 min = confirm it's real, not a blip
      - alert: SLOBurnRateCritical
        expr: |
          (
            (1 - slo:availability:ratio1h) / (1 - 0.999) > 14.4
          )
          and
          (
            (1 - slo:availability:ratio5m) / (1 - 0.999) > 14.4
          )
        for: 2m
        labels:
          severity: critical
          slo: availability
        annotations:
          summary: "Availability SLO burning 14x faster than budget allows"
          description: |
            Service {{ $labels.service_name }} is consuming error budget 
            at {{ $value | humanize }}x the sustainable rate.
            At this rate, the 30-day budget will be exhausted in ~2 days.
          runbook_url: "https://runbooks.example.com/slo-critical"
      
      # ============================================
      # Medium burn alerts (significant degradation)  
      # ============================================
      
      # 6x burn rate over 6 hours
      - alert: SLOBurnRateHigh
        expr: |
          (
            (1 - slo:availability:ratio6h) / (1 - 0.999) > 6
          )
          and
          (
            (1 - slo:availability:ratio30m) / (1 - 0.999) > 6
          )
        for: 5m
        labels:
          severity: warning
          slo: availability
        annotations:
          summary: "Availability SLO burning 6x faster than sustainable"
          description: |
            Service {{ $labels.service_name }} has elevated error rate.
            Budget will exhaust in ~5 days at current rate.
      
      # ============================================
      # Slow burn alerts (chronic issues)
      # ============================================
      
      # 3x burn rate over 1 day - catches slow degradation
      - alert: SLOBurnRateElevated
        expr: |
          (
            (1 - slo:availability:ratio1d) / (1 - 0.999) > 3
          )
          and
          (
            (1 - slo:availability:ratio2h) / (1 - 0.999) > 3
          )
        for: 15m
        labels:
          severity: warning
          slo: availability
        annotations:
          summary: "Elevated error rate consuming budget faster than expected"
          description: |
            Service {{ $labels.service_name }} has sustained elevated errors.
            Investigate for chronic issues or recent changes.
```

SLO Dashboard
```
{
  "title": "SLO Overview Dashboard",
  "uid": "slo-overview",
  "tags": ["slo", "reliability", "sre"],
  "timezone": "browser",
  "refresh": "30s",
  "panels": [
    {
      "title": "Overall SLO Status",
      "type": "stat",
      "gridPos": { "h": 4, "w": 6, "x": 0, "y": 0 },
      "targets": [{
        "expr": "avg(slo:availability:ratio30d) * 100",
        "legendFormat": "Availability %"
      }],
      "options": {
        "colorMode": "value",
        "graphMode": "none",
        "textMode": "value"
      },
      "fieldConfig": {
        "defaults": {
          "unit": "percent",
          "decimals": 2,
          "thresholds": {
            "mode": "absolute",
            "steps": [
              { "color": "red", "value": 99 },
              { "color": "yellow", "value": 99.5 },
              { "color": "green", "value": 99.9 }
            ]
          }
        }
      }
    },
    {
      "title": "Error Budget Remaining",
      "type": "gauge",
      "gridPos": { "h": 4, "w": 6, "x": 6, "y": 0 },
      "targets": [{
        "expr": "(1 - ((1 - slo:availability:ratio30d) / 0.001)) * 100",
        "legendFormat": "Budget %"
      }],
      "options": {
        "showThresholdLabels": false,
        "showThresholdMarkers": true
      },
      "fieldConfig": {
        "defaults": {
          "unit": "percent",
          "min": 0,
          "max": 100,
          "thresholds": {
            "mode": "absolute", 
            "steps": [
              { "color": "red", "value": 0 },
              { "color": "orange", "value": 25 },
              { "color": "yellow", "value": 50 },
              { "color": "green", "value": 75 }
            ]
          }
        }
      }
    }
  ]
}
```

Burn rate dashboard
```
{
  "title": "Current Burn Rate",
  "type": "timeseries",
  "gridPos": { "h": 8, "w": 12, "x": 0, "y": 4 },
  "targets": [
    {
      "expr": "(1 - slo:availability:ratio1h) / 0.001",
      "legendFormat": "1h Burn Rate"
    },
    {
      "expr": "(1 - slo:availability:ratio6h) / 0.001", 
      "legendFormat": "6h Burn Rate"
    },
    {
      "expr": "(1 - slo:availability:ratio1d) / 0.001",
      "legendFormat": "1d Burn Rate"
    }
  ],
  "options": {
    "tooltip": { "mode": "multi" },
    "legend": { "displayMode": "table", "placement": "right" }
  },
  "fieldConfig": {
    "defaults": {
      "custom": {
        "lineWidth": 2,
        "fillOpacity": 10,
        "spanNulls": true
      },
      "thresholds": {
        "mode": "absolute",
        "steps": [
          { "color": "green", "value": 0 },
          { "color": "yellow", "value": 1 },
          { "color": "orange", "value": 3 },
          { "color": "red", "value": 6 }
        ]
      }
    },
    "overrides": [{
      "matcher": { "id": "byName", "options": "1x Sustainable" },
      "properties": [{
        "id": "custom.lineStyle",
        "value": { "fill": "dash", "dash": [10, 10] }
      }]
    }]
  }
}
```
