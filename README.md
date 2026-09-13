# mi.pizarra

Sistema personal de tareas, pendientes y notas, autohosteado con
[Vikunja](https://vikunja.io/) (opensource, AGPL-3.0).

Vikunja incluye API + interfaz web en una sola imagen Docker, listas de
tareas, tableros Kanban, fechas límite, recordatorios, etiquetas,
proyectos compartidos y una app de escritorio/móvil opcional que se
conecta a esta misma instancia.

## Requisitos

- Docker y Docker Compose instalados.

## Puesta en marcha

1. Copiar el archivo de variables de entorno y completarlo:

   ```bash
   cp .env.example .env
   ```

   Editar `.env` y definir:
   - `VIKUNJA_DB_PASSWORD`: contraseña de la base de datos.
   - `VIKUNJA_JWT_SECRET`: clave secreta aleatoria (`openssl rand -hex 32`).
   - `VIKUNJA_PUBLIC_URL`: URL pública donde vas a acceder (o dejar el
     valor local si solo lo usás en tu red).
   - `VIKUNJA_PORT`: puerto local donde se expone (por defecto `3456`).

2. Levantar el stack:

   ```bash
   docker compose up -d
   ```

3. Abrir `http://localhost:3456` (o el puerto/host que hayas configurado)
   y crear tu usuario administrador desde la pantalla de registro.

4. Una vez creado tu usuario, si es de uso personal, deshabilitá el
   registro público (ya viene deshabilitado por defecto en este
   `docker-compose.yml` vía `VIKUNJA_SERVICE_ENABLEREGISTRATION=false`).

## Datos y backups

Los datos persisten en dos volúmenes Docker:

- `db-data`: base de datos Postgres (tareas, proyectos, usuarios, etc).
- `vikunja-data`: archivos adjuntos subidos a las tareas.

Para hacer un backup completo:

```bash
docker compose exec db pg_dump -U vikunja vikunja > backup-$(date +%F).sql
docker run --rm -v mi_pizarra_vikunja-data:/data -v "$PWD":/backup \
  alpine tar czf /backup/vikunja-files-$(date +%F).tar.gz -C /data .
```

(El nombre exacto del volumen de archivos puede variar según el nombre
del proyecto Docker Compose; verificalo con `docker volume ls`.)

## Actualizar

```bash
docker compose pull
docker compose up -d
```

## Acceso remoto

Para acceder desde fuera de tu red local, lo más simple es poner un
reverse proxy con TLS delante (Caddy, Traefik o Nginx + Let's Encrypt)
apuntando al puerto configurado en `VIKUNJA_PORT`, y actualizar
`VIKUNJA_PUBLIC_URL` en `.env` con el dominio real.
