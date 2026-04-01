# Policy as Code

## Context & Problem

Some authorization decisions depend on attributes and conditions, not just roles or relationships:

- "Trades can only be executed during market hours"
- "Positions exceeding $10M require additional approval"
- "Users in the EU cannot access US-only data feeds"

These are **policy decisions** — they combine identity, resource attributes, and environmental context. Hardcoding them in application logic makes them invisible, untestable, and scattered.

Policy-as-code externalizes these rules into a dedicated policy engine (OPA, Cedar) where they can be versioned, tested, and audited independently of application code.

## Design Decisions

### When to Use a Policy Engine vs. Application Logic

| Decision Type | Tool |
|---|---|
| "Does user have role X?" | RBAC (in-app) |
| "Can user access resource Y?" | OpenFGA |
| "Is this action allowed given conditions A, B, C?" | Policy engine (OPA/Cedar) |

Use a policy engine when the authorization decision depends on **dynamic attributes** that are not captured by static roles or relationships.

### OPA (Open Policy Agent)

OPA evaluates policies written in Rego. The application sends a query with context, OPA returns allow/deny:

```rego
# policy/trading.rego

package trading

import rego.v1

default allow := false

# Allow trade execution during market hours by authorized roles
allow if {
    input.action == "execute_trade"
    input.user.role in {"portfolio_manager", "admin"}
    market_is_open
    within_position_limit
}

market_is_open if {
    now := time.now_ns()
    hour := time.clock(now)[0]
    day := time.weekday(now)
    day in {"Monday", "Tuesday", "Wednesday", "Thursday", "Friday"}
    hour >= 9
    hour < 16
}

within_position_limit if {
    input.trade.notional_value < 10000000  # $10M
}

within_position_limit if {
    input.trade.notional_value >= 10000000
    input.user.approval_level == "senior"
}
```

### Application Integration

```python
import httpx


class PolicyEngine:
    def __init__(self, opa_url: str = "http://localhost:8181") -> None:
        self._client = httpx.AsyncClient(base_url=opa_url)

    async def check(self, policy_path: str, input_data: dict) -> bool:
        response = await self._client.post(
            f"/v1/data/{policy_path}",
            json={"input": input_data},
        )
        result = response.json()
        return result.get("result", {}).get("allow", False)


# Usage
policy = PolicyEngine()

allowed = await policy.check("trading/allow", {
    "action": "execute_trade",
    "user": {"role": "portfolio_manager", "approval_level": "senior"},
    "trade": {"notional_value": 15_000_000, "instrument": "AAPL"},
})
```

### Policy Testing

Policies are code — they need tests:

```rego
# policy/trading_test.rego

package trading_test

import rego.v1
import data.trading

test_allow_trade_during_market_hours if {
    trading.allow with input as {
        "action": "execute_trade",
        "user": {"role": "portfolio_manager", "approval_level": "standard"},
        "trade": {"notional_value": 5000000},
    }
}

test_deny_trade_outside_hours if {
    not trading.allow with input as {
        "action": "execute_trade",
        "user": {"role": "portfolio_manager"},
        "trade": {"notional_value": 5000000},
    }
    # Note: this test depends on when it runs — use mock time in practice
}

test_deny_large_trade_without_senior_approval if {
    not trading.within_position_limit with input as {
        "trade": {"notional_value": 15000000},
        "user": {"approval_level": "standard"},
    }
}
```

Run with: `opa test policy/ -v`

## Local Development

```yaml
services:
  opa:
    image: openpolicyagent/opa:latest
    ports: ["8181:8181"]
    command: run --server --watch /policies
    volumes:
      - ./policy:/policies
```

OPA watches the policy directory and hot-reloads on changes.

## Failure Modes

| Failure | Cause | Mitigation |
|---|---|---|
| OPA unavailable | Service down | Fail-closed (deny), circuit breaker |
| Policy bug allows unauthorized action | Rego logic error | Policy unit tests, policy review process |
| Stale context | Application sends outdated attributes | Refresh context before policy check |
| Performance on hot path | OPA round-trip on every request | Cache decisions with short TTL, bundle policies into application |

## Related Documents

- [Authorization RBAC](../api/authorization-rbac.md) — role-based checks (simpler cases)
- [OpenFGA Modeling](openfga-modeling.md) — relationship-based checks
- [Attribute-Based Access](attribute-based-access.md) — ABAC patterns
