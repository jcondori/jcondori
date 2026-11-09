# Paso 6: Fail2ban para Nginx y las aplicaciones

> Serie [Seguridad de un servidor web](README.md) · Anterior: [paso 5, un sitio por aplicación](05-nginx-sitio.md)
> · Siguiente: [paso 7, Odoo](07-odoo-seguridad.md)

Jails que leen los logs de Nginx y de las aplicaciones: bloquean a quien insiste contra un login y a quien escanea rutas
conocidas. Usan la base del [paso 2](02-fail2ban-base.md): mismos valores por defecto, mismo bloqueo con nftables, y la
jail `recidive` las cubre también sin tocar nada.

Se repite con cada aplicación nueva, después de su paso 5.

## Requisitos

- Paso 2 hecho: Fail2ban instalado, con `jail.local` y las jails `sshd` y `recidive`.
- Pasos 4 y 5 hechos **y verificados**. En concreto:
  - Los sitios usan la zona `login` con `limit_req` (si no, `nginx-limit-req` no detecta nada).
  - La aplicación registra la IP real del cliente en su log (`proxy_mode = True` en Odoo). Si no, todas las líneas dicen
    `127.0.0.1`, que está en `ignoreip`, y la jail no bloquea a nadie.
  - Los logs ya existen y tienen líneas reales. Una jail cuyo log no existe no arranca
    (`Have not found any log file for ... jail`).

## Jails

| Jail | Qué bloquea | Log | Límite |
|---|---|---|---|
| `nginx-limit-req` | Quien choca varias veces con la zona `login` | `/var/log/nginx/error.log` | 10 en 10 min, 6 h |
| `nginx-scan` | Peticiones a rutas trampa (`.env`, `.git`, `wp-login.php`…) | `/var/log/nginx/access.log` | 2 en 1 h, 1 día |
| `nginx-noise` | Tráfico sin destino: por IP u otro host (`444`), mal formado (`400`), TLS rechazado | `access.log` + `tls-reject.log` | 5 en 1 h, 1 día |
| `odoo-login` | Logins fallidos de Odoo | Log de Odoo | 5 en 10 min, 1 h |

## 6.1 Nginx: límite de login

Cada vez que Nginx rechaza una petición por `limit_req`, escribe `limiting requests` en `error.log`. Esta jail usa el
filtro `nginx-limit-req` que trae Fail2ban y bloquea a quien lo provoca varias veces seguidas.

`/etc/fail2ban/jail.d/nginx.local`:

```ini
[nginx-limit-req]
enabled  = true
port     = http,https
logpath  = /var/log/nginx/error.log
maxretry = 10
findtime = 10m
bantime  = 6h
```

`maxretry` es más alto que en SSH: un usuario real que se equivoca de contraseña varias veces también llega al límite de
Nginx, y no conviene bloquearlo horas por eso.

## 6.2 Nginx: escaneos de rutas conocidas

Los bots prueban rutas de WordPress, archivos `.env` y `.git`, phpMyAdmin, etc. Ninguna aplicación propia las usa, así
que quien las pide está escaneando.

El filtro `nginx-botsearch` que trae Fail2ban lee `error.log` y solo funciona en sitios que sirven archivos con `root`.
En un proxy, el 404 lo responde la aplicación y no queda en `error.log`. Por eso se usa un filtro propio sobre
`access.log`.

`/etc/fail2ban/filter.d/nginx-scan.conf`:

```ini
[Definition]
failregex = ^<HOST> - \S+ \[[^\]]*\] "(?:GET|POST|HEAD) /(?:[^" ]*/)?(?:\.env|\.git/|wp-login\.php|wp-admin|wp-config|xmlrpc\.php|phpmyadmin|pma/|cgi-bin/|vendor/phpunit|boaform|HNAP1|phpinfo\.php|actuator/)[^"]*" (?:301|302|400|401|403|404|444)
ignoreregex =
```

Dos detalles de la expresión que no se ven a simple vista:

- **`\[[^\]]*\]`, con `*` y no `+`.** Fail2ban detecta la fecha, la **quita de la línea** y aplica `failregex` sobre lo
  que queda: el `[08/Nov/2026:19:20:00 +0000]` llega como `[]`. Con `+` el filtro no coincide con nada (0 de 4352
  líneas en un servidor con 1300 escaneos reales), y tampoco con la línea suelta de 6.4.
