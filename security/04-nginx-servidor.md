# Paso 4: Nginx, configuración del servidor

> Serie [Seguridad de un servidor web](README.md) · Anterior: [paso 3, Nginx y certificado](../manuals/ngnix_ssl_wildcard.md)
> · Siguiente: [paso 5, un sitio por aplicación](05-nginx-sitio.md)

Configuración de Nginx que se hace **una sola vez por máquina** y vale para cualquier aplicación: versión oculta, TLS,
tiempos, logs, redirección a HTTPS y servers por defecto. Lo que cambia según la aplicación va en el
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
| Logs: cada petición con host, tiempos, TLS e ID; handshakes rechazados; 90 días | `00-base.conf`, `default-ssl-reject.conf`, `/etc/logrotate.d/nginx` |

## Antes de tocar nada

Respalda cada archivo antes de editarlo y valida antes de recargar. Un `nginx -t` fallido no corta el servicio, pero un
`reload` con una configuración rota sí puede dejar el sitio caído tras el próximo reinicio.

```bash
ts=$(date +%Y%m%d%H%M%S)
cp -a /etc/nginx/nginx.conf /root/nginx.conf.bak.$ts
cp -a /etc/nginx/sites-available/default /root/nginx-default.bak.$ts
cp -a /etc/logrotate.d/nginx /root/logrotate-nginx.bak.$ts
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

# Log de accesos: formato "combined" (Fail2ban lo lee igual) + campos de diagnóstico al final. Ver 4.5.
log_format main_ext '$remote_addr - $remote_user [$time_local] "$request" $status $body_bytes_sent '
                    '"$http_referer" "$http_user_agent" '
                    'host="$host" rt=$request_time urt="$upstream_response_time" '
                    'us="$upstream_status" tls="$ssl_protocol" conn=$connection rid=$request_id';
access_log /var/log/nginx/access.log main_ext;
```

`conf.d/*.conf` se incluye dentro del mismo bloque `http { }` que `nginx.conf`, y Nginx rechaza una directiva repetida
en el mismo nivel (`"ssl_protocols" directive is duplicate`). Antes de crear el archivo, comenta en `nginx.conf` las que
ya estén ahí (las líneas ya comentadas no se tocan):

```bash
sed -i -E 's/^(\s*)(server_tokens|ssl_protocols|ssl_ciphers|ssl_prefer_server_ciphers|ssl_session_cache|ssl_session_timeout|ssl_session_tickets|client_body_timeout|client_header_timeout|send_timeout|keepalive_timeout|access_log)\b/\1# \2/' /etc/nginx/nginx.conf
nginx -t
```

`access_log` es el caso traicionero: **no** da error si se repite, porque es acumulativa. Con la línea de `nginx.conf`
activa, cada petición se escribiría dos veces en `access.log` (una por formato), y Fail2ban contaría doble.

Notas:

- No actives `ssl_stapling`: Let's Encrypt dejó de publicar OCSP en 2025, y Nginx avisaría en cada recarga.
- El nombre empieza por `00-` para que se cargue antes que los sitios (`conf.d/*.conf` va en orden alfabético) y el
  `map` y la zona ya existan cuando un sitio los usa.
- **Muchos subdominios:** con varios sitios (o subdominios largos) Nginx puede fallar al recargar con
  `could not build server_names_hash, you should increase server_names_hash_bucket_size`. Añade en este archivo
  `server_names_hash_bucket_size 128;` (o 64) y recarga.

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
proxy_set_header X-Request-ID $request_id;
```

Sin estas cabeceras, la aplicación ve todas las peticiones como `127.0.0.1` por HTTP: no puede limitar por IP ni marcar
las cookies como `Secure`. Y en el paso 6, Fail2ban no podría bloquear por los logs de la aplicación.

`X-Request-ID` lleva a la aplicación el mismo ID que Nginx escribe en `rid=` (4.5). Si la aplicación lo registra en su
log, una petición se sigue de Nginx a la aplicación con un solo `grep`. Nginx lo genera siempre: un `X-Request-ID` que
mande el cliente se sustituye, así que no se puede falsificar.

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

    # Handshakes rechazados (acceso por IP): no llegan a ser peticiones, así que solo quedan aquí (nivel info).
    error_log /var/log/nginx/tls-reject.log info;
}
```

