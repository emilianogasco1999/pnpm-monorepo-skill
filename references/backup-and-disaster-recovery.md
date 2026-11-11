# Estrategia de Backups Automatizados y Disaster Recovery

Esta referencia técnica define la arquitectura de respaldos desatendidos (24/7), sincronización multi-destino y procedimientos de recuperación ante desastres para entornos en producción.

---

## 1. Topología del Servicio de Backup

El sistema cuenta con un contenedor dedicado (`.docker/backup/Dockerfile`) que interactúa con la base de datos PostgreSQL y el almacenamiento de objetos S3 (MinIO) a través de la red interna de Docker.

```
┌────────────────────────────────────────────────────────┐
│                   DOCKER ENGINE PROD                   │
│                                                        │
│  ┌────────────┐   (Red Interna)    ┌────────────────┐  │
│  │ postgres   │ ◄───────────────── │     backup     │  │
│  └────────────┘    pg_dump         │  (Alpine 3.20) │  │
│                                    │  - crond       │  │
│  ┌────────────┐    (Volumen :ro)   │  - pg_dump 16  │  │
│  │ minio data │ ─────────────────► │  - rclone      │  │
│  └────────────┘                    └───────┬────────┘  │
└────────────────────────────────────────────┼───────────┘
                                             │
             ┌───────────────────────────────┴───────────────────────────────┐
             ▼                                                               ▼
   ┌────────────────────┐                                          ┌────────────────────┐
   │  Disco Servidor    │                                          │  Nube Externa      │
   │  /backups/         │                                          │  (Google Drive,    │
   │  (Últimos 5)       │                                          │   AWS S3, R2)      │
   └────────────────────┘                                          └────────────────────┘
```

---

## 2. Componentes del Contenedor de Backup

### 2.1. Dockerfile Especializado (`.docker/backup/Dockerfile`) y `entrypoint.sh`
Construido sobre Alpine Linux mínimo, incluyendo solo las utilidades necesarias:
- `postgresql16-client`: Garantiza compatibilidad de versión del motor de volcado.
- `rclone`: Soporte para más de 40 proveedores de almacenamiento en la nube.
- `gzip`, `tar`, `findutils`: Compresión eficiente y rotación de archivos.
- **`entrypoint.sh`**: Script de inicio que registra dinámicamente la expresión cron (`BACKUP_CRON`) en `/etc/crontabs/root` redirigiendo logs a `stdout/stderr` del proceso principal (`/proc/1/fd/1`), ejecuta opcionalmente un respaldo inmediato al arrancar si `BACKUP_ON_STARTUP="true"`, e inicia `crond -f -l 2` en primer plano.

### 2.2. Script Orquestador (`backup.sh`)

1. **Volcado Relacional (`pg_dump`)**:
   ```bash
   export PGPASSWORD="${POSTGRES_PASSWORD}"
   pg_dump -h "${POSTGRES_HOST:-postgres}" \
           -p "${POSTGRES_PORT:-5432}" \
           -U "${POSTGRES_USER}" \
           -d "${POSTGRES_DB}" \
           --no-owner \
           --no-privileges | gzip > "${DESTINO}/db_${POSTGRES_DB}_${FECHA}.sql.gz"
   ```
2. **Empaquetado de Almacenamiento S3**:
   ```bash
   tar -czf "${DESTINO}/archivos_minio_${FECHA}.tar.gz" -C "/minio_data" .
   ```
3. **Copia Remota Multi-Cuenta (Rclone)**:
   ```bash
   rclone copy "${DESTINO}" "gdrive1:Backups_Finca/${FECHA}" --fast-list --retries 3
   rclone copy "${DESTINO}" "gdrive2:Backups_Finca/${FECHA}" --fast-list --retries 3
   ```
4. **Política de Rotación y Limpieza**:
   - **Local**: Se ordenan las carpetas cronológicamente y se eliminan las más antiguas para conservar únicamente los **últimos 5 backups**.
   - **Remota**: Se utiliza `rclone delete --min-age 5d` para limpiar automáticamente los respaldos que excedan el período de retención.

---

## 3. Configuración en `docker-compose.prod.yml`

```yaml
backup:
  container_name: finca_backup_prod
  restart: unless-stopped
  build:
    context: ../backup
    dockerfile: Dockerfile
  environment:
    POSTGRES_HOST: postgres
    POSTGRES_USER: ${POSTGRES_USER}
    POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
    POSTGRES_DB: ${POSTGRES_DB}
    BACKUP_CRON: ${BACKUP_CRON:-0 3 * * *}      # Ejecución diaria a las 03:00 AM
    BACKUP_ON_STARTUP: ${BACKUP_ON_STARTUP:-false}
  volumes:
    - ../../backups:/backups                     # Persistencia local en host
    - minio_prod_data:/minio_data:ro             # Acceso de solo lectura a archivos
    - ../backup/rclone.conf:/root/.config/rclone/rclone.conf:ro
  depends_on:
    postgres:
      condition: service_healthy
    minio:
      condition: service_started
```

---

## 4. Ejecución de Respaldo Manual / Bajo Demanda

Para generar un backup inmediato sin esperar el cron de la madrugada:

```bash
docker compose -f .docker/prod/docker-compose.prod.yml exec backup /scripts/backup.sh
```

O en desarrollo local:
```bash
pnpm dev:backup
```

---

## 5. Procedimiento de Disaster Recovery (Recuperación)

En caso de fallo total de disco, corrupción de datos o migración a un nuevo VPS:

### Paso 1: Levantar los servicios base limpios
```bash
docker compose -f .docker/prod/docker-compose.prod.yml up -d postgres minio
```

### Paso 2: Localizar el archivo de respaldo más reciente
```bash
# Ejemplo: backups/2026-10-14_03-00-00/
ls -l backups/
```

### Paso 3: Restaurar la base de datos PostgreSQL
```bash
gunzip -c backups/2026-10-14_03-00-00/db_finca_don_osvaldo_db_*.sql.gz | \
docker exec -i finca_postgres_prod psql -U finca_user -d finca_don_osvaldo_db
```

### Paso 4: Restaurar los archivos de almacenamiento S3
```bash
docker run --rm \
  -v finca-don-osvaldo_minio_prod_data:/dest \
  -v $(pwd)/backups/2026-10-14_03-00-00:/src:ro \
  alpine:3.20 tar -xzf /src/archivos_minio_*.tar.gz -C /dest
```

### Paso 5: Levantar las aplicaciones (API + Web)
```bash
docker compose -f .docker/prod/docker-compose.prod.yml up -d
```
Verificar el estado con `docker compose -f .docker/prod/docker-compose.prod.yml ps`.
