# Multi-instancia Odoo con Docker

## Red Docker

Todos los contenedores comparten una sola red bridge con nombre fijo. Esto permite que Nginx y Odoo se comuniquen por
nombre de contenedor.

```bash
docker network create odoo_network
```

## PostgreSQL

Un solo contenedor. Las bases de datos de cada instancia viven aquí.

> **Producción:** cambia `POSTGRES_PASSWORD` y `db_password` por credenciales seguras.

> **Permisos:** se usa un named volume (`postgres_data`), Docker gestiona los permisos automáticamente.
> Si en algún momento cambias a un bind mount (carpeta del host), el proceso `postgres` corre con `uid=999, gid=999`
> y necesitarás `chown -R 999:999` en esa carpeta.

```bash
docker run -d \
  --name odoo_db \
  --network odoo_network \
  --restart unless-stopped \
  -e POSTGRES_USER=odoo \
  -e POSTGRES_PASSWORD=odoo \
  -e POSTGRES_DB=postgres \
  -v postgres_data:/var/lib/postgresql/data \
  -p 127.0.0.1:5432:5432 \
  postgres:17
```

## Estructura de carpetas

```
/opt/odoo/
└── empresa01/
    ├── data/        ← archivos de Odoo (adjuntos, sesiones, etc.)
    ├── addons/      ← módulos personalizados
    ├── logs/        ← aquí verás el log desde tu servidor
    └── odoo.conf
```

Antes de lanzar el contenedor asegúrate de crear las carpetas y darles los permisos correctos.

**UIDs del contenedor `odoo:18`** (fijos por imagen, no cambian al recrear el contenedor):

| Usuario | UID | GID |
|---------|-----|-----|
| odoo    | 100 | 101 |

> Puedes verificarlo en cualquier momento con `docker exec <nombre> id`

- `data` y `logs` necesitan `chown 100:101` porque Odoo escribe en ellas
- `addons` solo necesita existir — se monta como `:ro`, Odoo no escribe en ella
- `odoo.conf` no necesita permisos especiales — también se monta como `:ro`

```bash
mkdir -p /opt/odoo/empresa01/data
chown -R 100:101 /opt/odoo/empresa01/data

mkdir -p /opt/odoo/empresa01/logs
chown -R 100:101 /opt/odoo/empresa01/logs

mkdir -p /opt/odoo/empresa01/addons
```

Crea el archivo de configuración en la ruta del host que se monta en el contenedor:

```bash
nano /opt/odoo/empresa01/odoo.conf
```

```ini
[options]
addons_path = /usr/lib/python3/dist-packages/odoo/addons,/mnt/extra-addons

# Solo puede ver su propia DB — clave para el aislamiento
db_name = empresa01

db_host = odoo_db
db_port = 5432
db_user = odoo
db_password = odoo

list_db = False
admin_passwd = empresa01_password

gevent_port = 8072
without_demo = True
proxy_mode = True

logfile = /var/log/odoo/odoo.log
logrotate = True
log_level = info
```

## Odoo

```bash
docker run -d \
  --name empresa01 \
  --network odoo_network \
  --restart unless-stopped \
  -e HOST=odoo_db \
  -e PORT=5432 \
  -e USER=odoo \
  -e PASSWORD=odoo \
  -v /opt/odoo/empresa01/data:/var/lib/odoo \
  -v /opt/odoo/empresa01/logs:/var/log/odoo \
  -v /opt/odoo/empresa01/odoo.conf:/etc/odoo/odoo.conf:ro \
  -v /opt/odoo/empresa01/addons:/mnt/extra-addons:ro \
  -p 8069:8069 \
  -p 8072:8072 \
  odoo:18
```

```bash
docker stop empresa01 && docker rm empresa01

# Verificar que levantó correctamente
docker logs -f empresa01
```

> **Múltiples instancias:** el puerto del contenedor siempre es `8069/8072`, solo cambia el puerto del host (izquierda):
> ```
> empresa01 → -p 8069:8069  -p 8072:8072
> empresa02 → -p 8070:8069  -p 8073:8072
> empresa03 → -p 8071:8069  -p 8074:8072
> ```
> Recuerda también actualizar `gevent_port` en cada `odoo.conf` para que coincida con el puerto longpolling asignado.

## Nginx

Instalado directamente en el host (no en Docker), para que sea más fácil de gestionar los certificados SSL y
agregar/quitar virtual hosts.

**Estructura:**

```
/etc/nginx/
├── nginx.conf
└── conf.d/
    ├── empresa01.conf
    ├── empresa02.conf
    └── empresa03.conf
```

### Instalación

```bash
sudo apt update
sudo apt install nginx
```

### Caché de archivos estáticos

Para activar la caché de estáticos de Odoo, define la zona en el bloque `http` de `/etc/nginx/nginx.conf`:

```bash
sudo nano /etc/nginx/nginx.conf
```

Añade dentro del bloque `http { ... }`, antes de la línea `include /etc/nginx/conf.d/*.conf;`:

```nginx
proxy_cache_path /var/cache/nginx/odoo
    levels=1:2
    keys_zone=odoo_cache:10m
    max_size=1g
    inactive=60m
    use_temp_path=off;
```

- `keys_zone=odoo_cache:10m` — nombre de la zona (referenciado desde los virtual hosts) y tamaño de la tabla de claves en RAM.
- `max_size=1g` — límite de espacio en disco para la caché.
- `inactive=60m` — archivos no solicitados en ese tiempo se eliminan automáticamente.

Crea la carpeta y ajusta los permisos:

```bash
sudo mkdir -p /var/cache/nginx/odoo
sudo chown www-data:www-data /var/cache/nginx/odoo
```

### Virtual host

Crea el archivo de configuración del virtual host:

```bash
sudo nano /etc/nginx/conf.d/empresa01.conf
```

Ejemplo de virtual host (`/etc/nginx/conf.d/empresa01.conf`):

```nginx
upstream empresa01 {
    server 127.0.0.1:8069;
}

upstream empresa01_websocket {
    server 127.0.0.1:8072;
}

server {
    listen 80;
    server_name empresa01.ejemplo.com;

    # Redirigir a HTTPS
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name empresa01.ejemplo.com;

    # Los certificados deben generarse previamente (Let's Encrypt, self-signed, o CA propia).
    # Ajusta estas rutas según donde los hayas almacenado.
    ssl_certificate     /ruta/a/fullchain.pem;
    ssl_certificate_key /ruta/a/privkey.pem;

    access_log /var/log/nginx/empresa01.access.log;
    error_log  /var/log/nginx/empresa01.error.log;

    proxy_read_timeout 720s;
    proxy_connect_timeout 720s;
    proxy_send_timeout 720s;

    proxy_set_header X-Forwarded-Host $host;
    proxy_set_header X-Forwarded-For  $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_set_header X-Real-IP $remote_addr;

    location /websocket {
        proxy_pass http://empresa01_websocket;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }

    location / {
        proxy_pass http://empresa01;
    }

    location ~* /web/static/ {
        proxy_cache       odoo_cache;   # zona definida en nginx.conf (ver sección "Caché de archivos estáticos")
        proxy_cache_valid 200 90m;
        proxy_buffering   on;
        proxy_pass http://empresa01;
    }

    gzip on;
    gzip_types text/css text/plain application/xml application/json application/javascript;
}
```

Verifica la sintaxis y aplica los cambios:

```bash
sudo nginx -t
sudo systemctl reload nginx
```

## Backup

```bash
docker exec odoo_db pg_dump -U odoo empresa01 > /opt/backups/empresa1_$(date +%Y%m%d).sql
```

```bash
# Primero asegúrate que la DB existe (o créala)
docker exec -it odoo_db psql -U odoo -c "CREATE DATABASE empresa01 OWNER odoo;"

# Luego restaurar
docker exec -i odoo_db psql -U odoo empresa01 < /opt/backups/empresa1_20260224.sql
```

```bash
# Backup comprimido (recomendado para producción)
docker exec odoo_db pg_dump -U odoo empresa01 | gzip > /opt/backups/empresa1_$(date +%Y%m%d).sql.gz

# Restaurar desde el comprimido
gunzip -c /opt/backups/empresa1_20260224.sql.gz | docker exec -i odoo_db psql -U odoo empresa01
```

## SSH Tunnel para PostgreSQL

El puerto 5432 está vinculado a `127.0.0.1` en el servidor, por lo que no es accesible desde internet. Para conectarte
desde tu máquina local se usa un túnel SSH que redirige un puerto local hacia el servidor.

**Requisito:** tener acceso SSH al servidor.

### Abrir el túnel

```bash
ssh -L 5432:127.0.0.1:5432 usuario@tu-servidor.com
```

Mientras esta sesión SSH esté activa, cualquier conexión a `localhost:5432` en tu PC se redirige al PostgreSQL del servidor.

### Conectar con un cliente (psql, DBeaver, pgAdmin, etc.)

| Campo    | Valor       |
|----------|-------------|
| Host     | localhost   |
| Puerto   | 5432        |
| Usuario  | odoo        |
| Password | odoo        |

### Túnel en segundo plano (sin sesión interactiva)

```bash
ssh -fNL 5432:127.0.0.1:5432 usuario@tu-servidor.com
```

- `-f` — pasa el proceso a segundo plano
- `-N` — no ejecuta ningún comando remoto, solo mantiene el túnel

Para cerrar el túnel:

```bash
# Buscar el proceso
ps aux | grep "5432:127.0.0.1:5432"

# Matar el proceso con su PID
kill <PID>
```
