# Day 10 Reliability Final Report

## 1. Architecture summary

The system is built as a highly reliable API gateway for LLM providers. It implements a layered reliability pipeline:

1. **Gateway Layer**: Evaluates incoming prompts, checks the cache first, then orchestrates routing through circuit-breaker-wrapped provider clients.
2. **Semantic Caching Layer**: Uses n-gram cosine similarity (character 3-grams and word tokens) to identify semantically equivalent queries, preventing duplicate expensive provider calls. Features privacy filters and false-hit detection to safeguard data integrity. Supports both in-memory and shared Redis backends.
3. **Circuit Breaker Layer**: Implements a 3-state machine (`CLOSED`, `OPEN`, `HALF_OPEN`) per provider. Trips on repeated provider failures to prevent overloading upstream systems, falling back to backup models.
4. **Fallback Layer**: Falls back to secondary provider models, and finally to a static fallback response if all upstream providers are unreachable or open.

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

### Detailed Request Flow (Quy trình xử lý đầy đủ)
The gateway handles incoming requests through the following step-by-step execution pipeline:

1. **Cache Check & Privacy Guardrail**: The gateway invokes `cache.get(prompt)`.
   - **Privacy Filtering**: If the query matches `PRIVACY_PATTERNS` (e.g., contains words like `balance`, `password`, `ssn`, `credit card`, `user 123`), it is blocked from cache storage and lookup to prevent sensitive data leaks.
   - **Similarity Matching**: If a cached response exists with an n-gram cosine similarity score $\ge$ threshold (`0.92`), and it is not flagged as a false hit, the response is returned immediately with `route=cache_hit` (0ms latency, $0 cost).
2. **False-Hit Guard**: Semantically similar queries containing different 4-digit numbers (e.g., year numbers like 2024 vs 2026) are rejected as false hits and logged. This prevents serving outdated/incorrect year-specific information.
3. **Primary Provider Execution with Circuit Breaker**: If the cache misses, the gateway queries the primary provider (Provider A).
   - **Fail-Fast**: If Provider A's circuit breaker is in the `OPEN` state, the request is immediately skipped to prevent retry storms.
   - **Call & Track**: If the breaker is `CLOSED` or `HALF_OPEN`, Provider A is queried. A success calls `record_success()` (closing the circuit). An exception calls `record_failure()` (tripping the circuit to `OPEN` after 3 consecutive failures).
4. **Backup Fallback Routing**: If the primary provider fails or its breaker is open, the gateway attempts to query the backup provider (Provider B) via its own independent circuit breaker. On success, the response is cached, and the gateway returns the response with `route="fallback"`.
5. **Static Fallback**: If all configured providers fail or are blocked, the gateway recovers gracefully by returning a static degraded message (`"The service is temporarily degraded. Please try again soon."`), with `route="static_fallback"` and the last encountered error. Upstream raw errors are never exposed directly to clients.
6. **Self-Healing Recovery**: An `OPEN` circuit breaker waits for `reset_timeout_seconds = 2` seconds, then transitions to `HALF_OPEN` to permit a single probe request. If the probe succeeds, the circuit closes (`CLOSED`), resetting failure counters. If it fails, the circuit immediately re-opens (`OPEN`).

## 2. Configuration

| Setting | Value | Reason |
|---|---:|---|
| failure_threshold | 3 | Opens the circuit after 3 consecutive failures to avoid retry storms on down providers. |
| reset_timeout_seconds | 2 | Gives a down provider 2 seconds to recover before probing it. |
| success_threshold | 1 | Restores the circuit to CLOSED immediately after a single successful probe in HALF_OPEN. |
| cache TTL | 300 | Caches query responses for 5 minutes to balance freshness and cache hit rates. |
| similarity_threshold | 0.92 | High threshold to prevent cache collisions on similar-looking queries with different intent. |
| load_test requests | 100 | Performs 100 requests per scenario to gather robust statistics. |

### False-Hit Detection Examples
To ensure semantic queries do not return false positives when key qualifiers differ, the cache validates matches against a numeric mismatch check (`_looks_like_false_hit`):
- **Example 1 (Different Years)**:
  - Cache contains: `"Summarize refund policy for 2024 deadline"`
  - Query: `"Summarize refund policy for 2026 deadline"`
  - *Result*: The similarity score is high due to overlapping tokens (`Summarize`, `refund`, `policy`, `deadline`), but the different 4-digit years (`2024` vs `2026`) trigger the false-hit detector. The cache blocks the response, logs it, and returns `None` to force a fresh model lookup.
- **Example 2 (Same Year, Correct Hit)**:
  - Cache contains: `"Summarize refund policy for 2024 deadline"`
  - Query: `"Summarize refund policy for 2024"`
  - *Result*: Since both keys share the same year (`2024`), the false-hit guard does not trigger, and the correct cached response is successfully served.

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

