# Multiambiente de desarrollo Odoo con `odev`

Guía para montar en un servidor Linux decenas de repositorios y varias versiones de Odoo, sin Docker y sin un servicio por
ambiente. Cada ambiente es un repositorio con su conf, su BD y su filestore. Está apagado hasta que lo inicias, y al reiniciar
el servidor no arranca ninguno.

> **Solo para desarrollo y pruebas.** El tráfico viaja por HTTP sin cifrar y `odev` se ejecuta como `root`. No lo uses en
> producción. Antes de restaurar datos de clientes, ten en cuenta que viajarán sin cifrar.

## Cómo funciona

```text
navegador ── HTTP :80 ──► nginx ── upstream fijo ──► 127.0.0.1:8069
                                                          ▲
                                                          │ odev start levanta ahí el ambiente
                                                          │
/opt/odoo/repos/mi-repo ── odev ──► systemd-run (unidad temporal odev-mi-repo.service)
        │                               │
        │ el nombre de la carpeta       │ usa
        ▼ identifica el ambiente        ▼
/opt/odoo/confs/mi-repo.conf ──► src/<versión>, venvs/<versión>, BD mi-repo, data/mi-repo
```

| Decisión | Motivo |
|---|---|
| Un conf por carpeta (`confs/<nombre>.conf`) | `odev` lo deduce de la carpeta actual, así que no hace falta un core de Odoo modificado |
| `db_name` fijo en cada conf | Evita usar por error la BD de otro ambiente |
| Unidades temporales de systemd | Dan límite de memoria y parada limpia, y no quedan activadas al reiniciar |
| Un solo ambiente encendido, siempre en el puerto 8069 | nginx no cambia nunca: `odev` solo administra Odoo, y no hacen falta subdominios ni DNS |
| Core compartido por versión | Cada ambiente ocupa poco: su repo, su BD y su filestore |
| `git fetch --depth 1` sobre un almacén bare | Sin historial y con objetos compartidos entre versiones |
| `uv` para Python y venvs | Cada versión de Odoo usa su Python, sin depender del que trae el sistema |

## Requisitos

