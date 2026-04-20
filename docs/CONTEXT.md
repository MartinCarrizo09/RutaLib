# RutaLib — contexto del proyecto

> Este archivo es el punto de entrada para cualquier sesión de trabajo.
> Pegarlo al inicio de cada conversación con Claude para retomar sin perder contexto.

---

## Qué es RutaLib

App móvil de accesibilidad urbana para personas con movilidad reducida.
Mapa colaborativo donde la comunidad reporta el estado de rampas, veredas y barreras urbanas.
Nació a partir de una nota periodística de El Doce (Córdoba) sobre el problema de accesibilidad en la ciudad.

**No es un producto comercial.** Es un movimiento con tecnología adentro.
El objetivo es generar evidencia pública suficiente para que la política no pueda ignorar el problema.

---

## Equipo

- **Martín Carrizo** — Founder, Tech Lead. Ing. en Sistemas (UTN FRC, graduación 2027).
  Founder de Digital Axios. Experiencia en RAG, Vertex AI, arquitecturas de backend.
- **Claude** — Co-developer. Trabaja junto a Martín en VS Code vía Claude Code extension.

---

## Stack decidido

| Capa | Tecnología | Motivo |
|---|---|---|
| Mobile | React Native (Expo) | iOS + Android desde un repo |
| Backend | Python · FastAPI | Ecosistema IA, async nativo |
| DB | PostgreSQL 16 + PostGIS 3.x | Consultas geoespaciales nativas |
| Migraciones | Alembic | Standard con SQLAlchemy |
| Cache / Queue | Redis | Cola async + cache heatmap |
| Worker | Celery o RQ | Pipeline IA no bloquea request |
| IA | Vertex AI — **`gemini-2.5-flash`** | `gemini-2.0-flash` y `1.5-flash*` devuelven 404 en `growth-projects`. En `us-central1`. Es *thinking model* → pasar `thinkingConfig.thinkingBudget: 0` en el pipeline de clasificación para bajar costo/latencia |
| Mapas | OpenStreetMap + MapLibre | Gratis, sin depender de Google para tiles |
| Geocoding | Google Geocoding API | Reverse geocoding dirección legible |
| Places | Google Places API | Autocomplete cuando no hay EXIF |
| Notificaciones | Firebase Cloud Messaging | Push iOS + Android |
| Offline (mobile) | SQLite local | Cola de reportes sin conexión |

---

## Infra externa (GCP) — estado real

Todo verificado end-to-end el **2026-04-19**.

### Cuenta y proyecto
- **GCP project:** `growth-projects`
- **Cuenta Google owner:** `growthimbar@gmail.com` — **NO** es la cuenta personal de Martín (`mtmartincarrizo@gmail.com`). Es una cuenta separada dedicada a la infra de RutaLib.
- Si se pierde acceso a `growthimbar@gmail.com`, se pierde el proyecto completo. Password en password manager + 2FA activado.

### APIs habilitadas
- `aiplatform.googleapis.com` — Vertex AI
- `storage.googleapis.com` — Cloud Storage
- `directions-backend.googleapis.com` — Google Directions
- `geocoding-backend.googleapis.com` — Google Geocoding
- `places-backend.googleapis.com` — Google Places
- `cloudresourcemanager.googleapis.com` — management

### Recursos creados
- **Bucket GCS:** `gs://rutalib-photos-dev` en `us-central1`, storage class `Standard`, uniform bucket access, soft-delete OFF (dev). Write/read/delete verificado.
- **API Key Maps (dev):** nombre `RutaLib Maps Dev`, restringida a `Directions + Geocoding + Places`, sin restricción de aplicación (para dev). Guardada en `backend/.env`, ignorada por git.

### Autenticación local
- `gcloud auth login growthimbar@gmail.com` — CLI auth
- `gcloud auth application-default login` — **ADC** para que el código Python (librerías google-cloud-*) pueda llamar a Vertex AI sin service account. ADC en `C:\Users\Martin\AppData\Roaming\gcloud\application_default_credentials.json`.
- Config gcloud aislada: `rutalib` (no pisar el `default` que apunta a `proinnovate-483814` de otro laburo de Martín).

### Pendiente
- **Firebase Cloud Messaging** — cuando arranque la app mobile, no antes.
- **`SECRET_KEY` del `.env`** — quedó expuesta en chat durante setup; rotar antes de cualquier deploy público (aunque está git-ignored).
- **Cuenta personal como `Owner`** en IAM del proyecto — para no depender solo de `growthimbar@gmail.com`.

