# Seismic Platform

Plataforma de monitoreo de eventos sísmicos en tiempo real.
Consume la API pública de USGS cada 3 minutos, procesa los eventos,
los almacena en MongoDB y expone una API REST + WebSockets con dashboard Angular.

## Requisitos

- Docker >= 24.0
- Docker Compose >= 2.20

## Levantar el sistema

```bash
# 1. Clonar / descomprimir el proyecto
cd "entrevista quipux"

# 2. Crear el archivo de variables de entorno
cp infra/.env.example infra/.env

# 3. Editar infra/.env con tus valores (ver sección Variables de entorno)
# Como mínimo cambiar: MONGO_PASSWORD, JWT_SECRET_KEY, SEED_PASSWORD

# 4. Levantar todos los servicios
cd infra
docker compose up --build
```

El sistema estará listo cuando veas en los logs:
```
seismic_api      | api_ready
seismic_ingesta  | ingestion_service_started
```

## URLs de acceso

| Servicio | URL | Credenciales |
|---|---|---|
| **Dashboard Angular** | http://localhost:4200 | admin / (SEED_PASSWORD del .env) |
| **API REST (Swagger)** | http://localhost:8000/docs | — |
| **Airflow** | http://localhost:8080 | admin / (AIRFLOW_ADMIN_PASSWORD del .env) |
| **Grafana** | http://localhost:3000 | admin / (GF_SECURITY_ADMIN_PASSWORD del .env) |
| **Prometheus** | http://localhost:9090 | — |

## Variables de entorno

Copiar `infra/.env.example` a `infra/.env` y completar:

| Variable | Descripción | Obligatoria |
|---|---|---|
| `MONGO_PASSWORD` | Contraseña de MongoDB | ✅ |
| `JWT_SECRET_KEY` | Secret para firmar JWT (mín. 32 chars) | ✅ |
| `SEED_PASSWORD` | Contraseña del usuario admin inicial | ✅ |
| `AIRFLOW_ADMIN_PASSWORD` | Contraseña del admin de Airflow | ✅ |
| `GF_SECURITY_ADMIN_PASSWORD` | Contraseña de Grafana | ✅ |
| `INGESTION_INTERVAL_SECONDS` | Intervalo de ingesta en segundos (default: 180) | ❌ |
| `METRICS_CACHE_TTL_SECONDS` | TTL del caché Redis en segundos (default: 60) | ❌ |

> **Nunca commitear el `.env` real** — está en `.gitignore`.

## Arquitectura

```
USGS API → Ingesta (cada 3 min) → MongoDB ← FastAPI → Angular
                                      ↓
                              Redis (caché + pub/sub)
                                      ↓
                              WebSocket → Angular (tiempo real)
                                      
Airflow (cada hora) → Lee MongoDB → Genera reporte → MongoDB
FastAPI → Prometheus → Grafana
```

Ver `docs/arquitectura.md` para el diagrama completo y justificación de decisiones.

## Estructura del proyecto

```
entrevista quipux/
├── infra/                    # docker-compose.yml, configs Prometheus/Grafana
├── seismic-backend/          # FastAPI + ingesta + Airflow (Python)
├── seismic-frontend/         # Dashboard Angular v20
└── docs/                     # Documentación técnica completa
```

## API Endpoints

### Auth
| Método | Endpoint | Descripción |
|---|---|---|
| POST | `/auth/login` | Login con username/password → JWT |
| POST | `/auth/refresh` | Renovar access token con refresh token |
| GET | `/auth/me` | Usuario autenticado actual |

### Datos (requieren Bearer token)
| Método | Endpoint | Descripción |
|---|---|---|
| GET | `/earthquakes` | Lista de sismos con filtros y paginación |
| GET | `/metrics` | Métricas por ventana horaria (caché Redis) |
| GET | `/reports` | Reportes horarios generados por Airflow |
| WS | `/ws/events` | Stream en tiempo real de sismos nuevos |
| GET | `/health` | Health check del servicio |
| GET | `/metrics` | Métricas Prometheus de la app |

### Parámetros de `/earthquakes`
| Parámetro | Tipo | Descripción |
|---|---|---|
| `magnitude_min` | float | Magnitud mínima |
| `magnitude_max` | float | Magnitud máxima |
| `magnitude_range` | string | micro\|menor\|ligero\|moderado\|fuerte\|mayor |
| `from_date` | datetime | Desde (ISO 8601) |
| `to_date` | datetime | Hasta (ISO 8601) |
| `page` | int | Página (default: 1) |
| `page_size` | int | Tamaño de página (default: 20, max: 100) |
| `sort_by` | string | Campo de ordenamiento (default: event_time) |
| `order` | string | asc\|desc (default: desc) |

## Colección Postman

Importar el archivo `docs/seismic-platform.postman_collection.json` en Postman.
La colección incluye todos los endpoints con ejemplos y variables de entorno.

## Comandos útiles

```bash
# Ver logs de un servicio específico
docker compose logs -f api
docker compose logs -f ingesta

# Reiniciar solo el backend
docker compose restart api ingesta

# Detener todo y eliminar volúmenes (reset completo)
docker compose down -v

# Ejecutar solo el backend (sin frontend)
docker compose up mongodb redis api ingesta airflow
```

## Tecnologías

| Capa | Tecnología |
|---|---|
| API REST | FastAPI + Pydantic + Motor (async) |
| Base de datos | MongoDB 7 |
| Caché / Pub-Sub | Redis 7 |
| Scheduler | Apache Airflow 2.10 |
| Observabilidad | Prometheus + Grafana |
| Frontend | Angular v20 (Signals + TanStack Query + RxJS) |
| Contenerización | Docker Compose |

## Documentación

- [Arquitectura y decisiones técnicas](docs/arquitectura.md)
- [Decisiones ADR](docs/decisiones.md)
- [Estimaciones y plan de trabajo](docs/estimaciones.md)
- [Pasos que seguí](docs/pasos-que-segui.md)
- [Rol del líder técnico](docs/rol-lider-tecnico.md)
- [Recorrido por el código](docs/recorrido-codigo.md)
