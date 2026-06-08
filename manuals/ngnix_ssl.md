# Instalación de Nginx con SSL (Certbot)

Este manual documenta los pasos para instalar Nginx, emitir un certificado SSL gratuito con
Let's Encrypt (vía Certbot) y resolver el problema más común al hacerlo: el bloqueo de los
puertos 80/443 por las reglas de `iptables`. Se usa como ejemplo el dominio `jkanime.jcondori.com`.

## Instalación de Nginx

```bash
sudo apt update
sudo apt install nginx -y
```

## Instalación de Certbot

Se instala Certbot junto con su plugin para Nginx, que permite emitir y configurar el certificado
automáticamente sobre los `server blocks` existentes.

```bash
sudo apt install certbot python3-certbot-nginx -y
```

## Validar acceso al dominio

Antes de solicitar el certificado, conviene confirmar que el dominio resuelve y que Nginx responde
correctamente por HTTP:

```bash
curl http://jkanime.jcondori.com
```

## En caso de error por no acceso (iptables)

Si la petición anterior falla por timeout o conexión rechazada, es posible que las reglas de
`iptables` estén bloqueando el tráfico hacia los puertos 80 (HTTP) y 443 (HTTPS).

### Listar las reglas actuales

```bash
sudo iptables -L INPUT --line-numbers
```

### Permitir tráfico HTTP y HTTPS

```bash
sudo iptables -I INPUT 5 -p tcp --dport 80 -j ACCEPT
sudo iptables -I INPUT 6 -p tcp --dport 443 -j ACCEPT
```

> **Nota:** Los números `5` y `6` indican la posición donde se inserta la regla dentro de la
> cadena `INPUT`. Deben ajustarse según el listado obtenido con `iptables -L INPUT --line-numbers`,
> de forma que la regla de `ACCEPT` quede antes de cualquier regla de `DROP`/`REJECT` que afecte
> a esos puertos.

### Persistir las reglas

```bash
sudo netfilter-persistent save
```

> **Nota:** Sin este paso, las reglas agregadas con `iptables -I` se pierden al reiniciar el
> servidor.

## Emisión del certificado SSL

```bash
sudo certbot --nginx -d jkanime.jcondori.com
```

> **Nota:** El plugin `python3-certbot-nginx` modifica automáticamente el `server block` del
> dominio en Nginx, agregando la configuración SSL y, opcionalmente, la redirección de HTTP a
> HTTPS.

## Verificación

### Comprobar que el sitio responde por HTTPS

```bash
curl https://jkanime.jcondori.com
```

### Revisar el estado y la fecha de expiración del certificado

```bash
sudo certbot certificates
```

### Probar la renovación automática (modo simulación)

```bash
sudo certbot renew --dry-run
```

> **Nota:** Este comando no renueva el certificado, solo simula el proceso para confirmar que la
> renovación automática funcionará correctamente cuando se acerque la fecha de expiración.
