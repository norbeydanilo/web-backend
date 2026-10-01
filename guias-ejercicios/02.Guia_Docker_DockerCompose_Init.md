# Introducción a Docker y Docker Compose
## Guía práctica para principiantes

**Computación en la nube**

---

## ¿Por qué Docker?

Imagina que desarrollas una aplicación en tu equipo con Node.js 20, pero el servidor de producción tiene Node.js 18 y la app falla. O que un compañero no puede ejecutar tu proyecto porque le faltan dependencias del sistema operativo. Este problema se conoce como **"funciona en mi máquina"** (*works on my machine*).

Docker resuelve esto empaquetando la aplicación junto con todo lo que necesita para ejecutarse: el runtime, las dependencias del sistema, las variables de entorno y la configuración. El resultado es una unidad llamada **contenedor** que se comporta de la misma manera en cualquier máquina que tenga Docker instalado.

En el contexto del curso, Docker es el paso previo al despliegue en Azure: primero construyes y pruebas el contenedor en local, luego lo subes a la nube.

---

## Conceptos fundamentales

### Imagen
Una imagen es una **plantilla inmutable** que describe todo lo necesario para ejecutar una aplicación: el sistema base, las dependencias, el código y el comando de inicio. Es como la receta de un plato.

### Contenedor
Un contenedor es una **instancia en ejecución de una imagen**. De la misma imagen puedes crear uno, diez o cien contenedores ejecutándose al mismo tiempo. Es el plato ya preparado.

### Dockerfile
Un archivo de texto con instrucciones que le dicen a Docker cómo construir una imagen. Es la receta escrita.

### Registry
Un repositorio donde se almacenan y distribuyen imágenes. Docker Hub es el más conocido (hub.docker.com). Azure tiene el suyo propio: Azure Container Registry (ACR), que veremos más adelante en el curso.

---

## Parte 1 — Instalación

### 1.1 Instalar Docker Desktop

Docker Desktop incluye Docker Engine, Docker CLI y Docker Compose en una sola instalación.

