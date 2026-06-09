# Nginx con SSL wildcard y renovación automática — acme.sh (GoDaddy)

Este manual documenta cómo emitir un certificado SSL wildcard (`*.jcondori.com`) con **Let's Encrypt**
usando `acme.sh` y la API de GoDaddy, con renovación automática sin intervención manual.

## Instalación de Nginx

```bash
sudo apt update
sudo apt install nginx -y
```

## Certificado wildcard con renovación automática — acme.sh (GoDaddy)

Un certificado wildcard (`*.jcondori.com`) cubre todos los subdominios con un único certificado.
La renovación automática requiere el challenge DNS-01, que acme.sh gestiona directamente contra
la API de GoDaddy sin intervención manual.

### Obtener las credenciales API de GoDaddy

Ingresar a <https://developer.godaddy.com/keys>, crear una clave de tipo **Production** y copiar
el **Key** y el **Secret**.

### Instalación de acme.sh

```bash
curl https://get.acme.sh | sh -s email=luiscondorijara@gmail.com
```

> **Nota:** El instalador agrega automáticamente un cron diario que gestiona las renovaciones.

### Exportar las credenciales de GoDaddy

```bash
export GD_Key="TU_API_KEY"
export GD_Secret="TU_API_SECRET"
```

> **Nota:** acme.sh guarda estas credenciales internamente tras la primera emisión, por lo que
> no es necesario exportarlas en cada renovación.

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

```bash
~/.acme.sh/acme.sh --renew -d "jcondori.com" --force
```

## Revocar y reemitir con una CA diferente

Si el certificado fue emitido con ZeroSSL y se necesita cambiarlo a Let's Encrypt:

```bash
~/.acme.sh/acme.sh --revoke -d "jcondori.com" --ecc
~/.acme.sh/acme.sh --remove -d "jcondori.com" --ecc
```

Luego volver a ejecutar el paso de emisión con `--server letsencrypt` y `--force` para sobreescribir
la clave de dominio existente:

```bash
~/.acme.sh/acme.sh --issue \
  --dns dns_gd \
  --server letsencrypt \
  --force \
  -d "jcondori.com" \
  -d "*.jcondori.com"
```