---

## Arquitectura — resumen

Tres capas:

**Cliente:** App React Native · Dashboard web público · Widget embebible para medios

**Backend Python (FastAPI):**
- `reports` service — recibe foto + geo, encola en Redis
- `ai_pipeline` — Vertex AI clasifica la foto async (worker)
- `map` service — tiles, heatmap, rutas accesibles
- `places` service — sello "Lugar Accesible", verificación
- `report_generator` — PDF municipal automático

**Datos y externos:** PostgreSQL+PostGIS · Redis · Object Storage · Vertex AI · Google APIs

### Flujo crítico — reporte de barrera
1. Usuario saca foto → GPS capturado en EXIF
2. Si no hay GPS → Places Autocomplete (campo opcional)
3. App guarda en SQLite local primero (nunca se pierde)
4. Upload multipart → API responde `202 Accepted` inmediato
5. Worker async clasifica con Gemini Vision (3-8s)
6. Si confidence >= 0.7 → `approved` → pin en mapa + push a usuarios afectados
7. Si confidence < 0.7 → `review` → cola de moderación humana
8. Sin conexión → se sincroniza al reconectar

---

## Modelo de datos — entidades principales

- `user` — perfil + `accessibility_profile` jsonb
- `report` — núcleo del sistema, barrera puntual con GIST index en `location`
- `report_photo` — separada, permite 1..N fotos por reporte
- `place` — lugar físico con `accessibility_score` 0-100
- `place_review` — reseña con `accessibility_tags` jsonb
- `route` — LINESTRING del camino habitual del usuario

**Tipos geo:** `GEOGRAPHY(POINT, 4326)` para puntos, `GEOGRAPHY(LINESTRING, 4326)` para rutas.

