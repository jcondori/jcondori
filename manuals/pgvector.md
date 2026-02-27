# Compilar extension pgvector para postgres

### Repositorio oficial: https://github.com/pgvector/pgvector

### Instalar "Visual Studio" con la carga de trabajo "Desarrollo para el escritorio con C++"
![pgvector.png](../sources/pgvector.png)

### Abrir la herramienta `x64 Native Tools Command Prompt for VS` como administrador que se debio instalar con visual studio

### Clonar repositorio en una ubicación temporal
```bash
git clone git@github.com:pgvector/pgvector.git
cd pgvector
```
> **Nota:** Puedes navegar a una ruta especifica `cd /D C:\Users\jcondori\AppData\Local\Temp`.

### Establecemos variable de entorno
```bash
set "PGROOT=E:\Program Files\PostgreSQL\17"
``` 

### Ejecutamos la compilación
```bash
nmake /F Makefile.win
```
> **Nota:** Si compila correctamente deberia terminar en `1 archivo(s) copiado(s).`.

### Instalamos la extension
```bash
nmake /F Makefile.win install
```

### Por ultimo reiniciamos postgres y validamos si se instalo
```sql
SELECT name, default_version, installed_version FROM pg_available_extensions WHERE name = 'vector';
```
```sql
CREATE EXTENSION vector;
```
