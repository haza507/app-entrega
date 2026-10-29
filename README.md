# Sistema de Gestión de Planificaciones Docentes

Monolito full-stack (SvelteKit + Bun + PostgreSQL + MinIO) para que los docentes de un colegio entreguen sus planificaciones en PDF y coordinación las revise, devuelva o apruebe. Ver [PLAN.md](PLAN.md) para la arquitectura completa, el esquema de base de datos y las decisiones tomadas durante la construcción.

## Stack

SvelteKit 2 / Svelte 5 · TypeScript · Tailwind CSS 4 · PostgreSQL + Drizzle ORM · Better Auth · MinIO (S3) · Zod · Quill · Playwright (generación de PDF) · Vitest + Playwright (e2e) · Bun · Docker Compose.

## Desarrollo local

Requisitos: [Bun](https://bun.sh), Docker Desktop.

```bash
bun install
docker compose up -d        # Postgres + MinIO
bun run db:generate         # (solo si cambiaste el schema)
bun run db:migrate
bun run db:seed             # crea admin, director, 2 coordinadores, 10 docentes y datos de ejemplo
bun run dev
```

Luego crea el bucket de MinIO una sola vez (no se crea solo):

```bash
docker exec <contenedor-minio> mc alias set local http://localhost:9000 minioadmin minioadmin
docker exec <contenedor-minio> mc mb local/planning-documents
```

La app queda en `http://localhost:5173`. La consola de MinIO en `http://localhost:9001` (usuario/clave: `minioadmin`/`minioadmin`).

### Credenciales de prueba (`bun run db:seed`)

| Rol           | Correo                                           | Contraseña      |
| ------------- | ------------------------------------------------ | --------------- |
| Administrador | `admin@colegio.edu`                              | `Admin1234!`    |
| Director      | `director@colegio.edu`                           | `Director1234!` |
| Coordinador   | `coord1@colegio.edu` / `coord2@colegio.edu`      | `Coord1234!`    |
| Docente       | `docente1@colegio.edu` … `docente10@colegio.edu` | `Docente1234!`  |

> No hay proveedor de correo configurado (ver PLAN.md §10): el enlace de recuperación de contraseña se imprime en la consola del servidor en desarrollo, y las contraseñas iniciales de nuevas cuentas las define el administrador al crearlas, no se generan ni se envían solas.

### Scripts

```bash
bun run dev            # servidor de desarrollo
bun run build           # build de producción
bun run preview         # sirve el build de producción localmente
bun run check           # type-check (svelte-check)
bun run lint            # prettier --check + eslint
bun run format          # prettier --write
bun run test            # tests unitarios (Vitest)
bun run test:e2e        # tests end-to-end (Playwright; primera vez: bunx playwright install chromium)
bun run db:generate     # genera una migración a partir del schema Drizzle
bun run db:migrate      # aplica migraciones pendientes
bun run db:studio       # explorador visual de la base de datos
bun run db:seed         # datos de desarrollo (ver arriba)
bun run check-deadlines # notifica planificaciones con fecha límite próxima (ver "Tareas programadas")
```

## Producción

La app corre igual que en desarrollo, con `NODE_ENV=production` y variables de entorno propias (ver `.env.example`). El adapter configurado es `@sveltejs/adapter-node`, así que el build produce un servidor Node/Bun independiente en `build/`.

### Despliegue recomendado (servidor Linux)

1. **Base de datos y almacenamiento**: levanta `postgres` y `minio` con el mismo `docker-compose.yml` (usa los puertos oficiales 5432/9000/9001 salvo que ya estén ocupados) o con instancias gestionadas equivalentes.
2. **Build de la app**:
   ```bash
   bun install --production
   bun run build
   bun run db:migrate
   ```
3. **Arranque**: `NODE_ENV=production bun ./build/index.js` (o `node build/index.js`, el adapter-node es compatible con ambos runtimes), idealmente supervisado con `systemd` o un gestor de procesos.
4. **Reverse proxy + HTTPS**: pon nginx (o Caddy) delante, terminando TLS y reenviando a la app (`proxy_pass http://127.0.0.1:3000;` por defecto). Asegúrate de que `ORIGIN` y `BETTER_AUTH_URL` en el entorno de producción apunten al dominio público real (Better Auth valida el origen de las peticiones).
5. **Variables de entorno**: copia `.env.example`, genera un `BETTER_AUTH_SECRET` real (32+ caracteres aleatorios), y usa credenciales de Postgres/MinIO distintas de las de desarrollo.
6. **Chromium para generar PDF**: si un docente redacta la planificación con el editor en vez de subir un PDF, el servidor la renderiza a PDF con Playwright/Chromium (ver PLAN.md §15). Instálalo una vez en el servidor: `bunx playwright install --with-deps chromium`.

## Tareas programadas

El proyecto no incluye colas ni un scheduler embebido (ver PLAN.md §10), así que el aviso de fecha límite próxima (`DEADLINE_APPROACHING`) se implementa como un script independiente que hay que programar externamente:

```bash
bun run check-deadlines
```

Revisa los períodos académicos activos y notifica a cada docente que tenga una planificación sin enviar (`DRAFT` o `RETURNED`) cuya fecha límite esté dentro de la ventana configurada en `/admin/settings` (por defecto 3 días). Es seguro ejecutarlo varias veces: no repite una notificación ya enviada para la misma planificación y fecha límite.

Prográmalo una vez al día, por ejemplo:

```cron
# crontab -e — todos los días a las 07:00
0 7 * * * cd /ruta/al/proyecto && bun run check-deadlines >> /var/log/check-deadlines.log 2>&1
```

En Windows, usa el Programador de tareas con `bun.exe run check-deadlines` como acción, mismo directorio de trabajo.

## Backups

Los volúmenes de Docker (`postgres_data`, `minio_data`) **no son un backup**: si el disco del servidor falla, se pierden igual. Hay que respaldarlos activamente y fuera del propio servidor.

### PostgreSQL

```bash
# Backup (dump comprimido)
docker exec <contenedor-postgres> pg_dump -U app -Fc planificaciones > backup-$(date +%Y%m%d).dump

# Restore (a una base vacía)
docker exec -i <contenedor-postgres> pg_restore -U app -d planificaciones --clean --if-exists < backup-20260101.dump
```

Automatiza el dump diario (cron) y copia el archivo resultante a almacenamiento externo (otro servidor, S3, etc.).

### MinIO

Los PDFs viven únicamente en el bucket `planning-documents`; nunca en PostgreSQL. Usa `mc mirror` para sincronizarlo a otra ubicación:

```bash
# Una vez: registrar el destino del backup (otro MinIO, o cualquier endpoint S3)
mc alias set backup-target https://backup-endpoint ACCESS_KEY SECRET_KEY

# Backup incremental
mc mirror --overwrite local/planning-documents backup-target/planning-documents

# Restore (en sentido inverso)
mc mirror --overwrite backup-target/planning-documents local/planning-documents
```

Corre el `mirror` de backup con la misma frecuencia que el dump de Postgres — un backup de uno sin el otro deja el sistema en un estado inconsistente (planificaciones en la base de datos sin su PDF, o archivos huérfanos sin fila asociada).
