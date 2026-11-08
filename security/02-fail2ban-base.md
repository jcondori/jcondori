# Paso 2: Fail2ban base (SSH y reincidentes)

> Serie [Seguridad de un servidor web](README.md) · Anterior: [paso 1, firewall y SSH](README.md#paso-1-firewall-y-ssh)
> · Siguiente: [paso 3, Nginx y certificado](../manuals/ngnix_ssl_wildcard.md)

Fail2ban lee los logs, cuenta los fallos por IP (intentos de SSH, logins fallidos, escaneos) y bloquea en el firewall de
la VM las IPs que pasan del límite durante un tiempo. Probado en Ubuntu 24.04 con Fail2ban 1.0.2 y nftables.

En este paso se instala y se activan las dos jails que no dependen de nada más: `sshd` y `recidive`. Las de Nginx y las
aplicaciones van en el [paso 6](06-fail2ban-web.md), cuando ya existen sus logs.

> **No reemplaza el paso 1.** Con SSH solo por llave, los bots no pueden entrar aunque Fail2ban no exista. Lo que aporta
> es menos ruido en los logs, menos carga y, en el paso 6, un freno a la fuerza bruta contra los logins de las
> aplicaciones, que sí aceptan contraseñas.

## Cómo funciona

```text
log (journal, nginx, odoo)
        │  filtro (regex) detecta un fallo y saca la IP
        ▼
   jail: maxretry fallos dentro de findtime
        │  acción
        ▼
   nftables bloquea la IP durante bantime
```

| Pieza | Función |
|---|---|
| Filtro (`filter.d/*.conf`) | Expresión regular que reconoce un fallo en una línea de log y extrae la IP (`<HOST>`) |
| Jail (`jail.local`, `jail.d/*.local`) | Une un filtro con un log y define `maxretry`, `findtime` y `bantime` |
| Acción (`banaction`) | Cómo se bloquea la IP. Aquí, nftables |

## Requisitos

- Paso 1 hecho: SSH entra solo con llave.
- Ubuntu 22.04 o superior (o Debian 12).
- Acceso alternativo al servidor por si te bloqueas a ti mismo: la consola serie de GCP o la consola web del proveedor.

## Instalación

```bash
apt update
apt install -y fail2ban nftables
systemctl status fail2ban --no-pager
```

En Ubuntu 24.04, la versión inicial del paquete fallaba al arrancar con `ModuleNotFoundError: No module named
'asynchat'` (Python 3.12). Si aparece, actualiza el paquete: la versión corregida ya está en los repositorios.

## Configuración

**Nunca edites `jail.conf`**: se sobrescribe al actualizar el paquete. Todo va en `jail.local` y en `jail.d/*.local`.

El paquete de Ubuntu/Debian ya trae `jail.d/defaults-debian.conf`, que activa `sshd` con `backend = systemd` y
`banaction = nftables`. No lo borres: los archivos `.local` de este paso se cargan después y mandan sobre él.

### Valores por defecto

Valen para todas las jails, también las del paso 6, salvo que la jail los cambie.

`/etc/fail2ban/jail.local`:

```ini
[DEFAULT]
# IPs que nunca se bloquean. AJUSTAR: tu IP fija o tu VPN, si tienes.
# Una IP doméstica dinámica no sirve: mañana será de otro.
ignoreip = 127.0.0.1/8 ::1

# Bloqueo con nftables.
banaction = nftables-multiport
banaction_allports = nftables-allports

# 5 fallos en 10 minutos = 1 hora de bloqueo.
maxretry = 5
findtime = 10m
bantime  = 1h

# Cada reincidencia duplica el bloqueo (1 h, 2 h, 4 h…), hasta 1 semana.
bantime.increment = true
bantime.factor    = 1
bantime.maxtime   = 1w

# Leer el journal de systemd cuando la jail no indica un archivo.
backend = auto
```

`bantime.increment` hace que un bot que vuelve tras cumplir su bloqueo reciba 2 h, luego 4 h, y así hasta 1 semana.
Fail2ban recuerda los bloqueos en su base de datos (`/var/lib/fail2ban/fail2ban.sqlite3`), así que sobreviven a los
reinicios.

La duplicación la hace la fórmula por defecto (`bantime × 2^n`, con `n` = bloqueos anteriores de esa IP).
`bantime.factor` es un multiplicador **encima** de esa fórmula, no el ritmo de crecimiento: con `factor = 2` todo
saldría al doble (2 h, 4 h, 8 h…). Déjalo en `1`.

**Por qué estos tiempos:** con 3 fallos por bloqueo, una IP consigue unos 9 intentos el primer día (1 h + 2 h + 4 h de
bloqueo) y luego `recidive` la saca una semana: del orden de 10 intentos por semana y por IP. Para adivinar una
contraseña eso es nada, así que bloqueos más largos no añaden protección real (las botnets cambian de IP y las IPs
domésticas se reasignan a otra gente). Lo que sí aumentan es el daño de un falso positivo: tú mismo u otra persona
detrás de la misma IP.

### SSH

`/etc/fail2ban/jail.d/sshd.local`:

```ini
[sshd]
enabled  = true
backend  = systemd
mode     = aggressive
maxretry = 3
```

- `backend = systemd` lee el journal. En Debian 12 no existe `/var/log/auth.log` (no trae rsyslog), y con el backend por
  archivo la jail no arranca.
- `mode = aggressive` también cuenta los intentos que fallan antes de la autenticación: usuario inválido, llave
  rechazada, conexión cortada en el intercambio de claves. Con SSH solo por llave, casi todo el ruido es de ese tipo.

### Reincidentes

Bloquea por una semana, en **todos los puertos**, a las IPs que Fail2ban ya bloqueó varias veces en cualquier jail. Lee
el log del propio Fail2ban, así que en el paso 6 cubre también las jails web sin tocar nada.

`/etc/fail2ban/jail.d/recidive.local`:

```ini
[recidive]
enabled   = true
logpath   = /var/log/fail2ban.log
banaction = %(banaction_allports)s
maxretry  = 3
findtime  = 1d
bantime   = 1w
```

### Aplicar

```bash
fail2ban-client -t            # valida toda la configuración
systemctl restart fail2ban
systemctl enable fail2ban
fail2ban-client status        # debe listar: recidive, sshd
```

`fail2ban-client -t` puede avisar `'allowipv6' not defined in 'Definition'`. Es inofensivo: usa el valor por defecto.

## Verificación del paso 2

Comprueba que el filtro de SSH reconoce los fallos. Se prueba sobre las últimas 24 h del journal, volcadas a un archivo,
y con `mode=aggressive`, igual que la jail:

```bash
journalctl _COMM=sshd --since "-24h" -o short > /root/ssh-24h.log
fail2ban-regex /root/ssh-24h.log "sshd[mode=aggressive]"
rm /root/ssh-24h.log
```

No uses `fail2ban-regex systemd-journal ...`: lee el journal **entero**. Con el tope de 200 MB del paso 1 tarda unos
segundos, pero sin él (1,4 GB en una VM de 1 GB de RAM) no termina y hay que matarlo. Ojo al matarlo con
`pkill -f fail2ban-regex` dentro de `ssh host '...'`: el patrón coincide con la propia línea de comandos de la sesión y
la mata; usa `pkill -f "[f]ail2ban-regex"`.

La salida dice cuántas líneas coinciden (`Failregex: N total`). En un servidor con IP pública, a las pocas horas ya hay
bots probando: `fail2ban-client status sshd` debe mostrar fallos y alguna IP bloqueada.

```bash
fail2ban-client status sshd
nft list ruleset | grep -A5 f2b
```

## Uso diario

```bash
# Jails activas y resumen de una
fail2ban-client status
fail2ban-client status sshd

# IPs bloqueadas en todas las jails
fail2ban-client banned

# Desbloquear una IP (en todas las jails o en una)
fail2ban-client unban 203.0.113.7
fail2ban-client set sshd unbanip 203.0.113.7

# Bloquear a mano
fail2ban-client set sshd banip 203.0.113.7

# Log de Fail2ban
tail -f /var/log/fail2ban.log
```

## Si te bloqueas a ti mismo

1. Entra por otra red (datos del móvil) o por la consola serie del proveedor.
2. `fail2ban-client unban <tu IP>`.
3. Si tienes una IP fija o una VPN, añádela a `ignoreip` en `jail.local` y `systemctl reload fail2ban`.

## Notas

- **Firewall de la nube:** Fail2ban bloquea en nftables, dentro de la VM. El firewall de GCP sigue dejando pasar el
  tráfico hasta la VM, y es la VM quien lo descarta. Es suficiente para este uso.
- **Memoria:** Fail2ban usa unos 30-60 MB. En un servidor pequeño es asumible, pero conviene vigilarlo con
  `systemctl status fail2ban`.

## Desinstalar

```bash
systemctl disable --now fail2ban
apt purge -y fail2ban
rm -rf /etc/fail2ban /var/lib/fail2ban
```

Al detener el servicio, Fail2ban borra sus reglas de nftables: ninguna IP queda bloqueada.

**Siguiente:** [paso 3, Nginx y certificado wildcard con acme.sh](../manuals/ngnix_ssl_wildcard.md).
