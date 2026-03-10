# 📚 Docs Module – README

## 1. Module Identity
The **`docs/`** directory is the single source of truth for all user‑facing and developer documentation of the **FileCraft** system. It bundles markdown files that describe:
- the overall architecture and deployment strategy,
- the FastAPI configuration model,
- public API endpoints and their contracts,
- runtime settings, and
- operational best‑practice guides.

These files are consumed by the documentation generator (e.g., MkDocs or a custom CI step) to produce the public documentation site for **Retail Docs System v2.0**.

---
## 2. Interface Contract
The folder exports a **static documentation contract**. Each markdown file is considered a public artefact that may be referenced by other modules, CI pipelines, or external tools.

| File | Primary Export / Purpose | Key Sections | Consumers |
|------|--------------------------|--------------|-----------|
| `README.md` | entry‑point for the docs module – provides a high‑level overview and navigation guide. | Introduction, navigation table | Documentation site index |
| `api-endpoints.md` | exhaustive list of HTTP endpoints, request/response schemas, and example curl commands. | Endpoint table, security notes | API developers, external integrators |
| `configuration-summary.md` | concise table‑style summary of all configurable parameters (useful for quick reference). | Parameter table, default values | Ops team, CI validation scripts |
| `configuration.md` | **Full FastAPI configuration guide** – contains the most detailed description of every setting, its source code mapping, and usage examples. | Overview, categories (API, Server, Security, CORS, Rate‑Limiting, File Processing, DB, Redis/Celery, Feature Flags, External Services), validation, extension steps | Developers, configuration validators, automated tests |
| `deployment.md` | step‑by‑step deployment instructions for Docker, Kubernetes, and bare‑metal environments. | Prerequisites, Docker compose, Helm values, CI/CD pipeline snippets | DevOps, release engineers |

> **Contract rule**: Any change that adds, removes, or modifies a public setting, endpoint, or behaviour **must** be reflected in the corresponding markdown file.

---
## 3. Logic Flow – How the Files Interact
1. **Source‑code ↔ Documentation Mapping**
   - core configuration classes (`app/core/config.py`, `config/settings/base.py`) expose settings that are documented in `configuration.md`.
   - constants such as `MAX_UPLOAD_SIZE` and `IMAGE_FORMATS` from `app/helpers/constants.py` are referenced verbatim in the *File Processing Configuration* section of `configuration.md`.
   - Celery broker/backend URLs defined in `app/celery_app.py` (`REDIS_URL`, `CELERY_BROKER_URL`, `CELERY_RESULT_BACKEND`) are documented under *Redis and Celery Configuration*.
2. **Markdown Composition**
   - `README.md` contains a **Table of Contents** that links to the other markdown files using relative paths (e.g., `[Configuration Guide](configuration.md)`).
   - `configuration-summary.md` is generated (or manually kept) by extracting the **key/value pairs** from the detailed sections of `configuration.md`. It provides a quick‑lookup table for CI linting.
   - `api-endpoints.md` imports endpoint signatures from the FastAPI routers (`app/router/...`) and adds example payloads. The OpenAPI schema produced by `app/core/api_config.py` is used to keep this file in sync.
3. **CI / Documentation Generation**
   - A pre‑commit hook runs an **AST‑based impact analyzer**. When a public setting or endpoint changes, the hook flags the affected markdown file(s) and fails the build if the documentation is outdated.
   - The documentation generator reads the markdown files, resolves internal links, and publishes a static HTML site under `/docs`.
   - During deployment, `deployment.md` is parsed by the CI pipeline to inject environment‑specific values (e.g., `HOST`, `PORT`, `ENVIRONMENT`).

---
## 4. Dependencies
The docs module **depends** on the following project components to stay accurate:

- **Configuration layer** (`app/core/config.py`, `config/settings/base.py`): provides the source of truth for every setting mentioned in `configuration.md` and `configuration-summary.md`.
- **OpenAPI helpers** (`app/core/api_config.py`): supplies title, version, server URLs, and security scheme definitions that are mirrored in the *Enhanced Documentation* section.
- **Constants module** (`app/helpers/constants.py`): defines upload limits and supported image formats that appear in the *File Processing Configuration* table.
- **Celery setup** (`app/celery_app.py`): its broker and backend URLs are documented under *Redis and Celery Configuration*.
- **Exception hierarchy** (`app/exceptions/__init__.py`): informs the error‑response examples shown in `api-endpoints.md`.
- **Dependency utilities** (`app/dependencies.py`): the uptime endpoint example in `api-endpoints.md` references the `get_uptime` dependency.
- **Middleware** (`app/middleware/__init__.py`): global error handling behaviour is described in the *Security Headers* and *Rate Limiting Headers* sections.

> **Note**: The documentation module does **not** import any runtime code at execution time; it only **references** code symbols for traceability.

---
## 5. Maintenance Guidelines (localized)
1. **When adding a new setting** – update `config/settings/base.py`, then immediately edit `configuration.md` (add to the appropriate category) and run the *summary generator* to keep `configuration-summary.md` in sync.
2. **When exposing a new endpoint** – add the router to `app/router/...`, ensure the OpenAPI schema reflects the change, then document the route in `api-endpoints.md` with request/response examples.
3. **When modifying Celery or Redis configuration** – adjust `app/celery_app.py` and reflect those changes in the *Redis and Celery Configuration* block of `configuration.md`.
4. **Run the docs CI check** – `./scripts/check_docs.sh` (or the pre‑commit hook) will fail if any markdown file is out‑of‑date with respect to the source code.

---
### Quick Reference Table
| Artifact | Location | Update Trigger |
|----------|----------|----------------|
| `configuration.md` | `docs/` | Any change in `app/core/config.py`, `config/settings/base.py`, `app/helpers/constants.py` |
| `api-endpoints.md` | `docs/` | New router, altered path, or changed request model |
| `deployment.md` | `docs/` | CI/CD pipeline changes, Docker/K8s manifest updates |
| `configuration-summary.md` | `docs/` | Regenerated after each `configuration.md` edit |

---
*End of localized README for the `docs` module.*