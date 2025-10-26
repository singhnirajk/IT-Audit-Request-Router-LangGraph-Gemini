
# IT Audit Request Router — LangGraph + Gemini (Colab)

## Purpose
Rules-first, LLM-second router for IT audit intake. Classifies requests to **IT Audit Desk** or **Global Audit Intake**, attaches framework-specific evidence checklists, and emits ticket/email stubs + JSON audit log.

## What’s Included
- **Notebook**: `IT_Audit_Request_Router_LangGraph_Gemini.ipynb` (Colab-ready)
- **Graph**: sanitize → rules → (conditional) Gemini verify → decide → artifacts
- **Frameworks**: SOX, SOC 1/2, HIPAA, COBIT, PCI DSS, ISO 27001, NIST CSF
- **Outputs**: `decision`, `evidence`, `ticket` (stub), `email` (stub), `log`

## Quick Start (Colab)
1. **Upload** the notebook.
2. **Set API key** *before* the config cell:
   ```python
   %env GOOGLE_API_KEY=YOUR_GEMINI_KEY
   ```
3. **Run cells** top to bottom:
   - Install
   - Config (auto-discovers a valid model for your key)
   - Domain / Models / Rules
   - Gemini verifier
   - Graph
   - Demo
4. **Result**: `it_audit_router_gemini_run.json` written in the runtime.

## Configuration
- **API key**: environment variable `GOOGLE_API_KEY` must be present before the config cell runs.
- **Model selection**: the config cell enumerates your project’s **supported** models via `genai.list_models()` and picks the best available in this order:
  1. `models/gemini-1.5-flash-latest`
  2. `models/gemini-1.5-pro-latest`
  3. `models/gemini-1.5-flash`
  4. `models/gemini-1.5-pro`
  5. `models/gemini-pro`
  6. `models/gemini-1.0-pro`
- The chosen `MODEL` is used by the LLM node. **Do not hard-code** model IDs; rely on auto-discovery to avoid 404s.

## Routing Policy
- **Rules-first**: keyword match on title+description maps to frameworks and assigns confidence.
- **LLM-second**: if no frameworks or low confidence (`< 0.7`), call Gemini to verify and extract frameworks.
- **Decision**: prefer rules when confident; otherwise use LLM verdict; else route to Global Intake.

## Evidence Library (examples)
- **SOX**: ICFR controls/owners, narratives/flowcharts, ToD/ToE, populations & samples
- **SOC 2**: TSC mapping, security/availability/confidentiality evidence, IR records, vendor/subservice list
- **HIPAA**: Privacy/Security/Breach policies, ePHI inventory & data flow, risk analysis/plan, access logs/audit trails
- **PCI DSS**: CDE diagram, segmentation evidence, ASV scans/pentest, key management/encryption docs
- **ISO 27001**: ISMS scope/SoA, Annex A evidence, internal audit results, management review & CAPA
- **NIST CSF**: Current vs Target profile, tier assessment, risk register, IR/BC/DR runbooks

## Artifacts
- **Ticket stub**: payload for ServiceNow (system string + JSON)
- **Email stub**: to IT Audit Desk or Global Audit Intake
- **Audit log**: `request_id`, `route`, `confidence`, `reason`, UTC timestamp

## Placeholders to Edit
- `IT_AUDIT_EMAIL`, `GLOBAL_INTAKE_EMAIL` in the graph cell
- Extend `KEYWORDS` and `EVIDENCE_LIBRARY` with org-specific terms/evidence
- Replace stubs with real integrations:
  - ServiceNow Table API (create incident/task/record)
  - SMTP or mail gateway

## Error Handling Guide
- **403 Forbidden (API disabled)**: enable **Generative Language API** for your GCP project. Link billing if required.
- **404 Not Found (model id)**: your key can’t access the hard-coded model. Use the **auto-discovery** config to select a supported model from `list_models()`.
- **429 Rate limit / quota**: add billing, switch to a project with available quota, or reduce demo requests. Free tier with 0 limits will always 429.
- **Pydantic init error**: use **keyword** arguments, not positional, when creating `AuditRequest`.
- **Hanging at key prompt**: set `GOOGLE_API_KEY` via `%env` (recommended) and remove any `getpass` code.

## Minimal Demo Snippet
```python
# key must already be set via %env GOOGLE_API_KEY=...

req = AuditRequest(
    request_id="REQ-9001",
    requester_name="Alice Smith",
    requester_email="alice@example.com",
    business_unit="Payments",
    title="SOC 2 Type II evidence for trust criteria",
    description="Need SOC 2 Type II support: security, availability controls, IR runbooks, vendor management."
)

state = {"req": req, "sanitized": None, "rule_result": None, "llm_result": None,
         "decision": None, "evidence": [], "ticket": None, "email": None, "log": {}}
out = app.invoke(state)
print(out["decision"])
print(out["ticket"].payload)
```

## Security Notes
- PII redaction runs **before** logging/persistence (emails are kept for comms only).
- Keep keys in secrets/variables; do not commit them to source control.
- Validate and sanitize any uploaded attachments before processing.

## Extensibility
- Swap Gemini with another provider by editing the **LLM verifier** only. The graph, rules, and artifacts remain unchanged.
- Front this with FastAPI for form intake; persist logs to S3/Snowflake/Postgres.
- Add per-framework evidence generators (e.g., SOC 2 TSC to control-matrix expansion).

## Files Written
- `it_audit_router_gemini_run.json` — demo run outputs

## Colab Runtime Order
1) Install → 2) Config → 3) Domain → 4) Models/Rules → 5) Gemini verifier → 6) Graph → 7) Demo
