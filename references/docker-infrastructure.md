# Infraestructura y Contenedores en Monorepo (Docker)

Esta referencia técnica describe los estándares de contenedorización, orquestación y persistencia de servicios para monorepos con soporte Docker.

---

## 1. Filosofía de Desarrollo Híbrido

En desarrollo, el monorepo adopta un modelo **híbrido optimizado para Developer Experience (DX)**:

1. **Servicios de Apoyo en Docker**:
   - Bases de datos relacionales (ej: PostgreSQL).
   - Servidores de almacenamiento de objetos (ej: MinIO / S3 compatible).
   - Herramientas visuales de administración (ej: PgAdmin).
   - Tareas automáticas de respaldo (backup).
2. **Aplicaciones en Host (Node.js local con pnpm)**:
   - Corren directamente en la máquina del desarrollador (`pnpm dev:api`, `pnpm dev:web`).
   - **Ventaja**: Hot Module Replacement (HMR) ultrarrápido, depuración interactiva y sin overhead de virtualización para herramientas de frontend/backend.

---

## 2. Topología Modular de Docker Compose

Para evitar archivos de compose masivos y difíciles de mantener, se utiliza una estructura modular en `.docker/`:

```
<monorepo-root>/
├── docker-compose.yml                # Punto de entrada raíz (utiliza include)
└── .docker/
    ├── dev/
    │   └── docker-compose.dev.yml    # Servicios para entorno de desarrollo local
    ├── prod/
    │   └── docker-compose.prod.yml   # Orquestación de producción (apps + servicios)
    └── backup/
        └── Dockerfile                # Imagen para scripts de respaldo periódico
```

### Inclusión en la Raíz (`docker-compose.yml`):

```yaml
include:
  - path: .docker/dev/docker-compose.dev.yml
```

Permite que comandos simples como `docker compose up -d` desde la raíz utilicen la configuración de desarrollo sin necesidad de flags de ruta extensos.

### Plantilla Canónica de Desarrollo (`.docker/dev/docker-compose.dev.yml`):

Incluye obligatoriamente los 4 servicios base de apoyo (`postgres`, `pgadmin`, `minio` con `alpine/minio:latest-release`, y `backup` apuntando a `../backup`):

```yaml
name: <project-slug>

services:
  postgres:
    image: postgres:16-alpine
    container_name: <prefix>_postgres_dev
    environment:
      POSTGRES_USER: ${POSTGRES_USER:-<prefix>_user}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:-<prefix>_password_secret}
      POSTGRES_DB: ${POSTGRES_DB:-<prefix>_db}
    ports:
      - '${POSTGRES_PORT:-5432}:5432'
    volumes:
      - postgres_dev_data:/var/lib/postgresql/data
    healthcheck:
      test:
        [
          'CMD-SHELL',
          'pg_isready -U ${POSTGRES_USER:-<prefix>_user} -d ${POSTGRES_DB:-<prefix>_db}',
        ]
      interval: 5s
      timeout: 5s
      retries: 5

  pgadmin:
    image: dpage/pgadmin4:8.14
    container_name: <prefix>_pgadmin_dev
    environment:
      PGADMIN_DEFAULT_EMAIL: ${PGADMIN_EMAIL:-admin@<domain>.com}
      PGADMIN_DEFAULT_PASSWORD: ${PGADMIN_PASSWORD:-admin123}
    ports:
      - '${PGADMIN_PORT:-5050}:80'
    depends_on:
      postgres:
        condition: service_healthy

  minio:
    image: alpine/minio:latest-release
    container_name: <prefix>_minio_dev
    command: server /data --console-address ":9001"
    environment:
      MINIO_ROOT_USER: ${MINIO_ROOT_USER:-<prefix>_minio_admin}
      MINIO_ROOT_PASSWORD: ${MINIO_ROOT_PASSWORD:-<prefix>_minio_secret}
    ports:
      - '${MINIO_PORT:-9000}:9000' # Puerto S3 API (lo usa el backend)
      - '${MINIO_CONSOLE_PORT:-9001}:9001' # Consola Web (para vos)
    volumes:
      - minio_dev_data:/data

  backup:
    container_name: <prefix>_backup_dev
    restart: unless-stopped
    build:
      context: ../backup
      dockerfile: Dockerfile
    environment:
      POSTGRES_HOST: postgres
      POSTGRES_USER: ${POSTGRES_USER:-<prefix>_user}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:-<prefix>_password_secret}
      POSTGRES_DB: ${POSTGRES_DB:-<prefix>_db}
      MINIO_HOST: minio
      MINIO_ROOT_USER: ${MINIO_ROOT_USER:-<prefix>_minio_admin}
      MINIO_ROOT_PASSWORD: ${MINIO_ROOT_PASSWORD:-<prefix>_minio_secret}
      BACKUP_CRON: ${BACKUP_CRON:-0 3 * * *}
      BACKUP_ON_STARTUP: ${BACKUP_ON_STARTUP:-false}
    volumes:
      - ../../backups:/backups
      - minio_dev_data:/minio_data:ro
      - ../backup/rclone.conf:/root/.config/rclone/rclone.conf:ro
    depends_on:
      postgres:
        condition: service_healthy
      minio:
        condition: service_started

volumes:
  postgres_dev_data:
    name: <project-slug>_postgres_dev_data
  minio_dev_data:
```

