# Apuntes de Prácticas: Despliegue en VPS con Docker

Apuntes y guía paso a paso para configurar un servidor **VPS** y desplegar aplicaciones sobre él utilizando **Docker** y **Cloudflare Tunnel**.

Los ficheros están estructurados de forma secuencial: cada capítulo amplía los conocimientos del anterior, por lo que se recomienda leerlos en orden.

## Contenido

- [1.VPS.md](1.VPS.md): Qué es un VPS y SSH. Conexión inicial, creación de un usuario con permisos `sudo`, seguridad con el firewall `ufw` y acceso con claves SSH sin contraseña.
- [2.Git.md](2.Git.md): Instalación y configuración de Git **dentro** del VPS para automatizar despliegues con `git pull`.
- [3.Docker.md](3.Docker.md): Fundamentos de Docker: diferencia entre **Dockerfile, imagen y contenedor**. Instalación, permisos de usuario y comandos más usados.
- [4.N8n_Docker.md](4.N8n_Docker.md): Despliegue de **n8n con `docker-compose`**, solución de errores de permisos (`EACCES`) y cifrado HTTPS con Cloudflare Tunnel.
- [5.App_web_Docker.md](5.App_web_Docker.md): Despliegue de una **aplicación web propia con `Dockerfile` y `docker run`**, gestión de volúmenes persistentes y procedimiento de actualización.
- [6.Backup.md](6.Backup.md): Creación de copias de seguridad comprimidas (`.tar.gz`) de los datos persistentes de los contenedores y su descarga al PC local.

## Los dos modelos de despliegue

- **n8n (Fichero 4):** Imagen pública preconstruida + `docker-compose` (configuración declarativa mediante archivo YAML).
- **App propia (Fichero 5):** Imagen personalizada construida desde cero con `Dockerfile` + `docker run` (ejecución directa).

## Recordatorios clave de seguridad y administración

- Abre el puerto SSH en el firewall **antes** de activarlo (`ufw allow OpenSSH`), de lo contrario quedarás bloqueado fuera del VPS.
- Tras añadir tu usuario al grupo `docker`, debes **cerrar y volver a abrir la sesión SSH** para aplicar los permisos.
- Cada conexión requiere su propia clave SSH: una va desde tu **PC al VPS** y otra se genera **en el VPS hacia GitHub**.
- Si modificas el código de una app propia, **debes reconstruir la imagen de Docker** (`docker build`). La imagen no se actualiza automáticamente al hacer `git pull`.
- Todos los datos que no estén asociados a un **volumen** (`-v`) se eliminarán al destruir el contenedor.
- Ante cualquier fallo en un servicio, inspecciona primero los registros con `docker logs nombre_contenedor`.

---
