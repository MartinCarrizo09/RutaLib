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
| IA | Vertex AI (Gemini Vision) | Ya usado en otros proyectos de Martín |
| Mapas | OpenStreetMap + MapLibre | Gratis, sin depender de Google para tiles |
| Geocoding | Google Geocoding API | Reverse geocoding dirección legible |
| Places | Google Places API | Autocomplete cuando no hay EXIF |
| Notificaciones | Firebase Cloud Messaging | Push iOS + Android |
| Offline (mobile) | SQLite local | Cola de reportes sin conexión |

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
- **Versionado de reportes:** reporte nuevo de tipo `resolved`/`improved`, nunca editar el original
- **Soft delete:** sí — campo `deleted_at timestamptz` nullable en `report` y `user`
- **Particionado:** no desde el día uno, esperar a 1M reportes

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
├── apps/
│   └── mobile/              # React Native (Expo)
│       ├── src/
│       │   ├── screens/
│       │   ├── components/
│       │   └── services/    # API client, SQLite offline
│       └── app.json
├── backend/                 # Python · FastAPI
│   ├── app/
│   │   ├── api/             # Routers: reports, places, map, auth
│   │   ├── services/        # ai_pipeline, geocoding, heatmap
│   │   └── models/          # SQLAlchemy + PostGIS
│   ├── alembic/             # Migraciones
│   └── pyproject.toml
├── infra/
│   └── docker-compose.yml   # Postgres+PostGIS, Redis, backend local
├── docs/
│   ├── CONTEXT.md           # ← este archivo
│   ├── architecture.md
│   ├── data-model.md
│   └── api-spec.md
└── CONTEXT.md               # copia en raíz para encontrarlo rápido
```

---

## Estado del proyecto

- [x] Concepto y estrategia de impacto definidos
- [x] Stack tecnológico decidido
- [x] Arquitectura documentada (`docs/architecture.md`)
- [x] Modelo de datos documentado (`docs/data-model.md`)
- [x] API spec documentada (`docs/api-spec.md`)
- [x] Decisiones pendientes del modelo cerradas
- [ ] `docker-compose.yml` — entorno local
- [ ] `pyproject.toml` — setup backend Python
- [ ] Estructura de carpetas inicial en el repo
- [ ] Primera migración Alembic (tablas base)
- [ ] Scaffold React Native (Expo)

---

## Próximo paso al retomar

Arrancar con el entorno local:
1. `infra/docker-compose.yml` — Postgres+PostGIS + Redis
2. `backend/pyproject.toml` — dependencias Python
3. Estructura de carpetas del backend
4. Primera migración Alembic

---

## Links

- Repo: https://github.com/MartinCarrizo09/RutaLib
- Video que originó la idea: nota de El Doce sobre accesibilidad en Córdoba
