# TRUST-AI: An AI-Driven Security Platform for Risk Management

> Graduation project. A platform that turns **detection into a decision**: an ML model detects a threat, a risk engine scores it, a role-based permission system controls who sees it, and every event is recorded in an audit trail.

## Why TRUST-AI

Small and medium businesses usually run detection tools, risk spreadsheets, and access control as three separate things. That leaves four recurring gaps:

1. Phishing links are checked by eye, if at all.
2. Risk decisions rely on personal judgment instead of a consistent, numeric method.
3. Users hold more access than their role needs (a Least Privilege violation).
4. Security events are scattered, so *"who did what, and when?"* is slow to answer.

TRUST-AI connects the pieces into one loop:

```
Detect  ->  Score  ->  Enforce  ->  Record  ->  Explain
 (ML)     (Risk)    (Roles)      (Audit)     (Dashboard)
```

## Architecture

```
Flutter app (Frontend)  --\
                           >--  ASP.NET Core API (Backend)
Security (tests, tools) --/       |  exposes /api/v1/* endpoints (docs/api-contract.yaml)
                                  |  authentication, role enforcement,
                                  |  risk scoring, masking, audit logging
                                  |
                                  |-- consumes: POST /api/v1/predictions <- AI Service (FastAPI, port 8001)
                                  |-- Audit Log + results                 -> SQL Server
```

The backend is the single entry point. It exposes the `/api/v1` endpoints that the Frontend and the Security team call, and the only external service it consumes is the AI Service (FastAPI). For each scan it takes the prediction from the AI Service, applies the risk rules and role-based masking defined by the Security team, writes the audit entry, and returns the final result based on the user's role.

| Component | Technology | Responsibility |
|---|---|---|
| Mobile / Web app | Flutter (Dart) | Role-adaptive UI, reports, alerts |
| Main API | ASP.NET Core, Entity Framework Core | Endpoints for Frontend and Security, auth, roles, consuming the AI Service, risk scoring, masking, database, audit logging |
| AI Service | Python, FastAPI, scikit-learn | Phishing URL detection with a confidence score (consumed by the backend only) |
| Security (rules, docs, tests) | Python, Markdown | Risk rules and masking spec, security docs, API security tests that call the backend endpoints |
| Database | SQL Server | Users, roles, audit logs, detection results |

## Repository layout

```
ai_service/          Phishing URL model + FastAPI service        (port 8001)
security_service/    Risk rules + masking reference, security docs/tests/tools
requirements.txt     Python dependencies for both services
run_tests.py         Runs all test suites
```

> The repository currently also contains `gateway/`, `run_all.py`, and `demo.py`. These are a temporary stand-in for the backend and will be removed once the ASP.NET Core API replaces them.

## Services
> The AI Service endpoint is internal (Backend to AI Service only).
> The Frontend and the Security team must use only the Backend endpoints in `docs/api-contract.yaml`.

### AI Service: phishing URL detection

`POST /api/v1/predictions`

```json
// request
{ "inputType": "url", "payload": "http://example.com/login" }

// response
{ "label": "phishing", "confidence": 0.94, "explanation": "..." }
```

- `label` is `phishing` or `legitimate`.
- `confidence` is the probability of the **predicted** label.
- Any `inputType` other than `url` returns `400`.
- The model is trained on a labeled public dataset of 11,430 URLs (`ai_service/dataset_phishing.csv`) using lexical URL features. Current cross-validated accuracy is about 89.6% (F1 about 0.896).
- If the saved model does not match the installed scikit-learn version, the service retrains automatically from the dataset on startup.

Useful scripts (run from inside `ai_service/`):

| Script | Purpose |
|---|---|
| `train.py` | Retrain the model and sanity-check it on URLs outside the dataset |
| `compare_models.py` | Compare Random Forest, Gradient Boosting, Decision Tree, and Logistic Regression with 5-fold cross-validation |

### Security: risk engine and privacy masking

