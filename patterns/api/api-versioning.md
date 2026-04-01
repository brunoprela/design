# API Versioning

## Context & Problem

APIs evolve. Fields are added, renamed, or removed. Response structures change. Without versioning, any change risks breaking existing clients. With too many versions, maintenance becomes a burden.

## Design Decisions

### URL Path Versioning (Recommended)

```
/api/v1/positions
/api/v2/positions
```

Pros: explicit, visible in logs and routing, easy to implement with FastAPI router prefixes.
Cons: URL changes between versions (but clients must update the version anyway).

```python
# Version-specific routers
app.include_router(positions_v1.router, prefix="/api/v1/positions")
app.include_router(positions_v2.router, prefix="/api/v2/positions")
```

### Versioning Rules

1. **Max 2 active versions** — supporting more than 2 creates maintenance burden
2. **Deprecation timeline** — announce deprecation with 3-month minimum before removal
3. **Sunset header** — include `Sunset: <date>` header on deprecated versions
4. **Changelog** — maintain a changelog per version transition

### What Triggers a New Version

**New version required:**
- Removing a field from a response
- Changing a field's type
- Changing the meaning of a field
- Restructuring the response shape

**No new version needed (backward compatible):**
- Adding a new optional field to a response
- Adding a new endpoint
- Adding a new optional query parameter
- Adding a new enum value

### Implementation Pattern

```python
# modules/positions/v1/schemas.py
class PositionResponseV1(BaseModel):
    portfolio_id: str
    instrument_id: str
    quantity: Decimal
    market_value: Decimal


# modules/positions/v2/schemas.py
class PositionResponseV2(BaseModel):
    portfolio_id: str
    instrument: InstrumentDetail  # expanded from plain ID
    quantity: Decimal
    market_value: MoneyAmount     # structured with currency
    unrealized_pnl: MoneyAmount   # new field


# The service layer is shared — only the schema layer differs
# V1 routes map service output to V1 schemas
# V2 routes map service output to V2 schemas
```

## Failure Modes

| Failure | Cause | Mitigation |
|---|---|---|
| Version proliferation | Every small change gets a new version | Only version for breaking changes |
| Abandoned old versions | V1 runs but is unmaintained | Deprecation policy with hard sunset dates |
| Inconsistent behavior | V1 and V2 have different bugs | Shared service layer, version only at the schema/route level |

## Related Documents

- [OpenAPI Contracts](openapi-contracts.md) — versioned API specs
- [Contract-First Design](../../principles/contract-first-design.md) — compatibility rules
