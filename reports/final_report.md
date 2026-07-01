# Day 10 Reliability Report

## 1. Architecture summary

Describe your gateway, circuit breaker, fallback chain, and cache layers.
Include a simple diagram (text/ASCII is fine):

The gateway orchestrates the reliable execution of incoming requests. It first attempts a semantic cache lookup. If the query is clean (non-PII) and has high similarity to a cached query without false-hit mismatches (differing years/IDs), it returns the response immediately. On a cache miss, the gateway routes the request through a chain of providers (Primary, then Backup). Each provider is wrapped in a Circuit Breaker that monitors consecutive failures. If a provider fails repeatedly, its circuit trips to `OPEN`, causing future requests to fail fast and route directly to the next provider, avoiding retry storms. If all providers fail or are blocked, the gateway falls back to a static degraded response.

```
User Request
    |
    v
[Gateway] ---> [Cache check] ---> HIT? return cached
    |                                 |
    v                                 v MISS
[Circuit Breaker: Primary] -------> Provider A
    |  (OPEN? skip)
    v
[Circuit Breaker: Backup] --------> Provider B
    |  (OPEN? skip)
    v
[Static fallback message]
```

## 2. Configuration

| Setting | Value | Reason |
|---|---:|---|
| failure_threshold | 3 | Tripping threshold to tolerate short network blips while blocking prolonged outages. |
| reset_timeout_seconds | 2 | Cooling period before probing the provider again in HALF_OPEN. |
| success_threshold | 1 | Single successful probe restores the circuit to CLOSED. |
| cache TTL | 300 | Caches query responses for 5 minutes to balance data freshness and hits. |
| similarity_threshold | 0.92 | High threshold to prevent false semantic hits on similar text with different intents. |
| load_test requests | 100 | Performs 100 requests per scenario to obtain stable statistics. |

## 3. SLO definitions

Define your target SLOs and whether your system meets them:

| SLI | SLO target | Actual value | Met? |
|---|---|---:|---|
| Availability | >= 99% | 99.67% | **Yes** |
| Latency P95 | < 2500 ms | 317.21 ms | **Yes** |
| Fallback success rate | >= 95% | 98.39% | **Yes** |
| Cache hit rate | >= 10% | 73.00% | **Yes** |
| Recovery time | < 5000 ms | 2491.29 ms | **Yes** |

## 4. Metrics

Paste or summarize `reports/metrics.json`.

| Metric | Value |
|---|---:|
| availability | 0.9967 |
| error_rate | 0.0033 |
| latency_p50_ms | 278.61 |
| latency_p95_ms | 317.21 |
| latency_p99_ms | 321.12 |
| fallback_success_rate | 0.9839 |
| cache_hit_rate | 0.7300 |
| estimated_cost_saved | 0.2190 |
| circuit_open_count | 8 |
| recovery_time_ms | 2491.29 |

## 5. Cache comparison

Run simulation with cache enabled vs disabled. Fill in both columns:

| Metric | Without cache | With cache | Delta |
|---|---:|---:|---|
| latency_p50_ms | 281.08 ms | 278.61 ms | -2.47 ms (-0.88%) |
| latency_p95_ms | 318.69 ms | 317.21 ms | -1.48 ms (-0.46%) |
| estimated_cost | $0.124034 | $0.033222 | -$0.090812 (-73.21%) |
| cache_hit_rate | 0 | 0.7300 | +73.00% |

*Note: Latencies are only calculated for requests that hit a provider (latency > 0). This explains why the P50 and P95 latencies are similar between cache-enabled and cache-disabled scenarios. However, the cache-enabled scenario yields a massive 73.21% reduction in estimated costs because 73.00% of the queries were served from the cache without calling the upstream providers.*

## 6. Redis shared cache

Explain why shared cache matters for production:

- Why in-memory cache is insufficient for multi-instance deployments: In-memory cache is isolated inside a single application process. When running multiple gateway instances behind a load balancer, they cannot share their local caches. This leads to duplicate provider calls, higher overall costs, and cache inconsistency.
- How `SharedRedisCache` solves this: Externalizes the cache storage to a centralized Redis cluster. All gateway instances write to and read from the same Redis server, ensuring that a cached response from one instance is immediately available to all other instances.

### Evidence of shared state

Show that two separate cache instances can see the same data:

```
pytest tests/test_redis_cache.py -k test_shared_state_across_instances -v
============================= test session starts ==============================
platform darwin -- Python 3.14.3, pytest-9.0.3, pluggy-1.6.0
collected 6 items / 5 deselected / 1 selected

tests/test_redis_cache.py::test_shared_state_across_instances PASSED     [100%]
======================= 1 passed, 5 deselected in 0.04s ========================
```

### Redis CLI output

```bash
# docker compose exec redis redis-cli KEYS "rl:cache:*"
1) "rl:cache:dacb2b833659"
2) "rl:cache:3dab98c0e49e"
3) "rl:cache:98332d0d1c9c"
4) "rl:cache:8baa2cfa11fa"
5) "rl:cache:844ef0143a5c"
6) "rl:cache:b2a52f7dc795"
7) "rl:cache:d354658dc020"
8) "rl:cache:fff10da1c72c"
9) "rl:cache:0bc3b1acf73d"
10) "rl:cache:9e413fd814eb"
11) "rl:cache:095946136fea"
12) "rl:cache:734852f3cf4a"
```

### In-memory vs Redis latency comparison (optional)

| Metric | In-memory cache | Redis cache | Notes |
|---|---:|---:|---|
| latency_p50_ms | 276.48 ms | 278.61 ms | Redis adds a slight TCP overhead but ensures shared state. |
| latency_p95_ms | 318.95 ms | 317.21 ms | The difference is negligible under simulated network latency. |

## 7. Chaos scenarios

| Scenario | Expected behavior | Observed behavior | Pass/Fail |
|---|---|---|---|
| primary_timeout_100 | All traffic fallback to backup, circuit opens | Primary circuit trips to OPEN, traffic redirects to Backup, availability is maintained | Pass |
| primary_flaky_50 | Circuit oscillates, mix of primary and fallback | Primary circuit opens and closes dynamically, distributing requests across providers | Pass |
| all_healthy | All requests via primary, no circuit opens | Primary circuit remains CLOSED, all queries go to Primary provider | Pass |
| primary_dead_backup_flaky | Traffic flows to backup, backup trips occasionally, fallback to static | Primary circuit remains OPEN, backup circuit oscillates, degraded messages served on failure | Pass |

## 8. Failure analysis

Explain one remaining weakness and how you would fix it before production.

- What could still go wrong?
  If the Redis cluster is unreachable, network connection timeouts or exceptions thrown by the `redis` client will cause the Gateway requests to fail completely, bringing down the service.
- What would you change? (e.g., Redis circuit state, per-user rate limiting, quality SLO)
  Implement a local backup cache (in-memory) as a secondary fallback layer (graceful cache degradation). If Redis client throws a connection error, catch the exception, bypass the shared cache, and use the local in-memory cache or route directly to providers. Also, store circuit breaker failure counters in Redis so that tripping a circuit on one instance trips it globally.

## 9. Next steps

List 2-3 concrete improvements you would make:

1. **Implement Graceful Cache Degradation**: Add exception handling for Redis connection errors to fall back automatically to in-memory caching.
2. **Global Redis-backed Circuit Breakers**: Move circuit breaker states (failure counters and timestamps) to Redis so that all instances share a unified, real-time view of provider health.
3. **Cost Budget-aware Routing**: Add budget tracking to automatically redirect traffic to cheaper models or enforce strict caching once a budget limit is reached.