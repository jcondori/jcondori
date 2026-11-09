# Paso 7: Odoo

> Serie [Seguridad de un servidor web](README.md) · Anterior: [paso 6, Fail2ban web](06-fail2ban-web.md)
> · Último paso de la serie

Configuración de servidor para una instancia de Odoo publicada detrás de Nginx: el `.conf` de Odoo, rutas de Odoo en
Nginx, jails propias de Odoo y el proceso de Odoo. Solo aplica a sitios Odoo. Las líneas con `AJUSTAR` cambian según
el servidor; el resto se deja tal cual.

## Requisitos

- Paso 5 hecho: el sitio de Odoo existe (ejemplo del [paso 5.4](05-nginx-sitio.md#54-ejemplo-odoo)).
- Paso 6 hecho: la jail `odoo-login` está activa.
- Nginx y Fail2ban corren **en el host**, no en un contenedor. Odoo puede correr en el host o en Docker.

## Modo de ejecución

El paso cubre tres formas de correr Odoo. Elige la tuya; donde las instrucciones cambian, cada sección lo indica.

| Modo | Cómo corre Odoo |
|---|---|
| **Usuario dedicado** | Servicio systemd con `User=odoo` (o el usuario que tengas). |
| **root** | Servicio systemd sin `User=`, o con `User=root`. |
| **Docker** | Contenedor (imagen oficial `odoo` o propia) con `docker compose`. |

Valores que se usan en todo el paso:

| Valor | Usuario dedicado / root | Docker | AJUSTAR |
|---|---|---|---|
| `.conf` de Odoo | `/etc/odoo/odoo.conf` | en el host: `/opt/odoo/config/odoo.conf`, montado en `/etc/odoo/odoo.conf` | ruta del `.conf` |
| Log de Odoo (lo lee Fail2ban) | `/var/log/odoo/odoo.log` | en el host: `/var/log/odoo/odoo.log`, montado en `/var/log/odoo` | `logfile` |
| Dueño de los archivos | `odoo` (o `root`) | UID del usuario del contenedor (ver 7.4) | usuario |
| Reinicio | `systemctl restart odoo` | `docker compose restart odoo` | servicio |
| Puerto HTTP de Odoo | `8069` | `8069`, publicado en `127.0.0.1` | `http_port` |
| Sitio Nginx | `/etc/nginx/conf.d/odoo.example.com.conf` | igual | archivo del paso 5 |

En Docker, los comandos `docker compose` se ejecutan en la carpeta del `docker-compose.yml` y `odoo` es el nombre del
servicio en ese archivo (AJUSTAR).

## 7.1 Configuración de Odoo

En la sección `[options]` del `.conf`:

```ini
[options]
; AJUSTAR: 127.0.0.1 en host; 0.0.0.0 en Docker (el puerto se publica solo en loopback, ver abajo)
http_interface = 127.0.0.1
; AJUSTAR: puerto del proxy_pass del sitio
http_port = 8069
proxy_mode = True
list_db = False
; AJUSTAR: nombre de la base
db_name = midb
; vacío a propósito: deshabilita la contraseña maestra. Nunca borrar la línea.
admin_passwd =
; ruta dentro del proceso: en Docker, la del volumen montado
logfile = /var/log/odoo/odoo.log
log_level = info
```

Los comentarios van siempre en línea propia: Odoo no quita un `;` al final de la línea y lo toma como parte del valor.

| Clave | Efecto |
|---|---|
| `http_interface` | Dónde escucha Odoo. En host, solo loopback. En Docker, todas las interfaces **del contenedor**; el aislamiento lo da la publicación del puerto. |
| `proxy_mode = True` | Odoo toma la IP de `X-Forwarded-For`. Sin esto el log dice la IP del proxy (`127.0.0.1` o la del bridge de Docker) y las jails no bloquean a nadie. |
| `list_db = False` | Desactiva el selector y la lista de bases. |
| `db_name` | Fija la base que sirve Odoo: no se elige ni se adivina otra. |
| `admin_passwd =` (vacío) | Deshabilita la contraseña maestra del gestor de bases: Odoo rechaza cualquier valor. |
| `logfile` | Ruta fija del log que leen las jails `odoo-*`. |
| `log_level = info` | Registra los logins fallidos sin volcar datos de `debug`. |

`admin_passwd` va **escrito y vacío**, nunca ausente. Sin la línea, Odoo usa el valor por defecto `admin`, y con el
gestor de bases activo el primer visitante puede fijar la contraseña maestra que quiera. Con `list_db = False` el gestor
ya está cerrado; la línea vacía lo mantiene cerrado aunque `list_db` se vuelva a activar.

Si hay varias instancias en el mismo servidor, cada `.conf` lleva su propio `logfile`, y cada uno va en el `logpath` de
las jails (7.3).

### Docker: puerto, configuración y log

En el `docker-compose.yml`, el servicio de Odoo publica el puerto **solo en loopback** y monta el `.conf` y la carpeta
del log desde el host:

```yaml
services:
  odoo:                                   # AJUSTAR: nombre del servicio
    image: odoo:18                        # AJUSTAR: imagen y versión
    ports:
      - "127.0.0.1:8069:8069"             # nunca "8069:8069": Docker abre el puerto por fuera del firewall del host
    volumes:
      - /opt/odoo/config/odoo.conf:/etc/odoo/odoo.conf:ro
      - /var/log/odoo:/var/log/odoo
      - odoo-data:/var/lib/odoo
```

Sin `127.0.0.1:` delante, Docker publica el puerto en todas las interfaces y añade sus propias reglas de nftables, que
`ufw` no filtra.

Sin el volumen de `/var/log/odoo`, la imagen oficial escribe el log en la salida estándar del contenedor y Fail2ban no
tiene archivo que leer.

### Aplica

Usuario dedicado / root:

```bash
systemctl restart odoo                     # AJUSTAR: nombre del servicio
ss -tlnp | grep 8069                       # 127.0.0.1:8069, nunca 0.0.0.0
```

Docker:

```bash
mkdir -p /var/log/odoo
docker compose up -d odoo
ss -tlnp | grep 8069                       # 127.0.0.1:8069 (docker-proxy), nunca 0.0.0.0
tail -n 5 /var/log/odoo/odoo.log           # el log se escribe en el host
```

## 7.2 Nginx: rutas de Odoo

Igual en los tres modos: Nginx corre en el host y apunta a `127.0.0.1:8069`.

### Zona para la API

Añade a `/etc/nginx/conf.d/00-base.conf` (junto a la zona `login` del paso 4):

```nginx
limit_req_zone $binary_remote_addr zone=odoo_rpc:10m rate=60r/m;
```

### Bloques del sitio

En el `server` del 443 del sitio de Odoo, **sustituye** el bloque `location = /web/login` del paso 5.4 por estos:

```nginx
    # Gestor de bases de datos: cerrado
    location ^~ /web/database {
        return 403;
    }

    # Importador de módulos (solo lo usa odoo-bin desde fuera)
    location ^~ /base_import_module {
        return 404;
    }

    # Login, 2FA, reseteo y registro: límite estricto
    location ~ ^/web/(login|login/totp|session/authenticate|reset_password|signup)$ {
        limit_req zone=login burst=5 nodelay;
        limit_req_status 429;
        proxy_pass http://127.0.0.1:8069;      # AJUSTAR: puerto de Odoo
        include snippets/proxy.conf;
    }

    # API externa: XML-RPC, JSON-RPC y JSON-2
    location ~ ^/(xmlrpc|jsonrpc|json/2)(/|$) {
        limit_req zone=odoo_rpc burst=20 nodelay;
        limit_req_status 429;
        proxy_pass http://127.0.0.1:8069;      # AJUSTAR: puerto de Odoo
        include snippets/proxy.conf;
    }
```

Sin integraciones externas por API, cierra la API en vez de limitarla. Sustituye el último bloque por:

```nginx
    location ~ ^/(xmlrpc|jsonrpc|json/2)(/|$) {
        return 403;
    }
```

O ábrela solo a IPs conocidas, dentro del mismo bloque y antes del `proxy_pass`:

```nginx
        allow 198.51.100.10;                   # AJUSTAR: IP del integrador
        deny  all;
```

Aplica:

```bash
nginx -t && systemctl reload nginx
```

## 7.3 Fail2ban: jails de Odoo

Igual en los tres modos: Fail2ban lee el log de Odoo **en el host** (en Docker, el del volumen montado) y bloquea en el
host, que es donde entra el tráfico a Nginx.

Dos jails que se suman a `odoo-login` (paso 6):

| Jail | Qué bloquea | Log | Límite |
|---|---|---|---|
| `odoo-reset-password` | Peticiones repetidas de reseteo de contraseña | Log de Odoo | 10 en 1 h, 1 día |
| `odoo-api-401` | API key inválida en `/json/2` (Odoo 19 o superior) | `access.log` de Nginx | 10 en 10 min, 1 día |

`/etc/fail2ban/filter.d/odoo-reset-password.conf`:

```ini
[Definition]
failregex = ^\s*\d+ INFO \S+ odoo\.addons\.auth_signup\.controllers\.main: Password reset attempt for <[^>]*> by user <[^>]*> from <HOST>\s*$
ignoreregex =
```

`/etc/fail2ban/filter.d/odoo-api-401.conf`:

```ini
[Definition]
failregex = ^<HOST> - \S+ \[[^\]]*\] "(?:GET|POST|PUT|DELETE) /json/2\S* HTTP[^"]*" 401 
ignoreregex =
```

El filtro de la API espera el formato `combined` (o `main_ext` del paso 4.5). Con otro `log_format`, ajusta la
expresión.

Añade a `/etc/fail2ban/jail.d/odoo.local`:

```ini
[odoo-reset-password]
enabled  = true
port     = http,https
filter   = odoo-reset-password
logpath  = /var/log/odoo/odoo.log
maxretry = 10
findtime = 1h
bantime  = 1d

[odoo-api-401]
enabled  = true
port     = http,https
filter   = odoo-api-401
logpath  = /var/log/nginx/access.log
maxretry = 10
findtime = 10m
bantime  = 1d
```

AJUSTAR `logpath`: en `odoo-reset-password`, la ruta **en el host** del `logfile` de cada instancia de Odoo (varios,
uno por línea); en `odoo-api-401`, el `access_log` del sitio de Odoo. La jail `odoo-login` del paso 6 usa la misma ruta
del log de Odoo.

Prueba los filtros contra líneas sueltas:

```bash
fail2ban-regex '2026-11-08 19:20:00,123 1234 INFO midb odoo.addons.auth_signup.controllers.main: Password reset attempt for <admin> by user <public> from 203.0.113.7' /etc/fail2ban/filter.d/odoo-reset-password.conf
fail2ban-regex '203.0.113.7 - - [08/Nov/2026:19:20:00 +0000] "GET /json/2/res.partner HTTP/1.1" 401 0 "-" "curl"' /etc/fail2ban/filter.d/odoo-api-401.conf
```

Cada una debe dar `1 matched`. Aplica:

```bash
fail2ban-client -t
systemctl reload fail2ban
fail2ban-client status        # debe listar también: odoo-api-401, odoo-reset-password
```

## 7.4 Archivos y proceso

### Permisos del `.conf`

El `.conf` guarda la contraseña de la base en claro. Solo lo lee el usuario que corre Odoo.

Usuario dedicado:

```bash
chown odoo:odoo /etc/odoo/odoo.conf     # AJUSTAR: usuario y ruta
chmod 600 /etc/odoo/odoo.conf
```

root:

```bash
chown root:root /etc/odoo/odoo.conf     # AJUSTAR: ruta
chmod 600 /etc/odoo/odoo.conf
```

Docker: el archivo del host lo lee el usuario **del contenedor**. Mira su UID y dale el archivo:

```bash
docker compose exec odoo id                                # p. ej. uid=100(odoo) gid=101(odoo)
chown 100:101 /opt/odoo/config/odoo.conf                   # AJUSTAR: uid:gid de la salida anterior
chmod 600 /opt/odoo/config/odoo.conf
chown 100:101 /var/log/odoo                                # el contenedor escribe aquí el log
```

Si el contenedor corre como root (`uid=0`), usa `root:root`.

### Permisos del código

Usuario dedicado / root: el core y los addons son de `root` o del usuario de Odoo, nunca escribibles por otros:

```bash
chmod -R o-w /opt/odoo                  # AJUSTAR: carpeta del core y los addons
```

Docker: igual sobre las carpetas de addons que montes como volumen (`/mnt/extra-addons`); móntalas con `:ro`.

### Usuario dedicado / root: unidad systemd

`systemctl edit odoo` (AJUSTAR: nombre del servicio) y pega:

```ini
[Service]
NoNewPrivileges=true
PrivateTmp=true
PrivateDevices=true
ProtectSystem=full
ProtectHome=true
ProtectKernelTunables=true
ProtectKernelModules=true
ProtectControlGroups=true
RestrictSUIDSGID=true
LockPersonality=true
```

- `ProtectSystem=full` deja `/usr`, `/boot` y `/etc` en solo lectura. Odoo escribe en su `data_dir` y su `logfile`, que
  no deben estar ahí.
- `ProtectHome=true` oculta `/home`, `/root` y `/run/user`. AJUSTAR: si Odoo, su venv, su `data_dir` o su `.conf` están
  bajo `/home` o `/root` (habitual cuando corre como root), quita esa línea.

Aplica:

```bash
systemctl daemon-reload
systemctl restart odoo
systemctl is-active odoo                # active
systemd-analyze security odoo           # la puntuación baja respecto a la de antes
```

Si no arranca, `journalctl -u odoo -n 50` dice qué ruta no puede escribir.

### Docker: opciones del contenedor

Añade al servicio de Odoo en el `docker-compose.yml`:

```yaml
services:
  odoo:
    security_opt:
      - no-new-privileges:true
    cap_drop:
      - ALL
    tmpfs:
      - /tmp
```

- `no-new-privileges` es el equivalente de `NoNewPrivileges=true`.
- `cap_drop: ALL` quita las capacidades de Linux; Odoo no necesita ninguna. La imagen oficial ya arranca como usuario
  `odoo`. AJUSTAR: si tu imagen arranca como root y cambia de usuario en el entrypoint (`gosu`, `su-exec`, `chown`),
  añade `cap_add: [SETUID, SETGID, CHOWN]`.
- No añadas `privileged: true` ni montes `/var/run/docker.sock` en el contenedor de Odoo.

Aplica:

```bash
docker compose up -d odoo
docker compose ps odoo                  # running
docker compose logs --tail 50 odoo      # sin errores de permisos
```

## Verificación del paso 7

En el servidor:

```bash
grep -E "^(http_interface|proxy_mode|list_db|db_name|admin_passwd|logfile|log_level)\b" /etc/odoo/odoo.conf   # AJUSTAR: ruta del .conf en el host
stat -c "%a %U" /etc/odoo/odoo.conf     # 600 y el usuario que corre Odoo
ss -tlnp | grep 8069                    # 127.0.0.1:8069
fail2ban-client status | grep -E "odoo-login|odoo-reset-password|odoo-api-401"
```

Desde fuera del servidor:

```bash
curl -s -o /dev/null -w "%{http_code}\n" https://odoo.example.com/web/database/manager            # 403
curl -s -o /dev/null -w "%{http_code}\n" https://odoo.example.com/web/database/selector           # 403
curl -s -o /dev/null -w "%{http_code}\n" https://odoo.example.com/base_import_module/login_upload # 404
curl -s -o /dev/null -w "%{http_code}\n" --max-time 5 http://<IP-del-servidor>:8069/              # sin respuesta
```

Y en el log de Odoo (en el host), tras un login fallido desde fuera, la línea `Login failed` lleva la IP del cliente,
no `127.0.0.1` ni la IP del bridge de Docker (`172.x.x.x`).

No repitas pruebas de login ni de reseteo con las jails activas: bloquean tu IP.

## Deshacer

```bash
rm /etc/fail2ban/filter.d/odoo-reset-password.conf /etc/fail2ban/filter.d/odoo-api-401.conf
# borra los bloques [odoo-reset-password] y [odoo-api-401] de /etc/fail2ban/jail.d/odoo.local
systemctl reload fail2ban
```

Usuario dedicado / root:

```bash
rm -r /etc/systemd/system/odoo.service.d
systemctl daemon-reload && systemctl restart odoo
```

Docker: quita `security_opt`, `cap_drop` y `tmpfs` del `docker-compose.yml` y `docker compose up -d odoo`.

En Nginx, vuelve al bloque `location = /web/login` del paso 5.4, quita la zona `odoo_rpc` y recarga
(`nginx -t && systemctl reload nginx`).

---

Fin de la serie. Índice: [README](README.md).
