# Rotación de logs de Odoo

`logrotate` es una herramienta del sistema que gestiona la rotación automática de archivos de log. Permite mantener los logs organizados por fecha, evitando que crezcan indefinidamente y consuman todo el espacio en disco.

## Documentación

- Man page oficial: https://linux.die.net/man/8/logrotate

## Configuración

### Crear o editar el archivo de configuración

```bash
nano /etc/logrotate.d/odoo
```

> **Nota:** El archivo debe pertenecer a `root` con permisos `644`. Logrotate ignora archivos con permisos de escritura para otros usuarios.

### Configuración de ejemplo

```bash
/var/log/odoo/odoo-server.log {
    su odoo odoo
    daily
    rotate 36500
    dateext
    dateformat .%Y-%m-%d.log
    copytruncate
    missingok
    notifempty
}
```

### Descripción de parámetros

| Directiva | Descripción |
|-----------|-------------|
| `su odoo odoo` | Ejecuta la rotación con el usuario y grupo `odoo` |
| `daily` | Rota el log una vez al día |
| `rotate 36500` | Conserva hasta 36500 archivos rotados (~100 años) |
| `dateext` | Agrega la fecha como extensión al nombre del archivo rotado |
| `dateformat .%Y-%m-%d.log` | Formato de la fecha en el nombre: `.2025-01-31.log` |
| `copytruncate` | Copia el log activo y luego lo vacía, sin necesidad de reiniciar Odoo |
| `missingok` | No genera error si el archivo de log no existe |
| `notifempty` | No rota el log si está vacío |

> **Nota:** `copytruncate` evita reiniciar el servicio Odoo tras la rotación. Sin embargo, puede perder las líneas escritas entre la copia y el truncado del archivo (ventana de milisegundos).

## Verificación

### Modo debug (sin modificar archivos)

```bash
logrotate -d /etc/logrotate.d/odoo
```

> **Nota:** Este comando simula la rotación y muestra lo que haría, sin modificar ningún archivo.

### Forzar una rotación de prueba

```bash
logrotate -f /etc/logrotate.d/odoo
```

### Verificar el resultado

```bash
ls -lh /var/log/odoo/
```