- Linux con systemd, `sudo` y acceso `root`. Probado solo como `root`.
- Debian o Ubuntu para los comandos de esta guía; en otra distribución cambian los paquetes y las rutas.
- Una llave SSH de `root` con acceso de GitHub a `odoo/odoo`, `odoo/enterprise` y `odoo/design-themes`, y a tus repositorios.
- `odoo/enterprise` es privado y `odev` lo exige. Sin acceso, consulta [Solo Community](#solo-community).
- `odev` solo administra Odoo: no toca nginx ni fail2ban. Ambos son opcionales; sin nginx, el ambiente encendido solo es
  accesible en `127.0.0.1:8069`.

**Probado en:** Ubuntu 26.04.1 x86_64 con 4 CPU, 8 GB de RAM y 96 GB de disco, Postgres 18, `uv` 0.13.7, Odoo 19.0 (Python
3.12) y Odoo 20.0 (Python 3.14).

**No probado:** esta instalación completa sobre una máquina limpia (la guía sale de un servidor ya montado), Odoo 16.0 a 18.0,
otras distribuciones, WSL, ARM y usuarios distintos de `root`.

### Capacidad de referencia

Medido con dos versiones instaladas:

| Elemento | Tamaño |
|---|---|
| `src/` (dos cores y su almacén) | 5,3 GB |
| `venvs/` (dos venvs) | 301 MB |
| `python/` (dos intérpretes) | 110 MB |

El gasto real lo marcan las BD y los filestores. Con 8 GB de RAM, `odev` avisa a partir de tres ambientes encendidos o si
quedan menos de 1,5 GB disponibles, pero no lo impide.

## Instalación

Ejecuta los pasos como `root` y en este orden: fail2ban necesita los logs que crea nginx.

### 1. Instalar paquetes

```bash
apt-get update
apt-get install -y git curl python3 python3-dev python3-venv build-essential pkg-config \
  libffi-dev libjpeg-dev libldap-dev libpq-dev libsasl2-dev libssl-dev libxml2-dev libxmlsec1-dev libxslt1-dev zlib1g-dev \
  fontconfig libx11-6 libxext6 libxrender1 xfonts-75dpi xfonts-base \
  postgresql postgresql-client nginx fail2ban
```

**wkhtmltopdf** lo usa Odoo para los PDF. Instala la build con Qt parcheado de
[wkhtmltopdf/packaging](https://github.com/wkhtmltopdf/packaging/releases), porque la de `apt` no trae ese Qt y los PDF
pueden salir sin encabezados ni pies. En el servidor de referencia, los binarios 0.12.6.1 están en `/usr/local/bin`.

**Chrome** solo hace falta para los tests con tours (`HttpCase`):

```bash
curl -fsSLO https://dl.google.com/linux/direct/google-chrome-stable_current_amd64.deb
apt-get install -y ./google-chrome-stable_current_amd64.deb
```

### 2. Instalar `uv`

```bash
cd "$(mktemp -d)"
B=https://github.com/astral-sh/uv/releases/latest/download/uv-x86_64-unknown-linux-gnu.tar.gz
curl -fsSLO "$B"
curl -fsSLO "$B.sha256"
sha256sum -c uv-x86_64-unknown-linux-gnu.tar.gz.sha256
tar -xzf uv-x86_64-unknown-linux-gnu.tar.gz
install -m 755 uv-x86_64-unknown-linux-gnu/uv uv-x86_64-unknown-linux-gnu/uvx /usr/local/bin/
uv --version
```

Debe quedar fuera de `/opt/odoo` y en una ruta del `secure_path` de `sudo`, porque `odev` lo ejecuta como `odoo`.

### 3. Crear el usuario y las carpetas

```bash
useradd --system --home-dir /opt/odoo --shell /bin/bash odoo
mkdir -p /opt/odoo/{bin,src,venvs,repos,confs,data,logs,python,docs,.cache}
chown -R odoo:odoo /opt/odoo
chmod 750 /opt/odoo
```

| Ruta | Contenido |
|---|---|
| `/opt/odoo/bin/odev` | Script de operación, enlazado en `/usr/local/bin/odev` |
| `/opt/odoo/src/<versión>/` | `odoo`, `enterprise` y `design-themes`, como worktrees de `src/.store` |
| `/opt/odoo/venvs/<versión>/` | venv por versión; `venvs/<nombre>/` si el repo trae `requirements.txt` |
| `/opt/odoo/repos/<nombre>/` | Repositorio clonado |
| `/opt/odoo/confs/<nombre>.conf` | Conf del ambiente |
| `/opt/odoo/data/<nombre>/` | `data_dir`: filestore y sesiones |
| `/opt/odoo/logs/<nombre>.log` | Log del servidor; las BD temporales usan `logs/<nombre>/<bd>.log` |
| `/opt/odoo/python/` | Intérpretes descargados por `uv` |

### 4. Configurar GitHub para `root`

`odev` clona y descarga como `root`; después entrega los archivos a `odoo`.

```bash
ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519_github -N '' -C "odev@$(hostname)"
cat >> ~/.ssh/config <<'EOF'
Host github.com
    User git
    IdentityFile ~/.ssh/id_ed25519_github
    IdentitiesOnly yes
EOF
cat ~/.ssh/id_ed25519_github.pub
```

Agrega la llave pública a tu cuenta de GitHub y comprueba el acceso. El segundo comando valida `enterprise`:

```bash
ssh -T git@github.com
git ls-remote git@github.com:odoo/enterprise.git HEAD
```

### 5. Configurar Postgres

El rol `odoo` no necesita ser superusuario y se autentica por `peer`, sin contraseña:

```bash
sudo -u postgres createuser --createdb odoo
```

Ajustes para varias BD en 8 GB de RAM. Sube `shared_buffers` si tienes más memoria y cambia `18` por tu versión:

```bash
cat > /etc/postgresql/18/main/conf.d/odev.conf <<'EOF'
# Ajuste para desarrollo con varias BD Odoo en 8 GB de RAM
shared_buffers = 1GB
work_mem = 16MB
maintenance_work_mem = 256MB
EOF
systemctl restart postgresql
```

### 6. Configurar swap

Con 8 GB, instalar varios módulos a la vez puede agotar la memoria. Omite este paso si el equipo tiene RAM de sobra.

```bash
fallocate -l 8G /swapfile
chmod 600 /swapfile
mkswap /swapfile
swapon /swapfile
echo '/swapfile none swap sw 0 0' >> /etc/fstab
echo 'vm.swappiness = 10' > /etc/sysctl.d/90-odev.conf
sysctl --system
```

### 7. Configurar nginx

Límites de peticiones y conexiones:

```bash
cat > /etc/nginx/conf.d/odoo-security.conf <<'EOF'
# Medidas de seguridad para el proxy de Odoo (contexto http).

# Rate limiting por IP
limit_req_zone $binary_remote_addr zone=odoo_login:10m rate=10r/m;
limit_req_zone $binary_remote_addr zone=odoo_rpc:10m rate=60r/m;
limit_req_zone $binary_remote_addr zone=odoo_general:10m rate=30r/s;
limit_conn_zone $binary_remote_addr zone=odoo_conn:10m;
limit_req_status 429;
limit_conn_status 429;

# Timeouts y buffers contra slowloris / cabeceras abusivas
client_header_timeout 15s;
client_body_timeout 30s;
send_timeout 60s;
keepalive_timeout 30s;
client_header_buffer_size 1k;
large_client_header_buffers 4 16k;
EOF
```

Sitio. El upstream es fijo: `odev` no modifica nginx.

```bash
cat > /etc/nginx/sites-available/odoo <<'EOF'
# Odoo que esté en 8069: el ambiente de odev start
upstream odoo {
    server 127.0.0.1:8069;
}

map $http_upgrade $connection_upgrade {
    default upgrade;
    ""      close;
}

server {
    listen 80 default_server;
    listen [::]:80 default_server;
    server_name _;

    client_max_body_size 200m;

    proxy_read_timeout 720s;
    proxy_connect_timeout 720s;
    proxy_send_timeout 720s;

    proxy_set_header Host $host;
    proxy_set_header X-Forwarded-Host $host;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_set_header X-Real-IP $remote_addr;

    # Cabeceras que Odoo no envía (X-Frame-Options y nosniff ya vienen de Odoo)
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;
    add_header Permissions-Policy "camera=(), microphone=(), geolocation=(), payment=(), usb=()" always;

    access_log /var/log/nginx/odoo.access.log;
    error_log /var/log/nginx/odoo.error.log;

    limit_conn odoo_conn 50;
    limit_req zone=odoo_general burst=200 nodelay;

    # Solo métodos que usa Odoo
    if ($request_method !~ ^(GET|HEAD|POST|OPTIONS)$) {
        return 405;
    }

    # Odoo apagado: nginx responde 502
    error_page 502 504 = @odoo_off;
    location @odoo_off {
        default_type text/html;
        add_header Retry-After 30 always;
        return 503 '<!doctype html><html lang="es"><meta charset="utf-8"><title>Sin ambiente</title><body style="font-family:sans-serif;margin:3rem"><h1>Ning&uacute;n ambiente encendido</h1><p>Inicia uno con <code>odev start</code> y vuelve a cargar esta p&aacute;gina.</p></body></html>';
    }

    # Gestor de bases de datos bloqueado (además de list_db = False)
    location ^~ /web/database {
        return 403;
    }

    # Archivos ocultos y rutas típicas de escáneres: cerrar sin respuesta
    location ~ /\. {
        return 444;
    }
    location ~* (wp-admin|wp-login|wp-content|xmlrpc\.php|phpmyadmin|\.php$|\.asp|\.env|cgi-bin) {
        return 444;
    }

    # Login, reseteo y registro: límite estricto contra fuerza bruta
    location ~ ^/web/(login|session/authenticate|reset_password|signup)$ {
        limit_req zone=odoo_login burst=5 nodelay;
        proxy_pass http://odoo;
        proxy_redirect off;
    }

    # API externa (XML-RPC / JSON-RPC / JSON-2)
    location ~ ^/(xmlrpc|jsonrpc|json/2)(/|$) {
        limit_req zone=odoo_rpc burst=20 nodelay;
        proxy_pass http://odoo;
        proxy_redirect off;
    }

    location /websocket {
        proxy_pass http://odoo;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection $connection_upgrade;
    }

    location / {
        proxy_pass http://odoo;
        proxy_redirect off;
    }

    gzip on;
    gzip_types text/css text/plain text/xml application/xml application/javascript application/json image/svg+xml;
}
EOF
ln -sf /etc/nginx/sites-available/odoo /etc/nginx/sites-enabled/odoo
rm -f /etc/nginx/sites-enabled/default
nginx -t
systemctl reload nginx
```

Sin ambiente encendido no hay nada en `127.0.0.1:8069`, y nginx muestra la página de aviso. Para publicar por HTTPS, adapta
este sitio con tu certificado: `odev` no depende de cómo lo publiques. Los
ambientes usan `workers = 0`, por lo que `/websocket` va al mismo puerto que el resto.

### 8. Configurar fail2ban

Filtro de intentos fallidos de inicio de sesión en Odoo:

```bash
cat > /etc/fail2ban/filter.d/odoo-login.conf <<'EOF'
[Definition]
failregex = ^.* INFO \S+ odoo\.addons\.base\.models\.res_users: Login failed for login:.* from <HOST>$
ignoreregex =
EOF
```

Jails. Cambia `<tu-ip>` por tu IP pública para no bloquearte durante las pruebas, o quítalo si es dinámica:

```bash
cat > /etc/fail2ban/jail.d/odoo.local <<'EOF'
[DEFAULT]
ignoreip = 127.0.0.1/8 ::1 <tu-ip>
bantime = 1h
findtime = 10m
banaction = nftables-multiport
banaction_allports = nftables-allports

[sshd]
enabled = true
backend = systemd
maxretry = 5

[nginx-limit-req]
enabled = true
port = http,https
logpath = /var/log/nginx/odoo.error.log
maxretry = 10

[odoo-login]
enabled = true
port = http,https
filter = odoo-login
logpath = /opt/odoo/logs/*.log
maxretry = 5
EOF
```

Crea un log vacío. Sin él, la jail `odoo-login` falla mientras no exista ningún ambiente (`Have not found any log file`):

```bash
sudo -u odoo touch /opt/odoo/logs/_fail2ban.log
fail2ban-client -t
systemctl restart fail2ban
```

fail2ban resuelve `/opt/odoo/logs/*.log` solo al cargar la jail, y `odev` no lo recarga. Después de cada `odev new`, recarga la
jail para que vigile el log del ambiente nuevo:

```bash
fail2ban-client -t && fail2ban-client reload odoo-login
```

### 9. Configurar logrotate

```bash
cat > /etc/logrotate.d/odev <<'EOF'
/opt/odoo/logs/*.log /opt/odoo/logs/*/*.log {
    su odoo odoo
    weekly
    rotate 4
    maxsize 200M
    compress
    missingok
    notifempty
    copytruncate
}
EOF
logrotate -d /etc/logrotate.d/odev
```

### 10. Instalar `odev`

`odev` es un único script de Python, sin dependencias externas. Cópialo desde un servidor ya montado:

```bash
scp root@servidor-existente:/opt/odoo/bin/odev /tmp/odev
install -o odoo -g odoo -m 755 /tmp/odev /opt/odoo/bin/odev
ln -sf /opt/odoo/bin/odev /usr/local/bin/odev
```

Ajusta las constantes propias de cada servidor. Cambia `dev.ejemplo.com` por tu dominio o IP:

```bash
sed -i "s|^PUBLIC_URL = .*|PUBLIC_URL = 'http://dev.ejemplo.com'|" /opt/odoo/bin/odev
sed -i "s|^LEGACY_DBS = .*|LEGACY_DBS = {}|" /opt/odoo/bin/odev
odev --help
```

| Constante | Valor de referencia | Para qué sirve |
|---|---|---|
| `ROOT` | `/opt/odoo` | Carpeta base. Si la cambias, cambia también las rutas de fail2ban y logrotate |
| `PUBLIC_URL` | Tu dominio | Solo se usa en los mensajes |
| `PUBLIC_PORT` | `8069` | Puerto de `odev start`; debe coincidir con el upstream de nginx |
| `BASE_PORT` | `8100` | Primer puerto local que se asigna a los ambientes |
| `PYTHONS` | Ver [Versiones](#versiones-de-odoo) | Python por defecto de cada versión de Odoo |
| `LEGACY_DBS` | `{}` | BD ajenas a `odev` que no deben tocarse sin `--force` |
| `MAX_ACTIVE` | `3` | Ambientes encendidos a partir de los cuales avisa |
| `MIN_AVAILABLE_MB` | `1500` | RAM disponible por debajo de la cual avisa |

### 11. Comprobar la instalación

Crea un ambiente de prueba desde un repositorio tuyo, indicando la versión:

```bash
odev core 20.0
odev new git@github.com:org/repo.git 20.0 --name prueba
odev -e prueba -i base --new
odev -e prueba start
```

Con el ambiente encendido debe responder `200`, y al detenerlo, `503`:

```bash
curl -s -o /dev/null -w '%{http_code}\n' http://127.0.0.1/web/login
odev -e prueba stop
curl -s -o /dev/null -w '%{http_code}\n' http://127.0.0.1/web/login
```

La primera vez, `odev core` tarda varios minutos. Si el repo es de pruebas, elimínalo al terminar con
`odev rm prueba --yes`. Esto borra el repo, el conf, la BD, el filestore y los logs del ambiente.

## Uso diario

```bash
odev new git@github.com:org/repo.git    # detecta la versión desde los manifests
cd /opt/odoo/repos/repo
odev -i base --new                      # crea la BD
odev start                              # en el puerto 8069, el que publica nginx
odev -u mi_modulo
odev logs
odev stop
```

Dentro de `/opt/odoo/repos/<nombre>`, el ambiente se deduce de la carpeta. Fuera, se indica antes del comando:
`odev -e repo stop`.

| Comando | Qué hace |
|---|---|
| `odev ps` | Estado, puerto, memoria y tamaños de cada ambiente |
| `odev repos [--sync]` | Repo y submódulos del ambiente; `--sync` clona los que falten y regenera `addons_path` |
| `odev start [-d BD] [--new] [--force] [--memory 3G]` | Inicia en segundo plano en el puerto 8069; solo uno a la vez |
| `odev stop` | Apaga y libera el puerto 8069 |
| `odev run [opciones]` | Servidor en primer plano en el `http_port` del conf, sin publicar; termina con Ctrl+C |
| `odev -i <mods>` / `odev -u <mods>` | Instala o actualiza y termina (`--stop-after-init`) |
| `odev logs [-n N] [--no-follow] [--db BD]` | Sigue el log |
| `odev shell [-d BD]` | Shell de Odoo |
| `odev restore <zip\|dump\|sql[.gz]> [--db BD] [--force]` | Restaura y neutraliza |
| `odev snapshot [etiqueta]` | Copia la BD y el filestore a `<bd>__snap_<etiqueta>` |
| `odev snapshot --restore <etiqueta>` | Vuelve a esa copia, con el ambiente apagado |
| `odev clone <destino> [--from BD]` | Copia BD y filestore |
| `odev clean [--all]` | Borra BD `__tmp_` y logs de más de 14 días |
| `odev test <mods> [--tags T] [--keep]` | Prueba en una BD temporal |
| `odev build [--rm]` | Dos pasos de un build de Odoo.sh |
| `odev rm <nombre> --yes [--force]` | Elimina repo, conf, BD, filestore y logs |

Cualquier otro subcomando va directo a `odoo-bin`, por ejemplo `odev neutralize`. La referencia completa está en
`odev --help`.

### Publicación

- `odev` no toca nginx: `odev start` levanta el ambiente en el puerto 8069, al que nginx apunta siempre.
- Solo puede haber un ambiente encendido. Si el 8069 está ocupado, `odev start` indica qué ambiente lo usa; para cambiar de
  ambiente, detén el anterior con `odev stop`.
- `odev run`, `-i`/`-u` y los tests usan el `http_port` del conf, uno libre o ninguno, así que pueden correr a la vez.
- Los ambientes comparten dominio, así que al cambiar de ambiente puede pedirse iniciar sesión otra vez.

### Conf generado

`odev new` crea `/opt/odoo/confs/<nombre>.conf` con permisos `640` y una `admin_passwd` aleatoria:

| Opción | Valor |
|---|---|
| `addons_path` | `odoo/addons`, `enterprise`, `design-themes` y las carpetas con módulos (hasta 3 niveles) del repo y de cada submódulo |
| `data_dir` | `/opt/odoo/data/<nombre>` |
| `db_user` / `db_name` | `odoo` / `<nombre>`, o el valor de `--db` |
| `list_db` | `False` |
| `http_port` / `http_interface` | Primer puerto libre desde `BASE_PORT`, para `odev run` (`odev start` usa 8069) / `127.0.0.1` |
| `proxy_mode` | `True` |
| `workers` / `max_cron_threads` | `0` / `0`: un solo proceso y sin crons |
| `logfile` | `/opt/odoo/logs/<nombre>.log` |

`odev` deduce la versión del ambiente de la ruta `/src/<versión>/odoo/` de `addons_path`. Si editas el conf, conserva esa ruta.

### Submódulos

`odev new` no inicializa los submódulos dentro del repo. Clona cada uno aparte, en `repos/<nombre-del-repo>`, y después los
submódulos de cada uno, sin límite de niveles. Un repo que aparece varias veces se clona una sola vez.

```text
repos/proyecto/              ── .gitmodules: sub/base -> org/base, ext/tools -> org/tools
repos/base/                  ── .gitmodules: lib -> org/lib (branch = dev)
repos/tools/
repos/lib/                   ── en la rama dev

addons_path = <core>, repos/proyecto, repos/base, repos/tools, repos/lib
```

- **Rama:** la `branch` de `.gitmodules` (`.` es la rama del repo padre) o, si no la indica, la rama por defecto. No es el
  commit que fija el padre, así que puede no coincidir con Odoo.sh. Si la rama no existe, usa la rama por defecto y avisa.
- **Compartidos:** si la carpeta ya existe con el mismo origen, se reutiliza sin cambiar su rama ni sus cambios, y avisa si
  está en otra rama. Todos los ambientes que la usan ven el mismo código. Si la ocupa otro repo con el mismo nombre, se usa
  `<dueño>-<nombre>`.
- **Conf:** `addons_path` incluye las carpetas con módulos del repo y de cada submódulo, en ese orden. Las rutas de los
  submódulos dentro del padre quedan vacías y se omiten. La primera línea, `# odev-repos: ...`, registra los submódulos: no
  la borres.
- **Carpeta actual:** dentro de un submódulo, `odev` usa su ambiente. Si lo usan varios, pide `-e`.
- **Cambios en `.gitmodules`:** después de un `git pull` del padre que agrega o quita submódulos, ejecuta `odev repos --sync`.
  Clona los que falten y regenera `addons_path`, lo que reemplaza cualquier edición manual de esa línea. No borra las carpetas
  de los retirados ni actualiza los ya clonados: en esos se hace `git pull` a mano.
- **`odev rm`:** borra los submódulos que solo usa ese ambiente, revisando antes si tienen cambios sin subir, y conserva los
  compartidos. No elimina un ambiente cuyo repo es submódulo de otro.
- **`odev build`:** incluye los módulos de los submódulos.

## Versiones de Odoo

`odev new` usa la versión indicada o, si no se indica, la más frecuente en los `version` de los manifests. Si no identifica
ninguna, solo clona el repo, sin conf ni BD. Para crearle el ambiente después:

```bash
odev new <nombre> <version>
```

`odev rm` solo elimina ambientes: un repo clonado sin ambiente se borra a mano.

Para corregir una versión equivocada, con el ambiente apagado:

```bash
odev version <version>
odev version <version> --drop-db
```

Sin `--drop-db`, se detiene si el ambiente ya tiene BD. `--drop-db` borra la BD, sus copias `<bd>__*` y sus filestores **sin
hacer respaldo**. El comando no migra datos.

Para administrar los cores:

```bash
odev core
odev core 18.0
odev core 20.0 --update
```

El primero lista las versiones, el segundo descarga el core y crea su venv, y el tercero actualiza el core y sus dependencias.

| Odoo | Python por defecto | Estado |
|---|---|---|
| 20.0 | 3.14 | Probado |
| 19.0 | 3.12 | Probado |
| 18.0 | 3.12 | Sin probar |
| 17.0 | 3.11 | Sin probar |
| 16.0 | 3.10 | Sin probar |

Se cambia con `--python`. Si un repo trae `requirements.txt` con contenido, recibe un venv propio en `venvs/<nombre>`.

## Bases de datos

Sin `-d` se usa la BD del conf. Con `-d <bd>` se usa otra, con estas reglas:

- Si la BD no existe, da error salvo con `--new`. Sin esta protección, Odoo la crearía sola, incluso con un nombre mal escrito.
- Si la BD pertenece a otro ambiente, da error salvo con `--force`. Se consideran suyas `<bd>` y `<bd>__*`.
- `odev clean` solo borra BD `<bd>__tmp_<fecha>`. Nunca borra la principal ni las `__snap_`.

`odev restore` siempre neutraliza. Si no puede, elimina la BD para evitar correos o envíos reales.

## Pruebas y builds

- `odev test <módulos>` instala sin demo en una BD temporal y la borra al terminar. Falla si no corrió ningún test de esos
  módulos, porque el total del log incluye suites de `web` y no sirve para detectarlo.
- `odev build` ejecuta los dos pasos de Odoo.sh sobre los módulos instalables del repo: instalación sin demo con sus tests y
  `module force-demo`. Falla si no corrieron tests.

`odev build` usa el core del servidor, así que no replica un build concreto: para eso necesitas el mismo commit del core y las
mismas dependencias.

## Instalar pgvector (opcional)

Solo hace falta para el módulo `ai` de Odoo. Para que el rol `odoo`, que no es superusuario, pueda crear la extensión:

```bash
apt-get install -y postgresql-18-pgvector
echo 'trusted = true' >> /usr/share/postgresql/18/extension/vector.control
```

Una actualización del paquete puede revertir el cambio. Si reaparece `permission denied to create extension vector`, repite
la segunda línea.

## Solo Community

`odev` espera `enterprise` en el core. Sin acceso a ese repositorio hay que cambiar dos líneas del script. **No se ha probado.**

- Quitar `'enterprise'` de `CORE_REPOS`.
- Quitar `SRC / version / 'enterprise'` de la lista `core` en la función `cmd_new`.

## Seguridad

- `odev` da acceso `root`: cualquiera que pueda ejecutarlo controla el servidor.
- Odoo escucha solo en `127.0.0.1` y se conecta a Postgres por socket. Lo único que esta guía expone es nginx, junto con SSH.
- El gestor de BD está bloqueado por `list_db = False` y por nginx.
- No guardes las `admin_passwd` ni las llaves en este repositorio.
- Restringe los puertos con el firewall de tu proveedor. El servidor de referencia no usa `ufw` y deja que fail2ban gestione
  sus propias reglas de nftables.

## Respaldo

Los ambientes se reconstruyen con sus repositorios. Lo que no se recupera solo:

| Qué | Dónde |
|---|---|
| Cambios sin subir | `/opt/odoo/repos/<nombre>`. `odev rm` se detiene si los hay, salvo con `--force` |
| Datos | BD de Postgres y `/opt/odoo/data/<nombre>/filestore` |
| Script | `/opt/odoo/bin/odev`. No está versionado: guarda una copia |
| Configuración | `/opt/odoo/confs` y los archivos de nginx, fail2ban, Postgres y logrotate |

## Problemas frecuentes

| Síntoma | Causa | Solución |
|---|---|---|
| `Ningún ambiente encendido` en el navegador | No hay ambiente encendido, o el proceso se detuvo | `odev ps` y `odev start` |
| fail2ban no bloquea los intentos fallidos de un ambiente nuevo | La jail no se recargó después de `odev new` | `fail2ban-client reload odoo-login` |
| `el puerto 8069 lo usa ...` | Ya hay un ambiente encendido, u otro proceso en ese puerto | `odev -e <ese> stop`, o detén el otro proceso |
| `Have not found any log file for odoo-login jail` | No hay ningún log en `/opt/odoo/logs` | Crea `_fail2ban.log` (paso 8) |
| `no se pudo descargar enterprise <versión>` | Sin acceso de GitHub o versión inexistente | Revisa la llave y `git ls-remote` (paso 4) |
| Fallan las dependencias al crear el venv | Falta una librería de desarrollo | Instala el paquete que indica el error y ejecuta `odev core <versión> --update`, que también actualiza el core |
| `<nombre> no tiene conf` | Repo clonado sin ambiente | `odev new <nombre> <version>` |
| `<nombre> es submódulo de varios ambientes` | La carpeta la comparten varios ambientes | Indica el ambiente con `-e` |
| Falta un módulo de un submódulo nuevo | El padre agregó un submódulo después de `odev new` | `odev repos --sync` y reinicia el ambiente |
| `la BD <bd> no existe; usa --new` | Protección contra crear una BD por error | Agrega `--new` si es intencional |
| `la BD <bd> pertenece a ...` | La BD es de otro ambiente | Corrige `-d`, o agrega `--force` si es intencional |
| Build o test con 0 tests | Un hook desinstala módulos durante la instalación (`Reloading registry once more after uninstalling modules`) | No desinstales módulos en hooks |
| `permission denied to create extension vector` | pgvector no está marcada como `trusted` | Repite el comando de [pgvector](#instalar-pgvector-opcional) |
| `Killed` o código 137 al instalar | Falta de memoria | Instala por tandas y revisa la swap (paso 6). En `odev start`, sube `--memory` |
