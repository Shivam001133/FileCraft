# FileCraft – Distributed File Processing API

## Vision
FileCraft is a high‑performance, open‑source service for converting and processing files—images, audio, video, and documents—through a unified RESTful API. Designed for both local development and production deployment, it abstracts complex media handling (FFmpeg, compression, encoding/decoding) behind simple endpoints while supporting asynchronous processing via Celery.

The platform aims to provide developers with a reliable, extensible foundation for any workflow that requires on‑the‑fly file transformation, allowing seamless integration into micro‑service architectures or monolithic back‑ends alike.

---

## Architecture
The repository follows a classic FastAPI‑centric layout:

- **`app/`** – Core application package containing FastAPI entry‑point, dependency injection, global middleware, and custom exception hierarchy.
- **`app/core/`** – Configuration, logging, security utilities and OpenAPI customisation.
- **`app/router/`** – Grouped FastAPI routers for converters, encoders, decoders, and authentication.
- **`app/services/`** – Synchronous processing logic for each media type (image, audio, video, compression) and abstract base classes.
- **`app/services/decoders/` & `app/services/encoders/`** – Low‑level implementations (Base64, Hex, JWT, URL, Hash).
- **`app/tasks/`** – Celery task definitions that delegate heavy workloads to background workers.
- **`app/helpers/`** – Utility modules (constants, validators, converters).
- **`config/`** – Central configuration objects (environment variables, database settings – currently unused but ready for extension).
- **`Dockerfile` & `.env.*`** – Containerisation and environment configuration for both Docker and local development.

FastAPI serves HTTP requests, routes them through the appropriate router, and either processes them synchronously (lightweight tasks) or enqueues a Celery task for intensive operations. Celery workers communicate with Redis (configured via `REDIS_URL`) for broker and result backend.

---

## Key Components
| Directory | Responsibility |
|-----------|----------------|
| `app/main.py` | Creates the FastAPI instance, injects middle‑wares, registers routers, and supplies the custom OpenAPI schema. |
| `app/router/` | Declarative API endpoints grouped by domain (images, audio, video, encoders/decoders). |
| `app/services/` | Core business logic – file validation, format conversion, compression, and interaction with FFmpeg. |
| `app/services/encoders/` & `app/services/decoders/` | Implement stateless encoding/decoding algorithms (Base64, Hex, JWT, URL, Hash). |
| `app/tasks/` | Celery task wrappers that invoke service methods asynchronously. |
| `app/middleware/` | Global exception handling and request‑logging middleware. |
| `app/helpers/` | Constants (e.g., `MAX_UPLOAD_SIZE`), file validation helpers, and generic converters. |
| `app/dependencies.py` | FastAPI dependency providers such as application uptime tracking. |
| `app/exceptions/` | Custom exception hierarchy (`FileCraftException`, `FileValidationError`, etc.) that standardises error responses. |
| `config/` | Pydantic‑based configuration (`AppConfig`) loaded from environment variables. |
| `celery_app.py` | Celery application factory, broker/backend configuration, and task routing. |

---

## Tech Stack
| Category | Tool / Library |
|----------|----------------|
| Language | **Python 3.12+** |
| Web Framework | **FastAPI** |
| ASGI Server | **uvicorn** (implicit in Docker/`start_local.sh`) |
| Data Validation | **Pydantic** (used in `app/schemas/`) |
| Background Jobs | **Celery** |
| Message Broker / Result Store | **Redis** (configured via `REDIS_URL`) |
| Media Processing | **FFmpeg** (external binary) |
| Logging | **standard `logging`** module, custom `app/core/app_logging.py` |
| Containerisation | **Docker** |
| Configuration Management | **pydantic BaseSettings** (`app/core/config.py`) |
| Security | FastAPI security utilities (`app/core/security.py`) |
| Testing (not shown) | **pytest** (commonly used in FastAPI projects) |

---

