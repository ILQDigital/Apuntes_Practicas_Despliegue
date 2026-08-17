# Guía de Cloudflare y Cloudflare Tunnels para Principiantes

Esta guía explica de forma clara y sencilla qué es **Cloudflare**, para qué sirve y cómo se ha utilizado a lo largo de este proyecto para conectar nuestros dominios a los contenedores Docker en el VPS de forma segura, rápida y gratuita.

---

## 1. ¿Qué es Cloudflare y para qué sirve?

**Cloudflare** es un servicio que actúa como un intermediario o escudo de seguridad entre los usuarios de Internet y tu servidor (VPS). 

Cuando alguien escribe tu dominio en el navegador (ejemplo: `https://midominio.es`), la petición no va directamente a la IP de tu VPS, sino que pasa primero por la red global de Cloudflare.

```
[ Usuario en Internet ]  --->  [ Escudo de Cloudflare ]  --->  [ Tu Servidor VPS (Docker) ]
```

### Funciones principales que nos ofrece:

1. **Gestor de DNS (La "agenda" de Internet)**: Traduce tu nombre de dominio (`midominio.es`) a la dirección IP donde se encuentra tu servicio.
2. **Certificados SSL / HTTPS gratuitos**: Cifra la conexión entre tus visitantes y la web (mostrando el candado de seguridad en la barra de navegación) sin tener que configurar certificados manualmente en el servidor.
3. **Escudo de seguridad y Anti-DDoS**: Oculta la dirección IP real de tu VPS. Si sufres un ataque informático, Cloudflare lo bloquea en sus servidores antes de que afecte a tu VPS.
4. **Cloudflare Tunnel (`cloudflared`)**: La tecnología que permite exponerte a Internet **sin abrir puertos en tu firewall**.

---

## 2. ¿Por qué usamos Cloudflare Tunnel (`cloudflared`) en nuestros despliegues?

En un despliegue tradicional, para publicar una web debías abrir el puerto 80 (HTTP) y 443 (HTTPS) en el firewall de tu VPS (`ufw allow 80`), dejando tu servidor expuesto a escaneos de vulnerabilidades y ataques directos a tu IP.

Con **Cloudflare Tunnel**:
- Tu servidor VPS realiza una **conexión de salida segura** hacia Cloudflare.
- **No necesitas abrir ningún puerto de entrada en el firewall del VPS**.
- El tráfico entra seguro por Cloudflare y viaja de forma cifrada a través del túnel hasta tu servidor.
- **Un solo túnel sirve para múltiples aplicaciones**: Puedes tener `n8n`, una web de `WordPress` y una `App en Python` ejecutándose en puertos diferentes de Docker (`5678`, `8081`, `8080`) y gestionarlas todas bajo el mismo túnel.

---

## 3. Guía Paso a Paso: Configuración completa de Cloudflare

### Paso 1: Conectar tu dominio a Cloudflare