Junto con el `444` del puerto 80, el acceso por IP queda bloqueado en los dos puertos y sin revelar el dominio: el
certificado tampoco se llega a enviar. Los clientes sin SNI (muy antiguos) no
podrán entrar a ningún sitio; en la práctica no existen.

**Valida que el `default_server` del 443 sea este, no una aplicación.** Solo puede haber un `default_server` por puerto.
Si un sitio quedó con `listen 443 ssl default_server` y `server_name _` (a propósito o por descuido), es él quien
responde a cualquier `https://<IP>` o host desconocido, sirviendo la aplicación con el certificado wildcard. Corrígelo:
quítale `default_server`, dale un `server_name` explícito (su subdominio) y deja que este server de rechazo sea el único
por defecto. Compruébalo:

```bash
grep -rn "default_server" /etc/nginx/
```

En el 443 debe aparecer **solo** el `default-ssl-reject.conf`. Tras el cambio, la aplicación responde únicamente a su
subdominio nombrado; el acceso por IP cae en el rechazo.

## 4.5 Logs

Objetivo: que **cada petición** quede registrada con lo necesario para medir ataques y depurar, y que se conserve lo
bastante para ver tendencias. Nginx ya registra cada petición HTTP en `access.log` (también las que acaban en `444`),
pero con el formato `combined` faltan datos clave y quedan fuera los intentos que no llegan a ser petición.

### Formato `main_ext`

Definido en `00-base.conf` (4.1). Las primeras columnas son **idénticas** a `combined`, así que los filtros de Fail2ban
(paso 6) y cualquier herramienta que lea ese formato siguen funcionando. Los campos nuevos van al final:

| Campo | Variable | Para qué |
|---|---|---|
| `host=` | `$host` | Qué se pidió: el subdominio, o la IP (escaneo). Sin él no se distinguen los sitios en un log compartido |
| `rt=` | `$request_time` | Duración total en segundos, cliente incluido |
| `urt=` | `$upstream_response_time` | Lo que tardó la aplicación. `rt` alto con `urt` bajo = cliente lento o red |
| `us=` | `$upstream_status` | Código que dio la aplicación. `-` = Nginx respondió solo (444, 429, redirect del 80) |
| `tls=` | `$ssl_protocol` | `TLSv1.3`/`TLSv1.2`; `-` = petición por HTTP |
| `conn=` | `$connection` | Número de conexión: agrupa las peticiones de un mismo keep-alive. Coincide con el `*N` de `error.log` |
| `rid=` | `$request_id` | ID único de la petición, el mismo que recibe la aplicación en `X-Request-ID` (4.2) |

Ejemplo real:

```text
190.0.2.10 - - [08/Nov/2026:20:55:07 +0000] "GET /.git/config HTTP/1.1" 302 33 "-" "curl/8.16.0" host="app.jcondori.com" rt=0.002 urt="0.002" us="302" tls="TLSv1.3" conn=15895 rid=11b9ca63dc80a2ff6b8ac4374f0bb26b
```

Un solo `access.log` para todos los sitios, a propósito: las jails del paso 6 leen un archivo, y `host=` ya separa los
sitios. Si un sitio necesita su propio archivo, pon `access_log /var/log/nginx/<sitio>.log main_ext;` en su server y
añade esa ruta al `logpath` de las jails, o Fail2ban dejará de ver sus escaneos.

### Lo que no es una petición

| Evento | Dónde queda |
|---|---|
| Handshake TLS rechazado (acceso por IP al 443, 4.4) | `tls-reject.log`: `handshake rejected while SSL handshaking, client: <IP>` |
| Petición rechazada por `limit_req` | `error.log` (`limiting requests`) **y** `access.log` con `429` |
| Petición mal formada (basura, TLS enviado al puerto 80) | `access.log` con `400` y petición `"-"` |
| Conexión que se abre y se cierra sin mandar nada | En Nginx, solo con `error_log` a nivel `info` (ver abajo) |
| Escaneo de puertos (la conexión TCP ni se completa) | Nunca llega a Nginx: solo lo ve el firewall (ver abajo) |