- **302 y 401.** Una aplicación con login suele responder a cualquier ruta desconocida con una redirección a su página
  de login (302) o con un 401, no con un 404. En un servidor con una app así, el 60 % de los escaneos eran 302.

Añade a `/etc/fail2ban/jail.d/nginx.local`:

```ini
[nginx-scan]
enabled  = true
port     = http,https
filter   = nginx-scan
logpath  = /var/log/nginx/access.log
maxretry = 2
findtime = 1h
bantime  = 1d
```

Con `maxretry = 2` basta con dos peticiones a rutas trampa. Si alguna aplicación tuya usa una de esas rutas (por
ejemplo, un WordPress real), quítala del filtro.

El filtro espera el formato `combined` por defecto de Nginx (empieza por la IP). El formato `main_ext` del
[paso 4.5](04-nginx-servidor.md#45-logs) empieza igual y solo añade campos al final, así que sirve sin cambios
(probado). Con otro `log_format` distinto, ajusta la expresión.

### Tráfico sin destino: `nginx-noise`

Tras el paso 4, quien entra por IP (o con un `Host` que no es un sitio) no recibe nada: `444` en el 80 y handshake
rechazado en el 443. Lo mismo quien manda basura o TLS al puerto 80 (`400`). No obtienen información, pero pueden seguir
llamando. Esta jail bloquea a quien insiste, para reducir ruido en los logs. Gana poco en seguridad (ya no consiguen
nada), y el riesgo es bajo: un usuario legítimo nunca entra por IP.

Requiere el `tls-reject.log` del [paso 4.4](04-nginx-servidor.md#44-puerto-443-por-defecto-cortar-el-handshake).

`/etc/fail2ban/filter.d/nginx-noise.conf`:

```ini
[Definition]
failregex = ^<HOST> - \S+ \[[^\]]*\] "[^"]*" (?:400|444) 
            handshake rejected while SSL handshaking, client: <HOST>, server:
ignoreregex =
```

Un solo filtro con una expresión por log: cada línea solo coincide con la de su archivo.

Añade a `/etc/fail2ban/jail.d/nginx.local`:

```ini
[nginx-noise]
enabled  = true
port     = http,https
filter   = nginx-noise
logpath  = /var/log/nginx/access.log
           /var/log/nginx/tls-reject.log
maxretry = 5
findtime = 1h
bantime  = 1d
```

`maxretry = 5` y no 2 como `nginx-scan`: un `444` o un `400` solo no delata a nadie (un navegador antiguo, un enlace con
la IP), cinco en una hora sí. Las rutas trampa pedidas por IP cuentan en las dos jails; da igual, la primera que llegue
al límite bloquea.

Lo que sigue sin bloquearse, a propósito: escaneos al dominio con rutas fuera de la lista de `nginx-scan` (la
aplicación responde y no filtra nada) y botnets que hacen 1-2 intentos por IP (ningún límite por IP las frena; eso
requiere un WAF o CDN delante, fuera de esta serie).

## 6.3 Odoo: logins fallidos

Odoo escribe una línea por cada login fallido, con la IP del cliente. El formato cambia según la versión:

```text
# Odoo 18 o anterior
2026-11-08 19:20:00,123 1234 INFO midb odoo.addons.base.models.res_users: Login failed for db:midb login:admin from 203.0.113.7
# Odoo 19 o superior
2026-11-08 19:20:00,123 1234 INFO midb odoo.addons.base.models.res_users: Login failed for login:admin from 203.0.113.7
```

`/etc/fail2ban/filter.d/odoo-login.conf` (el `db:` es opcional, cubre los dos formatos):

```ini
[Definition]
failregex = ^\s*\d+ INFO \S+ odoo\.addons\.base\.models\.res_users: Login failed for (?:db:\S+ )?login:.* from <HOST>\s*$
ignoreregex =
```

Las líneas `from n/a` (login sin petición HTTP: consola, scripts) no coinciden, a propósito.

`/etc/fail2ban/jail.d/odoo.local`:

```ini
[odoo-login]
enabled  = true
port     = http,https
filter   = odoo-login
logpath  = /var/log/odoo/odoo.log
maxretry = 5
findtime = 10m
bantime  = 1h
```

AJUSTAR `logpath` al `logfile` del `.conf` de Odoo. Si hay varias instancias, una jail por log o varios `logpath` en
líneas separadas.

### Otras aplicaciones

El mismo patrón sirve para cualquier aplicación que registre los logins fallidos con la IP del cliente:

1. Busca en su log una línea de login fallido real.
2. Escribe un filtro en `filter.d/<app>-login.conf` con `<HOST>` donde va la IP.
3. Pruébalo con `fail2ban-regex` (ver 6.4) hasta que coincida.
4. Crea su jail en `jail.d/<app>.local`, como la de Odoo.

Si la aplicación no registra los fallos, `nginx-limit-req` (6.1) ya la cubre a nivel de Nginx.

## 6.4 Probar los filtros

Antes de activar las jails, comprueba que cada filtro reconoce las líneas reales del log:

```bash
fail2ban-regex /var/log/nginx/error.log /etc/fail2ban/filter.d/nginx-limit-req.conf
fail2ban-regex /var/log/nginx/access.log /etc/fail2ban/filter.d/nginx-scan.conf
fail2ban-regex /var/log/nginx/access.log /etc/fail2ban/filter.d/nginx-noise.conf
fail2ban-regex /var/log/nginx/tls-reject.log /etc/fail2ban/filter.d/nginx-noise.conf
fail2ban-regex /var/log/odoo/odoo.log /etc/fail2ban/filter.d/odoo-login.conf
```

La salida dice cuántas líneas coinciden (`Failregex: N total`). Con `--print-all-matched` muestra cuáles. Cero
coincidencias habiendo fallos reales en el log = el filtro no sirve para ese formato.

Si el log todavía no tiene fallos, provoca uno **desde una IP que no te importe bloquear** (datos del móvil) o prueba el
filtro contra una línea suelta:

```bash
fail2ban-regex '203.0.113.7 - - [08/Nov/2026:19:20:00 +0000] "GET /.env HTTP/1.1" 404 0 "-" "curl"' /etc/fail2ban/filter.d/nginx-scan.conf
```

## 6.5 Aplicar

```bash
fail2ban-client -t            # valida toda la configuración
systemctl reload fail2ban
fail2ban-client status        # debe listar también: nginx-limit-req, nginx-noise, nginx-scan, odoo-login
```

## Verificación del paso 6

```bash
fail2ban-client status nginx-scan
fail2ban-client status nginx-noise    # "File list" debe incluir access.log y tls-reject.log
fail2ban-client status odoo-login
fail2ban-client banned
```

En un servidor con IP pública, en un día ya hay bots pidiendo `/.env` o `/wp-login.php`: `nginx-scan` debe mostrar
alguna IP bloqueada.

**No repitas la prueba de límite de login del paso 5** con las jails activas: tras unos intentos, `nginx-limit-req`
bloquea tu IP durante 6 h. Si pasa, ver "Si te bloqueas a ti mismo" en el [paso 2](02-fail2ban-base.md).

## Notas

- **Proxy o CDN delante (Cloudflare, balanceador):** Nginx verá la IP del proxy, no la del cliente, y Fail2ban
  bloquearía al proxy entero. Antes hay que configurar `real_ip_header` y `set_real_ip_from` en Nginx para que el log
  tenga la IP real. Y aun así, bloquear en el servidor no frena el tráfico que llega por el proxy: ahí el bloqueo se
  hace en el propio CDN.
- **IP compartida (oficina, NAT):** todos los usuarios de una oficina salen por la misma IP. Si uno falla el login
  varias veces, el bloqueo los saca a todos; y si se repite 3 veces en un día, `recidive` (paso 2) bloquea la oficina
  entera una semana en todos los puertos. Si los usuarios de una aplicación trabajan desde una oficina con IP fija,
  añádela a `ignoreip` en `jail.local`.
- **Logs rotados:** logrotate rota los logs de Nginx y Odoo; Fail2ban sigue el archivo nuevo solo, no hay que hacer
  nada.

## Deshacer

```bash
rm /etc/fail2ban/jail.d/nginx.local /etc/fail2ban/jail.d/odoo.local
rm /etc/fail2ban/filter.d/nginx-scan.conf /etc/fail2ban/filter.d/nginx-noise.conf /etc/fail2ban/filter.d/odoo-login.conf
systemctl reload fail2ban
```

Las IPs que esas jails tenían bloqueadas quedan libres al recargar. `sshd` y `recidive` (paso 2) siguen activas.

---

Con una aplicación nueva: [paso 5](05-nginx-sitio.md) y luego este paso 6. Si es Odoo:
[paso 7](07-odoo-seguridad.md). Índice: [README](README.md).
