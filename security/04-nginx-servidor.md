# Paso 4: Nginx, configuración del servidor

> Serie [Seguridad de un servidor web](README.md) · Anterior: [paso 3, Nginx y certificado](../manuals/ngnix_ssl_wildcard.md)
> · Siguiente: [paso 5, un sitio por aplicación](05-nginx-sitio.md)

Configuración de Nginx que se hace **una sola vez por máquina** y vale para cualquier aplicación: versión oculta, TLS,
tiempos, redirección a HTTPS y servers por defecto. Lo que cambia según la aplicación va en el
[paso 5](05-nginx-sitio.md). Probado en Ubuntu 24.04 con Nginx 1.24.

A diferencia de Certbot, acme.sh **no edita la configuración de Nginx**: no crea la redirección de HTTP a HTTPS ni
incluye ajustes de TLS. Todo eso se escribe a mano en este paso.

## Requisitos

- Paso 3 hecho: Nginx instalado y el certificado wildcard en `/etc/ssl/acme/jcondori.com.cer` y
  `/etc/ssl/acme/jcondori.com.key`, con el `--reloadcmd` de acme.sh recargando Nginx.
- Si el certificado viene de **Certbot** y no de acme.sh, lee antes la sección [Si usas Certbot](#si-usas-certbot):
  cambia cómo se hace el punto 4.3.

## Resumen

| Medida | Archivo |
|---|---|
| Ajustes globales: versión oculta, TLS 1.2/1.3, tiempos, zona de `limit_req` | `conf.d/00-base.conf` |
| Cabeceras del proxy, reutilizables por todos los sitios | `snippets/proxy.conf` |
| Puerto 80: dominio redirige a HTTPS, IP se bloquea | `sites-available/default` |
| Puerto 443 por defecto: cortar el handshake TLS | `conf.d/default-ssl-reject.conf` |

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

## 4.1 Ajustes globales

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

# Zona para limitar intentos de login por IP. Cada sitio la usa con limit_req en su ruta de login (paso 5).
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

## 4.2 Cabeceras del proxy

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
las cookies como `Secure`. Y en el paso 6, Fail2ban no podría bloquear por los logs de la aplicación.

## 4.3 Puerto 80: dominio a HTTPS, IP bloqueada

acme.sh valida el dominio por DNS (GoDaddy), así que el puerto 80 no hace falta para emitir ni renovar el certificado.
Solo sirve para que quien escriba `http://` con el dominio llegue a HTTPS. Ningún server debe servir contenido en el 80.

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

# Cualquier otro host (p.ej. por IP): cerrar la conexión sin responder.
server {
    listen 80 default_server;
    listen [::]:80 default_server;
    server_name _;
    return 444;
}
```

| Acceso | Resultado |
|---|---|
| `http://app.jcondori.com/ruta` | `301` a `https://app.jcondori.com/ruta` |
| `http://<IP>/ruta` | Conexión cerrada, sin respuesta |
| `http://otro-dominio.com/` apuntando a tu IP | Conexión cerrada, sin respuesta |

`444` es un código propio de Nginx: no envía nada, ni cabeceras ni cuerpo, y cierra la conexión. Quien escanea por IP no
recibe ninguna pista: ni el dominio (una redirección lo revelaría en la cabecera `Location`) ni el software del
servidor. Es el mismo criterio que el 443 por IP (4.4).

Si algún día quieres que quien escriba la IP a mano llegue al sitio, cambia `return 444;` por
`return 301 https://jcondori.com$request_uri;`. Redirige al **dominio**, no a `https://$host`: con `$host` el
navegador iría a `https://<IP>`, que el 4.4 rechaza. A cambio, cualquier bot que escanee la IP verá tu dominio.

Con esto, el `.conf` de cada sitio (paso 5) solo necesita el server del 443.

## 4.4 Puerto 443 por defecto: cortar el handshake

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

Junto con el `444` del puerto 80, el acceso por IP queda bloqueado en los dos puertos y sin revelar el dominio: el
certificado tampoco se llega a enviar. Los clientes sin SNI (muy antiguos) no
podrán entrar a ningún sitio; en la práctica no existen.

## Si usas Certbot

Para un servidor que ya tiene el certificado con Certbot (`certbot --nginx`) en vez de acme.sh. Certbot sí edita la
configuración de Nginx, y eso cambia tres cosas de este paso.

### TLS (4.1): nada que cambiar

Certbot añade `include /etc/letsencrypt/options-ssl-nginx.conf;` en el server de cada sitio. Ese archivo define
`ssl_protocols`, `ssl_ciphers` y `ssl_session_*` en el nivel `server`, y `00-base.conf` los define en el nivel `http`.
No chocan: Nginx solo rechaza una directiva repetida **en el mismo nivel**, y la del server manda sobre la de `http`
para ese sitio. Los valores de Certbot también son TLS 1.2/1.3, así que el resultado es el mismo. Deja el `include`.

### Puerto 80 (4.3): no reemplaces el archivo entero

`certbot --nginx` suele escribir el server del 443 del sitio **dentro de** `sites-available/default`, junto con su
propio server del 80 (`if ($host = ...) { return 301 ... }` y `return 404;`). Reemplazar el archivo entero, como dice el
4.3, borraría el sitio.

1. Mira qué hay en el archivo:

   ```bash
   grep -nE "server_name|listen|managed by Certbot" /etc/nginx/sites-available/default
   ```

2. Si contiene el server del 443 de un sitio, muévelo a su propio archivo `conf.d/<subdominio>.conf` (como la plantilla
   del [paso 5](05-nginx-sitio.md), con las rutas de Certbot) y valida con `nginx -t`.
3. Cambia los servers del 80 por los del 4.3, con una diferencia: en el del dominio, la redirección va dentro de
   `location /` y no a nivel de server, y se deja pasar el desafío de Let's Encrypt:

   ```nginx
   # Dominio y subdominios: desafío de Let's Encrypt aparte; todo lo demás, a HTTPS.
   server {
       listen 80;
       listen [::]:80;
       server_name jcondori.com *.jcondori.com;

       location /.well-known/acme-challenge/ {
           root /var/www/html;
       }

       location / {
           return 301 https://$host$request_uri;
       }
   }
   ```

   Certbot renueva por HTTP (desafío `http-01`): Let's Encrypt pide `http://<dominio>/.well-known/acme-challenge/...`.
   Un `return 301` a nivel de server se ejecuta antes que cualquier `location` y mandaría también esa petición a HTTPS,
   donde la aplicación respondería 404 y la renovación fallaría. El server por defecto con `return 444` (IP y otros
   hosts) no cambia: Let's Encrypt siempre pide por el dominio.

4. Prueba la renovación:

   ```bash
   certbot renew --dry-run
   ```

### Sitios nuevos (paso 5)

- Certificado: `ssl_certificate /etc/letsencrypt/live/<subdominio>/fullchain.pem;` y
  `ssl_certificate_key /etc/letsencrypt/live/<subdominio>/privkey.pem;`. Con `http-01`, Certbot no emite wildcard: un
  certificado por subdominio.
- Emítelo con `certbot certonly --nginx -d <subdominio>`. Con `certonly`, Certbot no toca la configuración y el `.conf`
  del sitio queda como lo escribiste. Sin `certonly`, volvería a meter sus bloques.
- La renovación la hace el timer de Certbot (`systemctl list-timers | grep certbot`) y recarga Nginx sola.

## Verificación del paso 4

Desde fuera del servidor (cambia la IP):

```bash
# Por IP en el 80: debe fallar sin respuesta (curl exit 52, "Empty reply from server")
curl -s -o /dev/null http://<IP>/; echo "exit=$?"

# Por IP en el 443: debe fallar (curl exit 35, handshake rechazado)
curl -sk -o /dev/null https://<IP>/; echo "exit=$?"

# Dominio en el 80: 301 a la misma URL por HTTPS
curl -s -o /dev/null -w "%{http_code} %{redirect_url}\n" http://jcondori.com/
```

Y que la renovación de acme.sh sigue recargando Nginx sin errores:

```bash
~/.acme.sh/acme.sh --renew -d "jcondori.com" --force
```

TLS y cabeceras se verifican sobre un sitio real, al final del paso 5.

## Deshacer

```bash
cp -a /root/nginx.conf.bak.<ts> /etc/nginx/nginx.conf
cp -a /root/nginx-default.bak.<ts> /etc/nginx/sites-available/default
rm /etc/nginx/conf.d/00-base.conf /etc/nginx/conf.d/default-ssl-reject.conf /etc/nginx/snippets/proxy.conf
nginx -t && systemctl reload nginx
```

Los sitios del paso 5 usan `$connection_upgrade` y la zona `login`, que vienen de `00-base.conf`, e incluyen
`snippets/proxy.conf`. Si borras esos archivos con sitios activos, `nginx -t` falla: borra o adapta primero los sitios.

**Siguiente:** [paso 5, un sitio por aplicación](05-nginx-sitio.md).
