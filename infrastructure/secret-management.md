# Secret Management

## Context & Problem

Applications need secrets — database passwords, API keys, JWT signing keys, encryption keys. Hardcoding them in source code or committing `.env` files is the most common cause of credential leaks. Even in "private" repos, credentials in git history persist after deletion.

Secret management ensures that secrets are stored securely, accessed programmatically, rotated without downtime, and never appear in source control.

## Design Decisions

### Environments Have Different Needs

| Environment | Secret Source | Complexity |
|---|---|---|
| **Local dev** | `.env` file (gitignored) + docker-compose | Lowest — developer convenience |
| **CI/CD** | GitHub Actions secrets / CI environment variables | Medium — automated, no human access |
| **Production** | Vault / AWS Secrets Manager / GCP Secret Manager | Highest — audit trail, rotation, access control |

### Local Development

Use a `.env` file that is **gitignored** and a `.env.example` that is committed:

```bash
# .env.example (committed — no real secrets)
POSTGRES_PASSWORD=dev
REDIS_URL=redis://localhost:6379
KAFKA_BOOTSTRAP_SERVERS=localhost:9092
JWT_SECRET=change-me-in-production
BLOOMBERG_API_KEY=your-key-here
OPENAI_API_KEY=your-key-here
```

```bash
# .env (gitignored — real local values)
POSTGRES_PASSWORD=dev
REDIS_URL=redis://localhost:6379
KAFKA_BOOTSTRAP_SERVERS=localhost:9092
JWT_SECRET=local-dev-secret-not-for-production
BLOOMBERG_API_KEY=sandbox-key-abc123
OPENAI_API_KEY=sk-dev-...
```

```gitignore
# .gitignore
.env
*.pem
*.key
credentials.json
```

### Application Configuration

Use `pydantic-settings` to load secrets from environment variables with validation:

```python
from pydantic_settings import BaseSettings
from pydantic import SecretStr


class Settings(BaseSettings):
    model_config = {"env_file": ".env", "env_file_encoding": "utf-8"}

    # Database
    database_url: str = "postgresql+asyncpg://app:dev@localhost:5432/app"

    # Kafka
    kafka_bootstrap_servers: str = "localhost:9092"
    schema_registry_url: str = "http://localhost:8081"

    # Auth
    jwt_secret: SecretStr  # SecretStr masks the value in logs and repr
    jwt_algorithm: str = "RS256"

    # External APIs
    bloomberg_api_key: SecretStr | None = None
    openai_api_key: SecretStr | None = None

    # Redis
    redis_url: str = "redis://localhost:6379"


settings = Settings()

# Access secret value explicitly
secret = settings.jwt_secret.get_secret_value()

# repr is safe — secrets are masked
print(settings)
# > Settings(database_url='...', jwt_secret=SecretStr('**********'), ...)
```

### CI/CD Secrets

```yaml
# .github/workflows/test.yml
jobs:
  test:
    runs-on: ubuntu-latest
    env:
      DATABASE_URL: postgresql://app:test@localhost:5432/test
      JWT_SECRET: ${{ secrets.JWT_SECRET_TEST }}
      BLOOMBERG_API_KEY: ${{ secrets.BLOOMBERG_SANDBOX_KEY }}
    steps:
      - uses: actions/checkout@v4
      - run: pytest
```

Rules:
- Never echo secrets in CI logs
- Use separate secrets for test/staging/production
- Rotate CI secrets periodically
- Minimize the number of people with access to production secrets

### Production: Secret Manager Integration

```python
# infrastructure/secrets.py

from typing import Protocol


class SecretProvider(Protocol):
    async def get_secret(self, name: str) -> str: ...


class AWSSecretProvider:
    """Fetches secrets from AWS Secrets Manager."""

    def __init__(self, region: str = "us-east-1") -> None:
        import boto3
        self._client = boto3.client("secretsmanager", region_name=region)

    async def get_secret(self, name: str) -> str:
        response = self._client.get_secret_value(SecretId=name)
        return response["SecretString"]


class EnvSecretProvider:
    """Fetches secrets from environment variables (local dev / CI)."""

    async def get_secret(self, name: str) -> str:
        import os
        value = os.environ.get(name)
        if not value:
            raise ValueError(f"Secret {name} not found in environment")
        return value
```

The application uses `SecretProvider` — local dev uses `EnvSecretProvider`, production uses `AWSSecretProvider`. Wired in the composition root.

### Secret Rotation

Secrets should be rotatable without application restart:

1. **Short-lived tokens** — JWT access tokens expire in minutes, refresh tokens in hours
2. **Database credentials** — rotate via Vault dynamic secrets (new credentials per deployment)
3. **API keys** — dual-key rotation: add new key, deploy, remove old key
4. **Encryption keys** — key versioning (encrypt with latest, decrypt with any version)

## Pre-commit Hook: Secret Detection

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/Yelp/detect-secrets
    rev: v1.4.0
    hooks:
      - id: detect-secrets
        args: ["--baseline", ".secrets.baseline"]
```

This scans staged files for patterns that look like secrets (high-entropy strings, known key patterns) and blocks the commit.

## Failure Modes

| Failure | Cause | Mitigation |
|---|---|---|
| Secret committed to git | `.env` not in `.gitignore` | Pre-commit hook, `.gitignore` template, git-secrets |
| Secret in logs | Logging the config object | Use `SecretStr`, scrub logs |
| Secret expired | API key or certificate expired | Monitoring on expiration, alerting 30 days before |
| Secret manager unavailable | Cloud service outage | Cache secrets locally with TTL, circuit breaker |
| Over-broad access | Too many services share one key | Per-service credentials, principle of least privilege |

## Related Documents

- [Authentication MFA](../patterns/api/authentication-mfa.md) — JWT signing keys
- [OAuth2 Flows](../patterns/api/oauth2-flows.md) — client secrets
- [Docker Compose Patterns](docker-compose-patterns.md) — `.env` integration
- [Dependency Inversion](../principles/dependency-inversion.md) — SecretProvider as a Protocol