1. Crea una cuenta gratuita en [Cloudflare.com](https://www.cloudflare.com/).
2. Haz clic en **Añadir un sitio** e introduce tu dominio (ej. `midominio.es`). Selecciona el **Plan Gratuito (Free)**.
3. Cloudflare escaneará los registros DNS existentes.
4. Copia los dos **Servidores de Nombres (Nameservers)** que te asignará Cloudflare (ej. `aria.ns.cloudflare.com` y `bob.ns.cloudflare.com`).
5. Accede al panel donde compraste tu dominio (DonDominio, GoDaddy, Namecheap, etc.) y reemplaza los Nameservers de tu proveedor por los dos que te dio Cloudflare.
6. Espera unos minutos hasta que Cloudflare confirme que tu dominio está protegido.

---

### Paso 2: Crear el Túnel en Cloudflare Zero Trust

1. En el panel de Cloudflare, ve a la sección lateral izquierda y selecciona **Zero Trust**.
2. Ve a **Networks** > **Tunnels** y haz clic en **Create a Tunnel**.
3. Elige la opción **Cloudflared** y ponle un nombre identificativo a tu túnel (ej. `VPS-Principal`).
4. En la pantalla de instalación del conector, selecciona **Debian / Ubuntu** y **64-bit**.
5. Cloudflare te dará un comando con una clave o token único. Copia e instala el servicio en tu VPS vía SSH:

```bash
# 1. Descargar clave GPG e instalar repositorio oficial
curl -fsSL https://pkg.cloudflare.com/cloudflare-public-v2.gpg | sudo tee /usr/share/keyrings/cloudflare-public-v2.gpg >/dev/null
echo 'deb [signed-by=/usr/share/keyrings/cloudflare-public-v2.gpg] https://pkg.cloudflare.com/cloudflared any main' | sudo tee /etc/apt/sources.list.d/cloudflared.list

# 2. Instalar el paquete cloudflared
sudo apt-get update && sudo apt-get install cloudflared -y

# 3. Registrar e instalar el servicio del túnel con tu token
sudo cloudflared service install el_token_proporcionado_por_cloudflare...
```

> ⚠️ **IMPORTANTE**: El token de Cloudflare es secreto. No lo subas a ningún repositorio público de Git.

---

### Paso 3: Crear las Rutas de Aplicación (*Public Hostnames*)

Una vez instalado el túnel en el VPS, solo debes indicarle a Cloudflare a qué subdominio y puerto local de Docker debe enviar las peticiones.

En la pestaña **Public Hostname** de tu túnel en Cloudflare Zero Trust, haz clic en **Add a public hostname**:

#### Ejemplo 1: Enrutar n8n
- **Subdomain**: `n8n`
- **Domain**: `midominio.es`
- **Type**: `HTTP`
- **URL**: `localhost:5678`  *(o la IP interna del puerto del contenedor n8n)*

#### Ejemplo 2: Enrutar Web WordPress
- **Subdomain**: *(déjalo en blanco si es el dominio principal `midominio.es` o pon `web` para un subdominio)*
- **Domain**: `midominio.es`
- **Type**: `HTTP`
- **URL**: `localhost:8081`  *(el puerto expuesto en tu `docker-compose.yml`)*

#### Ejemplo 3: Enrutar App en Python
- **Subdomain**: `app`
- **Domain**: `midominio.es`
- **Type**: `HTTP`
- **URL**: `localhost:8080`

---

## 4. Resumen de uso en las prácticas del proyecto

A lo largo de nuestros apuntes y despliegues, Cloudflare se ha utilizado de la siguiente forma:

- 📄 **[4.N8n_Docker.md](4.N8n_Docker.md)**: Instalamos `cloudflared` por primera vez en el VPS y configuramos el primer subdominio (`n8n.midominio.es`) para acceder de forma segura por HTTPS a la plataforma de automatizaciones en el puerto `5678`.
- 📄 **[5.App_web_Docker.md](5.App_web_Docker.md)**: Reutilizamos el túnel creado anteriormente sin necesidad de instalar nada nuevo en el VPS, agregando un segundo *Public Hostname* (`app.midominio.es`) apuntando al puerto `8080`.
- 📄 **[6.Crear_web_Docker.md](6.Crear_web_Docker.md)**: Eliminamos los registros A/CNAME antiguos que apuntaban a un hosting viejo y asociamos el dominio principal a la web migrada en WordPress en el puerto `8081`.

---

## 5. Mantenimiento y Resolución de problemas comunes

### El servicio `cloudflared` se detiene tras actualizar la VPS

Si tras una actualización del sistema operativo del VPS tu túnel deja de funcionar y el servicio da error:

1. **Verificar el estado del servicio**:
   ```bash
   systemctl status cloudflared
   ```

2. **Revisar los registros de error**:
   ```bash
   sudo journalctl -u cloudflared -n 50 --no-pager
   ```
   *Si observas el mensaje `Failed to read token file: open /etc/cloudflared/token: no such file or directory`, el token fue eliminado en la actualización.*

3. **Solución**: Recrea el archivo introduciendo el token y reinicia el servicio:
   ```bash
   sudo nano /etc/cloudflared/token
   # Pega el token y guarda los cambios

   sudo systemctl restart cloudflared
   ```
