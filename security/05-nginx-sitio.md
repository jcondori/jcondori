# Paso 5: Nginx, un sitio por aplicación

> Serie [Seguridad de un servidor web](README.md) · Anterior: [paso 4, configuración del servidor](04-nginx-servidor.md)
> · Siguiente: [paso 6, Fail2ban web](06-fail2ban-web.md)

Un archivo por subdominio con el server del 443: certificado, cabeceras, proxy y límites. Las líneas con `AJUSTAR`
cambian según la aplicación; el resto se deja tal cual. Se repite con cada aplicación nueva.

## Requisitos

- Paso 4 hecho: existen `conf.d/00-base.conf` (zona `login`, `$connection_upgrade`) y `snippets/proxy.conf`.
- La aplicación corriendo en un puerto local.

## 5.1 Plantilla

`/etc/nginx/conf.d/<subdominio>.jcondori.com.conf`:

```nginx
server {
    listen 443 ssl;
    listen [::]:443 ssl;
    server_name app.jcondori.com;                      # AJUSTAR: subdominio

    ssl_certificate     /etc/ssl/acme/jcondori.com.cer;
    ssl_certificate_key /etc/ssl/acme/jcondori.com.key;

    # Cabeceras de seguridad. AJUSTAR: borra las que ya envíe la aplicación.
    add_header Strict-Transport-Security "max-age=31536000" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;

    client_max_body_size 10m;                          # AJUSTAR: subida más grande que acepta la aplicación

    location / {
        proxy_pass http://127.0.0.1:3000;              # AJUSTAR: puerto de la aplicación
        include snippets/proxy.conf;
    }

    # Límite de intentos de login. AJUSTAR: ruta de login de la aplicación, o borra el bloque.
    location = /login {
        limit_req zone=login burst=5 nodelay;
        limit_req_status 429;
        proxy_pass http://127.0.0.1:3000;              # AJUSTAR: mismo puerto que arriba
        include snippets/proxy.conf;
    }
}
```

El redirect del puerto 80 ya lo cubre el server `*.jcondori.com` del paso 4.

## 5.2 Qué ajustar por aplicación

| Línea | Qué decidir |
|---|---|
| `server_name` | El subdominio del sitio |
| `proxy_pass` | Puerto local de la aplicación, en `127.0.0.1` |
| Cabeceras `add_header` | Si la aplicación ya las envía, bórralas aquí (ver abajo) |
| `client_max_body_size` | La subida más grande que la aplicación acepta |
| `location = /login` | La ruta de login de la aplicación; sin login propio, borra el bloque |
| Peticiones largas | `proxy_read_timeout` solo en el `location` que lo necesite |
| Archivos estáticos con `root` | Solo entonces, bloquear archivos ocultos (ver abajo) |

### Cabeceras

Pon cada cabecera **en un solo sitio**: o la envía la aplicación (por ejemplo, un middleware de Express) o la añade
Nginx. Si la ponen los dos, el navegador la recibe duplicada. Para ver qué envía la aplicación sin Nginx delante:

```bash
curl -sI http://127.0.0.1:3000/
```

- `always` las envía también en respuestas de error (4xx, 5xx).
- Un `location` que define su propio `add_header` **no hereda** ninguno del server: hay que repetirlos todos ahí.
- `X-Frame-Options "SAMEORIGIN"` deja que la aplicación se muestre en iframes propios (el editor web de Odoo los usa).
  Si la aplicación nunca va en un iframe, `DENY` es más estricto.
- La `Content-Security-Policy` no va en la plantilla: depende de qué scripts externos carga cada aplicación. Mejor que la
  defina la aplicación.
- HSTS solo en el 443, nunca en el 80. `includeSubDomains` obliga a HTTPS a todos los subdominios durante todo el
  `max-age`, también los de otros servidores: añádelo solo si ninguno necesita seguir por HTTP.

### Archivos ocultos

Solo si el sitio sirve archivos con `root` (no en un proxy puro). Bloquea `.git`, `.env`, `.htpasswd` y deja el
listado de directorios apagado (`autoindex off` es el default; no lo actives):

```nginx
location ~ /\. {
    deny all;
}
```

Si el sitio publica archivos en `/.well-known/` (por ejemplo `security.txt`), usa `location ~ /\.(?!well-known)`.