### Decisiones cerradas del modelo
Confirmadas antes de la primera migración Alembic — congeladas salvo razón fuerte. Detalle completo en [data-model.md](./data-model.md#decisiones-cerradas).

- **Versionado de reportes:** reporte nuevo con `report_kind ∈ {resolved, improved}` y FK `resolves_report_id` apuntando al `barrier` original. Nunca se edita el original. Historial inmutable para métricas y PDF municipal.
- **Soft delete:** columna `deleted_at timestamptz NULL` en `report` y `user`. Índices parciales con `WHERE deleted_at IS NULL`. Filtro aplicado en el repo layer del backend, no en cada query.
- **Particionado:** tabla `report` monolítica hasta 1M filas. A 800k (80%) → ticket de deuda técnica para migrar a partición mensual por `created_at`.

---

## API — resumen de endpoints

Base URL: `http://localhost:8000/api/v1`
Auth: JWT Bearer. Errores: RFC 7807. IDs: UUID v4. Timestamps: ISO 8601 UTC.

| Método | Endpoint | Descripción |
|---|---|---|
| POST | /auth/register | Registro |
| POST | /auth/login | Login |
| POST | /reports | Crear reporte (multipart) |
| GET | /reports?bbox=... | Reportes en bounding box |
| POST | /reports/:id/confirm | Confirmar barrera (crowdsourcing) |
| GET | /map/heatmap?bbox=... | Heatmap por zona |
| POST | /map/route | Ruta accesible entre dos puntos |
| GET | /places?near=... | Lugares accesibles cercanos |
| POST | /places/:id/review | Review con accessibility_tags |
| GET | /me | Perfil propio |
| POST | /me/routes | Guardar ruta habitual |

---

## Estructura de carpetas (target)

```
rutalib/
├── .gitignore               # ← ya creado, ignora .env y credenciales
├── apps/
│   └── mobile/              # React Native (Expo) — todavía no existe
│       ├── src/
│       │   ├── screens/
│       │   ├── components/
│       │   └── services/    # API client, SQLite offline
│       └── app.json
├── backend/                 # Python · FastAPI
│   ├── .env                 # ← git-ignored, con secretos reales
│   ├── .venv/               # git-ignored, con deps instaladas
│   ├── Dockerfile           # stub
│   ├── pyproject.toml       ✅ completo (incluye pins: pydantic[email], bcrypt<5)
│   ├── alembic.ini          ✅
│   ├── alembic/
│   │   ├── env.py           ✅ async + fix SSL condicional
│   │   └── versions/
│   │       └── 0001_initial_schema.py  ✅ aplicada
│   └── app/
│       ├── __init__.py      ✅
│       ├── main.py          ✅ /health + router auth montado
│       ├── core/
│       │   ├── __init__.py  ✅
│       │   ├── config.py    ✅ settings Pydantic
│       │   └── security.py  ✅ bcrypt hash + JWT encode/decode
│       ├── api/
│       │   ├── __init__.py  ✅
│       │   ├── deps.py      ✅ DbSession, CurrentUser, get_current_user
│       │   └── auth.py      ✅ /register, /login, /me
│       ├── services/
│       │   └── __init__.py  ✅ (vacío, falta ai_pipeline/geocoding/heatmap)
│       └── models/
│           ├── __init__.py  ✅ re-exporta Base + todos los modelos
│           ├── base.py      ✅ engine async + SSL condicional + get_db
│           ├── user.py      ✅
│           ├── report.py    ✅
│           ├── place.py     ✅
│           └── route.py     ✅
├── infra/
│   └── docker-compose.yml   ✅ db:5433 + redis + backend
└── docs/
    ├── CONTEXT.md           # ← este archivo
    ├── architecture.md
    ├── data-model.md
    └── api-spec.md
```

---

## Estado del proyecto

### Hecho
- [x] Concepto y estrategia de impacto definidos
- [x] Stack tecnológico decidido
- [x] Arquitectura documentada (`docs/architecture.md`)
- [x] Modelo de datos documentado (`docs/data-model.md`)
- [x] API spec documentada (`docs/api-spec.md`)
- [x] Decisiones pendientes del modelo cerradas
- [x] Repo inicializado y pusheado a GitHub
- [x] `.gitignore` configurado (ignora `.env`, credenciales, node_modules, etc.)
- [x] GCP project `growth-projects` creado
- [x] APIs habilitadas (Maps trio + Vertex AI + Cloud Storage)
- [x] Bucket `rutalib-photos-dev` creado y testeado (R/W/D)
- [x] gcloud configurado con config aislada `rutalib`
- [x] ADC local configurado (Python puede llamar a Vertex sin service account)
- [x] **Vertex AI / Gemini probado end-to-end** — modelo `gemini-2.5-flash`
- [x] `backend/.env` con todos los valores reales (excepto Firebase)
- [x] Estructura de carpetas del backend creada (`app/`, `core/`, `api/`, `services/`, `models/` con `__init__.py`)
- [x] `backend/pyproject.toml` completo (FastAPI, SQLAlchemy async, asyncpg, geoalchemy2, alembic, redis/rq, google-cloud-*, passlib/jose, pydantic-settings)
- [x] Modelos SQLAlchemy 2.x escritos: `base.py` (engine + `AsyncSessionLocal` + `get_db`), `user.py`, `report.py` (con `Report` + `ReportPhoto` + CHECK constraint), `place.py` (con `Place` + `PlaceReview`), `route.py`
- [x] `backend/app/models/__init__.py` re-exporta `Base` y todos los modelos — necesario para que `Base.metadata` conozca las tablas cuando Alembic corre `env.py`
- [x] `backend/app/main.py` — FastAPI app + `/health` + CORS. Routers comentados hasta que existan
- [x] `backend/app/core/config.py` — `Settings` con Pydantic, lee de `.env`
- [x] `infra/docker-compose.yml` — servicios `db` (postgis/postgis:16-3.4), `redis` (redis:7-alpine) y `backend` con healthchecks y `depends_on`
- [x] `backend/alembic.ini` + `alembic/env.py` (async, inyecta URL desde `settings.database_url`)
- [x] `backend/alembic/versions/0001_initial_schema.py` — migración hand-written con todas las tablas del data-model, extensión `postgis`, índices GIST + parciales con `WHERE deleted_at IS NULL`, y el CHECK constraint `chk_resolves_matches_kind`
- [x] Decisiones del modelo cerradas definitivamente (ver [data-model.md § Decisiones cerradas](./data-model.md#decisiones-cerradas))
- [x] **Docker Compose funcionando** — `db` (postgis/postgis:16-3.4) y `redis` (redis:7-alpine) arrancan healthy
- [x] **`backend/.venv`** creado con `python -m venv` + `pip install -e .` — todas las deps instaladas
- [x] **Migración 0001 aplicada** ✅ — verificado con `\dt`: tablas `user`, `place`, `place_review`, `report`, `report_photo`, `route` + `alembic_version`, todos los índices correctos, CHECK constraint vigente
- [x] **Auth endpoints funcionando end-to-end** ✅ — `POST /api/v1/auth/register` (201 + JWT), `POST /api/v1/auth/login` (200 + JWT), `GET /api/v1/auth/me` (200 + perfil). Errores correctos: 409 duplicado, 401 bad pwd, 401 sin token
- [x] Hashing passwords con `passlib[bcrypt]` + bcrypt 4.x (pin, ver gotcha #10)
- [x] JWT con `python-jose`, HS256, `sub = user_id`, expiración según `ACCESS_TOKEN_EXPIRE_MINUTES`
- [x] Primer usuario real en DB: `martin@rutalib.dev` (UUID `6f3feafc-f50e-4e19-a5c1-2d1f870495bc`)

### En curso / siguiente
- [ ] `backend/app/api/reports.py` — `POST /reports` (multipart: foto + geo + descripción), `GET /reports?bbox=...`, `POST /reports/:id/confirm`
- [ ] `backend/app/api/places.py` — `GET /places?near=...`, `POST /places/:id/review`
- [ ] `backend/app/api/map.py` — `GET /map/heatmap?bbox=...`, `POST /map/route`
- [ ] `backend/app/services/geocoding.py` — wrapper de Google Geocoding API usando `GOOGLE_MAPS_API_KEY`
- [ ] `backend/app/services/ai_pipeline.py` — Gemini 2.5 Flash para clasificar fotos de barreras
- [ ] `backend/app/services/heatmap.py` — agregación GIST + cache Redis
- [ ] Worker RQ arrancado con `docker compose up worker` (todavía no existe el servicio)
- [ ] Seed mínimo opcional: un par de reports dummy para probar el mapa

---

## Setup local — comandos concretos para retomar

Asumiendo el repo clonado, desde la raíz:

```bash
# 1. Levantar DB (Postgres 16 + PostGIS + Redis)
cd infra && docker compose up db redis -d

# 2. Venv + deps (una sola vez)
cd ../backend
python -m venv .venv
.venv/Scripts/python.exe -m pip install -e .   # Windows
# o source .venv/bin/activate && pip install -e .   # Linux/Mac

# 3. Aplicar migración
.venv/Scripts/alembic.exe upgrade head

# 4. Arrancar el backend
.venv/Scripts/uvicorn.exe app.main:app --reload
# → http://localhost:8000/health
# → http://localhost:8000/docs (Swagger)
```

---

## ⚠️ Gotchas del setup local (Windows) — evitar re-descubrirlas

Estas cosas rompieron el setup y costaron debug. Documentadas para que la próxima sesión no pierda tiempo.

### 1. Puerto Docker: **5433**, no 5432
Martín tiene **PostgreSQL 17 nativo instalado como servicio Windows** (`postgresql-x64-17`) que se autoarranca y escucha en 5432. No se tocó (puede estar en uso por otros proyectos).

- `docker-compose.yml` mapea `"5433:5432"` (host:container).
- `backend/.env` tiene `DATABASE_URL=...@localhost:5433/rutalib`.
- `backend/alembic.ini` tiene la misma URL por consistencia visual (env.py igual la sobreescribe).
- Dentro de Docker, el servicio `backend` usa `db:5432` (red interna de Docker, no mapeo de host).

### 2. `connect_args={"ssl": False}` obligatorio en local
El proxy de Docker Desktop en Windows **corta el handshake SSL** que asyncpg intenta por default. Síntomas: `ConnectionDoesNotExistError: connection was closed in the middle of operation`.

Fix aplicado en dos lugares:
- `backend/alembic/env.py` → `async_engine_from_config(..., connect_args={"ssl": False})`
- `backend/app/models/base.py` → `create_async_engine(..., connect_args={"ssl": False} if _is_local else {})`

Para **producción con DB cloud (Supabase/Neon/Cloud SQL)** el flag condicional `_is_local` (detecta `localhost`/`127.0.0.1`) evita mandar `ssl=False`, así se usa SSL normal.

### 3. `spatial_index=False` en columnas `Geography`
Por default **GeoAlchemy2 genera automáticamente** un índice GIST en cada columna `Geography`. Combinado con los `op.create_index(...)` explícitos de la migración → **CREATE INDEX duplicado** → falla con `relation "idx_X_location" already exists`.

Fix: todas las columnas `Geography` tienen `spatial_index=False`, tanto en la migración como en los modelos SQLAlchemy (`place.py`, `report.py`, `route.py`). Los índices se crean SOLO vía los `op.create_index()` manuales, que además son parciales (`WHERE deleted_at IS NULL`).

### 4. `app/models/__init__.py` re-exporta Base
Vacío genera `ImportError` al correr `alembic upgrade head` (env.py hace `from app.models import Base`). Debe tener:
```python
from app.models.base import Base, engine, AsyncSessionLocal, get_db
from app.models.user import User
from app.models.place import Place, PlaceReview
from app.models.report import Report, ReportPhoto
from app.models.route import Route
```
Así `Base.metadata` conoce todas las tablas cuando Alembic corre.

### 5. `pyproject.toml` — discovery explícito de packages
`pip install -e .` falla con "multiple packages found" porque setuptools ve `app/` y `alembic/`. Agregado al final del pyproject:
```toml
[build-system]
requires = ["setuptools>=61.0"]
build-backend = "setuptools.build_meta"

[tool.setuptools.packages.find]
include = ["app*"]
exclude = ["alembic*", "tests*"]
```

### 6. `CREATE EXTENSION "uuid-ossp"` en la migración es dead weight
La migración lo crea pero nadie lo usa (todas las PKs usan `gen_random_uuid()` que es nativo en PG 13+). Se puede remover en una próxima migración, no bloquea nada.

### 7. Tabla `user` — palabra reservada PG
Funciona con quoting. Si en un query crudo te olvidás las comillas, falla. SQLAlchemy las agrega automáticamente via el modelo `User`, no es problema en ORM queries.

### 8. PostGIS `tiger` schema en el search_path
La imagen `postgis/postgis:16-3.4` instala las extensiones **postgis_tiger_geocoder** y **topology**, y agrega ambos al `search_path` (`"$user", public, topology, tiger`). La `tiger` schema tiene una tabla llamada **`place`** (geocoder USA). No nos afecta porque nuestras tablas están en `public`, pero tener presente si alguna query hace `SELECT * FROM place` sin schema prefix → puede resolver a `tiger.place`.

### 9. `pydantic.EmailStr` requiere `email-validator`
`from pydantic import EmailStr` funciona, pero cuando FastAPI registra una ruta con un modelo que usa `EmailStr`, Pydantic genera el schema y **ahí** falla con `ImportError: email-validator is not installed`. Síntoma: uvicorn crashea al arrancar (no es error de request).

Fix: dependency `pydantic[email]>=2.7.0` en pyproject (en lugar de `pydantic>=2.7.0`). El extra `[email]` trae `email-validator` + `dnspython`.

### 10. `passlib 1.7.4` es incompatible con `bcrypt 5.x`
`bcrypt 5.0` removió la truncación silenciosa de passwords >72 bytes. `passlib 1.7.4` tiene un auto-test interno (`detect_wrap_bug`) que genera un hash con un password largo al arrancar el primer hash → `ValueError: password cannot be longer than 72 bytes`. Los passwords reales pueden ser cortos, pero el test interno siempre corre.

Síntoma: `POST /auth/register` devuelve 500, traceback apunta a `_finalize_backend_mixin`.

Fix aplicado: pin `bcrypt<5` en pyproject.toml. Cuando passlib saque una versión nueva con soporte oficial para bcrypt 5.x (issue abierto en su repo), se puede soltar el pin.

### 11. uvicorn `--reload` + Windows → procesos huérfanos en port 8000
Con `--reload`, uvicorn spawna subprocess. Si el parent muere sin limpiar, el child queda como **orphan** heredando el socket. `taskkill` al parent no mata al child → puerto 8000 sigue ocupado. Hay que `taskkill` también los `python.exe` hijos (se ven con `wmic process ... parentprocessid=<uvicorn_pid>`).

**Workaround:** en local preferir `uvicorn app.main:app --host 127.0.0.1 --port 8000` (sin `--reload`) y reiniciar manual tras cambios, o usar `uvicorn --reload-dir app` que limita el watcher.

---

## Troubleshooting rápido

| Síntoma | Causa probable | Fix |
|---|---|---|
| `connection was closed in the middle of operation` | SSL handshake roto en Windows | `connect_args={"ssl": False}` |
| `password authentication failed` | Puerto 5432 ocupado por PG nativo | Usar puerto **5433** |
| `relation "idx_X_location" already exists` | GeoAlchemy2 auto-index | `spatial_index=False` en columna Geography |
| `ImportError: cannot import name 'Base'` | `app/models/__init__.py` vacío | Re-exportar Base + modelos |
| `multiple top-level packages discovered` | Falta config setuptools | `[tool.setuptools.packages.find]` con `include=["app*"]` |
| `pg_isready` ok pero asyncpg falla | DB inicializando PostGIS tras wipe | Esperar 5s extra, los init scripts de `postgis_tiger` tardan |
| `ImportError: email-validator is not installed` | `pydantic` sin extra `[email]` | `pydantic[email]>=2.7.0` en deps |
| `password cannot be longer than 72 bytes` en registro/login | bcrypt 5.x rompe passlib 1.7.4 | Pin `bcrypt<5` |
| `error 10048: solo se permite un uso de cada dirección de socket` | uvicorn `--reload` huérfano | Matar subprocess python children de uvicorn + reintentar |
- [ ] Scaffold React Native (Expo)
- [ ] Firebase Cloud Messaging (cuando arranque mobile)
- [ ] Rotar `SECRET_KEY` antes de cualquier release público

---

## Comandos útiles

### Cambiar entre proyectos gcloud
Martín tiene dos proyectos GCP activos — no confundir:

```bash
gcloud config configurations activate rutalib    # → growth-projects · growthimbar@gmail.com
gcloud config configurations activate default    # → proinnovate-483814 · mtmartincarrizo@gmail.com
gcloud config configurations list                # ver todas
gcloud config list                               # ver la activa
```

### Test rápido de Vertex AI
```bash
TOKEN=$(gcloud auth application-default print-access-token)
curl -s -X POST \
  "https://us-central1-aiplatform.googleapis.com/v1/projects/growth-projects/locations/us-central1/publishers/google/models/gemini-2.5-flash:generateContent" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"contents":[{"role":"user","parts":[{"text":"Reply OK"}]}],"generationConfig":{"maxOutputTokens":50,"thinkingConfig":{"thinkingBudget":0}}}'
```

### Test bucket GCS
```bash
echo "hi" | gcloud storage cp - gs://rutalib-photos-dev/_test.txt
gcloud storage cat gs://rutalib-photos-dev/_test.txt
gcloud storage rm gs://rutalib-photos-dev/_test.txt
```

---

## Próximo paso al retomar

Toda la infra externa está lista. **Falta código y entorno local**:

1. Completar `infra/docker-compose.yml` — Postgres 16 + PostGIS 3.x + Redis 7
2. Completar `backend/pyproject.toml` — FastAPI, SQLAlchemy 2.x, asyncpg, alembic, google-cloud-aiplatform, google-cloud-storage, redis, python-jose (JWT)
3. Crear estructura `backend/app/{api,services,models,core}`
4. Primera migración Alembic con las tablas del data-model
5. Primer endpoint andando: `POST /reports` (sin IA todavía, solo persistencia)
6. Recién después: wire del worker + `ai_pipeline` con `gemini-2.5-flash`

---

## Variables en `backend/.env` (git-ignored)

El archivo existe localmente con valores reales. Nunca se commitea. Campos actuales:

```
DATABASE_URL                 postgresql+asyncpg://rutalib:rutalib_dev@localhost:5432/rutalib
REDIS_URL                    redis://localhost:6379/0
SECRET_KEY                   (generada — rotar antes de release)
ACCESS_TOKEN_EXPIRE_MINUTES  60
REFRESH_TOKEN_EXPIRE_DAYS    30
GOOGLE_MAPS_API_KEY          (AIza... real, restringida a Maps trio)
GOOGLE_CLOUD_PROJECT         growth-projects
VERTEX_AI_LOCATION           us-central1
VERTEX_AI_MODEL              gemini-2.5-flash
AI_CONFIDENCE_THRESHOLD      0.7
GCS_BUCKET_NAME              rutalib-photos-dev
FIREBASE_CREDENTIALS_PATH    ./firebase-credentials.json  (archivo todavía no existe)
ENVIRONMENT                  development
DEBUG                        true
```

---

## Regla operativa — commits

En este repo **los commits se firman solo como Martin Carrizo**. Nunca agregar trailer `Co-Authored-By: Claude ...` ni ninguna mención a Claude/AI. El autor del commit es el git config local (`Martin <mtmartincarrizo@gmail.com>`).

---

## Links

- Repo: https://github.com/MartinCarrizo09/RutaLib
- Nota que originó la idea: nota de El Doce sobre accesibilidad en Córdoba
