# Seguridad de un servidor web

Serie de manuales para dejar seguro un servidor Linux que publica aplicaciones (Node, Odoo, etc.) detrás de Nginx con
HTTPS. Se siguen **en orden**: cada paso da por hecho el anterior. Probado en Ubuntu 24.04.

## Orden

| Paso | Qué | Manual |
|---|---|---|
| 1 | Firewall de la nube y SSH solo con llave | En este README (es manual, se hace a mano) |
| 2 | Fail2ban: SSH y reincidentes | [02-fail2ban-base.md](02-fail2ban-base.md) |
| 3 | Nginx y certificado wildcard con acme.sh (si ya usa Certbot, se salta: ver paso 4) | [ngnix_ssl_wildcard.md](../manuals/ngnix_ssl_wildcard.md) |
| 4 | Nginx: configuración del servidor (una vez) | [04-nginx-servidor.md](04-nginx-servidor.md) |
| 5 | Nginx: un sitio por aplicación | [05-nginx-sitio.md](05-nginx-sitio.md) |
| 6 | Fail2ban: jails de Nginx y de las aplicaciones | [06-fail2ban-web.md](06-fail2ban-web.md) |

**Por qué este orden:**

- SSH y el firewall van primero: el servidor queda expuesto desde el primer minuto.
- Fail2ban para SSH (paso 2) no depende de nada más, así que va pronto.
- Las jails web de Fail2ban (paso 6) van al final: leen los logs de Nginx y de las aplicaciones, que antes no existen, y
  sus filtros se prueban con líneas reales. Además, las pruebas del paso 5 provocan bloqueos a propósito, y con esas
  jails activas te bloquearías a ti mismo.

**Aplicación nueva en un servidor ya montado:** repite los pasos 5 y 6, en ese orden.

---

## Paso 1: firewall y SSH

### Firewall de la nube

Solo deben estar abiertos al mundo:

| Puerto | Uso |
|---|---|
| tcp/22 | SSH |
| tcp/80 | Solo redirección a HTTPS (acme.sh valida por DNS, no usa este puerto) |
| tcp/443 | Sitios |

Los puertos internos de las aplicaciones (3000, 8069, etc.) **nunca** se abren en el firewall.

En GCP (VPC network → Firewall), las reglas se aplican por **etiqueta de red**:

- La VM lleva solo las etiquetas `http-server` y `https-server` (reglas `default-allow-http` y `default-allow-https`).
- Revisa que no tenga etiquetas de otras reglas. Por ejemplo, una regla de juegos que abre el 3000 lo expondría si
  alguien pone esa etiqueta a la VM.
- `default-allow-ssh` aplica a todas las VMs. Para cerrarlo más, limítala a tu IP fija o al rango de IAP
  (`35.235.240.0/20`, si entras con `gcloud compute ssh --tunnel-through-iap`). Cambiarla afecta a todas las VMs del
  proyecto.

No hace falta activar `ufw` en la VM: el firewall de la nube ya filtra. Fail2ban (pasos 2 y 6) usa nftables por su
cuenta.

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
