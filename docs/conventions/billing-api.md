# Billing API Reference

GitHub Actions billing usage endpoint documentation.

## Endpoint History

The billing usage endpoint has moved multiple times. Only one path works:

| Endpoint | Result |
|----------|--------|
| `/user/settings/billing/usage` | 404 (needs username in path) |
| `/user/settings/billing/actions` | 404 (wrong path) |
| `/users/{user}/settings/billing/actions` | **410** (permanently removed) |
| `/users/{user}/settings/billing/usage` | **Works** |

## Authentication

**Required OAuth scope**: `user` — refresh with `gh auth refresh -h github.com -s user`

## Response Format

```json
{
  "usageItems": [
    {
      "date": "2026-04-01T00:00:00Z",
      "product": "actions",
      "sku": "Actions Linux",
      "quantity": 45.0,
      "unitType": "Minutes",
      "repositoryName": "my-repo"
    }
  ]
}
```

Key filters: `unitType == "Minutes"` + `product == "actions"` for Actions minutes. Dates are first-of-month (calendar month aggregation).

## Usage

**Don't query this endpoint ad-hoc** — use `budget-monitor.sh --quick` for a one-line summary or `--check` for a full report with per-repo breakdown. The script handles fallback estimation when the billing API is unavailable.