#### Por qué el `error_log` global se queda en su nivel por defecto

- **Nginx no lo considera un error.** Desde 2013, una conexión cerrada sin datos ya no genera un `400` en
  `access.log`: los navegadores abren conexiones de reserva que a veces no usan. Solo aparece a nivel `info`
  (`client closed connection while waiting for request`).
- **CIS NGINX Benchmark (3.3)** pide `error_log ... info` en sus versiones 1 y 2; la v3 acepta `notice` o `info` según la
  política de la organización y avisa del volumen. Si hay que cumplir CIS, `error_log /var/log/nginx/error.log info;`
  en `nginx.conf` es todo lo que hace falta: Fail2ban (paso 6) sigue leyendo `error.log` igual.
- **Medido en un servidor pequeño** (20 min con un log `info` aparte): 22 líneas, unos 300 KB al día. 19 eran
  peticiones mal formadas que ya estaban en `access.log` como `400`; conexiones vacías, **ninguna**. En un servidor con
  mucho tráfico, además, cada keep-alive que se cierra deja una línea (`closed keepalive connection`).

Conclusión: a nivel `info` Nginx apenas añade nada al `access.log`, así que se queda en su nivel por defecto. El nivel
`info` se aplica solo al server del 4.4, que no tiene otro tráfico. Si alguna vez hace falta medirlo, se puede añadir
un log aparte durante un rato, sin tocar el `error.log` (a nivel `http`, una directiva `error_log` reemplaza a la
heredada, por eso van las dos):

```nginx
# conf.d/zz-measure-info.conf  (temporal: bórralo después)
error_log /var/log/nginx/error.log;
error_log /var/log/nginx/info-measure.log info;
```

#### Intentos que no llegan a Nginx: el firewall

Los escaneos de puertos y las conexiones que se cortan antes de la petición se registran en la capa de red, no en el
servidor web. Dónde verlos depende del proveedor:

- **Firewall de la nube con logging** (la mayoría: GCP, AWS, Azure, Oracle Cloud; algunos VPS no lo ofrecen): activa el
  registro de la regla que permite el tráfico y, si se quiere ver los escaneos a puertos cerrados, una regla de
  **denegación explícita** con logging (el *deny* implícito no suele poder registrarse). El coste lo cobra el servicio de
  logs del proveedor por volumen; reduce los metadatos si se puede. Se consultan en el panel de logs del proveedor, no en
  la VM.
- **Sin firewall de nube** (filtras con nftables/ufw en la VM): añade una regla `log` antes del `drop` para registrar los
  intentos a puertos cerrados en el journal.

En GCP, en concreto (VPC network → Firewall, o `gcloud`): **Firewall Rules Logging** en `default-allow-http`/`-https`/`-ssh`
registra cada conexión permitida; una **deny-all explícita** de prioridad baja con logging registra los escaneos; **VPC
Flow Logs** da una muestra de la subred. Se consultan en Logs Explorer (`logName:"compute.googleapis.com%2Ffirewall"`).

Nada de esto se activa en esta serie: el volumen y el coste dependen del proyecto.

### Retención

Ubuntu rota los logs de Nginx a diario y guarda **14 días**: poco para ver tendencias o investigar algo que se descubre
tarde. Súbelo a 90 (comprimidos ocupan poco: un servidor pequeño con escaneos genera ~1 MB al día sin comprimir):

```bash
sed -i 's/^\(\s*\)rotate 14$/\1rotate 90/' /etc/logrotate.d/nginx
logrotate -d /etc/logrotate.d/nginx 2>&1 | grep "rotating pattern"   # debe decir (90 rotations)
```

El patrón `/var/log/nginx/*.log` incluye `tls-reject.log` sin tocar nada. Fail2ban sigue al archivo nuevo tras cada
rotación (paso 6).

### Privacidad

Los logs guardan la URL completa, con su query string. Una aplicación que acepte secretos en la URL (`?token=...`)
los deja escritos durante 90 días: mejor que vayan en una cabecera o en el cuerpo, que Nginx no registra. Los logs son
`640 www-data:adm`; no los copies fuera del servidor sin pensarlo.

