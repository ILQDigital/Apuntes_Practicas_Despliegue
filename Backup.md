# Guía Completa: Copias de Seguridad (Backups) en VPS con Docker

Esta guía explica cómo realizar copias de seguridad (*backups*) de aplicaciones y sitios web desplegados en Docker sobre un servidor VPS, cubriendo tanto directorios persistentes (*bind mounts* / volúmenes) como bases de datos relacionales (MySQL/MariaDB), y su posterior transferencia segura al equipo local.

---

## Estrategia de Copias de Seguridad

Dependiendo del tipo de aplicación desplegada en Docker, la estrategia de copia de seguridad varía:

| Tipo de Aplicación / Datos | Estrategia Recomendada | Herramienta |
| :--- | :--- | :--- |
| **Archivos estáticos y volúmenes** (*bind mounts*, descargas, configs) | Compresión del directorio del host | `tar -czf` |
| **Bases de datos SQL** (MariaDB, MySQL) | Volcado limpio desde el contenedor | `docker exec ... mysqldump` |
| **Aplicaciones Web completas** (ej. WordPress) | Exportación de BD (`.sql`) + compresión de archivos (`wp-content`) | `mysqldump` + `tar` / `docker cp` |

---

## 1. Copia de Seguridad de Archivos y Volúmenes (`tar.gz`)

Para respaldar carpetas persistentes mapeadas desde el host al contenedor (*bind mounts*):

### 1.1. Identificar la ruta de datos en el VPS
Inspecciona los puntos de montaje del contenedor con `docker inspect`:

```bash
docker inspect nombre_contenedor --format '{{json .Mounts}}'
```

Identifica el campo `"Source"` (ej. `/home/uservps/nombre_carpeta`), que indica la ruta real donde residen los datos persistentes en el disco del VPS.

### 1.2. Crear el directorio de copias de seguridad
```bash
mkdir -p ~/backups
cd ~/backups
```

### 1.3. Generar el archivo comprimido (`.tar.gz`)
Se recomienda incluir la fecha automáticamente en el nombre del archivo:

```bash
tar -czf backup_app_$(date +%Y%m%d).tar.gz -C /home/uservps/nombre_carpeta .
```

#### Desglose de parámetros:
- `-c` (*create*): Crea un nuevo archivo de copia de seguridad.
- `-z` (*gzip*): Comprime en formato Gzip para optimizar espacio.
- `-f` (*file*): Especifica el nombre del archivo comprimido resultante.
- `-C /ruta`: Cambia la ubicación de trabajo al directorio indicado antes de realizar la copia, evitando incluir rutas absolutas innecesarias dentro del paquete comprimido.
- `.` (*punto*): Indica que debe empaquetar todo el contenido de esa carpeta.
- `$(date +%Y%m%d)`: Añade automáticamente la fecha actual (ej. `20260817`).

---

## 2. Copia de Seguridad de Sitios Web y Bases de Datos (WordPress / MariaDB)

Para aplicaciones web que dependen de una base de datos relacional (como WordPress, Drupal o Laravel), la copia de seguridad debe realizarse en **dos partes**:

### 2.1. Exportación de la base de datos SQL (`mysqldump`)
Para garantizar la integridad transaccional de los datos, ejecuta el volcado directamente desde el contenedor de MariaDB/MySQL hacia el VPS:

```bash
docker exec nombre_contenedor_db mysqldump -u usuario_db -p"clave_db" nombre_db > ~/backups/db_$(date +%Y%m%d).sql
```

> 📘 **Tutorial completo de despliegue web**: Para ver el proceso completo de instalación, migración y volcado en WordPress, consulta la guía detallada en [6.Crear_web_Docker.md](6.Crear_web_Docker.md).

### 2.2. Respaldo de los archivos de la web (`wp-content` / subidas)
Puedes empaquetar la carpeta de archivos subidos y plugins directamente desde el host o usando `docker cp`:

```bash
# Opción A: Comprimir la carpeta persistente directamente desde el VPS
tar -czf ~/backups/wp_content_$(date +%Y%m%d).tar.gz -C /home/uservps/carpeta-web/wp-content .

# Opción B: Copiar la carpeta directamente desde el contenedor de WordPress al host
docker cp nombre_contenedor_web:/var/www/html/wp-content ~/backups/wp-content-backup
```

---

## 3. Transferencia Segura del Backup al PC Local (`scp`)

> [!IMPORTANT]
> Los pasos anteriores se ejecutan **dentro del VPS**. La descarga a tu equipo personal se ejecuta desde la **terminal de tu ordenador local**.

1. Abre la terminal en tu ordenador local.
2. Navega a la carpeta local donde desees almacenar el respaldo (ej. `Descargas` o `Backups`).
3. Transfiere el archivo mediante `scp`:

```bash
scp usuariovps@ipvps:/home/uservps/backups/backup_app_20260817.tar.gz .
```

*(El punto `.` final indica que el archivo se guardará en la ubicación local en la que te encuentras).*

---

## 4. Verificación e Inspección del Backup

Comprueba el contenido del directorio y verifica que el archivo no esté vacío:

```bash
# Comprobar tamaño del archivo en el VPS
ls -lh ~/backups/

# Inspeccionar la lista de archivos dentro del paquete comprimido sin extraerlo
tar -tzf ~/backups/backup_app_20260817.tar.gz | head -n 20
```

---

## 5. Restaurar una Copia de Seguridad

Si necesitas restaurar los datos por una migración o incidente:

### 5.1. Restaurar archivos comprimidos (`.tar.gz`)
```bash
tar -xzf backup_app_20260817.tar.gz -C /home/uservps/nombre_carpeta
```

### 5.2. Restaurar base de datos SQL (`.sql`)
```bash
docker exec -i nombre_contenedor_db mariadb -u usuario_db -p"clave_db" nombre_db < ~/backups/db_20260817.sql
```

---

## 6. (Opcional) Automatización con Cron Jobs

Para programar copias de seguridad automáticas diarias (por ejemplo, a las 03:00 AM) en el VPS:

1. Edita la tabla de tareas de cron:
```bash
crontab -e
```

2. Añade la regla indicando la ejecución del respaldo:
```bash
0 3 * * * tar -czf /home/uservps/backups/backup_$(date +\%Y\%m\%d).tar.gz -C /home/uservps/nombre_carpeta .
```
