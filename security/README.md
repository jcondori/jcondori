# Seguridad de un servidor web

Serie de manuales para dejar seguro un servidor Linux que publica aplicaciones (Node, Odoo, etc.) detrás de Nginx con
HTTPS. Se siguen **en orden**: cada paso da por hecho el anterior.

### Versiones probadas

| Componente | Versiones |
|---|---|
| SO | Ubuntu 24.04 y 26.04 |
| Fail2ban | 1.0.2 (24.04), 1.1.0 (26.04) |
| Nginx | 1.24 (24.04), 1.28 (26.04) |

Los comandos valen para cualquiera de esas versiones salvo que se indique; las diferencias se señalan en el paso
correspondiente.

## Orden

| Paso | Qué | Manual |
|---|---|---|
| 1 | Firewall de la nube y SSH solo con llave | En este README (es manual, se hace a mano) |
| 2 | Fail2ban: SSH y reincidentes | [02-fail2ban-base.md](02-fail2ban-base.md) |
| 3 | Nginx y certificado wildcard con acme.sh (si ya usa Certbot, se salta: ver paso 4) | [ngnix_ssl_wildcard.md](../manuals/ngnix_ssl_wildcard.md) |
| 4 | Nginx: configuración del servidor (una vez) | [04-nginx-servidor.md](04-nginx-servidor.md) |
| 5 | Nginx: un sitio por aplicación | [05-nginx-sitio.md](05-nginx-sitio.md) |
| 6 | Fail2ban: jails de Nginx y de las aplicaciones | [06-fail2ban-web.md](06-fail2ban-web.md) |
| 7 | Odoo: endurecimiento de la aplicación (solo sitios Odoo) | [07-odoo-seguridad.md](07-odoo-seguridad.md) |

**Por qué este orden:**

- SSH y el firewall van primero: el servidor queda expuesto desde el primer minuto.
- Fail2ban para SSH (paso 2) no depende de nada más, así que va pronto.
- Las jails web de Fail2ban (paso 6) van al final: leen los logs de Nginx y de las aplicaciones, que antes no existen, y
  sus filtros se prueban con líneas reales. Además, las pruebas del paso 5 provocan bloqueos a propósito, y con esas
  jails activas te bloquearías a ti mismo.
- El paso 7 es la capa de aplicación de Odoo: solo aplica si el servidor publica Odoo, y da por hechos el proxy (paso 4)
  y las jails (paso 6). Un servidor que solo sirve otras aplicaciones lo omite.

**Aplicación nueva en un servidor ya montado:** repite los pasos 5 y 6, en ese orden; si la aplicación es Odoo, añade el
paso 7.

## Server ya configurado a mano

Esta serie asume un servidor nuevo y propone un layout canónico: `conf.d/00-base.conf`, `snippets/proxy.conf`, un solo
`access.log`, jails `nginx-scan`/`nginx-noise`. Un servidor que ya tiene Nginx y Fail2ban configurados a mano (confs por
aplicación, un `log_format` propio, logs separados, otras jails) puede cumplir lo mismo con otra estructura. No hace
falta rehacerlo: aplica lo que falta sin arrancar lo que funciona.

Antes de tocar nada, compara contra la **intención** de cada paso, no contra sus nombres de archivo:

| Intención del paso | Qué buscar en lo que ya hay |
|---|---|
| 4.1 Ajustes globales | `server_tokens off`, `ssl_protocols TLSv1.2 TLSv1.3`, timeouts contra slowloris. Pueden estar en `nginx.conf` en vez de `00-base.conf`. |
| 4.2 Cabeceras del proxy | `proxy_set_header X-Forwarded-*`, `X-Real-IP`. Pueden estar inline en cada sitio en vez de un snippet. |
| 4.4 default_server seguro | **Debe existir** un server por defecto en el 443 con `ssl_reject_handshake`. Si una aplicación es el `default_server`, se sirve a cualquiera por IP: corrígelo (ver paso 4.4). |
| 4.5 Logs | Un `log_format` que conserve el prefijo `combined` (para que los filtros funcionen) y retención de 90 días. El nombre del formato da igual. |
| 6 Jails web | Jails que lean los logs de Nginx y de las apps y bloqueen escaneos y logins fallidos. Con **logs separados por aplicación**, cada archivo debe ir en el `logpath` de la jail (varias rutas, una por línea), o esa app queda sin vigilar. |

Regla práctica: aplica el paso solo donde la intención no esté cubierta. Si ya está, valida que el valor sea el correcto
(p. ej. `fail2ban-regex` sobre el log real, paso 6.4) y sigue. Lo único innegociable es el **default_server seguro** del
paso 4.4: siempre debe haber uno, y nunca puede ser una aplicación.

