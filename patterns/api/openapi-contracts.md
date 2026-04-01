# OpenAPI Contracts

## Context & Problem

When teams build APIs without a shared contract, the frontend and backend diverge. Response field names change without notice, nullable fields appear unexpectedly, and integration tests are the only safety net — slow and late.

OpenAPI provides a machine-readable API specification. FastAPI generates it automatically from Pydantic models and route definitions. This generated spec serves as the single source of truth for clients, documentation, and contract testing.

## Design Decisions

### Code-First with Committed Spec

FastAPI generates the OpenAPI spec from code. The workflow:

1. Define Pydantic models and route signatures
2. FastAPI auto-generates the OpenAPI 3.1 JSON at `/openapi.json`
3. Export and commit the spec to version control
4. CI compares the committed spec against the generated spec — fails on drift

This gives the simplicity of code-first with the reliability of a committed contract.

### Schema Design Rules

```python
# Explicit, never generic
class TradeResponse(BaseModel):
    trade_id: str
    instrument_id: str
    quantity: Decimal
    price: Decimal
    executed_at: datetime

# Not this — dict, Any, or overly generic types leak implementation details
class BadResponse(BaseModel):
    data: dict  # what's in here? nobody knows
```

Rules:
- Every endpoint has explicit request and response models
- No `dict`, `Any`, or `object` in schemas
- Use `Decimal` for financial amounts, never `float`
- Use `datetime` for timestamps, never `str`
- Optional fields are explicitly `field: str | None = None`
- Enums for constrained values (`side: Literal["buy", "sell"]`)

### Error Response Format

Consistent error responses across all endpoints:

```python
class ErrorResponse(BaseModel):
    error: str           # machine-readable error code: "validation_error", "not_found"
    message: str         # human-readable description
    details: list[dict] | None = None  # field-level validation errors


# Register as default error responses
@app.exception_handler(RequestValidationError)
async def validation_handler(request: Request, exc: RequestValidationError):
    return JSONResponse(
        status_code=422,
        content=ErrorResponse(
            error="validation_error",
            message="Request validation failed",
            details=exc.errors(),
        ).model_dump(),
    )
```

## Generating and Committing the Spec

```python
# scripts/export_openapi.py

import json
from app.main import create_app

app = create_app()
spec = app.openapi()

with open("openapi.json", "w") as f:
    json.dump(spec, f, indent=2, default=str)
```

CI check:

```bash
# ci/check_openapi.sh
python scripts/export_openapi.py
diff openapi.json openapi.json.committed || (echo "OpenAPI spec drift detected" && exit 1)
```

## Failure Modes

| Failure | Cause | Mitigation |
|---|---|---|
| Spec drift | Code changes without updating committed spec | CI diff check |
| Breaking change unnoticed | Field removed or renamed | Backward compatibility linting (e.g., openapi-diff) |
| Vague schemas | `dict` or `Any` in response models | Linting rules, code review |
| Missing error docs | Error responses not documented | Explicit `responses={}` parameter on route decorators |

## Related Documents

- [FastAPI Modular Layout](fastapi-modular-layout.md) — where routes and schemas live
- [Contract-First Design](../../principles/contract-first-design.md) — the principle behind API contracts
- [API Versioning](api-versioning.md) — managing breaking changes
