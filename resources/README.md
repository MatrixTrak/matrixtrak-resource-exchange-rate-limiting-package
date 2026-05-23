# Exchange Rate Limit Package

This package contains everything you need to handle 429 (rate limit) errors correctly in your crypto trading bot.

## What's Inside

- **`rate-limit-config-template.yaml`** — Configuration template for major exchanges (Binance, Kraken, Coinbase, Bybit)
- **`rate-limit-decision-checklist.md`** — Pre-deployment checklist and testing procedures
- **`rate-limit-429-logging-schema.json`** — JSON schema for structured logging of 429 errors

## Quick Start

### 1. Copy the config template

```bash
cp rate-limit-config-template.yaml src/config/rate-limits.yaml
```

Edit the `rate-limits.yaml` file and customize for your exchanges:
- Adjust `limit` and `window_seconds` based on exchange documentation
- Set `base_wait_seconds` and `max_retries` based on your risk tolerance
- Ensure `jitter_enabled` is `true` to avoid thundering herd

### 2. Implement exponential backoff with jitter

The formula:
```
wait_time = base_wait_seconds * (2 ^ (attempt - 1)) + jitter(0, 1s)
capped_wait = min(wait_time, max_wait_seconds)
```

Example in pseudocode:
```python
import random
import time

def backoff_wait(attempt, base_wait, max_wait):
    exponential = base_wait * (2 ** (attempt - 1))
    jitter = random.uniform(0, 1.0)
    total_wait = min(exponential + jitter, max_wait)
    time.sleep(total_wait)
    return total_wait
```

### 3. Add logging using the schema

Every time you encounter a 429, log these fields from the schema:
- `timestamp` (ISO-8601)
- `exchange` (which exchange)
- `endpoint` (which API endpoint)
- `http_status` (429)
- `retry_after_header` (what the exchange told you)
- `retry_count` (which attempt)
- `wait_duration_ms` (how long you waited)

Example log entry (JSON):
```json
{
  "timestamp": "2026-01-27T14:30:20.123Z",
  "event_type": "rate_limit_429",
  "exchange": "binance",
  "endpoint": "/api/v3/order",
  "http_status": 429,
  "retry_after_header": "2",
  "request_weight": 1,
  "cumulative_weight_used": 1195,
  "retry_count": 1,
  "backoff_strategy": "exponential",
  "wait_duration_ms": 1523,
  "jitter_applied_ms": 523,
  "severity": "info"
}
```

### 4. Run the checklist before deploying

Before pushing your bot to production:
- [ ] Identify the rate-limit architecture (fixed window vs leaky bucket)
- [ ] Run burst test (send 100 requests, observe pattern)
- [ ] Verify Retry-After header is respected
- [ ] Check that jitter is working (add randomness)
- [ ] Verify circuit breaker stops requests after N failures
- [ ] Monitor 429 frequency in staging (should be 0 or very rare)

See `rate-limit-decision-checklist.md` for detailed steps.

## Common Mistakes (Do NOT Do)

- ❌ Retrying immediately after 429 (causes ban)
- ❌ Fixed wait instead of exponential backoff (predictable retry pattern)
- ❌ Ignoring Retry-After header (the exchange is telling you exactly how long)
- ❌ No jitter (all bots retry at same time = coordinated storm)
- ❌ Retrying forever (need max_retries + circuit breaker)
- ❌ No logging (can't debug issues)

## Troubleshooting

### I'm getting sustained 429s (more than 1 minute)

1. Check if the exchange rate-limit window changed
2. Check if you're on a shared IP with other bots
3. Check if your request pattern changed
4. **STOP trading and investigate** — don't retry blindly (risk of ban)

### I see a burst of 429s then silence

This is normal for fixed-window exchanges (e.g., Binance). You hit the limit, backed off, and succeeded on retry. The logging schema will help you understand the pattern.

### My retries are failing even after waiting

1. Verify `Retry-After` header value (are you waiting long enough?)
2. Check if you've actually hit a ban (see `ban_detected` severity in logs)
3. Reduce your request rate permanently (you're over the limit)
4. Contact exchange support if unsure

## Integration Points

### In Your Bot Code

```python
# At startup
config = load_yaml("config/rate-limits.yaml")

# When making API call
try:
    response = make_api_call(endpoint)
except RateLimitError as e:
    for attempt in range(1, config[exchange]["trading"]["max_retries"] + 1):
        wait_time = backoff_wait(attempt, 
                                  config[exchange]["trading"]["base_wait_seconds"],
                                  config[exchange]["trading"]["max_wait_seconds"])
        log_429_event(endpoint, attempt, wait_time, e.retry_after_header)
        
        response = make_api_call(endpoint)
        if response.ok:
            log("event_type", "rate_limit_retry_success")
            break
    else:
        log("event_type", "rate_limit_retry_failed")
        raise CircuitBreakerOpen(f"Max retries exceeded for {endpoint}")
```

### In Your Observability

- Set up a dashboard to track 429 frequency by exchange
- Alert if 429 rate exceeds threshold (e.g., > 1% of requests)
- Alert if consecutive 429s exceed 2 minutes (potential ban)

## Further Reading

- [Binance Rate Limits](https://binance-docs.github.io/apidocs/#general-info)
- [Kraken Rate Limits](https://docs.kraken.com/rest/#rate-limits)
- [Coinbase Rate Limits](https://docs.cloud.coinbase.com/advanced-trade/docs/rest-api-rate-limits)
- [Backoff and Jitter Paper](https://aws.amazon.com/blogs/architecture/exponential-backoff-and-jitter/)

## Support

If you hit issues:
1. Check the logging schema — make sure you're logging all required fields
2. Review the decision checklist — are you missing a test?
3. Run the burst test to identify fixed window vs leaky bucket
4. Share your logs with your team for analysis

Good luck! 🚀