### Plantilla Canónica de Producción (`.docker/prod/docker-compose.prod.yml`):

Incluye los 5 servicios de producción (`postgres` y consola `minio` protegidos en `127.0.0.1`, `api`, `web` con `INTERNAL_API_URL`, y `backup`), red dedicada y volúmenes nombrados:

```yaml
name: <project-slug>-prod

services:
  postgres:
    image: postgres:16
    container_name: <prefix>_postgres_prod
    restart: unless-stopped
    environment:
      POSTGRES_USER: ${POSTGRES_USER:-<prefix>_user}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:-<prefix>_password_secret}
      POSTGRES_DB: ${POSTGRES_DB:-<prefix>_db}
    volumes:
      - postgres_prod_data:/var/lib/postgresql/data
    ports:
      # Solo accesible desde el localhost del servidor para mantenimiento / scripts
      - '127.0.0.1:${POSTGRES_PORT:-5432}:5432'
    healthcheck:
      test:
        [
          'CMD-SHELL',
          'pg_isready -U ${POSTGRES_USER:-<prefix>_user} -d ${POSTGRES_DB:-<prefix>_db}',
        ]
      interval: 10s
      timeout: 5s
      retries: 5

  minio:
    image: alpine/minio:latest-release
    container_name: <prefix>_minio_prod
    restart: unless-stopped
    command: server /data --console-address ":9001"
    environment:
      MINIO_ROOT_USER: ${MINIO_ROOT_USER:-<prefix>_minio_admin}
      MINIO_ROOT_PASSWORD: ${MINIO_ROOT_PASSWORD:-<prefix>_minio_secret}
    ports:
      - '${MINIO_PORT:-9000}:9000'
      - '127.0.0.1:${MINIO_CONSOLE_PORT:-9001}:9001' # Consola Web protegida en localhost
    volumes:
      - minio_prod_data:/data
    healthcheck:
      test: ['CMD-SHELL', 'wget --spider -q http://localhost:9000/minio/health/live || exit 1']
      interval: 15s
      timeout: 5s
      retries: 3

  api:
    container_name: <prefix>_api_prod
    restart: unless-stopped
    build:
      context: ../..
      dockerfile: apps/api/Dockerfile
    environment:
      NODE_ENV: production
      PORT_API: 4000
      DATABASE_URL: postgresql://${POSTGRES_USER:-<prefix>_user}:${POSTGRES_PASSWORD:-<prefix>_password_secret}@postgres:5432/${POSTGRES_DB:-<prefix>_db}?schema=public
      JWT_SECRET: ${JWT_SECRET}
      JWT_REFRESH_SECRET: ${JWT_REFRESH_SECRET}
      ACCESS_TOKEN_EXPIRES: ${ACCESS_TOKEN_EXPIRES:-15m}
      REFRESH_TOKEN_EXPIRES: ${REFRESH_TOKEN_EXPIRES:-7d}
      CORS_ORIGIN: ${CORS_ORIGIN:-http://localhost:3000}
      S3_ENDPOINT: ${S3_ENDPOINT:-http://minio:9000}
      S3_ACCESS_KEY: ${MINIO_ROOT_USER:-<prefix>_minio_admin}
      S3_SECRET_KEY: ${MINIO_ROOT_PASSWORD:-<prefix>_minio_secret}
      S3_BUCKET: ${S3_BUCKET:-<prefix>-archivos}
      S3_REGION: ${S3_REGION:-us-east-1}
    ports:
      - '${PORT_API:-4000}:4000'
    depends_on:
      postgres:
        condition: service_healthy
      minio:
        condition: service_started

  web:
    container_name: <prefix>_web_prod
    restart: unless-stopped
    build:
      context: ../..
      dockerfile: apps/web/Dockerfile
      args:
        INTERNAL_API_URL: ${INTERNAL_API_URL:-http://api:${PORT_API:-4000}}
    environment:
      NODE_ENV: production
      PORT: 3000
      INTERNAL_API_URL: ${INTERNAL_API_URL:-http://api:${PORT_API:-4000}}
    ports:
      - '${PORT_WEB:-3000}:3000'
    depends_on:
      api:
        condition: service_healthy

  backup:
    container_name: <prefix>_backup_prod
    restart: unless-stopped
    build:
      context: ../backup
      dockerfile: Dockerfile
    environment:
      POSTGRES_HOST: postgres
      POSTGRES_USER: ${POSTGRES_USER:-<prefix>_user}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:-<prefix>_password_secret}
      POSTGRES_DB: ${POSTGRES_DB:-<prefix>_db}
      MINIO_HOST: minio
      MINIO_ROOT_USER: ${MINIO_ROOT_USER:-<prefix>_minio_admin}
      MINIO_ROOT_PASSWORD: ${MINIO_ROOT_PASSWORD:-<prefix>_minio_secret}
      BACKUP_CRON: ${BACKUP_CRON:-0 3 * * *}
      BACKUP_ON_STARTUP: ${BACKUP_ON_STARTUP:-false}
    volumes:
      - ../../backups:/backups
      - minio_prod_data:/minio_data:ro
      - ../backup/rclone.conf:/root/.config/rclone/rclone.conf:ro
    depends_on:
      postgres:
        condition: service_healthy
      minio:
        condition: service_started

networks:
  default:
    name: <prefix>_prod_network

volumes:
  postgres_prod_data:
    name: <project-slug>_postgres_prod_data
  minio_prod_data:
    name: <project-slug>_minio_prod_data
```