## 5.3 Configurar la aplicación

Dos cosas en cada aplicación:

1. **Escuchar solo en `127.0.0.1`**, no en `0.0.0.0`. Si alguien abre el puerto en el firewall por error, la aplicación
   sigue sin ser alcanzable desde fuera.
2. **Confiar en las cabeceras `X-Forwarded-*` del proxy**, para ver la IP real y saber que la petición llegó por HTTPS.
   Esto solo es seguro por el punto 1: si la aplicación fuera alcanzable directamente, cualquiera podría falsificar
   `X-Forwarded-For`. El paso 6 también lo necesita: sin la IP real en el log de la aplicación, Fail2ban no tiene a
   quién bloquear.

| Aplicación | Solo localhost | Confiar en el proxy |
|---|---|---|
| Node/Express | `app.listen(PORT, '127.0.0.1')` o `HOST=127.0.0.1` en `.env` | `app.set('trust proxy', 'loopback')` |
| Odoo | `http_interface = 127.0.0.1` en el `.conf` | `proxy_mode = True` en el `.conf` |

En `ss -tulnp`, el puerto de la aplicación debe aparecer como `127.0.0.1:<puerto>`.

## 5.4 Ejemplo: Odoo

Lo que cambia respecto a la plantilla:

```nginx
server {
    listen 443 ssl;
    listen [::]:443 ssl;
    server_name odoo.jcondori.com;

    ssl_certificate     /etc/ssl/acme/jcondori.com.cer;
    ssl_certificate_key /etc/ssl/acme/jcondori.com.key;

    add_header Strict-Transport-Security "max-age=31536000" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;

    client_max_body_size 500m;                 # importación de bases y adjuntos grandes

    location / {
        proxy_pass http://127.0.0.1:8069;
        include snippets/proxy.conf;
        proxy_read_timeout 720s;               # reportes y exportaciones largas
    }

    location /websocket {
        proxy_pass http://127.0.0.1:8072;      # gevent_port (Odoo 16 o superior)
        include snippets/proxy.conf;
        proxy_read_timeout 720s;
    }

    location = /web/login {
        limit_req zone=login burst=5 nodelay;
        limit_req_status 429;
        proxy_pass http://127.0.0.1:8069;
        include snippets/proxy.conf;
    }
}
```

En el `.conf` de Odoo: `proxy_mode = True`, `http_interface = 127.0.0.1` y `list_db = False` en producción (oculta el
gestor de bases de datos).

## Verificación del paso 5

Hazla **antes** del paso 6: la prueba del límite de login provoca bloqueos a propósito, y con las jails web de Fail2ban
activas bloquearía tu propia IP.

Desde fuera del servidor:

```bash
# Subdominio en el 80: 301 a la misma URL por HTTPS
curl -s -o /dev/null -w "%{http_code} %{redirect_url}\n" http://app.jcondori.com/

# Subdominio en el 443: responde la aplicación, cabecera Server sin versión, cabeceras sin duplicar
curl -sI https://app.jcondori.com/ | grep -iE "^(HTTP|server|strict-transport|x-frame|x-content|referrer)"

# TLS 1.1 rechazado y TLS 1.2 aceptado
openssl s_client -connect app.jcondori.com:443 -servername app.jcondori.com -tls1_1 </dev/null
openssl s_client -connect app.jcondori.com:443 -servername app.jcondori.com -tls1_2 </dev/null | grep -E "Protocol|Cipher"

# Límite de login: tras unos intentos seguidos debe aparecer 429
for i in $(seq 1 10); do curl -s -o /dev/null -w "%{http_code} " https://app.jcondori.com/login; done; echo
```

Y en el servidor, que la aplicación registra la IP real del cliente (no `127.0.0.1`) en su log.

Para una revisión externa de TLS: [SSL Labs](https://www.ssllabs.com/ssltest/) (objetivo: A o A+). Para las cabeceras:
[securityheaders.com](https://securityheaders.com/).

## Deshacer

Borra el `.conf` del sitio y recarga:

```bash
rm /etc/nginx/conf.d/<subdominio>.jcondori.com.conf
nginx -t && systemctl reload nginx
```

**Siguiente:** [paso 6, Fail2ban para Nginx y las aplicaciones](06-fail2ban-web.md).