1. Descarga Docker Desktop desde [docker.com/products/docker-desktop](https://www.docker.com/products/docker-desktop/)
2. Ejecuta el instalador y sigue los pasos
3. Reinicia el equipo si se solicita
4. Abre Docker Desktop y espera a que el ícono de la ballena en la barra de tareas se ponga en verde

### 1.2 Verificar la instalación

Abre una terminal (PowerShell, Command Prompt o Terminal) y ejecuta:

```bash
docker --version
docker compose version
```

Debes ver algo similar a:

```
Docker version 26.x.x, build ...
Docker Compose version v2.x.x
```

### 1.3 Primera prueba

```bash
docker run hello-world
```

Docker descargará la imagen `hello-world` desde Docker Hub y ejecutará un contenedor que imprime un mensaje de bienvenida. Si ves el mensaje, Docker está funcionando correctamente.

---

## Parte 2 — Primeros comandos

### 2.1 Ejecutar un contenedor de nginx

```bash
docker run -p 8080:80 nginx
```

Abre el navegador en `http://localhost:8080`. Verás la página de bienvenida de nginx.

> **¿Qué significa `-p 8080:80`?**
> Mapea el puerto 80 del contenedor (donde nginx escucha) al puerto 8080 de tu máquina. El formato es siempre `-p [puerto-local]:[puerto-contenedor]`.

Presiona `Ctrl+C` para detener el contenedor.

### 2.2 Ejecutar en segundo plano

Para que el contenedor corra sin bloquear la terminal, agrega `-d` (detached):

```bash
docker run -d -p 8080:80 --name mi-nginx nginx
```

### 2.3 Comandos básicos de gestión

```bash
# Ver contenedores en ejecución
docker ps

# Ver todos los contenedores (incluyendo detenidos)
docker ps -a

# Ver imágenes descargadas
docker images

# Detener un contenedor
docker stop mi-nginx

# Eliminar un contenedor
docker rm mi-nginx

# Eliminar una imagen
docker rmi nginx

# Ver logs de un contenedor
docker logs mi-nginx

# Entrar a la terminal de un contenedor en ejecución
docker exec -it mi-nginx bash
```

---

## Parte 3 — Construir una imagen con Dockerfile

Vamos a contenerizar la misma app Node.js del Taller 2. Si no la tienes, crea la carpeta `mi-app-docker` con estos tres archivos:

### 3.1 Estructura del proyecto

```
mi-app-docker/
├── Dockerfile
├── package.json
├── server.js
└── index.html
```

### 3.2 Archivos de la aplicación

**`package.json`**
```json
{
  "name": "mi-app-docker",
  "version": "1.0.0",
  "description": "App Node.js en contenedor Docker",
  "main": "server.js",
  "scripts": {
    "start": "node server.js"
  },
  "engines": {
    "node": ">=18.0.0"
  }
}
```

**`server.js`**
```javascript
const http = require('http');
const fs   = require('fs');
const path = require('path');

const PORT = process.env.PORT || 3000;

const server = http.createServer((req, res) => {

  if (req.url === '/estado') {
    res.writeHead(200, { 'Content-Type': 'application/json' });
    res.end(JSON.stringify({
      estado: 'activo',
      entorno: process.env.ENTORNO || 'local',
      runtime: 'Node.js en Docker',
      timestamp: new Date().toISOString()
    }));
    return;
  }

  const filePath = path.join(__dirname, 'index.html');
  fs.readFile(filePath, (err, content) => {
    if (err) {
      res.writeHead(500);
      res.end('Error al cargar la página');
      return;
    }
    res.writeHead(200, { 'Content-Type': 'text/html; charset=utf-8' });
    res.end(content);
  });
});

server.listen(PORT, () => {
  console.log(`Servidor corriendo en el puerto ${PORT}`);
});
```

**`index.html`**
```html
<!DOCTYPE html>
<html lang="es">
<head>
  <meta charset="UTF-8" />
  <title>Mi app en Docker</title>
  <style>
    * { box-sizing: border-box; margin: 0; padding: 0; }
    body {
      font-family: 'Segoe UI', Arial, sans-serif;
      background: linear-gradient(135deg, #2c3e50, #1a252f);
      min-height: 100vh;
      display: flex;
      align-items: center;
      justify-content: center;
    }
    .card {
      background: #fff;
      border-radius: 12px;
      padding: 48px 40px;
      max-width: 540px;
      width: 90%;
      box-shadow: 0 8px 32px rgba(0,0,0,0.3);
      text-align: center;
    }
    .badge {
      display: inline-block;
      background: #e8f4fd;
      color: #1565c0;
      font-size: 13px;
      font-weight: 600;
      padding: 4px 14px;
      border-radius: 20px;
      margin-bottom: 20px;
    }
    h1 { font-size: 26px; color: #2c3e50; margin-bottom: 12px; }
    p  { font-size: 15px; color: #555; line-height: 1.7; margin-bottom: 24px; }
    .info-grid {
      display: grid;
      grid-template-columns: 1fr 1fr;
      gap: 12px;
      margin-bottom: 24px;
    }
    .info-item { background: #f5f9ff; border-radius: 8px; padding: 12px 10px; }
    .info-label { font-size: 11px; color: #888; text-transform: uppercase; letter-spacing: 0.06em; margin-bottom: 4px; }
    .info-value { font-size: 14px; font-weight: 600; color: #2c3e50; }
    .status {
      display: inline-flex; align-items: center; gap: 7px;
      background: #eaf3de; color: #3b6d11;
      font-size: 13px; font-weight: 600;
      padding: 8px 18px; border-radius: 20px; margin-bottom: 16px;
    }
    .dot { width: 8px; height: 8px; border-radius: 50%; background: #3b6d11; }
    footer { font-size: 12px; color: #aaa; border-top: 1px solid #eee; padding-top: 14px; margin-top: 8px; }
  </style>
</head>
<body>
  <div class="card">
    <div class="badge">🐳 Corriendo en Docker</div>
    <h1>¡App Node.js en contenedor!</h1>
    <p>Esta aplicación está empaquetada en un contenedor Docker. El sistema operativo, el runtime y las dependencias están aislados de tu máquina.</p>
    <div class="info-grid">
      <div class="info-item">
        <div class="info-label">Tecnología</div>
        <div class="info-value">Node.js</div>
      </div>
      <div class="info-item">
        <div class="info-label">Entorno</div>
        <div class="info-value">Docker</div>
      </div>
      <div class="info-item">
        <div class="info-label">Puerto interno</div>
        <div class="info-value">3000</div>
      </div>
      <div class="info-item">
        <div class="info-label">Aislamiento</div>
        <div class="info-value">Contenedor</div>
      </div>
    </div>
    <div class="status"><span class="dot"></span>Contenedor activo</div>
    <footer>Computación en la nube · Introducción a Docker</footer>
  </div>
</body>
</html>
```

### 3.3 El Dockerfile

Crea el archivo `Dockerfile` (sin extensión) en la raíz del proyecto:

```dockerfile
# Imagen base: Node.js 20 en Alpine Linux (imagen mínima, ~50MB)
FROM node:20-alpine

# Directorio de trabajo dentro del contenedor
# Todos los comandos siguientes se ejecutan desde aquí
WORKDIR /app

# Copiar primero solo los archivos de dependencias
# Esto aprovecha el caché de Docker: si package.json no cambió,
# no se vuelve a ejecutar npm install en el siguiente build
COPY package.json ./

# Instalar dependencias
RUN npm install --omit=dev

# Copiar el resto del código
COPY server.js ./
COPY index.html ./

# Documentar el puerto que expone el contenedor
# (no publica el puerto — eso se hace al ejecutar con -p)
EXPOSE 3000

# Comando que se ejecuta al iniciar el contenedor
CMD ["node", "server.js"]
```

> **¿Por qué copiar `package.json` antes que el resto del código?**
> Docker construye imágenes por capas. Si copias todo de una vez y cambias una línea de `server.js`, Docker invalida el caché desde esa capa y vuelve a ejecutar `npm install`. Separando el `COPY package.json` del `COPY server.js`, `npm install` solo se re-ejecuta cuando cambian las dependencias, no cada vez que cambias el código.

### 3.4 Construir la imagen

Desde la carpeta `mi-app-docker`, ejecuta:

```bash
docker build -t mi-app-node:1.0 .
```

- `-t mi-app-node:1.0` → nombre e imagen (tag). El tag `1.0` es la versión.
- `.` → el contexto de build es la carpeta actual (donde está el Dockerfile).

Verás cada instrucción del Dockerfile ejecutándose como un paso numerado. La primera vez tardará un poco mientras descarga la imagen base `node:20-alpine`. Los builds siguientes serán más rápidos gracias al caché.

Verifica que la imagen fue creada:

```bash
docker images
```

### 3.5 Ejecutar el contenedor

```bash
docker run -d -p 3000:3000 --name mi-app-node mi-app-node:1.0
```

Abre el navegador en `http://localhost:3000`. Verás la página de la app.

Prueba también el endpoint de estado:

```bash
curl http://localhost:3000/estado
```

O abre `http://localhost:3000/estado` en el navegador.

### 3.6 Pasar variables de entorno

Puedes pasar variables al contenedor con `-e`:

```bash
docker stop mi-app-node
docker rm mi-app-node

ddocker run -d -p 3000:3000 -e ENTORNO=desarrollo -e PORT=3000 --name mi-app-node mi-app-node:1.0
```

Visita `/estado` y verás `"entorno": "desarrollo"` en la respuesta JSON.

> **Conexión con Azure:** en el Taller 2, Azure App Service inyectaba `PORT` como variable de entorno. Docker hace exactamente lo mismo: el contenedor lee `process.env.PORT` sin importar quién lo establezca — la plataforma local o Azure.

---

## Parte 4 — Docker Compose

### 4.1 ¿Para qué sirve Docker Compose?

Hasta ahora levantaste un solo contenedor con un comando largo. En proyectos reales, una aplicación típica necesita varios contenedores: el servidor web, la base de datos, un proxy inverso, un servicio de caché. Administrarlos uno por uno con `docker run` se vuelve inmanejable.

Docker Compose permite definir **todos los servicios de una aplicación en un solo archivo YAML** y levantarlos o apagarlos con un único comando.

### 4.2 Estructura de `docker-compose.yml`

Crea el archivo `docker-compose.yml` en la raíz del proyecto `mi-app-docker`:

```yaml
# Archivo: docker-compose.yml

services:

  # Servicio 1: la aplicación Node.js
  web:
    # Construye la imagen desde el Dockerfile en la carpeta actual
    build: .
    # Publica el puerto 3000 del contenedor en el puerto 3000 del host
    ports:
      - "3000:3000"
    # Variables de entorno disponibles dentro del contenedor
    environment:
      - PORT=3000
      - ENTORNO=local-compose
    # Reinicia el contenedor si falla
    restart: unless-stopped

  # Servicio 2: proxy inverso con nginx
  proxy:
    # Usa la imagen oficial de nginx sin construir nada
    image: nginx:alpine
    # nginx escucha en el 80 interno, lo exponemos en el 8080 del host
    ports:
      - "8080:80"
    # Monta el archivo de configuración de nginx desde el host al contenedor
    volumes:
      - ./nginx.conf:/etc/nginx/conf.d/default.conf:ro
    # Este servicio depende de que 'web' esté levantado antes
    depends_on:
      - web
    restart: unless-stopped
```

### 4.3 Configuración de nginx

Crea el archivo `nginx.conf` en la misma carpeta:

```nginx
server {
    listen 80;

    location / {
        # nginx redirige el tráfico al servicio 'web' en el puerto 3000
        # Docker Compose crea automáticamente una red donde los servicios
        # se resuelven por su nombre (web, proxy, etc.)
        proxy_pass http://web:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

> **Red automática en Compose:** cuando Docker Compose levanta los servicios, crea una red virtual privada donde cada servicio puede acceder a los demás por su nombre. Por eso nginx puede usar `http://web:3000` — `web` es el nombre del servicio definido en el YAML, no una IP fija.

### 4.4 Estructura final del proyecto

```
mi-app-docker/
├── Dockerfile
├── docker-compose.yml
├── nginx.conf
├── package.json
├── server.js
└── index.html
```

### 4.5 Levantar los servicios

```bash
docker compose up -d
```

- `up` → crea y levanta todos los servicios definidos en `docker-compose.yml`
- `-d` → en segundo plano (detached)

Docker construirá la imagen de `web` (si no existe) y descargará la de `nginx:alpine`. Luego levantará ambos contenedores.

Verifica que ambos servicios estén corriendo:

```bash
docker compose ps
```

### 4.6 Probar los dos puntos de acceso

| URL | Descripción |
|---|---|
| `http://localhost:3000` | Acceso directo a la app Node.js |
| `http://localhost:8080` | Acceso a través del proxy nginx |

Ambas deben mostrar la misma app. La diferencia es que en producción solo expondrías el puerto 8080 (nginx) y el puerto 3000 quedaría interno — el mundo exterior solo ve el proxy.

### 4.7 Ver logs de todos los servicios

```bash
# Logs de todos los servicios
docker compose logs

# Logs en tiempo real
docker compose logs -f

# Logs de un servicio específico
docker compose logs -f web
```

### 4.8 Detener y limpiar

```bash
# Detener los contenedores (sin eliminarlos)
docker compose stop

# Detener y eliminar contenedores y red
docker compose down

# Detener, eliminar contenedores, red e imágenes construidas
docker compose down --rmi local
```

---

## Parte 5 — Reconstruir después de cambios

Cuando modificas el código de la app, necesitas reconstruir la imagen:

```bash
# Reconstruir la imagen de 'web' y reiniciar solo ese servicio
docker compose up -d --build web
```

Prueba: modifica el título en `index.html`, guarda y ejecuta el comando anterior. Recarga el navegador y verás el cambio.

---

## Parte 6 — Comandos de referencia rápida

### Docker

```bash
# Construcción
docker build -t nombre:tag .            # Construir imagen
docker build --no-cache -t nombre:tag . # Construir sin caché

# Ejecución
docker run -d -p host:contenedor --name nombre imagen  # Ejecutar en background
docker run -it imagen bash              # Ejecutar interactivo con terminal

# Gestión de contenedores
docker ps                  # Contenedores en ejecución
docker ps -a               # Todos los contenedores
docker stop nombre         # Detener
docker start nombre        # Iniciar detenido
docker rm nombre           # Eliminar (debe estar detenido)
docker rm -f nombre        # Eliminar forzando detención

# Gestión de imágenes
docker images              # Listar imágenes
docker rmi nombre:tag      # Eliminar imagen
docker pull imagen:tag     # Descargar imagen sin ejecutar

# Inspección y logs
docker logs nombre         # Ver logs
docker logs -f nombre      # Logs en tiempo real
docker exec -it nombre sh  # Abrir shell en contenedor activo
docker inspect nombre      # Información detallada en JSON

# Limpieza
docker system prune        # Eliminar todo lo no usado (pide confirmación)
docker system prune -a     # Incluye imágenes no referenciadas
```

### Docker Compose

```bash
docker compose up -d             # Levantar todos los servicios
docker compose up -d --build     # Levantar reconstruyendo imágenes
docker compose down              # Bajar y eliminar contenedores y red
docker compose ps                # Ver estado de los servicios
docker compose logs -f           # Logs en tiempo real de todos
docker compose logs -f servicio  # Logs de un servicio específico
docker compose exec web sh       # Shell en el servicio 'web'
docker compose restart web       # Reiniciar un servicio
docker compose stop              # Detener sin eliminar
docker compose start             # Iniciar servicios detenidos
```

---

## Solución de problemas frecuentes

| Problema | Causa | Solución |
|---|---|---|
| `port is already allocated` | El puerto ya está en uso en tu máquina | Cambia el puerto del host: `-p 3001:3000` |
| `Cannot connect to Docker daemon` | Docker Desktop no está corriendo | Abre Docker Desktop y espera a que el ícono esté en verde |
| La app no refleja los cambios | La imagen no fue reconstruida | Ejecuta `docker compose up -d --build` |
| `no such file or directory` en el Dockerfile | Path incorrecto en `COPY` | Verifica que el archivo existe y que el `COPY` use la ruta relativa correcta |
| nginx devuelve 502 Bad Gateway | El servicio `web` aún no levantó | Espera unos segundos y recarga; en compose usa `depends_on` |
| El contenedor se reinicia en loop | Error en el CMD o en la app | Revisa `docker compose logs web` para ver el error |

---

## ¿Qué viene después?

En los próximos talleres del curso haremos:

1. **Subir la imagen a Azure Container Registry (ACR)** — el registro privado de imágenes de Azure
2. **Desplegar el contenedor en Azure Container Instances (ACI)** — desde la imagen en ACR, sin gestionar infraestructura
3. **Integrar el build de Docker en el pipeline CI/CD de Azure Pipelines** 

---

*Computación en la nube · Introducción a Docker y Docker Compose*