---

## 3. Persistencia de Datos y Volúmenes Nombrados

Toda base de datos y almacenamiento de archivos debe utilizar **volúmenes nombrados** de Docker para asegurar que los datos sobrevivan a reinicios o reconstrucciones de imágenes:

```yaml
volumes:
  postgres_dev_data:
    driver: local
  minio_dev_data:
    driver: local
```

- **Regla**: Nunca mapear carpetas del código fuente directamente como volumen de base de datos (`./data:/var/lib/postgresql/data`), ya que degrada el rendimiento de disco y puede provocar conflictos de permisos en Windows/Linux.

---

## 4. Redes Internas y Puertos de Servicio

1. **Resolución Interna**: Entre contenedores de Docker (por ejemplo, el script de backup conectándose a PostgreSQL), la comunicación se realiza mediante el nombre del servicio en la red de Docker (`postgres:5432`).
2. **Acceso desde el Host**: Las aplicaciones que corren localmente en la máquina física acceden a los servicios mediante los puertos expuestos (`localhost:5432` para BD, `localhost:9000` para API S3, `localhost:9001` para consola MinIO).

---

## 5. Comandos Estándar de Infraestructura

```bash
# Levantar únicamente los servicios de base de datos y storage en background
pnpm dev:db          # docker compose -f .docker/dev/docker-compose.dev.yml up -d

# Detener los contenedores preservando los datos
pnpm dev:down        # docker compose -f .docker/dev/docker-compose.dev.yml down

# Ejecutar una copia de seguridad bajo demanda
pnpm dev:backup      # docker compose -f .docker/dev/docker-compose.dev.yml run --rm backup

# Ver logs de un servicio específico
docker compose logs -f postgres
```
