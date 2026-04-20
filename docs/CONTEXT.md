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
│   ├── Dockerfile           # stub
│   ├── pyproject.toml       # stub
│   ├── alembic/             # dir vacío, esperando `alembic init`
│   └── app/
│       ├── __init__.py      ✅
│       ├── main.py          ✅
│       ├── core/
│       │   ├── __init__.py  ✅
│       │   └── config.py    ✅ settings Pydantic
│       ├── api/
│       │   └── __init__.py  ✅ (vacío, falta routers)
│       ├── services/
│       │   └── __init__.py  ✅ (vacío, falta ai_pipeline/geocoding/heatmap)
│       └── models/
│           ├── __init__.py  ✅
│           ├── base.py      ✅
│           ├── user.py      ✅
│           ├── report.py    ✅
│           ├── place.py     ✅
│           └── route.py     ✅
├── infra/
│   └── docker-compose.yml   # stub
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
- [x] Stubs escritos: `main.py`, `core/config.py`, `models/{base,user,report,place,route}.py`
- [x] Decisiones del modelo cerradas definitivamente (ver [data-model.md § Decisiones cerradas](./data-model.md#decisiones-cerradas))

### En curso / siguiente
- [ ] `infra/docker-compose.yml` — Postgres+PostGIS + Redis (hoy es stub)
- [ ] `backend/pyproject.toml` completo — FastAPI, SQLAlchemy 2.x, asyncpg, alembic, google-cloud-aiplatform, google-cloud-storage, redis, python-jose
- [ ] `backend/app/api/` — routers vacíos todavía (reports, places, map, auth, me)
- [ ] `backend/app/services/` — vacío (ai_pipeline, geocoding, heatmap)
- [ ] `alembic init` + primera migración con las tablas del data-model (incluir `deleted_at`, `report_kind`, `resolves_report_id` y el CHECK constraint)
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