Past metrics from `reports/metrics.json` running with Redis cache:

| Metric | Value |
|---|---:|
| availability | 0.9967 |
| error_rate | 0.0033 |
| latency_p50_ms | 278.61 |
| latency_p95_ms | 317.21 |
| latency_p99_ms | 321.12 |
| fallback_success_rate | 0.9839 |
| cache_hit_rate | 0.7300 |
| estimated_cost_saved | 0.219000 |
| circuit_open_count | 8 |
| recovery_time_ms | 2491.2856 |

## 5. Cache comparison

We ran the chaos simulation under the exact same parameters with the cache enabled vs disabled:

| Metric | Without cache | With cache | Delta |
|---|---:|---:|---|
| latency_p50_ms | 281.08 | 278.61 | -2.47 ms (-0.88%) |
| latency_p95_ms | 318.69 | 317.21 | -1.48 ms (-0.46%) |
| estimated_cost | 0.124034 | 0.033222 | -0.090812 (-73.21%) |
| cache_hit_rate | 0 | 0.7300 | +73.00% |

**Latency Delta Analysis**: Latency percentiles are calculated only for requests that are actually sent to the providers (latency > 0). Since cache hits do not go to the providers, they are not counted in the latency list. This explains why the P50 and P95 latencies are similar. However, the estimated cost decreased by **73.21%** because 73.00% of the requests were served from the cache without calling the expensive provider.

## 6. Redis shared cache

### Why in-memory cache is insufficient for multi-instance deployments
In-memory caching is isolated to a single application process. In a scaled environment running behind a load balancer, instances cannot share cached entries. A cache hit on Instance A is still a cache miss on Instance B. This leads to duplicate provider calls, higher overall latencies, and redundant cost overhead.

### How `SharedRedisCache` solves this
`SharedRedisCache` externalizes the cache to a shared Redis cluster. All instances communicate with Redis, making a cache entry stored by Instance A immediately available to Instance B. This guarantees cache consistency and maximizes the hit rate across the entire deployment.

### Evidence of shared state

We verified shared cache state across instances using a test fixture that writes to one cache instance and reads from another:
```bash
pytest tests/test_redis_cache.py -k test_shared_state_across_instances -v
```
Output:
```
tests/test_redis_cache.py::test_shared_state_across_instances PASSED     [100%]
```

### Redis CLI output

List of keys stored in Redis during the chaos simulation run (prefix: `rl:cache:`):
```
1) "rl:cache:d354658dc020"
2) "rl:cache:095946136fea"
3) "rl:cache:b2a52f7dc795"
4) "rl:cache:9e413fd814eb"
5) "rl:cache:844ef0143a5c"
6) "rl:cache:dacb2b833659"
7) "rl:cache:fff10da1c72c"
8) "rl:cache:0bc3b1acf73d"
9) "rl:cache:8baa2cfa11fa"
10) "rl:cache:734852f3cf4a"
11) "rl:cache:98332d0d1c9c"
12) "rl:cache:3dab98c0e49e"
```

## 7. Chaos scenarios

| Scenario | Expected behavior | Observed behavior | Pass/Fail |
|---|---|---|---|
| primary_timeout_100 | All traffic fallback to backup, circuit opens | Primary circuit opens, requests fallback, backup successfully processes them | Pass |
| primary_flaky_50 | Circuit oscillates, mix of primary and fallback | Primary circuit periodically opens and closes, splitting traffic dynamically | Pass |
| all_healthy | All requests via primary, no circuit opens | Primary breaker stays CLOSED, zero fallback requests, all served by primary | Pass |

## 8. Failure analysis

### What could still go wrong?
If Redis is down or experiences latency, `SharedRedisCache` requests may fail or slow down the gateway pipeline. The current implementation does not handle Redis exceptions gracefully, which would crash the gateway or cause timeouts.

### What would you change?
1. **Graceful Cache Degradation**: Fall back to in-memory caching or bypass the cache entirely (direct provider calls) if the Redis connection is lost.
2. **Distributed Circuit Breaker States**: Store circuit breaker metrics and state transitions in Redis (e.g., using Redis incrementers and counters) so that multiple gateway instances share a unified view of provider health.

## 9. Next steps

1. **Redis Graceful Degradation**: Catch connection errors and fall back to local `ResponseCache` or direct provider routing.
2. **Redis-backed Circuit Breakers**: Share circuit breaker failure counters in Redis so that tripping a circuit on one instance trips it globally.
3. **Budget-aware Routing**: Add cost tracking to the gateway to skip expensive providers entirely once a cost budget is reached.