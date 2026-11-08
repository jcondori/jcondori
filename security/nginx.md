# Configuración de seguridad para Nginx

Checklist para endurecer un Nginx que hace de proxy inverso con HTTPS delante de una o varias aplicaciones (Node, Odoo,
etc.), con el certificado emitido por **acme.sh**. Probado en Ubuntu 24.04 con Nginx 1.24.

Para instalar Nginx y emitir el certificado wildcard con acme.sh, ver
[ngnix_ssl_wildcard.md](../manuals/ngnix_ssl_wildcard.md). Esta guía usa sus rutas: el certificado en
`/etc/ssl/acme/jcondori.com.cer` y la llave en `/etc/ssl/acme/jcondori.com.key`.

A diferencia de Certbot, acme.sh **no edita la configuración de Nginx**: no crea la redirección de HTTP a HTTPS ni
incluye ajustes de TLS. Todo eso se escribe a mano, y por eso está en esta guía.

La guía tiene dos partes:

- **Parte 1, servidor:** se hace una sola vez por máquina y vale para cualquier aplicación.
- **Parte 2, por cada sitio:** una plantilla por subdominio, con lo que cambia según la aplicación marcado con `AJUSTAR`.

## Resumen

| Parte | Medida | Archivo |
|---|---|---|
| 1 | Firewall: solo 22, 80 y 443 abiertos | Nube (GCP/AWS) |
| 1 | Ajustes globales: versión oculta, TLS 1.2/1.3, tiempos, zona de `limit_req` | `conf.d/00-base.conf` |
| 1 | Cabeceras del proxy, reutilizables por todos los sitios | `snippets/proxy.conf` |
| 1 | Puerto 80: todo redirige a HTTPS | `sites-available/default` |
| 1 | Puerto 443 por defecto: cortar el handshake TLS | `conf.d/default-ssl-reject.conf` |
| 1 | SSH solo con llave | `sshd_config` |
| 2 | Sitio: certificado, cabeceras, proxy y límites | `conf.d/<subdominio>.conf` |
| 2 | Aplicación solo en `127.0.0.1` y confiando en el proxy | Configuración de la aplicación |

## Antes de tocar nada

Respalda cada archivo antes de editarlo y valida antes de recargar. Un `nginx -t` fallido no corta el servicio, pero un
`reload` con una configuración rota sí puede dejar el sitio caído tras el próximo reinicio.

```bash
ts=$(date +%Y%m%d%H%M%S)
cp -a /etc/nginx/nginx.conf /root/nginx.conf.bak.$ts
cp -a /etc/nginx/sites-available/default /root/nginx-default.bak.$ts
```

Después de cada cambio:

```bash
nginx -t && systemctl reload nginx
```

---

# Parte 1: servidor (una vez)

## 1.1 Firewall

Solo deben estar abiertos al mundo:

| Puerto | Uso |
|---|---|
| tcp/22 | SSH (solo con llave, ver 1.6) |
| tcp/80 | Solo redirección a HTTPS (acme.sh valida por DNS, no usa este puerto) |
| tcp/443 | Sitios |

Los puertos internos de las aplicaciones (3000, 8069, etc.) **nunca** se abren en el firewall. En GCP, las reglas se
aplican por etiqueta de red: revisa que la VM no tenga etiquetas de otras reglas (por ejemplo, una regla de juegos que
abre el 3000).

Para ver qué escucha realmente la máquina:

```bash
ss -tulnp
```

Hacia fuera solo deben aparecer `0.0.0.0` o `[::]` en los puertos 22, 80 y 443. El resto, en `127.0.0.1`.

## 1.2 Ajustes globales

Todo lo que vale para todos los sitios va en un solo archivo dentro del bloque `http { }`. Así no se repite en cada
subdominio.

`/etc/nginx/conf.d/00-base.conf`:

