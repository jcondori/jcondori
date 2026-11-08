# Nginx con SSL wildcard y renovación automática — acme.sh (GoDaddy)

> Paso 3 de la serie [Seguridad de un servidor web](../security/README.md) · Anterior:
> [paso 2, Fail2ban base](../security/02-fail2ban-base.md) · Siguiente:
> [paso 4, Nginx: configuración del servidor](../security/04-nginx-servidor.md)
>
> **Si el servidor ya tiene el certificado con Certbot** (`/etc/letsencrypt/live/...`), sáltate este manual y sigue con
> el paso 4, aplicando su sección [Si usas Certbot](../security/04-nginx-servidor.md#si-usas-certbot).

Este manual documenta cómo emitir un certificado SSL wildcard (`*.jcondori.com`) con **Let's Encrypt**
usando `acme.sh` y la API de GoDaddy, con renovación automática sin intervención manual.

Se usa el fork [`jcondori/acme.sh`](https://github.com/jcondori/acme.sh) en lugar del acme.sh oficial,
porque agrega a `dns_gd`:

- Soporte de **Personal Access Tokens** (PAT) de GoDaddy (`GD_Token`), que reemplazan al par Key/Secret.
- Token guardado **por certificado**: cada dominio puede usar un token distinto (por ejemplo, dominios
  en cuentas GoDaddy diferentes) sin que un certificado sobrescriba el token de otro.

## Instalación de Nginx

```bash
sudo apt update
sudo apt install nginx -y
```

## Certificado wildcard con renovación automática — acme.sh (GoDaddy)

Un certificado wildcard (`*.jcondori.com`) cubre todos los subdominios con un único certificado.
La renovación automática requiere el challenge DNS-01, que acme.sh gestiona directamente contra
la API de GoDaddy sin intervención manual.

### Obtener el Personal Access Token de GoDaddy

1. Ingresar a <https://developer.godaddy.com/personal-access-token> con la cuenta dueña del dominio.
2. Clic en **+ Generate Token** y completar:
   - **Name:** un nombre que identifique el servidor y el dominio.
   - **Expiration:** días hasta que el token vence.
   - **Scopes:** `domains.dns:update` y `domains.domain:read`.
3. Clic en **Generate Token** y copiar el token.

> **Importante:** GoDaddy muestra el token **una sola vez**; guardarlo en un gestor de contraseñas.
> Un token recién creado puede tardar hasta un minuto en ser aceptado por la API.

> **Nota:** Cuando el token vence, las renovaciones fallan. Anotar la fecha de vencimiento y
> reemplazarlo antes (ver [Cambiar el token](#cambiar-el-token-vencido-o-regenerado)).

### Instalación de acme.sh (fork)

Ejecutar como el usuario que gestionará los certificados (en este manual, `root`):

```bash
curl -L https://raw.githubusercontent.com/jcondori/acme.sh/master/acme.sh | sh -s -- --install-online -m luiscondorijara@gmail.com
```

> **Nota:** No usar `curl https://get.acme.sh | sh`: instala la versión oficial, sin soporte de `GD_Token`.
> El instalador agrega automáticamente un cron diario que gestiona las renovaciones.

Verificar que quedó instalado el fork:

```bash
grep '^PROJECT=' ~/.acme.sh/acme.sh
grep -c _gd_detect_root ~/.acme.sh/dnsapi/dns_gd.sh
```

La primera línea debe contener `jcondori` y la segunda mostrar un número mayor que 0.

#### Si el servidor ya tiene el acme.sh oficial

Ejecutar el mismo comando de instalación (sin `-m` si el correo ya estaba configurado). Reemplaza los
scripts de `~/.acme.sh/` y mantiene los certificados, la configuración y el cron. Luego verificar
como se indica arriba.

> **Nota:** `acme.sh --upgrade` no sirve para pasar al fork: el script oficial se actualiza desde
> el repositorio oficial.

#### Si el servidor usó la versión anterior del fork

La versión anterior guardaba el token en `~/.acme.sh/account.conf` como `SAVED_GD_Token`. La versión
actual lo lee de la configuración del certificado, así que la renovación fallaría. Moverlo:

```bash
grep '^SAVED_GD_Token=' ~/.acme.sh/account.conf
echo "GD_Token='TU_TOKEN'" >> ~/.acme.sh/jcondori.com_ecc/jcondori.com.conf
sed -i '/^SAVED_GD_Token=/d' ~/.acme.sh/account.conf
```

### Exportar el token de GoDaddy

```bash
export GD_Token="TU_TOKEN"
```

> **Nota:** acme.sh guarda el token en la configuración del certificado
> (`/root/.acme.sh/jcondori.com_ecc/jcondori.com.conf`) tras la primera emisión, por lo que no es
> necesario exportarlo en cada renovación. Solo se guarda si el token logró acceder a la zona DNS.

> **Alternativa:** el par clásico `GD_Key` / `GD_Secret` sigue funcionando, pero GoDaddy lo marcó como
> obsoleto. Si se exportan el token y el par, se usa el token.

### Emitir el certificado wildcard

```bash
~/.acme.sh/acme.sh --issue \
  --dns dns_gd \
  --server letsencrypt \
  -d "jcondori.com" \
  -d "*.jcondori.com"
```

> **Nota:** Por defecto acme.sh usa ZeroSSL como CA. El flag `--server letsencrypt` fuerza la
> emisión con Let's Encrypt. Los archivos quedan en `/root/.acme.sh/jcondori.com_ecc/` porque
> acme.sh usa curvas elípticas (ECC) por defecto.

Tras emitir, quitar el token de la sesión para no usarlo por error con otro dominio:

```bash
unset GD_Token
```

### Instalar el certificado en Nginx

```bash
sudo mkdir -p /etc/ssl/acme

~/.acme.sh/acme.sh --install-cert \
  -d "jcondori.com" \
  --key-file       /etc/ssl/acme/jcondori.com.key \
  --fullchain-file /etc/ssl/acme/jcondori.com.cer \
  --reloadcmd      "sudo systemctl reload nginx"
```

### Configurar Nginx para usar el certificado wildcard

Para cada nuevo subdominio, crear un archivo `.conf` en `/etc/nginx/conf.d/` apuntando a los
mismos archivos de certificado — el wildcard cubre todos los subdominios de `jcondori.com`.

```bash
sudo nano /etc/nginx/conf.d/misubdominio.jcondori.com.conf
```

Contenido del archivo:

```nginx
server {
    listen 443 ssl;
    server_name misubdominio.jcondori.com;

    ssl_certificate     /etc/ssl/acme/jcondori.com.cer;
    ssl_certificate_key /etc/ssl/acme/jcondori.com.key;
}
```

Validar y recargar Nginx:

```bash
sudo nginx -t && sudo systemctl reload nginx
```

### Verificar renovación

Sin el token exportado, igual que lo ejecuta el cron:

```bash
unset GD_Token
~/.acme.sh/acme.sh --renew -d "jcondori.com" --force
```

Si renueva correctamente, acme.sh está leyendo el token guardado del certificado.

## Otro dominio con un token distinto

Cada certificado guarda su propio token. Para un dominio de otra cuenta GoDaddy (o con otro token),
emitir un certificado aparte exportando su token:

```bash
export GD_Token="TOKEN_DE_OTRODOMINIO"

~/.acme.sh/acme.sh --issue \
  --dns dns_gd \
  --server letsencrypt \
  -d "otrodominio.com" \
  -d "*.otrodominio.com"

unset GD_Token

~/.acme.sh/acme.sh --install-cert \
  -d "otrodominio.com" \
  --key-file       /etc/ssl/acme/otrodominio.com.key \
  --fullchain-file /etc/ssl/acme/otrodominio.com.cer \
  --reloadcmd      "sudo systemctl reload nginx"
```

> **Nota:** Un mismo certificado no puede incluir dominios de tokens distintos: emitir un certificado
> por token. Si al renovar queda exportado un token que no tiene acceso al dominio, acme.sh usa el
> token guardado del certificado y no lo sobrescribe.

## Cambiar el token (vencido o regenerado)

Generar el nuevo token en GoDaddy y renovar con él exportado; reemplaza al guardado:

```bash
export GD_Token="NUEVO_TOKEN"
~/.acme.sh/acme.sh --renew -d "jcondori.com" --force
unset GD_Token
```

## Actualizar acme.sh

`--upgrade` descarga la rama `master` del fork, no la versión oficial:

```bash
~/.acme.sh/acme.sh --upgrade
```

> **Importante:** Mantener desactivada la actualización automática, para que un cambio en el fork no
> llegue a los servidores sin probarlo:
>
> ```bash
> ~/.acme.sh/acme.sh --upgrade --auto-upgrade 0
> ```

## Revocar y reemitir con una CA diferente

Si el certificado fue emitido con ZeroSSL y se necesita cambiarlo a Let's Encrypt:

```bash
~/.acme.sh/acme.sh --revoke -d "jcondori.com" --ecc
~/.acme.sh/acme.sh --remove -d "jcondori.com" --ecc
```

`--remove` archiva la configuración del certificado (`jcondori.com.conf.removed`), incluido el token
guardado, así que hay que exportarlo de nuevo. Luego volver a ejecutar el paso de emisión con
`--server letsencrypt` y `--force` para sobreescribir la clave de dominio existente:

```bash
export GD_Token="TU_TOKEN"

~/.acme.sh/acme.sh --issue \
  --dns dns_gd \
  --server letsencrypt \
  --force \
  -d "jcondori.com" \
  -d "*.jcondori.com"

unset GD_Token
```
