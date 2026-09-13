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

## Exponerlo con Cloudflare Tunnel

Para acceder desde fuera de tu red sin abrir puertos en el router, se
puede usar [Cloudflare Tunnel](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/).
Hay dos formas:

### Opción A: túnel rápido de prueba (sin cuenta, URL temporal)

Útil para probar ya mismo. La URL cambia cada vez que lo reiniciás, no
sirve para dejarlo fijo.

```bash
docker run --rm cloudflare/cloudflared:latest tunnel --url http://host.docker.internal:3456
```

Te va a imprimir una URL tipo `https://algo-random.trycloudflare.com`
en los logs. En Linux, si `host.docker.internal` no resuelve, corré
`cloudflared` directo en el host apuntando a `http://localhost:3456`
en lugar de usar Docker para este comando.

### Opción B: túnel permanente con URL fija (recomendado para dejarlo)

Requiere una cuenta gratuita de Cloudflare (no hace falta tener un
dominio propio, Cloudflare te puede dar un subdominio tipo
`*.cfargotunnel.com`, o podés usar uno tuyo si lo tenés).

1. Entrar a [Cloudflare Zero Trust](https://one.dash.cloudflare.com/) →
   **Networks → Tunnels → Create a tunnel** → elegir "Cloudflared".
2. Ponerle un nombre (ej. `mi-pizarra`) y copiar el **token** que te
   muestra (es un string largo).
3. Pegarlo en `.env` como `CLOUDFLARE_TUNNEL_TOKEN`.
4. En la misma pantalla de configuración del túnel, agregar un
   **Public Hostname**:
   - Subdominio/dominio: el que quieras (propio o el que te ofrezca
     Cloudflare).
   - Service: `HTTP` → `vikunja:3456` (nombre del servicio en la red
     interna de Docker Compose).
5. Levantar el túnel junto con el resto del stack:

   ```bash
   docker compose --profile tunnel up -d
   ```

6. Actualizar `VIKUNJA_PUBLIC_URL` en `.env` con la URL pública elegida
   y reiniciar: `docker compose up -d vikunja`.

Con esto la URL queda fija, el túnel se reconecta solo si se cae, y no
necesitás abrir ningún puerto en tu red.

## Alternativa: reverse proxy propio

Si preferís no depender de Cloudflare, otra opción es poner un reverse
proxy con TLS delante (Caddy, Traefik o Nginx + Let's Encrypt)
apuntando al puerto configurado en `VIKUNJA_PORT`, con el puerto 443
abierto en tu router.