```nginx
# --- Ajustes globales de seguridad (todos los sitios) ---

# No mostrar la versión de Nginx en la cabecera Server ni en las páginas de error.
server_tokens off;

# TLS: perfil "intermediate" de Mozilla, solo ECDHE (no necesita ssl_dhparam).
ssl_protocols TLSv1.2 TLSv1.3;
ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305;
ssl_prefer_server_ciphers off;
ssl_session_cache shared:SSL:10m;
ssl_session_timeout 1d;
ssl_session_tickets off;

# Cortar clientes lentos (slowloris). Un sitio con peticiones largas sube proxy_read_timeout en su location.
client_body_timeout 15s;
client_header_timeout 15s;
send_timeout 30s;
keepalive_timeout 30s;

# Zona para limitar intentos de login por IP. Cada sitio la usa con limit_req en su ruta de login.
limit_req_zone $binary_remote_addr zone=login:10m rate=10r/m;

# WebSocket: Connection "upgrade" solo cuando el cliente lo pide.
map $http_upgrade $connection_upgrade {
    default upgrade;
    ''      close;
}
```

`conf.d/*.conf` se incluye dentro del mismo bloque `http { }` que `nginx.conf`, y Nginx rechaza una directiva repetida
en el mismo nivel (`"ssl_protocols" directive is duplicate`). Antes de crear el archivo, comenta en `nginx.conf` las que
ya estén ahí (las líneas ya comentadas no se tocan):

```bash
sed -i -E 's/^(\s*)(server_tokens|ssl_protocols|ssl_ciphers|ssl_prefer_server_ciphers|ssl_session_cache|ssl_session_timeout|ssl_session_tickets|client_body_timeout|client_header_timeout|send_timeout|keepalive_timeout)\b/\1# \2/' /etc/nginx/nginx.conf
nginx -t
```

Notas:

- No actives `ssl_stapling`: Let's Encrypt dejó de publicar OCSP en 2025, y Nginx avisaría en cada recarga.
- El nombre empieza por `00-` para que se cargue antes que los sitios (`conf.d/*.conf` va en orden alfabético) y el
  `map` y la zona ya existan cuando un sitio los usa.

## 1.3 Cabeceras del proxy

Todos los sitios pasan las mismas cabeceras a su aplicación. Van en un snippet para incluirlo en cada `location`.

`/etc/nginx/snippets/proxy.conf`:

```nginx
proxy_http_version 1.1;
proxy_set_header Host $host;
proxy_set_header X-Real-IP $remote_addr;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
proxy_set_header X-Forwarded-Proto $scheme;
proxy_set_header X-Forwarded-Host $host;
proxy_set_header Upgrade $http_upgrade;
proxy_set_header Connection $connection_upgrade;
```

Sin estas cabeceras, la aplicación ve todas las peticiones como `127.0.0.1` por HTTP: no puede limitar por IP ni marcar
las cookies como `Secure`.

## 1.4 Puerto 80: todo redirige a HTTPS

acme.sh valida el dominio por DNS (GoDaddy), así que el puerto 80 no hace falta para emitir ni renovar el certificado.
Solo sirve para que quien escriba `http://` llegue a HTTPS. Ningún server debe servir contenido en el 80.

Nginx usa el server `default_server` para cualquier petición cuyo `Host` no coincide con ningún `server_name` (por
ejemplo, quien entra por la IP). En Ubuntu ese server muestra la página de bienvenida de Nginx.

Reemplaza el contenido de `/etc/nginx/sites-available/default` por dos servers:

```nginx
# Dominio y subdominios: misma URL, pero por HTTPS.
server {
    listen 80;
    listen [::]:80;
    server_name jcondori.com *.jcondori.com;
    return 301 https://$host$request_uri;
}

# Cualquier otro host (p.ej. por IP): al dominio principal por HTTPS.
server {
    listen 80 default_server;
    listen [::]:80 default_server;
    server_name _;
    return 301 https://jcondori.com$request_uri;
}
```

El server por defecto redirige al **dominio**, no a `https://$host`: con `$host` el navegador iría a `https://<IP>`, que
el punto 1.5 rechaza.

Con esto, el `.conf` de cada sitio solo necesita el server del 443.

## 1.5 Puerto 443 por defecto: cortar el handshake

Si no hay un server por defecto en el 443, Nginx usa el primer sitio y lo sirve a cualquiera que entre por
`https://<IP>`, con un certificado que no coincide. Crea uno que rechace la conexión sin mostrar ningún certificado.
Requiere Nginx 1.19.4 o superior.

`/etc/nginx/conf.d/default-ssl-reject.conf`:

```nginx
# HTTPS a un host que no es un sitio configurado (p.ej. por IP): cortar el handshake TLS.
server {
    listen 443 ssl default_server;
    listen [::]:443 ssl default_server;
    server_name _;
    ssl_reject_handshake on;
}
```

