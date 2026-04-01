# Attribute-Based Access Control (ABAC)

## Context & Problem

RBAC uses static roles. ReBAC (OpenFGA) uses relationships. ABAC uses **attributes** — properties of the user, resource, action, and environment — to make access decisions at runtime.

ABAC is the most flexible model. It can express:

- "Users in department=engineering can access resources with classification=internal"
- "Users can edit documents they created within the last 24 hours"
- "API calls from IP ranges outside the corporate network require MFA"

The tradeoff: ABAC is powerful but harder to reason about. It is easy to create policies that interact in unexpected ways.

## Design Decisions

### ABAC Components

```
Subject attributes:  user.department, user.clearance_level, user.location
Resource attributes: document.classification, portfolio.fund_type, data.sensitivity
Action attributes:   action.type (read, write, delete)
Environment:         time, ip_address, device_type
```

### When to Use ABAC

Use ABAC when access decisions require runtime evaluation of attributes that cannot be pre-computed into roles or relationships:

- Geographic restrictions (user.country, data.jurisdiction)
- Time-based access (market hours, after-hours restrictions)
- Classification-based access (confidential, internal, public)
- Dynamic conditions (user.risk_score > threshold)

### Implementation with Policy Engine

ABAC is best implemented via a policy engine (OPA, Cedar) rather than in application code. See [Policy as Code](policy-as-code.md) for the integration pattern.

```rego
# Attribute-based policy example
package document_access

default allow := false

allow if {
    input.action == "read"
    input.resource.classification == "internal"
    input.subject.department == input.resource.department
}

allow if {
    input.action == "read"
    input.resource.classification == "public"
}

allow if {
    input.action == "write"
    input.subject.id == input.resource.owner_id
    time.now_ns() - input.resource.created_at_ns < 86400000000000  # 24 hours
}
```

### Combining ABAC with RBAC and ReBAC

In practice, most systems use a layered approach:

```
1. RBAC  — "Does user have a role that permits this action type?"   (fast, coarse)
2. ReBAC — "Does user have a relationship to this resource?"        (OpenFGA)
3. ABAC  — "Do the attributes satisfy the policy conditions?"       (OPA/Cedar)
```

Check in order from cheapest to most expensive. Fail fast on the first denial.

## Failure Modes

| Failure | Cause | Mitigation |
|---|---|---|
| Policy conflict | Two policies give contradictory results | Explicit deny-overrides-allow ordering, policy testing |
| Attribute staleness | User attributes cached too long | Short TTL on attribute cache, refresh on sensitive operations |
| Debugging difficulty | Hard to explain why access was denied | Policy engine trace/explain mode, audit logging |
| Performance | Complex attribute evaluation on every request | Layer checks (RBAC first), cache attribute lookups |

## Related Documents

- [Policy as Code](policy-as-code.md) — implementing ABAC with OPA
- [Authorization RBAC](../api/authorization-rbac.md) — coarse-grained layer
- [OpenFGA Modeling](openfga-modeling.md) — relationship-based layer
