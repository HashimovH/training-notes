# 🚨 INCIDENT RESPONSE CHECKLIST

## PHASE 1: DETECTION (First 2 minutes)
- [ ] Acknowledge the alert in your paging system
- [ ] Open the service dashboard for affected service(s)
- [ ] Check: Is this a real incident or false positive?
- [ ] If real: Declare incident severity (SEV1/SEV2/SEV3)
- [ ] Start incident channel (Slack: #inc-YYYYMMDD-brief-description)

## PHASE 2: TRIAGE (Minutes 2-10)
- [ ] What is the user impact? (% affected, which features)
- [ ] When did it start? (Check metrics for change point)
- [ ] What changed recently? (Deploys, config, traffic spike)
- [ ] Run these diagnostic queries:

    # Error rate by service
    sum(rate(http_requests_total{status=~"5.."}[5m])) by (service)
    
    # Latency P99 by service  
    histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket[5m])) by (le, service))
    
    # Recent deployments
    changes(deployment_info[1h])

## PHASE 3: DIAGNOSIS (Minutes 10-30)
- [ ] Find a failing trace ID from logs/errors
- [ ] Open trace in your tracing UI
- [ ] Identify: Which span is slow/failing?
- [ ] Check span attributes for error details
- [ ] Correlate with infrastructure metrics (CPU, memory, connections)

## PHASE 4: MITIGATION (As soon as diagnosis is clear)
- [ ] Can we rollback? (Fastest option if deploy-related)
- [ ] Can we scale up? (If capacity-related)
- [ ] Can we failover? (If dependency-related)
- [ ] Can we feature-flag disable? (If feature-related)
- [ ] Implement mitigation and verify metrics recover

## PHASE 5: RESOLUTION & FOLLOW-UP
- [ ] Confirm metrics are back to baseline
- [ ] Update status page
- [ ] Send all-clear to stakeholders
- [ ] Schedule postmortem within 48 hours
- [ ] Create follow-up tickets for permanent fixes
