# Docs Module README

## 1. Module Identity
The **`docs/`** folder is the single source of truth for all human‑readable documentation that ships with **FileCraft**.  It provides developers, operators, and end‑users with:

- An overview of the project (`README.md`).
- Detailed API surface (`api-endpoints.md`).
- Configuration reference and summaries (`configuration.md` & `configuration-summary.md`).
- Full deployment instructions for Docker, Docker‑Compose, Kubernetes and production‑grade setups (`deployment.md`).

These markdown files are version‑controlled alongside the code base, ensuring that documentation drifts are detectable by the Documentation Consistency Enforcer.

---

## 2. Interface Contract (Exported Artifacts)
| Artifact | Purpose | Primary Consumers |
|----------|---------|-------------------|
| `README.md` | High‑level project introduction, quick‑start links, contribution guidelines. | New contributors, CI badge generators. |
| `api-endpoints.md` | Exhaustive list of FastAPI routes, HTTP methods, request/response schemas, authentication requirements. | API developers, SDK generators, external integrators. |
| `configuration.md` | In‑depth description of every environment variable and config object (e.g., `AppConfig`, `Redis` settings, Celery broker URLs). | DevOps, ops engineers, configuration validation scripts. |
| `configuration-summary.md` | Tabular summary of the most important runtime flags (e.g., `ENVIRONMENT`, `MAX_FILE_SIZE`). | Quick reference for operators, CI lint checks. |
| `deployment.md` | Step‑by‑step guide for Docker, Docker‑Compose, manual Docker runs, production Docker‑Compose, and Kubernetes manifests. Includes performance‑tuning snippets and monitoring hooks. | Site reliability engineers, CI deployment pipelines. |

*The folder does not expose executable code, but the above markdown files constitute the public contract of the documentation module.*

---

## 3. Logic Flow (How the Files Interact)
1. **Source of Truth** – The `configuration.md` file is authored from the `AppConfig` dataclass located in `app/core/config.py`.  When new fields are added (e.g., a new `REDIS_TLS_URL`), the **Documentation Enforcer** flags `configuration.md` for update.
2. **API Surface** – `api-endpoints.md` mirrors the routes registered in `app/router/**` modules (e.g., `images`, `audio`, `video`, `encoder_decoder`).  Each route entry includes:
   - Path and HTTP verb.
   - Request model (derived from `pydantic` schemas in `app/schemas`).
   - Response model (often `SystemCheckResponse` or custom error objects from `app/exceptions`).
3. **Deployment Guidance** – `deployment.md` references concrete values defined in:
   - `app/helpers/constants.py` for limits such as `MAX_UPLOAD_SIZE`.
   - `app/core/config.py` for defaults like `external_port`, `host`, and `environment`.
   - `app/celery_app.py` for broker/backend URLs (e.g., `REDIS_URL`).
   The deployment guide therefore reflects the *actual* runtime configuration rather than static placeholders.
4. **Summary Generation** – `configuration-summary.md` is a distilled table derived from the same source as `configuration.md`.  Its purpose is to provide a checklist for CI validation scripts that ensure required env‑vars are present before a container starts.
5. **README Coordination** – The top‑level `README.md` links to the other documentation files, acting as a navigation hub.  Any change that adds a new feature (e.g., a new *audio* processing pipeline) should trigger:
   - An addition to `api-endpoints.md` (new endpoint).
   - A new entry in `configuration.md` if new env‑vars are introduced.
   - An update to `deployment.md` if additional services (e.g., a new Redis queue) are required.

---

## 4. Dependencies (External Modules Referenced)
| Dependency | Reason for Dependency |
|------------|-----------------------|
| `app/core/config.py` | Provides the canonical `AppConfig` schema and runtime defaults used throughout the docs (environment, ports, security headers, etc.). |
| `app/helpers/constants.py` | Supplies file‑size limits and format tables that are explicitly documented in the configuration and deployment guides. |
| `app/celery_app.py` | Defines `REDIS_URL` and broker configuration; these values appear in the *Redis configuration* section of `deployment.md`. |
| `app/router/**` (e.g., `converters`, `encoder_decoder`) | Determines the list of public FastAPI routes that must be reflected in `api-endpoints.md`. |
| `app/exceptions` | Custom exception hierarchy (`FileCraftException`, `FileValidationError`, etc.) is referenced in the API error documentation sections. |
| `app/schemas/responses.py` | Response models such as `SystemCheckResponse` are described in the endpoint documentation. |
| CI / Lint tools (e.g., `markdownlint`, custom Doc‑Drift script) | Consume the markdown files to enforce consistency and detect stale sections. |

**Note:** The documentation module is *read‑only* from the application runtime perspective; it does not import any of these modules at execution time.  Its only dependency is the repository’s source tree, which it mirrors.

---

## 5. Maintenance Guidelines
- **When adding a new environment variable**: update `app/core/config.py`, then modify `configuration.md` *and* the corresponding row in `configuration-summary.md`.
- **When exposing a new API endpoint**: ensure the route is registered in a router module, then add an entry to `api-endpoints.md` with request/response schemas.
- **When changing file size limits** (`MAX_UPLOAD_SIZE`): edit `app/helpers/constants.py` and immediately adjust the limits mentioned in `deployment.md` and any relevant sections of `configuration.md`.
- **When modifying Celery or Redis settings**: reflect changes in the *Redis configuration* and *Celery configuration* blocks of `deployment.md`.
- Run the **Documentation Consistency Enforcer** CI job after each PR; any drift will be reported as a PR comment with a suggested diff.

---

*Generated by the Documentation Consistency Enforcer – confidence: high.*