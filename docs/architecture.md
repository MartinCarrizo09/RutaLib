# Arquitectura — Rutas Libres

## 1. Overview del sistema

Diagrama de alto nivel: usuarios móviles, backend, servicios externos y datos.

```mermaid
flowchart TB
    subgraph Mobile["📱 App Móvil (React Native / Expo)"]
        UI[UI: Mapa · Reportar · Rutas]
        LocalDB[(SQLite offline<br/>cola de reportes)]
        UI --- LocalDB
    end

    subgraph Backend["🐍 Backend (Python · FastAPI)"]
        API[API REST]
        AUTH[Auth / JWT]
        REPORTS[Reports Service]
        PLACES[Places Service]
        MAP[Map / Heatmap Service]
        AI[AI Pipeline]
        WORKER[Worker async<br/>Celery/RQ]

        API --> AUTH
        API --> REPORTS
        API --> PLACES
        API --> MAP
        REPORTS --> WORKER
        WORKER --> AI
    end

    subgraph Data["🗄️ Datos"]
        PG[(PostgreSQL<br/>+ PostGIS)]
        REDIS[(Redis<br/>cache + queue)]
        STORAGE[(Object Storage<br/>fotos)]
    end

    subgraph External["☁️ Servicios externos"]
        VERTEX[Vertex AI<br/>Gemini Vision]
        GMAPS[Google Maps<br/>Directions + Geocoding]
        FCM[Firebase Cloud<br/>Messaging]
    end

    Mobile -->|HTTPS| API
    REPORTS --> PG
    PLACES --> PG
    MAP --> PG
    MAP --> REDIS
    WORKER --> REDIS
    REPORTS --> STORAGE
    AI --> VERTEX
    PLACES --> GMAPS
    MAP --> GMAPS
    Backend -.push.-> FCM
    FCM -.-> Mobile
```

### Decisiones clave

- **PostgreSQL + PostGIS**: consultas geoespaciales nativas (radio, intersección de rutas, clustering). Alternativa considerada: MongoDB con índices 2dsphere — descartada por menor potencia en joins espaciales.
- **FastAPI**: tipado fuerte, OpenAPI automático, async nativo para orquestar llamadas a Vertex AI y Google Maps en paralelo.
- **Worker async** para el pipeline de IA: la clasificación de una foto puede tardar 3-8s, no puede bloquear el request.
- **Cola offline en mobile**: el usuario reporta donde no hay señal (subte, zonas rurales). Los reportes se encolan en SQLite y se sincronizan al reconectar.

---

## 2. Flujo del reporte con IA

Desde que el usuario saca la foto hasta que la barrera aparece en el mapa de otros usuarios.

```mermaid
sequenceDiagram
    autonumber
    actor U as Usuario
    participant M as App Móvil
    participant DB as SQLite local
    participant API as FastAPI
    participant S as Object Storage
    participant Q as Redis Queue
    participant W as Worker
    participant V as Vertex AI
    participant PG as PostgreSQL
    participant FCM as FCM Push

    U->>M: Saca foto + describe barrera
    M->>M: Captura GPS + timestamp
    M->>DB: Guarda report (status=pending)

    alt Hay conexión
        M->>API: POST /reports (multipart)
        API->>S: Upload foto → url
        API->>PG: INSERT report (status=queued)
        API->>Q: enqueue(report_id)
        API-->>M: 202 Accepted {report_id}
        M->>DB: Update status=queued

        W->>Q: dequeue(report_id)
        W->>PG: SELECT report
        W->>V: classify(photo_url, description)
        V-->>W: {type, severity, tags, confidence}

        alt confidence >= 0.7
            W->>PG: UPDATE report (status=approved, classification)
            W->>FCM: notify users in radius
            FCM-->>M: Push "Nueva barrera cerca"
        else confidence < 0.7
            W->>PG: UPDATE report (status=review)
            Note over W,PG: Cola de moderación humana
        end
    else Sin conexión
        M->>DB: Keep status=pending
        Note over M,DB: Sync retry al reconectar
    end
```

### Puntos críticos del flujo

- **Paso 3-4**: el reporte se persiste localmente *antes* de intentar subirlo. Si la app crashea o pierde señal, el dato no se pierde.
- **Paso 8**: el API responde `202 Accepted` inmediatamente. El cliente no espera a la IA.
- **Paso 14**: umbral de confianza en 0.7 — por debajo va a moderación humana. Ajustable según métricas de falsos positivos.
- **Paso 15**: notificación solo a usuarios con rutas guardadas que intersecten el radio del reporte (no broadcast global).

---

## 3. Próximos documentos

- [`data-model.md`](./data-model.md) — esquema de tablas, índices geoespaciales, relaciones.
- [`api-spec.md`](./api-spec.md) — endpoints, contratos, códigos de error.
