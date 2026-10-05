# Apuntes de Prácticas: Despliegue en VPS con Docker

Apuntes y guía paso a paso para configurar un servidor **VPS** y desplegar aplicaciones sobre él utilizando **Docker** y **Cloudflare Tunnel**.

Los ficheros están estructurados de forma secuencial: cada capítulo amplía los conocimientos del anterior, por lo que se recomienda leerlos en orden.

## Contenido

- [1.VPS.md](1.VPS.md): Qué es un VPS y SSH. Conexión inicial, creación de un usuario con permisos `sudo`, seguridad con el firewall `ufw` y acceso con claves SSH sin contraseña.
- [2.Git.md](2.Git.md): Instalación de Git **dentro** del VPS, clave SSH hacia GitHub y despliegues con `git pull`.
- [3.Docker.md](3.Docker.md): Fundamentos de Docker: diferencia entre **Dockerfile, imagen y contenedor**. Instalación, permisos de usuario y comandos más usados.
- [4.N8n_Docker.md](4.N8n_Docker.md): Despliegue de **n8n con Docker Compose**, solución de errores de permisos (`EACCES`) y cifrado HTTPS con Cloudflare Tunnel.
- [5.App_web_Docker.md](5.App_web_Docker.md): Despliegue de una **aplicación web propia con `Dockerfile`**, arrancada con `docker run` o con **Docker Compose**, con volumen, reinicio automático y actualización.
- [6.Crear_web_Docker.md](6.Crear_web_Docker.md): Migración y despliegue de un sitio **WordPress con MariaDB y Docker Compose**, resolución de errores comunes y vinculación con Cloudflare.
- [Backup.md](Backup.md): Guía de **copias de seguridad (backups)** de apps y webs en Docker: archivos (`tar`), bases de datos (`mysqldump`), descarga con `scp`, restauración y cron.
- [Cloudflare.md](Cloudflare.md): Guía explicativa para principiantes sobre **Cloudflare y Cloudflare Tunnels**, desde conectar el dominio hasta configurar las rutas de aplicación para los contenedores Docker.

## Los modelos de despliegue

- **n8n (fichero 4):** imagen pública + Docker Compose.
- **App propia (fichero 5):** imagen construida con `Dockerfile`, arrancada con `docker run` o Docker Compose.
- **WordPress (fichero 6):** dos imágenes públicas (WordPress + MariaDB) con Docker Compose.

## Recordatorios clave

- Abre el puerto SSH en el firewall **antes** de activarlo (`ufw allow OpenSSH`), o quedarás bloqueado fuera del VPS.
- Tras añadir tu usuario al grupo `docker`, cierra y vuelve a abrir la sesión SSH.
- Hay dos claves SSH distintas: **PC → VPS** y **VPS → GitHub**.
- Si cambias el `Dockerfile` o `requirements.txt` de una app propia, **reconstruye la imagen** (`docker compose up -d --build`).
- Lo que no esté en un **volumen** se pierde al borrar el contenedor. Copias de seguridad: [Backup.md](Backup.md).
- Usa `restart: unless-stopped` para que las apps arranquen solas si el VPS se reinicia.
- Ante cualquier fallo, mira primero `docker logs nombre_contenedor`.

---
