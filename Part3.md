# Part 3 – Production Scenario

## First 5 Minutes – Stop the Bleeding

My first priority would be to reduce the blast radius before investigating the root cause.

- Temporarily pause new payout requests using a feature flag or maintenance mode if duplicate payouts are suspected.
- Keep wallet balance and other read-only APIs available to minimize customer impact.
- Inform the support and engineering teams that payouts are temporarily paused while the incident is being investigated.

---

## Investigate the Application

I would start by reviewing the application logs for:
- Timeout errors
- Database exceptions
- Gateway failures
- Retry spikes
- Error rate

Useful commands:

```bash
kubectl logs <payment-service-pod> --since=30m
kubectl get pods
kubectl describe pod <payment-service-pod>
```

I would also verify whether a recent deployment or configuration change matches the time when latency started increasing.

---

## Check MongoDB Atlas

I would review the following metrics:

- p95 query latency
- Slow queries
- CPU utilization
- Memory usage
- Connection count
- Lock percentage
- Disk I/O
- Query Profiler

If I notice collection scans or slow queries, I would verify whether required indexes are missing.

---

## Check the Payment Gateway

I would check the gateway dashboard for:

- API response time
- Success vs failed payouts
- HTTP 429 (rate limiting)
- HTTP 5xx errors
- Service outage notifications

This helps determine whether the issue is caused by our application or the external payment provider.

---

## Decision Making

- If MongoDB is the bottleneck, I would optimize the affected queries, verify indexes, or scale the database.
- If the payment gateway is slow or unavailable, I would queue payout requests and process them once the gateway becomes healthy.
- If application servers are overloaded, I would scale the service and investigate expensive requests.

---

## What I Would Not Do

I would not immediately restart services or roll back the deployment without evidence, as this could interrupt ongoing payout processing and make the root cause harder to identify. I would first confirm whether the issue is in the application, database, or payment gateway before making production changes.