Esto también evita que el certificado revele el dominio a quien escanea IPs. Los clientes sin SNI (muy antiguos) no
podrán entrar a ningún sitio; en la práctica no existen.

## 1.6 SSH

No es de Nginx, pero va junto en cualquier servidor expuesto. En `sshd -T`:

| Opción | Valor |
|---|---|
| `passwordauthentication` | `no` |
| `kbdinteractiveauthentication` | `no` |
| `permitrootlogin` | `prohibit-password` (o `without-password`) o `no` |
| `permitemptypasswords` | `no` |
| `x11forwarding` | `no` |

```bash
sshd -T | grep -Ei "^(passwordauthentication|kbdinteractiveauthentication|permitrootlogin|permitemptypasswords|x11forwarding) "
```

Después de editar `/etc/ssh/sshd_config`, valida y recarga **sin cerrar la sesión actual**, y prueba una conexión nueva
en otra terminal:

```bash
sshd -t && systemctl reload ssh
```

## 1.7 Verificación del servidor

Desde fuera del servidor (cambia la IP):

```bash
# Por IP en el 80: 301 al dominio por HTTPS
curl -s -o /dev/null -w "%{http_code} %{redirect_url}\n" http://<IP>/

# Por IP en el 443: debe fallar (curl exit 35, handshake rechazado)
curl -sk -o /dev/null https://<IP>/; echo "exit=$?"
```

---

# Parte 2: por cada sitio

## 2.1 Plantilla

Un archivo por subdominio en `/etc/nginx/conf.d/<subdominio>.jcondori.com.conf`. Las líneas con `AJUSTAR` cambian según
la aplicación (ver 2.2); el resto se deja tal cual.

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

El redirect del puerto 80 ya lo cubre el server `*.jcondori.com` de la parte 1.

## 2.2 Qué ajustar por aplicación

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

## 2.3 Configurar la aplicación

Dos cosas en cada aplicación:

1. **Escuchar solo en `127.0.0.1`**, no en `0.0.0.0`. Si alguien abre el puerto en el firewall por error, la aplicación
   sigue sin ser alcanzable desde fuera.
2. **Confiar en las cabeceras `X-Forwarded-*` del proxy**, para ver la IP real y saber que la petición llegó por HTTPS.
   Esto solo es seguro por el punto 1: si la aplicación fuera alcanzable directamente, cualquiera podría falsificar
   `X-Forwarded-For`.

| Aplicación | Solo localhost | Confiar en el proxy |
|---|---|---|
| Node/Express | `app.listen(PORT, '127.0.0.1')` o `HOST=127.0.0.1` en `.env` | `app.set('trust proxy', 'loopback')` |
| Odoo | `http_interface = 127.0.0.1` en el `.conf` | `proxy_mode = True` en el `.conf` |

En `ss -tulnp`, el puerto de la aplicación debe aparecer como `127.0.0.1:<puerto>`.

## 2.4 Ejemplo: Odoo

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

## 2.5 Verificación del sitio

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

Y que la renovación de acme.sh sigue recargando Nginx sin errores:

```bash
~/.acme.sh/acme.sh --renew -d "jcondori.com" --force
```

Para una revisión externa de TLS: [SSL Labs](https://www.ssllabs.com/ssltest/) (objetivo: A o A+). Para las cabeceras:
[securityheaders.com](https://securityheaders.com/).

---

## Deshacer

Restaura el respaldo del archivo afectado, o borra el archivo nuevo, y recarga:

```bash
cp -a /root/nginx.conf.bak.<ts> /etc/nginx/nginx.conf
cp -a /root/nginx-default.bak.<ts> /etc/nginx/sites-available/default
rm /etc/nginx/conf.d/00-base.conf /etc/nginx/conf.d/default-ssl-reject.conf /etc/nginx/snippets/proxy.conf
nginx -t && systemctl reload nginx
```

Los sitios de la parte 2 usan `$connection_upgrade` y la zona `login`, que vienen de `00-base.conf`, e incluyen
`snippets/proxy.conf`. Si borras esos archivos con sitios activos, `nginx -t` falla: borra o adapta primero los sitios.

Para quitar un sitio, borra su `conf.d/<subdominio>.jcondori.com.conf` y recarga.