## Flow of Operation
1. **Client Request** – An HTTP request hits the FastAPI app (e.g., `POST /api/v1/images/convert`).
2. **Routing** – The request is dispatched to the appropriate router module (`app/router/converters/images.py`).
3. **Dependency Injection** – FastAPI injects shared dependencies such as `AppState` (uptime) and configuration values.
4. **Validation** – Request payloads are validated against Pydantic models defined in `app/schemas/requests.py`.
5. **Processing Decision** – If the operation is lightweight (e.g., Base64 encoding), the service method runs synchronously and returns a response. For CPU‑intensive jobs (video transcoding, large image conversion), the endpoint enqueues a Celery task via the `celery_app`. 
6. **Celery Worker** – Workers (`app/tasks/*.py`) pull the job from Redis, invoke the corresponding service (`app/services/video.py`, `app/services/image.py`, etc.), and store the result back in Redis.
7. **Result Retrieval** – The API can either poll a status endpoint or receive the final payload directly if the task completes within the request timeout.
8. **Response** – FastAPI returns a JSON payload containing success status, processed file URL/path, and any metadata.

---

## Interaction
### Primary API Endpoints
| Category | Method | Path | Description |
|----------|--------|------|-------------|
| Images | `POST` | `/api/v1/images/convert` | Convert an uploaded image to a target format (e.g., JPG → PNG). |
| Audio | `POST` | `/api/v1/audio/convert` | Transcode audio files (e.g., WAV → MP3). |
| Video | `POST` | `/api/v1/video/convert` | Transcode video files using FFmpeg. |
| Encode | `POST` | `/api/v1/encode/base64` | Encode a binary file to Base64 string. |
| Decode | `POST` | `/api/v1/decode/base64` | Decode a Base64 string back to a binary file. |
| Compression | `POST` | `/api/v1/compress` | Archive and compress files (ZIP, TAR, etc.). |
| Health | `GET` | `/health` | Simple health‑check returning service status. |

Clients can interact via `curl`, HTTP client libraries, or any API testing tool (Postman, Insomnia). Example for image conversion:
```bash
curl -X POST "http://127.0.0.1:8000/api/v1/images/convert" \
  -H "Content-Type: multipart/form-data" \
  -F "file=@/path/to/source.jpg" \
  -F "target_format=png"
```

---

## Deployment
### Docker (Production‑Ready)
1. **Build the image**
   ```bash
   docker build -t filecraft:latest .
   ```
2. **Create an `.env` file** (copy from `.env.example` or `.env.local`) and set required variables, e.g.:
   ```env
   REDIS_URL=redis://redis:6379/0
   PORT=8000
   HOST=0.0.0.0
   ```
3. **Run Redis** (if not already available)
   ```bash
   docker run -d --name redis -p 6379:6379 redis:7-alpine
   ```
4. **Start the application container**
   ```bash
   docker run -d --name filecraft \
     --env-file .env \
     -p 8000:8000 \
     --link redis:redis \
     filecraft:latest
   ```
   The API will be reachable at `http://<host>:8000` and the OpenAPI docs at `/docs`.

### Local Development (No Docker)
1. **Clone the repository**
   ```bash
   git clone <repository-url>
   cd FileCraft
   ```
2. **Create a virtual environment & install dependencies**
   ```bash
   python -m venv venv
   source venv/bin/activate
   pip install -r requirements.txt
   ```
3. **Set environment variables** – copy one of the provided env files:
   ```bash
   cp .env.local .env
   # Adjust values if needed (e.g., REDIS_URL, PORT)
   ```
4. **Start Redis locally** (optional for async tasks). For quick testing you can disable Celery by setting `task_always_eager=True` in `celery_app.py`.
5. **Run the FastAPI server**
   ```bash
   uvicorn app.main:create_application --host 0.0.0.0 --port 8000 --reload
   ```
6. **(Optional) Start Celery workers**
   ```bash
   celery -A celery_app.celery_app worker --loglevel=info -Q image_processing,audio_processing,video_processing,optimization
   ```
7. **Access the API** – Open `http://127.0.0.1:8000/docs` in a browser to explore the interactive documentation.

---

## Additional Resources
- **Detailed setup & troubleshooting** – see `README_LOCAL.md` and `README_DOCKER.md`.
- **Environment configuration** – all configurable items are declared in `app/core/config.py` and can be overridden via the `.env*` files.
- **Contribution guidelines** – follow the standard GitHub flow; ensure any public API change updates the OpenAPI schema and documentation.

*FileCraft is actively maintained. For questions or contributions, open an issue or submit a pull request.*
