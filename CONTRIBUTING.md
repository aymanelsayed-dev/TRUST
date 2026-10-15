# Contributing to TRUST-AI

## Repository layout (monorepo)

```
ai_service/          Phishing model + FastAPI service
security_service/    Risk Engine, masking, security docs and tests
backend/             ASP.NET Core API
frontend/            Flutter app
design/              Design files and exports
docs/                Dataset notes, reports, diagrams
notebooks/           EDA and evaluation notebooks
```

## Workflow

1. Pick an issue from the board and assign it to yourself. One issue, one owner.
2. Create a branch from `main`: `<area>/<short-description>`, for example `backend/jwt-roles` or `frontend/auth-screens`.
3. Commit small and often. Message format: `area: what changed` (example: `ai: add edge-case tests for long URLs`).
4. Open a Pull Request that references the issue (`Closes #12`).
5. At least **one teammate reviews** before merging. Nobody merges their own PR without a review.
6. Merge with **Squash and merge**, then delete the branch.

Never commit directly to `main`.

## Review rules

- Backend PRs: reviewed by Backend and, for auth or permission changes, also by a Security member.
- Frontend PRs: reviewed by the other Frontend member.
- Anything that changes an API contract (request or response fields): tag Backend, Frontend, and the service owner.
- Review within 24 hours so nobody is blocked.

## Security rules (this is a security project)

- No passwords, JWT signing keys, API keys, or connection strings in the repository, ever. Use environment variables, `.env` files (ignored by git), or ASP.NET user-secrets.
- If a secret is committed by mistake, tell the team immediately and rotate it. Deleting the commit is not enough.
- Do not log passwords, tokens, or raw uploaded content (see `security_service/docs/audit_log_policy.md`).

## Ownership

| Area | Owner |
|---|---|
| `ai_service/`, model evaluation | AI Engineer + Data Analyst |
| `security_service/risk_engine.py`, `masking.py` | Security 1 |
| `security_service/docs/`, `security_tests/`, `tools/` | Security 2 |
| `backend/` | Backend |
| `frontend/` | Frontend 1 and 2 |
| `design/` | UI/UX |

## API contracts

The JSON contracts between services are fixed in `README.md`. Changing a field name or type needs an issue and agreement from the owners on both sides before the change is merged.
