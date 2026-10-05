# Copias de seguridad (backups) de apps y webs en Docker

Guía para guardar una copia de las aplicaciones y webs que corren en Docker en tu VPS, y para recuperarlas si algo sale mal.

> [!NOTE]
> Los pasos 1 a 3 y 5 se ejecutan **dentro del VPS** (por SSH). El paso 4 se ejecuta en **tu ordenador**.

---

## 0. ¿Qué hay que guardar?

Un contenedor se puede borrar y volver a crear en cualquier momento, así que **no se guarda el contenedor**. Se guarda lo que él usa:

| Qué guardar | Dónde está | Cómo se copia |
| :-- | :-- | :-- |
| **Archivos de la app** (código, subidas, SQLite, configuración) | Un *volumen*: una carpeta del VPS conectada al contenedor | `tar` |
| **Base de datos** (MariaDB / MySQL, por ejemplo la de WordPress) | Dentro del contenedor de la base de datos | `mysqldump` |
| **Archivos de configuración** (`docker-compose.yml`, `.env`) | La carpeta del proyecto | `tar` (van incluidos con los archivos) |

Una web como WordPress necesita **las dos primeras**: archivos + base de datos.

> [!WARNING]
> Una copia guardada **solo en el VPS** no te salva si el VPS falla. Siempre descárgala también a tu ordenador (paso 4).

---

## 1. Copiar los archivos de la app (`tar`)

**1.1. Averigua dónde están los datos.** Pregunta a Docker qué volúmenes usa el contenedor:

```bash
docker inspect nombre_contenedor --format '{{json .Mounts}}'
```

Mira el campo `"Type"`; hay dos casos:

- **`bind`**: es una carpeta tuya del VPS (ej. `~/.n8n` en n8n o `~/nombre_carpeta` en la app Python). La ruta está en `"Source"`. Sigue con el paso 1.3.
- **`volume`**: es un volumen que gestiona Docker (es el caso del WordPress de [6.Crear_web_Docker.md](6.Crear_web_Docker.md): `wp_data` y `db_data`). Sigue con el paso 1.4.

**1.2. Crea una carpeta para las copias:**

```bash
mkdir -p ~/backups
```

**1.3. Comprime la carpeta (caso `bind`):**

```bash
tar -czf ~/backups/app_$(date +%Y%m%d).tar.gz -C /home/uservps/nombre_carpeta .
```

Qué significa cada parte:
- `tar`: programa que junta archivos en un solo paquete.
- `-c`: crear el paquete. `-z`: comprimirlo (`.gz`). `-f`: nombre del archivo resultante.
- `~/backups/app_....tar.gz`: dónde se guarda. `$(date +%Y%m%d)` pone la fecha (ej. `20261005`) para no sobrescribir copias anteriores.
- `-C /home/.../nombre_carpeta`: "entra primero en esta carpeta", para que el paquete no guarde rutas largas.
- `.`: "empaqueta todo lo que hay aquí".

**1.4. Copia un volumen gestionado por Docker (caso `volume`).** Su carpeta real queda escondida, así que se usa un contenedor temporal que lo lee y crea el `.tar.gz` en `~/backups`:

```bash
docker volume ls    # busca el nombre real, suele ser carpeta-web_wp_data
docker run --rm -v carpeta-web_wp_data:/datos -v ~/backups:/backup alpine tar -czf /backup/wp_$(date +%Y%m%d).tar.gz -C /datos .
```

- `--rm`: borra el contenedor temporal al terminar.
- `-v nombre_volumen:/datos`: conecta el volumen a la carpeta `/datos` del contenedor temporal.
- `-v ~/backups:/backup`: conecta tu carpeta de copias, para que el `.tar.gz` quede en el VPS.
- `alpine`: una imagen Linux muy pequeña que trae `tar`.

---

## 2. Copiar la base de datos (`mysqldump`)

Una base de datos **no se copia con `tar`** mientras está en marcha: podría quedar la copia a medias y estropeada. Se usa `mysqldump`, que genera un archivo `.sql` con todo su contenido:

```bash
docker exec -e MYSQL_PWD="clave_db" nombre_contenedor_db mysqldump -u usuario_db --single-transaction nombre_db > ~/backups/db_$(date +%Y%m%d).sql
```

Qué significa cada parte:
- `docker exec nombre_contenedor_db ...`: ejecuta un comando **dentro** del contenedor de la base de datos.
- `-e MYSQL_PWD="clave_db"`: le pasa la contraseña sin escribirla en el comando de `mysqldump` (así no queda visible en la lista de procesos).
- `mysqldump -u usuario_db nombre_db`: exporta la base de datos `nombre_db` con ese usuario.
- `--single-transaction`: hace una copia coherente sin parar la web.
- `> ~/backups/db_....sql`: el resultado se guarda en un archivo del VPS (no dentro del contenedor).

> [!TIP]
> Los valores `usuario_db`, `clave_db` y `nombre_db` están en tu `docker-compose.yml` o en el `.env`. Ejemplo completo con WordPress: [6.Crear_web_Docker.md](6.Crear_web_Docker.md).

**Resumen para WordPress:** haz el paso 1.4 (volumen `wp_data`) **y** el paso 2 (base de datos).

---

## 3. Comprobar que la copia sirve

```bash
ls -lh ~/backups/                                          # ¿tienen tamaño razonable (no 0)?
tar -tzf ~/backups/app_20261005.tar.gz | head -n 20        # lista el contenido sin extraerlo
head -n 5 ~/backups/db_20261005.sql                        # debe verse texto SQL
```

---

## 4. Descargar la copia a tu ordenador (`scp`)

Desde la terminal de **tu ordenador** (no del VPS), en la carpeta donde quieras guardarla:

```bash
scp usuariovps@ipvps:/home/uservps/backups/app_20261005.tar.gz .
scp usuariovps@ipvps:/home/uservps/backups/db_20261005.sql .
```

El `.` final significa "guárdalo en la carpeta donde estoy ahora".

---

## 5. Restaurar una copia

**5.1. Archivos.** `-x` = extraer (lo contrario de `-c`).

Si era una carpeta (`bind`):

```bash
tar -xzf ~/backups/app_20261005.tar.gz -C /home/uservps/nombre_carpeta
```

Si era un volumen de Docker (WordPress):

```bash
docker run --rm -v carpeta-web_wp_data:/datos -v ~/backups:/backup alpine tar -xzf /backup/wp_20261005.tar.gz -C /datos
```

**5.2. Base de datos** (el contenedor de la base de datos debe estar en marcha):

```bash
docker exec -i -e MYSQL_PWD="clave_db" nombre_contenedor_db mariadb -u usuario_db nombre_db < ~/backups/db_20261005.sql
```

- `-i`: permite que el contenedor **reciba** el archivo que le pasamos con `<`.
- Si tu contenedor es MySQL, cambia `mariadb` por `mysql`.

**5.3. Reiniciar la app** para que use los datos recuperados:

```bash
docker restart nombre_contenedor
```

---

## 6. (Opcional) Copia automática diaria con cron

`cron` es el programador de tareas de Linux. Para hacer la copia cada día a las 03:00:

```bash
crontab -e
```

Añade al final estas líneas (una para archivos y otra para la base de datos):

```bash
0 3 * * * tar -czf /home/uservps/backups/app_$(date +\%Y\%m\%d).tar.gz -C /home/uservps/nombre_carpeta .
5 3 * * * docker exec -e MYSQL_PWD="clave_db" nombre_contenedor_db mysqldump -u usuario_db --single-transaction nombre_db > /home/uservps/backups/db_$(date +\%Y\%m\%d).sql
```

- Los cinco primeros campos son `minuto hora día mes día_de_la_semana`; `0 3 * * *` = "a las 03:00, todos los días".
- Dentro de cron hay que escribir `\%` en vez de `%`, o la línea falla.
- En cron usa **rutas completas** (`/home/uservps/...`), no `~`.
- Las copias se acumulan y llenan el disco. Para borrar las de más de 14 días:

```bash
find /home/uservps/backups -type f -mtime +14 -delete
```
