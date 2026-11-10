# Multi-instancia Odoo con Docker

Varias instancias de Odoo (una por empresa) en un mismo servidor, con un solo PostgreSQL y Nginx en el host como proxy
inverso con HTTPS.

```
Internet ──443──► Nginx (host) ──127.0.0.1:8069/8072──► empresa01 (odoo:18) ──┐
                               ──127.0.0.1:8070/8073──► empresa02 (odoo:18) ──┼── odoo_network ──► odoo_db (postgres:17)
                                                                               ┘        (sin puertos publicados)
```

> **Regla de oro:** desde Internet solo deben responder los puertos 22, 80 y 443. PostgreSQL y los puertos de Odoo
> nunca se publican en `0.0.0.0`. Un PostgreSQL expuesto con contraseña débil termina, en horas.

## Red Docker

Todos los contenedores comparten una sola red bridge con nombre fijo. Esto permite que Odoo se conecte a PostgreSQL por
nombre de contenedor (`db_host = odoo_db`).

```bash
docker network create odoo_network
```

## PostgreSQL

Un solo contenedor. Las bases de datos de cada instancia viven aquí.

> **Sin `-p`:** PostgreSQL no publica ningún puerto en el host. Odoo llega a él por `odoo_network`. Para administrarlo,
> ver [Acceso a PostgreSQL](#acceso-a-postgresql).

> **Permisos:** se usa un named volume (`postgres_data`), Docker gestiona los permisos automáticamente.
> Si en algún momento cambias a un bind mount (carpeta del host), el proceso `postgres` corre con `uid=999, gid=999`
> y necesitarás `chown -R 999:999` en esa carpeta.

```bash
docker run -d \
  --name odoo_db \
  --network odoo_network \
  --restart unless-stopped \
  -e POSTGRES_USER=odoo \
  -e POSTGRES_PASSWORD='<contraseña generada>' \
  -e POSTGRES_DB=postgres \
  -v postgres_data:/var/lib/postgresql/data \
  postgres:17
```

> **`POSTGRES_PASSWORD` solo se usa al inicializar un volumen vacío.** Si después cambias la contraseña con
> `ALTER ROLE`, recrea el contenedor (mismo volumen) con el valor nuevo: el antiguo queda visible en `docker inspect`.

> **`pg_hba.conf`:** la imagen deja `host all all all scram-sha-256`. Mientras `odoo_db` no publique puertos no hace
> falta tocarlo: solo llegan los contenedores de `odoo_network` y el propio host, los mismos que permitiría limitarlo a
> la subred. La protección real es no publicar el puerto, el firewall y una contraseña fuerte.

### Rol por instancia (recomendado)

`POSTGRES_USER` crea un **superusuario**. Si todas las instancias usan ese rol, cualquiera de ellas puede leer las bases
de las otras y, si alguien obtiene la contraseña, ejecutar comandos en el contenedor. Para instalaciones nuevas, crea un
rol sin superusuario por instancia y deja el superusuario solo para administrar:

```bash
docker exec -it odoo_db psql -U odoo -c "CREATE ROLE empresa01 LOGIN CREATEDB PASSWORD '<contraseña generada>';"
docker exec -it odoo_db psql -U odoo -c "CREATE DATABASE empresa01 OWNER empresa01;"
```

y en el `odoo.conf` de esa instancia `db_user = empresa01`.

> En una instalación existente donde `odoo` es dueño de todo, no le quites `SUPERUSER` sin crear antes otro
> superusuario: te quedarías sin ninguno.

## Estructura de carpetas

```
/opt/odoo/
└── empresa01/
    ├── data/        ← archivos de Odoo (filestore/<db>, sesiones)
    ├── addons/      ← módulos personalizados (enterprise/, custom/)
    ├── logs/        ← aquí verás el log desde tu servidor
    └── odoo.conf
/opt/backups/        ← copias (ver sección Backup)
```

Antes de lanzar el contenedor asegúrate de crear las carpetas y darles los permisos correctos.

**UIDs del contenedor `odoo:18`** (fijos por imagen, no cambian al recrear el contenedor):

| Usuario | UID | GID |
|---------|-----|-----|
| odoo    | 100 | 101 |

> Puedes verificarlo en cualquier momento con `docker exec <nombre> id`. En el host el GID 101 puede aparecer con otro
> nombre (por ejemplo `lxd`); es normal.

- `data` y `logs` necesitan `chown 100:101` porque Odoo escribe en ellas
- `addons` solo necesita existir — se monta como `:ro`, Odoo no escribe en ella
- `odoo.conf` contiene contraseñas: `root:101` con permisos `640`, legible por Odoo pero no por otros usuarios del host

```bash
mkdir -p /opt/odoo/empresa01/{data,logs,addons}
chown -R 100:101 /opt/odoo/empresa01/data /opt/odoo/empresa01/logs
```

Crea el archivo de configuración en la ruta del host que se monta en el contenedor:

```bash
nano /opt/odoo/empresa01/odoo.conf
chown root:101 /opt/odoo/empresa01/odoo.conf
chmod 640 /opt/odoo/empresa01/odoo.conf
```

```ini
[options]
addons_path = /usr/lib/python3/dist-packages/odoo/addons,/mnt/extra-addons/enterprise,/mnt/extra-addons/custom

# Solo puede ver su propia DB — clave para el aislamiento
db_name = empresa01

db_host = odoo_db
db_port = 5432
db_user = odoo
db_password = <contraseña de PostgreSQL>

# Bloquea el gestor de bases de datos (web y RPC: crear, borrar, backup, restaurar), así que admin_passwd no se usa
list_db = False

# Siempre 8072: es el puerto DENTRO del contenedor, igual en todas las instancias
gevent_port = 8072
without_demo = True
proxy_mode = True

logfile = /var/log/odoo/odoo.log
log_level = info
```

> **Si activas `list_db = True`** (por ejemplo, para restaurar desde la web), define antes un `admin_passwd` fuerte:
> sin él, la master password por defecto es `admin`. Vuelve a `False` al terminar.

## Odoo

```bash
docker run -d \
  --name empresa01 \
  --network odoo_network \
  --restart unless-stopped \
  -v /opt/odoo/empresa01/data:/var/lib/odoo/.local/share/Odoo \
  -v /opt/odoo/empresa01/logs:/var/log/odoo \
  -v /opt/odoo/empresa01/odoo.conf:/etc/odoo/odoo.conf:ro \
  -v /opt/odoo/empresa01/addons:/mnt/extra-addons:ro \
  -p 127.0.0.1:8069:8069 \
  -p 127.0.0.1:8072:8072 \
  odoo:18
```

- **`127.0.0.1:`** delante de cada puerto: solo Nginx (en el mismo host) llega a Odoo. Sin él, Odoo queda accesible por
  HTTP desde Internet saltándose Nginx, y Docker **ignora ufw/firewalld** para los puertos publicados.
- **Sin `-e HOST/PORT/USER/PASSWORD`:** la conexión a la base de datos ya está en `odoo.conf`, que tiene prioridad. Las
  variables solo dejan la contraseña visible en `docker inspect` y se quedan desfasadas al rotarla.

> **Múltiples instancias:** el puerto del contenedor siempre es `8069/8072` y `gevent_port` siempre `8072`; solo cambia
> el puerto del host (izquierda):
> ```
> empresa01 → -p 127.0.0.1:8069:8069  -p 127.0.0.1:8072:8072
> empresa02 → -p 127.0.0.1:8070:8069  -p 127.0.0.1:8073:8072
> empresa03 → -p 127.0.0.1:8071:8069  -p 127.0.0.1:8074:8072
> ```

### Verificar que levantó correctamente

```bash
docker logs -f empresa01
tail -f /opt/odoo/empresa01/logs/odoo.log
```

### Recrear el contenedor

Necesario al cambiar puertos, montajes o la imagen. Los datos están en el host y en `postgres_data`, no se pierden.

```bash
docker stop empresa01 && docker rm empresa01
# volver a ejecutar el docker run de arriba
```

> La imagen declara `VOLUME /var/lib/odoo`, así que cada contenedor crea un volumen anónimo que queda huérfano al
> recrearlo. Límpialos con `docker volume prune` (solo borra volúmenes anónimos sin uso; revisa antes
`docker volume ls`).

### Actualizar módulos

```bash
docker exec empresa01 odoo -c /etc/odoo/odoo.conf -d empresa01 -u <modulo> --stop-after-init --no-http
docker restart empresa01
```

### Actualizar la imagen de Odoo

```bash
docker pull odoo:18
# hacer backup, recrear el contenedor y revisar el log
```

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

# Quitar el sitio por defecto
sudo rm /etc/nginx/sites-enabled/default
```

### Ajustes globales y caché de archivos estáticos

Edita el bloque `http { ... }` de `/etc/nginx/nginx.conf`:

```bash
sudo nano /etc/nginx/nginx.conf
```

Cambia `server_tokens` (no mostrar la versión de Nginx) y añade, antes de la línea `include /etc/nginx/conf.d/*.conf;`:

```nginx
server_tokens off;

# Por defecto es 1m: fallan adjuntos, importaciones y restauraciones más grandes
client_max_body_size 100m;

proxy_cache_path /var/cache/nginx/odoo
    levels=1:2
    keys_zone=odoo_cache:10m
    max_size=1g
    inactive=60m
    use_temp_path=off;
```

- `keys_zone=odoo_cache:10m` — nombre de la zona (referenciado desde los virtual hosts) y tamaño de la tabla de claves
  en RAM.
- `max_size=1g` — límite de espacio en disco para la caché.
- `inactive=60m` — archivos no solicitados en ese tiempo se eliminan automáticamente.

Crea la carpeta y ajusta los permisos:

```bash
sudo mkdir -p /var/cache/nginx/odoo
sudo chown www-data:www-data /var/cache/nginx/odoo
```

### Certificados

- Un dominio: [ngnix_ssl.md](ngnix_ssl.md) (Certbot).
- Wildcard o varios dominios con renovación automática: [ngnix_ssl_wildcard.md](ngnix_ssl_wildcard.md) (acme.sh).

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
    http2 on;
    server_name empresa01.ejemplo.com;

    # Los certificados deben generarse previamente (ver sección Certificados).
    ssl_certificate     /ruta/a/fullchain.pem;
    ssl_certificate_key /ruta/a/privkey.pem;

    add_header Strict-Transport-Security "max-age=31536000" always;

    access_log /var/log/nginx/empresa01.access.log;
    error_log  /var/log/nginx/empresa01.error.log;

    proxy_read_timeout 720s;
    proxy_connect_timeout 720s;
    proxy_send_timeout 720s;

    proxy_set_header X-Forwarded-Host $host;
    proxy_set_header X-Forwarded-For  $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_set_header X-Real-IP $remote_addr;

    # Gestor de bases de datos: nunca accesible desde Internet
    location ~* ^/web/database {
        deny all;
    }

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
        proxy_cache       odoo_cache;   # zona definida en nginx.conf (ver "Ajustes globales y caché")
        proxy_cache_valid 200 90m;
        proxy_buffering   on;
        proxy_pass http://empresa01;
    }

    gzip on;
    gzip_types text/css text/plain application/xml application/json application/javascript;
}
```

> **Un dominio, un archivo.** Si el mismo `server_name` aparece en dos virtual hosts, Nginx usa el primero e ignora el
> otro con el aviso `conflicting server name`. Para servir varios dominios con la misma instancia, ponlos juntos en el
> `server_name` de un solo archivo.

Verifica la sintaxis y aplica los cambios (no debe aparecer ningún `warn`):

```bash
sudo nginx -t
sudo systemctl reload nginx
```

## Firewall

Docker inserta sus propias reglas de iptables y **se salta ufw y firewalld**: un `-p 5432:5432` queda abierto a
Internet aunque ufw diga lo contrario. Aunque todos los puertos estén en `127.0.0.1`, añade un firewall que filtre
también el tráfico hacia los contenedores, para IPv4 e IPv6:

```bash
# Tráfico hacia el host por la interfaz pública: solo 22, 80, 443
iptables -N PUBLIC-IN
iptables -A PUBLIC-IN -m conntrack --ctstate RELATED,ESTABLISHED -j RETURN
iptables -A PUBLIC-IN -p icmp -j RETURN
iptables -A PUBLIC-IN -p udp --sport 67 --dport 68 -j RETURN      # DHCP
iptables -A PUBLIC-IN -p tcp -m multiport --dports 22,80,443 -j RETURN
iptables -A PUBLIC-IN -j DROP
iptables -I INPUT -i eth0 -j PUBLIC-IN

# Conexiones nuevas desde Internet hacia los contenedores: descartar
iptables -I DOCKER-USER -i eth0 -m conntrack --ctstate NEW -j DROP

# Repetir con ip6tables (DHCPv6: --sport 547 --dport 546, ICMP: -p ipv6-icmp)
```

- Con IPv6, `docker-proxy` atiende los puertos publicados por la cadena **INPUT**, no por `DOCKER-USER`: hacen falta las
  dos.
- Guarda las reglas en un script (`/usr/local/sbin/firewall-public.sh`) y aplícalo al arrancar con un servicio systemd
  `Type=oneshot`, `RemainAfterExit=yes`, `After=docker.service`.
- Antes de aplicarlo por SSH, programa una vuelta atrás por si te quedas fuera:
  `systemd-run --on-active=10min /usr/local/sbin/firewall-rollback.sh`.
- Si el proveedor tiene firewall propio (Hetzner Cloud Firewall, Security Groups…), configúralo también con la misma
  política.

Verifica **desde fuera del servidor** que solo responden 22, 80 y 443:

```bash
nmap -Pn -p 22,80,443,5432,8069,8070,8072,8073 <IP-del-servidor>
```

## SSH

Acceso solo con llave: [root_ssh_key.md](root_ssh_key.md). Comprueba la configuración efectiva (si otro archivo de
`sshd_config.d` se carga antes, su valor gana):

```bash
sshd -T | grep -E "^(passwordauthentication|kbdinteractiveauthentication|permitrootlogin) "
```

## Rotación de logs

Ver [odoo_logrotate.md](odoo_logrotate.md), con la ruta del host y `copytruncate`:

```bash
/opt/odoo/*/logs/odoo.log {
    daily
    rotate 30
    compress
    delaycompress
    dateext
    copytruncate
    missingok
    notifempty
}
```

## Backup

Un backup de Odoo tiene **dos partes**: la base de datos y el filestore (`data/filestore/<db>`, los adjuntos). Sin el
filestore, la base restaurada muestra adjuntos e imágenes rotos.

### Script diario

`/usr/local/sbin/odoo-backup.sh`:

```bash
#!/bin/bash
set -euo pipefail
D=/opt/backups/$(date +%F)
mkdir -p "$D"; chmod 700 /opt/backups "$D"

# Copia completa del clúster (roles incluidos)
docker exec odoo_db pg_dumpall -U odoo | gzip > "$D/pg_dumpall.sql.gz"

# Por instancia: base de datos (formato custom) + filestore
for conf in /opt/odoo/*/odoo.conf; do
  inst=$(basename "$(dirname "$conf")")
  db=$(sed -n 's/^db_name *= *//p' "$conf")
  docker exec odoo_db pg_dump -U odoo -Fc "$db" > "$D/$db.dump"
  tar -czf "$D/$db.filestore.tar.gz" -C "/opt/odoo/$inst/data/filestore" "$db"
done

# Retención: 14 días
find /opt/backups -mindepth 1 -maxdepth 1 -type d -mtime +14 -exec rm -rf {} +
```

```bash
chmod 700 /usr/local/sbin/odoo-backup.sh
echo '30 2 * * * root /usr/local/sbin/odoo-backup.sh >> /var/log/odoo-backup.log 2>&1' > /etc/cron.d/odoo-backup
```

> **Copia fuera del servidor:** un backup que solo está en el mismo servidor no sirve si se pierde el disco o hay
> ransomware. Sincroniza `/opt/backups` a otro sitio (Hetzner Storage Box, S3…) con `rclone` o `rsync`.

### Restaurar una instancia

```bash
docker stop empresa01

# Base de datos
docker exec odoo_db psql -U odoo -c "DROP DATABASE IF EXISTS empresa01;"
docker exec odoo_db psql -U odoo -c "CREATE DATABASE empresa01 OWNER odoo;"
docker exec -i odoo_db pg_restore -U odoo -d empresa01 --no-owner --role=odoo < /opt/backups/2026-10-24/empresa01.dump

# Filestore
rm -rf /opt/odoo/empresa01/data/filestore/empresa01
tar -xzf /opt/backups/2026-10-24/empresa01.filestore.tar.gz -C /opt/odoo/empresa01/data/filestore
chown -R 100:101 /opt/odoo/empresa01/data/filestore/empresa01

docker start empresa01
```

> **Prueba la restauración** de vez en cuando en una base nueva (`empresa01_restore`): un backup que nunca se restauró
> no está comprobado.

## Acceso a PostgreSQL

PostgreSQL no publica puertos en el host.

### Consola en el servidor

```bash
docker exec -it odoo_db psql -U odoo -d empresa01
```

### Cliente gráfico (DBeaver, pgAdmin…) con túnel SSH

El túnel apunta a la IP del contenedor dentro de `odoo_network`:

```bash
# En el servidor: IP del contenedor (cambia si se recrea)
docker inspect -f '{{.NetworkSettings.Networks.odoo_network.IPAddress}}' odoo_db

# En tu PC
ssh -L 5432:<IP-del-contenedor>:5432 usuario@tu-servidor.com
```

Mientras esta sesión SSH esté activa, cualquier conexión a `localhost:5432` en tu PC se redirige al PostgreSQL del
servidor.

| Campo    | Valor                           |
|----------|---------------------------------|
| Host     | localhost                       |
| Puerto   | 5432                            |
| Usuario  | odoo (o el rol de la instancia) |
| Password | la del gestor de contraseñas    |

### Túnel en segundo plano (sin sesión interactiva)

```bash
ssh -fNL 5432:<IP-del-contenedor>:5432 usuario@tu-servidor.com
```

- `-f` — pasa el proceso a segundo plano
- `-N` — no ejecuta ningún comando remoto, solo mantiene el túnel

Para cerrar el túnel:

```bash
# Buscar el proceso
ps aux | grep "ssh -fNL 5432"

# Matar el proceso con su PID
kill <PID>
```

## Verificación final

| Comprobación             | Comando                                                                            | Esperado                                                      |
|--------------------------|------------------------------------------------------------------------------------|---------------------------------------------------------------|
| Puertos publicados       | `docker ps --format "{{.Names}} {{.Ports}}"`                                       | `odoo_db` sin puertos; Odoo solo `127.0.0.1:`                 |
| Puertos desde fuera      | `nmap -Pn -p 22,80,443,5432,8069,8072 <IP>` (desde otra red)                       | Solo 22, 80, 443 abiertos                                     |
| Contraseñas en variables | `docker inspect <contenedor> --format '{{range .Config.Env}}{{println .}}{{end}}'` | Sin `PASSWORD` en Odoo; `POSTGRES_PASSWORD` igual a la actual |
| Nginx                    | `sudo nginx -t`                                                                    | Sin `warn`                                                    |
| Backup                   | `ls -la /opt/backups/$(date +%F)`                                                  | `.dump` y `.filestore.tar.gz` por instancia                   |
| Logs                     | `ls -la /opt/odoo/*/logs`                                                          | Archivos rotados por fecha                                    |
| SSH                      | `sshd -T \| grep passwordauthentication`                                           | `no`                                                          |
