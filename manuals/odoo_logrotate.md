# Rotación de logs de Odoo

### Documentación de la herramienta: https://linux.die.net/man/8/logrotate

### Crear o edita el archivo de configuración para logrotate de Odoo
```bash
nano /etc/logrotate.d/odoo
```

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

### Ejecutar depuración para verificar que no hay errores en la configuración.
```bash
logrotate -d /etc/logrotate.d/odoo
``` 
> **Nota:** No modifica los archivos logs, solo muestra lo que haría.