---

## Paso 1: firewall y SSH

### Firewall de la nube

Solo deben estar abiertos al mundo:

| Puerto | Uso |
|---|---|
| tcp/22 | SSH |
| tcp/80 | Redirección a HTTPS. Con acme.sh (valida por DNS) no hace falta para el certificado; con Certbot **sí**: renueva por HTTP en este puerto y cerrarlo rompe la renovación |
| tcp/443 | Sitios |

Los puertos internos de las aplicaciones (3000, 8069, etc.) **nunca** se abren en el firewall.

El filtrado se hace en el **firewall de la nube** (panel del proveedor), no en la VM. Según el proveedor:

- **Reglas por puerto/origen** (Contabo, Hetzner, DigitalOcean, AWS Security Groups, Oracle Cloud, etc.): abre solo 22,
  80 y 443 hacia `0.0.0.0/0` y cierra el resto. Si el proveedor trae todo abierto por defecto, crea una regla de
  denegación o restringe la lista de permitidos. Limita el 22 a tu IP fija o VPN si la tienes.
- **GCP (VPC network → Firewall), reglas por etiqueta de red:**
  - La VM lleva solo las etiquetas `http-server` y `https-server` (reglas `default-allow-http` y `default-allow-https`).
  - Revisa que no tenga etiquetas de otras reglas. Por ejemplo, una regla de juegos que abre el 3000 lo expondría si
    alguien pone esa etiqueta a la VM.
  - `default-allow-ssh` aplica a todas las VMs. Para cerrarlo más, limítala a tu IP fija o al rango de IAP
    (`35.235.240.0/20`, si entras con `gcloud compute ssh --tunnel-through-iap`). Cambiarla afecta a todas las VMs del
    proyecto.

Si el proveedor **no** tiene firewall de nube, entonces sí activa `ufw` o reglas nftables en la VM (abre 22/80/443,
deniega el resto). Con firewall de nube no hace falta `ufw`: ya filtra. En cualquier caso, Fail2ban (pasos 2 y 6) usa
nftables por su cuenta.

### SSH solo con llave

Antes de desactivar las contraseñas, comprueba que **entras con tu llave**. Si no, te quedas fuera.

Valores que debe mostrar `sshd -T`:

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

Para cambiar uno, edita `/etc/ssh/sshd_config`. Ojo con los archivos de `/etc/ssh/sshd_config.d/`: se cargan antes y el
primer valor que encuentra sshd es el que manda. Si un valor no cambia, búscalo ahí:

```bash
grep -rni "passwordauthentication" /etc/ssh/sshd_config /etc/ssh/sshd_config.d/
```

Valida y recarga **sin cerrar la sesión actual**, y prueba una conexión nueva en otra terminal antes de salir:

```bash
cp -a /etc/ssh/sshd_config /root/sshd_config.bak.$(date +%Y%m%d%H%M%S)
sshd -t && systemctl reload ssh
```

### Actualizaciones y logs del sistema

Las actualizaciones de seguridad deben instalarse solas. En Ubuntu vienen activas; compruébalo:

```bash
systemctl is-enabled unattended-upgrades
cat /etc/apt/apt.conf.d/20auto-upgrades   # las dos líneas en "1"
```

Si falta, `apt install -y unattended-upgrades && dpkg-reconfigure -plow unattended-upgrades`.

Pon un tope al journal de systemd. Sin él crece hasta el 10 % del disco (en una VM de 30 GB se vio en 1,4 GB), y
Fail2ban lo lee entero al probar el filtro de SSH (paso 2): en una VM de 1 GB de RAM esa prueba no termina.

```bash
mkdir -p /etc/systemd/journald.conf.d
printf '[Journal]\nSystemMaxUse=200M\n' > /etc/systemd/journald.conf.d/size.conf
systemctl restart systemd-journald
journalctl --vacuum-size=200M
journalctl --disk-usage
```

Usa `journalctl --vacuum-*`, nunca `rm` sobre `/var/log/journal`: systemd-journald tiene esos archivos abiertos.

### Verificación del paso 1

En la VM, ver qué escucha:

```bash
ss -tulnp
```

Hacia fuera (`0.0.0.0` o `[::]`) solo deben aparecer el 22 y, después del paso 3, el 80 y el 443. Todo lo demás, en
`127.0.0.1`.

**Siguiente:** [paso 2, Fail2ban base](02-fail2ban-base.md).
