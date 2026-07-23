# User Multi-Window Budget

Set multiple concurrent budget windows on a single user. Each window has its own limit and resets independently

## Supported durations

| Format | Meaning |
|--------|---------|
| `30s`  | 30 seconds |
| `5m`   | 5 minutes |
| `1h`   | 1 hour |
| `1d`   | 1 day |
| `7d`   | 7 days (weekly) |
| `1mo`  | 1 month |

Any positive integer can be combined with the unit, e.g. `2h`, `14d`, `3mo`

## Create a user with budget_limits

```bash
curl -X POST http://localhost:4000/user/new \
  -H "Authorization: Bearer sk-master-key" \
  -H "Content-Type: application/json" \
  -d '{
    "budget_limits": [
      {"budget_duration": "1h",  "max_budget": 1.00},
      {"budget_duration": "1d",  "max_budget": 10.00},
      {"budget_duration": "1mo", "max_budget": 100.00}
    ]
  }'
```

This creates a user with three independent budgets: $1/hour, $10/day, $100/month. A request is rejected as soon as any single window is exceeded

## Update an existing user

```bash
curl -X POST http://localhost:4000/user/update \
  -H "Authorization: Bearer sk-master-key" \
  -H "Content-Type: application/json" \
  -d '{
    "user_id": "user-abc-123",
    "budget_limits": [
      {"budget_duration": "1h",  "max_budget": 2.00},
      {"budget_duration": "1d",  "max_budget": 20.00}
    ]
  }'
```

## How it works

1. On each LLM call, spend is tracked per window using in-memory (and optionally Redis) counters keyed as `spend:user:<user_id>:window:<duration>`
2. Before a request is processed, the auth layer checks all windows; if any window's accumulated spend exceeds its `max_budget`, the request is rejected with a `budget_exceeded` error
3. A background reset job runs periodically and resets any window whose `reset_at` has passed, zeroing the spend counter and advancing `reset_at` to the next period

## Combining with the legacy single-window budget

`budget_limits` is independent of the existing `max_budget` + `budget_duration` fields. Both are enforced if both are set. For new setups, prefer `budget_limits` since it supports multiple windows in a single config