### Consultas útiles

```bash
cd /var/log/nginx
L="access.log"                       # o: zcat -f access.log* para todo el histórico

# Peticiones por código (cuánto es escaneo: 444, 404, 302 a login…)
awk '{print $9}' $L | sort | uniq -c | sort -rn

# Top IPs y top rutas que acaban en 404/444
awk '{print $1}' $L | sort | uniq -c | sort -rn | head
awk '$9==404||$9==444 {print $7}' $L | sed 's/?.*//' | sort | uniq -c | sort -rn | head -20

# Peticiones por host: escaneos por IP frente a cada sitio
grep -oE 'host="[^"]*"' $L | sort | uniq -c | sort -rn

# Peticiones lentas (> 2 s) y cuánto puso la aplicación
grep -E ' rt=([2-9]|[0-9]{2,})\.' $L

# Todo lo de una petición o una conexión
grep 'rid=<id>' $L
grep -E 'conn=15895 |\*15895 ' access.log error.log

# Handshakes rechazados por día
grep -c "handshake rejected" tls-reject.log
```

Para métricas continuas (paneles, alertas), estos mismos campos se pueden enviar a una herramienta externa (GoAccess,
Loki, New Relic…). Eso queda fuera de esta serie.

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
   del [paso 5](05-nginx-sitio.md), con las rutas de Certbot) y valida con `nginx -t`. Copia tal cual las líneas
   `# managed by Certbot` (`ssl_certificate*`, `include options-ssl-nginx.conf`, `ssl_dhparam`): el `include` trae cifrados DHE,
   que necesitan el `ssl_dhparam`.
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
   grep -E "authenticator|installer" /etc/letsencrypt/renewal/*.conf
   certbot renew --dry-run --no-random-sleep-on-renew
   ```

   La primera línea dice cómo renueva Certbot. Con `authenticator = nginx` (lo que deja `certbot --nginx`), el plugin
   inyecta su propio `location` exacto para el desafío, así que el `location /.well-known/acme-challenge/` de arriba no
   se usa, pero tampoco estorba. Con `webroot`, es el que responde.

   `--no-random-sleep-on-renew` hace falta cuando no hay terminal (por ejemplo, `ssh host 'certbot renew ...'`): sin él,
   Certbot espera hasta 8 minutos al azar antes de empezar (`Non-interactive renewal: random delay of ... seconds`).

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

En el servidor, que esas tres peticiones quedaron registradas, una sola vez cada una y con los campos de 4.5:

```bash
grep "<tu IP>" /var/log/nginx/access.log | tail -3       # deben terminar en host="..." ... rid=...
tail -1 /var/log/nginx/tls-reject.log                    # handshake rejected ..., client: <tu IP>
```

Para probar logs no pidas rutas trampa (`/.env`, `/.git/...`) desde tu IP si el paso 6 ya está hecho: con dos
peticiones `nginx-scan` te bloquea un día.

Y que la renovación de acme.sh sigue recargando Nginx sin errores:

```bash
~/.acme.sh/acme.sh --renew -d "jcondori.com" --force
```

TLS y cabeceras se verifican sobre un sitio real, al final del paso 5.

## Deshacer

```bash
cp -a /root/nginx.conf.bak.<ts> /etc/nginx/nginx.conf
cp -a /root/nginx-default.bak.<ts> /etc/nginx/sites-available/default
cp -a /root/logrotate-nginx.bak.<ts> /etc/logrotate.d/nginx
rm /etc/nginx/conf.d/00-base.conf /etc/nginx/conf.d/default-ssl-reject.conf /etc/nginx/snippets/proxy.conf
nginx -t && systemctl reload nginx
```

Los sitios del paso 5 usan `$connection_upgrade` y la zona `login`, que vienen de `00-base.conf`, e incluyen
`snippets/proxy.conf`. Si borras esos archivos con sitios activos, `nginx -t` falla: borra o adapta primero los sitios.

**Siguiente:** [paso 5, un sitio por aplicación](05-nginx-sitio.md).
