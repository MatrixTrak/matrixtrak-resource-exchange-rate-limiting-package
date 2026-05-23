# Exchange Rate Limit Decision Checklist

Use this before deploying a trading bot to any exchange.

## Pre-Deployment Checklist

### 1. Identify the Rate-Limit Architecture

- [ ] Exchange documentation explicitly states the rate-limit type (fixed window / leaky bucket / other)
- [ ] If not stated, run the burst test (see detection section below)
- [ ] Record the limit (requests per window) and window duration
- [ ] Identify whether limits are global or per-endpoint

**Example:**
```
Binance: 1200 requests per 1 minute (fixed window)
Kraken: 15 API calls per 15 seconds (fixed window)
Coinbase: 5 requests per 1 second per user_agent (leaky bucket)
```

### 2. Set Safe Defaults

- [ ] `max_retries` is set (usually 3–5)
- [ ] `base_wait_seconds` is non-zero (at least 1 second or 500ms for fast buckets)
- [ ] `max_wait_seconds` is reasonable (30–60 seconds, or 10 for leaky bucket)
- [ ] `jitter_enabled` is TRUE (to avoid thundering herd with other bots)
- [ ] `respect_retry_after` is TRUE (honor the Retry-After header from the exchange)

### 3. Implement Backoff

- [ ] Exponential backoff is implemented (2^n formula, not linear or fixed)
- [ ] Jitter is added to backoff (random 0–1 second typically)
- [ ] Stop rules are defined (after N retries, stop and escalate, not retry forever)

**Pseudocode:**
```
wait_time = base_wait_seconds * (2 ^ (attempt - 1))
jitter = random(0, 1_second)
total_wait = min(wait_time + jitter, max_wait_seconds)
if attempt > max_retries:
  STOP and escalate (create alert / incident)
```

### 4. Logging & Observability

- [ ] Every 429 error logs these fields:
  - `timestamp` (ISO-8601)
  - `exchange` (which exchange)
  - `endpoint` (which API endpoint)
  - `http_status` (429)
  - `retry_after_header` (what exchange told us)
  - `request_weight` (if applicable)
  - `cumulative_weight_used` (running total in window)
  - `retry_count` (which attempt)
  - `wait_duration_ms` (how long we waited)
  - `backoff_strategy` (exponential / fixed / other)

- [ ] A dashboard is built to show:
  - Rate limit hit frequency (how often we hit 429s)
  - Retry success rate (% of retries that eventually succeeded)
  - Ban detection (sustained 429s over 2+ minutes = likely ban)

### 5. Circuit Breaker / Ban Detection

- [ ] If 429s occur for more than 2–3 minutes continuously:
  - STOP all requests to that exchange
  - Log an alert (potential ban)
  - Do NOT retry blindly
  
- [ ] Circuit breaker is implemented:
  - "Open" state: all requests fail immediately (don't even try)
  - Tentative "half-open" state: try 1 request after 30 seconds
  - "Closed" state: resume normal operation

### 6. Testing Before Deployment

- [ ] Burst test: send 100 requests and observe the pattern
  - Fixed window: expect burst of 429s all at once, then silence
  - Leaky bucket: expect 429s spread over time
  
- [ ] Slow test: send requests at the declared rate (e.g., 1200/60sec = 20/sec) and verify no 429s
  
- [ ] Sustained test: run the bot for 1 hour and monitor 429 frequency (should be zero or very rare)

### 7. Incident Response

- [ ] What to do if 429s start appearing:
  1. Check if rate limit architecture changed (some exchanges change limits)
  2. Check if other bots on the same IP are running (shared infra issue)
  3. Check if our request pattern changed (did we add a new high-frequency strategy?)
  4. Check the logs: are retries eventually succeeding?
  5. If retries are failing: STOP trading, investigate separately

- [ ] Who gets alerted? (team member, on-call, etc.)
- [ ] Escalation path: if ban persists for > 1 hour, contact exchange support

## Detection Test (Run Before Deployment)

### Test 1: Fixed vs Leaky Bucket

```
1. Send 100 requests as fast as possible
2. Record the pattern of 429s:
   - All at once = Fixed window
   - Spread over time = Leaky bucket
3. Wait for the bucket duration (e.g., 60 seconds for Binance)
4. Try 100 requests again
5. If pattern repeats, your hypothesis is correct
```

### Test 2: Respecting Retry-After

```
1. Intentionally trigger a 429
2. Check the Retry-After header value
3. Verify your code waits at least that long
4. Verify that after waiting, the request succeeds
```

## Common Mistakes (Do NOT Do These)

- [ ] ❌ Retrying immediately after 429 (causes thundering herd + ban)
- [ ] ❌ Using fixed wait (e.g., always wait 5 seconds) instead of backoff (predictable retry patterns)
- [ ] ❌ Ignoring Retry-After header (the exchange is telling you exactly how long to wait)
- [ ] ❌ Implementing retry in the client AND the queue (double retries = more 429s)
- [ ] ❌ No jitter in backoff (all bots retry at the same time = coordinated thundering herd)
- [ ] ❌ Retrying forever (need a max_retries limit and a circuit breaker)
- [ ] ❌ No logging (can't debug 429 patterns without data)

## Quick Reference: Config by Exchange

| Exchange | Window Type | Limit | Duration | Base Wait | Max Retries | Jitter? |
|----------|-------------|-------|----------|-----------|-------------|---------|
| Binance  | Fixed       | 1200  | 60s      | 1s        | 3           | Yes     |
| Kraken   | Fixed       | 15    | 15s      | 2s        | 5           | Yes     |
| Coinbase | Leaky       | 5     | 1s       | 0.5s      | 3           | Yes     |
| Bybit    | Fixed       | 50    | 300s     | 1s        | 5           | Yes     |