The risk scoring and masking rules below are defined by the Security team and applied inside the backend. `security_service/` holds the reference implementation (`risk_engine.py`, `masking.py`) that the backend follows, and the Frontend and Security team reach the results only through the backend endpoints (for example `POST /api/v1/scans`), never by calling a separate security service.

Reference request and response shape of the risk engine (`POST /api/v1/assessments` in the reference implementation):

```json
// request
{ "asset": "login-page", "threat": "phishing", "likelihood": 0.94, "impact": 8.0, "source": "url-scan" }

// response
{ "riskScore": 7.52, "riskLevel": "High", "treatmentStrategy": "Treat", "explanation": "..." }
```

**Formula:** `Risk Score = effective likelihood x impact` (0 to 10).

- `likelihood` is the AI confidence (0 to 1). For a benign label (`legitimate`, `real`, `safe`) it is inverted (`1 - confidence`), because confidence describes the predicted label.
- `impact` (0 to 10) is sent by the backend per asset or input type.

| Risk Score | Level | Strategy |
|---|---|---|
| 9.0 and above | Critical | Terminate |
| 6.0 to 8.9 | High | Treat |
| 3.0 to 5.9 | Medium | Transfer |
| below 3.0 | Low | Tolerate |

**Uncertain Outcome:** when AI confidence is below `0.65`, the result is marked for manual review. The threshold is `UNCERTAIN_BELOW` in `security_service/risk_engine.py`.

Privacy masking (`POST /api/v1/mask` in the reference implementation) is applied by viewer role to emails, phone numbers, and card numbers:

| Role | What they see |
|---|---|
| Security Admin | Clear data |
| Analyst | Partially masked |
| Guest | Fully masked |

### Security documentation and tooling

| Path | Content |
|---|---|
| `security_service/docs/risk_matrix.md` | Risk levels and treatment mapping |
| `security_service/docs/audit_log_policy.md` | What is logged, what is never logged, retention, alert rules |
| `security_service/docs/stride_threat_model.md` | STRIDE threat model |
| `security_service/docs/owasp_checklist.md` | OWASP checklist |
| `security_service/security_tests/` | API security test scripts (call the backend `/api/v1` endpoints) |
| `security_service/tools/audit_log_analyzer.py` | Flags suspicious patterns in an audit log CSV |

## Roles and permissions

| Role | Permissions |
|---|---|
| **Guest** | View detection reports only |
| **Analyst** | Guest permissions + view the Risk Score per case + read the Audit Log |
| **Security Admin** | Analyst permissions + manage users and roles + handle incidents |

Each role gets only the access it needs (Least Privilege). Permissions are enforced on the server, not only hidden in the UI.

## Getting started

Requirements: Python 3.10 or newer.

```bash
python -m venv venv
venv\Scripts\activate          # Windows
# source venv/bin/activate     # macOS / Linux
pip install -r requirements.txt
python run_tests.py            # should end with ALL PASSED
```

Run the AI Service (the backend consumes it):

```bash
cd ai_service
uvicorn app:app --port 8001
```

Interactive API docs (Swagger): `http://localhost:8001/docs`. Health check: `/health`.

The Frontend and the Security tests connect to the backend (`http://localhost:5000`), not to the AI Service.

### Backend configuration

```
ExternalServices__AiServiceUrl=http://localhost:8001/
```

## Team

| Role | Members |
|---|---|
| Security | 2 |
| Frontend | 2 |
| Backend | 1 |
| AI | 1 |
| Data Analysis | 1 |

## Known limitations

- Only URL detection is supported; the AI Service rejects any other input type.
- The 89.6% accuracy comes from a public dataset with lexical features and should be re-checked on URLs from outside the dataset.
- The security test scripts were exercised against a mock backend, not yet against the real ASP.NET Core API.
- The risk thresholds, the strategy mapping, and the 0.65 uncertainty cut-off are team decisions that should be reviewed and confirmed.
- SHAP explanations are planned; the current `explanation` field is a fixed text.
## API Contract

العقد الرسمي بين Frontend و Backend: [`docs/api-contract.yaml`](docs/api-contract.yaml)
أي اسم endpoint أو حقل أو enum لازم يطابق العقد حرفياً.
