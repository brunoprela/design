# Tach Enforcement

## Context & Problem

Module boundaries defined by convention erode within weeks. A developer imports a repository class from another module because it is faster than going through the interface. A shared utility grows to depend on half the codebase. By the time anyone notices, the clean architecture exists only in the diagram.

Tach is a Python tool that enforces module boundaries through static analysis. It reads a configuration declaring what each module is allowed to depend on and reports violations. Running in CI, it makes boundary violations impossible to merge.

## Design Decisions

### What Tach Enforces

Tach analyzes Python import statements and checks them against declared dependencies:

- Module A is allowed to import from Module B → OK
- Module A is not allowed to import from Module C → **violation**
- Module A imports Module B's internal (non-interface) module → **violation** (if configured)

### Configuration

```toml
# tach.toml

[tool.tach]
root = "app"
exact = true          # only declared dependencies are allowed
exclude = ["tests"]   # exclude test directories

[[tool.tach.modules]]
path = "shared"
depends_on = []       # shared depends on nothing

[[tool.tach.modules]]
path = "modules.market_data"
depends_on = ["shared"]

[[tool.tach.modules]]
path = "modules.positions"
depends_on = ["shared", "modules.market_data"]

[[tool.tach.modules]]
path = "modules.risk"
depends_on = ["shared", "modules.positions", "modules.market_data"]

[[tool.tach.modules]]
path = "modules.compliance"
depends_on = ["shared", "modules.positions"]

[[tool.tach.modules]]
path = "modules.order_management"
depends_on = ["shared", "modules.positions", "modules.compliance"]
```

### Running Tach

```bash
# Check for violations
tach check

# Output on violation:
# ❌ modules.risk imports modules.compliance.service
#    modules.risk is not allowed to depend on modules.compliance

# Visualize dependency graph
tach show
```

### CI Integration

```yaml
# .github/workflows/lint.yml
jobs:
  boundaries:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: pip install tach
      - run: tach check
```

### Interface-Only Imports

For stricter enforcement, configure modules to only allow importing from another module's `interface.py`:

```toml
[[tool.tach.modules]]
path = "modules.positions"
depends_on = [
    "shared",
    "modules.market_data",  # can only import market_data's public interface
]
```

Then enforce in code review that the `__init__.py` of each module only re-exports the interface:

```python
# modules/market_data/__init__.py
from app.modules.market_data.interface import MarketDataReader, PriceSnapshot

__all__ = ["MarketDataReader", "PriceSnapshot"]
```

### Handling Violations

When Tach reports a violation, the options are:

1. **Fix the dependency** — use the module's public interface instead of its internals
2. **Add the dependency** — if the dependency is legitimate, declare it in `tach.toml`
3. **Refactor** — if the dependency reveals a boundary problem, move code to the right module
4. **Use events** — if two modules need to communicate but should not depend on each other, use events

## Failure Modes

| Failure | Cause | Mitigation |
|---|---|---|
| Config drift | New module added without tach config | CI fails on unknown modules (with `exact = true`) |
| Over-permissive | All modules allowed to depend on everything | Start strict, add dependencies only when justified |
| Bypassing via shared | Moving code to shared to avoid boundary rules | Keep shared small and stable, review shared changes carefully |
| Dynamic imports | `importlib.import_module()` bypasses static analysis | Code review, discourage dynamic imports for module access |

## Related Documents

- [Modular Monolith](../../principles/modular-monolith.md) — the architecture Tach enforces
- [Module Interfaces](module-interfaces.md) — the Protocols that form module boundaries
- [Shared Kernel](shared-kernel.md) — what belongs in the shared module
