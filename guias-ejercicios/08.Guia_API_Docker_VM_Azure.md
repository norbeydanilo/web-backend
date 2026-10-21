# API REST con Docker en una VM de Azure
## Guía práctica: despliegue de contenedor en IaaS

**Computación en la nube**

---

## ¿Qué vamos a hacer?

En guías anteriores construimos una API REST con Node.js y la ejecutamos de dos formas: directamente con `node index.js` en local y dentro de un contenedor Docker en local. En esta guía vamos a hacer lo mismo pero en una Máquina Virtual de Azure.

El flujo completo es:

```
Tu equipo                         VM en Azure (IaaS)
──────────────────                ─────────────────────────────
Código de la API    ──── scp ──→  Archivos en la VM
                                  Docker Engine instalado
                                  Imagen construida con docker build
                                  Contenedor corriendo con docker run
Postman             ──── HTTP ──→ IP pública:3000
```

La diferencia respecto al despliegue en App Service (PaaS) es que aquí nosotros instalamos Docker, gestionamos el contenedor y abrimos los puertos manualmente. Azure solo nos da la VM.

---

## Requisitos previos

- VM de Azure corriendo Ubuntu 22.04 con IP pública (guía de VM anterior)
- Llave SSH generada y configurada para conectarse a la VM
- Archivos de la API listos en tu equipo: `package.json`, `index.js` y `Dockerfile`
- Postman instalado en tu equipo

---

## Parte 1: Conectarse a la VM

Abre una terminal y conéctate a la VM:

**Mac / Linux:**
```bash
ssh -i ~/.ssh/vm-taller-key azureuser@[IP-PÚBLICA-DE-TU-VM]
```

**Windows PowerShell:**
```powershell
ssh -i "$HOME\.ssh\vm-taller-key" azureuser@[IP-PÚBLICA-DE-TU-VM]
```

Cuando aparezca el prompt `azureuser@vm-taller-intro:~$` estás dentro de la VM.

---

## Parte 2: Instalar Docker Engine en la VM

En la VM corremos Docker Engine, que es el motor de contenedores sin interfaz gráfica. Docker Desktop (lo que usamos en local) incluye Docker Engine más una UI para escritorio — en un servidor esa UI no tiene sentido.

### 2.1 Actualizar paquetes

```bash
sudo apt update && sudo apt upgrade -y
```

### 2.2 Instalar dependencias previas

```bash
sudo apt install -y ca-certificates curl gnupg
```

### 2.3 Agregar el repositorio oficial de Docker

```bash
# Crear carpeta para llaves de repositorios externos
sudo install -m 0755 -d /etc/apt/keyrings

# Descargar y guardar la llave GPG de Docker
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
  sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

sudo chmod a+r /etc/apt/keyrings/docker.gpg

# Agregar el repositorio de Docker a las fuentes de apt
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

### 2.4 Instalar Docker Engine

```bash
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

### 2.5 Verificar que Docker está corriendo

```bash
sudo systemctl status docker
```

Debes ver `Active: active (running)`. Presiona `q` para salir.

### 2.6 Permitir usar Docker sin sudo

Por defecto Docker requiere `sudo` en cada comando. Para evitarlo, agregamos el usuario actual al grupo `docker`:

```bash
sudo usermod -aG docker azureuser
```

Este cambio no aplica en la sesión actual. Cierra la conexión SSH y vuelve a conectarte:

```bash
exit
```

Conéctate nuevamente con el mismo comando `ssh` que usaste antes. Ahora puedes usar Docker sin `sudo`.

### 2.7 Confirmar la instalación

```bash
docker --version
docker run hello-world
```

Si ves el mensaje de bienvenida de Docker, la instalación fue exitosa.

---

## Parte 3: Copiar los archivos de la API a la VM

Vamos a copiar los archivos de la API desde tu equipo a la VM usando `scp` (Secure Copy), que funciona sobre SSH.

**Abre una nueva terminal en tu equipo** (no en la VM) y navega a la carpeta de tu API:

```bash
cd [ruta-de-tu-carpeta-mi-api]
```

### 3.1 Crear la carpeta en la VM

Primero crea la carpeta destino en la VM:

```bash
# Mac / Linux
ssh -i ~/.ssh/vm-taller-key azureuser@[IP-PÚBLICA] "mkdir -p ~/mi-api"

# Windows PowerShell
ssh -i "$HOME\.ssh\vm-taller-key" azureuser@[IP-PÚBLICA] "mkdir -p ~/mi-api"
```

### 3.2 Copiar los archivos con scp

```bash
# Mac / Linux
scp -i ~/.ssh/vm-taller-key \
  package.json index.js Dockerfile \
  azureuser@[IP-PÚBLICA]:~/mi-api/

# Windows PowerShell
scp -i "$HOME\.ssh\vm-taller-key" `
  package.json index.js Dockerfile `
  azureuser@[IP-PÚBLICA]:~/mi-api/
```

> `scp` usa la misma llave SSH que usas para conectarte. El formato del destino es `usuario@ip:ruta`.

### 3.3 Verificar que los archivos llegaron

Conéctate a la VM y comprueba:

```bash
ssh -i ~/.ssh/vm-taller-key azureuser@[IP-PÚBLICA]
ls -la ~/mi-api/
```

Debes ver los tres archivos: `package.json`, `index.js` y `Dockerfile`.

---

## Parte 4: Construir la imagen y ejecutar el contenedor

Ya estás dentro de la VM. Navega a la carpeta de la API:

```bash
cd ~/mi-api
```

### 4.1 Construir la imagen Docker

```bash
docker build -t mi-api:1.0 .
```

Verás cada paso del Dockerfile ejecutándose. La primera vez tarda un par de minutos porque descarga la imagen base `node:20-alpine`.

Verifica que la imagen fue creada:

```bash
docker images
```

### 4.2 Ejecutar el contenedor

```bash
docker run -d -p 3000:3000 --name mi-api-container --restart unless-stopped mi-api:1.0
```

- `-d` → corre en segundo plano
- `-p 3000:3000` → mapea el puerto 3000 del contenedor al 3000 de la VM
- `--restart unless-stopped` → el contenedor se reinicia automáticamente si la VM se reinicia

Verifica que está corriendo:

```bash
docker ps
```

Verifica los logs:

```bash
docker logs mi-api-container
```

Debes ver: `API corriendo en http://localhost:3000`

### 4.3 Probar desde dentro de la VM

Antes de abrir el puerto hacia internet, verifica que el contenedor responde internamente:

```bash
curl http://localhost:3000/[tu-entidad]
```

Reemplaza `[tu-entidad]` con el nombre de tu recurso (libros, videojuegos, mascotas, etc.). Si devuelve el JSON con tus datos, el contenedor está funcionando correctamente.

---

## Parte 5: Abrir el puerto 3000 en el NSG

El contenedor está corriendo dentro de la VM, pero el firewall de Azure (NSG) todavía no permite tráfico externo en el puerto 3000. Hay que agregar la regla.

1. En el portal de Azure, ve a tu VM → **Redes**
2. Haz clic en **Crear ACL del puerto** → **Regla de puerto de entrada**
3. Completa los campos:

| Campo | Valor |
|---|---|
| **Origen** | Any |
| **Intervalos de puertos de origen** | * |
| **Destino** | Any |
| **Servicio** | Custom |
| **Intervalos de puertos de destino** | `3000` |
| **Protocolo** | Any |
| **Acción** | Permitir |
| **Prioridad** | `330` |
| **Nombre** | `allow-3000` |

4. Haz clic en **Agregar**

Espera 30 segundos para que la regla se propague.

---

## Parte 6: Probar la API con Postman

Abre Postman en tu equipo. La URL base ahora es la IP pública de la VM en el puerto 3000:

```
http://[IP-PÚBLICA-DE-TU-VM]:3000
```

Prueba cada endpoint:

| Método | URL | Body |
|---|---|---|
| GET | `http://[IP]:3000/[entidad]` | No |
| GET | `http://[IP]:3000/[entidad]/1` | No |
| POST | `http://[IP]:3000/[entidad]` | JSON con los campos de tu entidad |
| PUT | `http://[IP]:3000/[entidad]/1` | JSON con los campos actualizados |
| DELETE | `http://[IP]:3000/[entidad]/1` | No |

Para POST y PUT: en Postman ir a **Body** → **raw** → seleccionar **JSON** en el menú desplegable de la derecha.

> Recuerda que los datos son quemados en memoria. Al reiniciar el contenedor, los cambios hechos con POST, PUT y DELETE se pierden y vuelven a los datos iniciales del array.

---

## Parte 7: Comandos útiles para gestionar el contenedor en la VM

```bash
# Ver contenedores en ejecución
docker ps

# Ver logs en tiempo real
docker logs -f mi-api-container

# Detener el contenedor
docker stop mi-api-container

# Reiniciar el contenedor
docker restart mi-api-container

# Eliminar el contenedor
docker rm -f mi-api-container

# Ver el uso de recursos del contenedor
docker stats mi-api-container
```

---

## Parte 8: Actualizar la API (redespliegue)

Si haces cambios en el código de la API en tu equipo y quieres actualizar el contenedor en la VM:

**Paso 1:** Copia los archivos modificados desde tu equipo a la VM:

```bash
# Mac / Linux
scp -i ~/.ssh/vm-taller-key \
  index.js \
  azureuser@[IP-PÚBLICA]:~/mi-api/

# Windows PowerShell
scp -i "$HOME\.ssh\vm-taller-key" `
  index.js `
  azureuser@[IP-PÚBLICA]:~/mi-api/
```

**Paso 2:** Conéctate a la VM y reconstruye la imagen:

```bash
ssh -i ~/.ssh/vm-taller-key azureuser@[IP-PÚBLICA]
cd ~/mi-api
docker stop mi-api-container
docker rm mi-api-container
docker build -t mi-api:1.0 .
docker run -d -p 3000:3000 --name mi-api-container --restart unless-stopped mi-api:1.0
```

**Paso 3:** Verifica que el cambio está activo:

```bash
curl http://localhost:3000/[tu-entidad]
```

> Esta secuencia manual de pasos es exactamente lo que un pipeline CI/CD (como Azure Pipelines) automatiza. Al hacer un `git push`, el pipeline copia el código, reconstruye la imagen y reinicia el contenedor sin intervención humana. Eso lo veremos más adelante en el curso.

---

## Parte 9: Detener la VM para conservar créditos

Cuando termines, recuerda detener la VM desde el portal:

1. Ve a tu VM en el portal de Azure
2. Haz clic en **Detener** en la barra superior
3. Confirma la acción

El estado debe cambiar a **Detenida (desasignada)**. El contenedor Docker queda guardado en el disco de la VM y vuelve a ejecutarse automáticamente cuando la VM se inicia gracias al flag `--restart unless-stopped`.

> Nota: cuando se reinicia la VM, la IP pública puede cambiar si no se configuró una IP estática. Anotar la nueva IP antes de intentar conectarse o hacer peticiones desde Postman.

---

## Comparación de los tres entornos donde corrimos la API

| | Local (node) | Local (Docker) | VM Azure (Docker) |
|---|---|---|---|
| Dónde corre | Tu equipo | Tu equipo | Datacenter de Azure |
| Accesible desde internet | No | No | Sí |
| Requiere Docker | No | Sí | Sí (Engine) |
| Requiere gestionar infraestructura | No | No | Sí (VM, NSG, disco) |
| Se apaga al cerrar el equipo | Sí | Sí | No (independiente) |
| Modelo de servicio | N/A | N/A | IaaS |

---

## Solución de problemas frecuentes

| Problema | Causa | Solución |
|---|---|---|
| `Got permission denied while trying to connect to Docker` | El usuario no está en el grupo `docker` | Ejecutar `sudo usermod -aG docker azureuser`, cerrar la sesión SSH y volver a conectarse |
| `scp: [ruta]: No such file or directory` | La carpeta destino no existe en la VM | Ejecutar `ssh ... "mkdir -p ~/mi-api"` antes del `scp` |
| Postman devuelve `Could not get any response` | El puerto 3000 no está abierto en el NSG o el contenedor no está corriendo | Verificar la regla del NSG y ejecutar `docker ps` en la VM |
| El contenedor aparece como `Exited` en `docker ps` | Error en la app al iniciar | Revisar `docker logs mi-api-container` para ver el error exacto |
| La IP cambió después de reiniciar la VM | No se configuró IP estática | Ir a Información general de la VM y anotar la nueva IP pública |
| `connection refused` en curl desde dentro de la VM | El contenedor no está corriendo o el puerto no está mapeado | Ejecutar `docker ps` y verificar que aparece `0.0.0.0:3000->3000/tcp` |

---

*Computación en la nube · API REST con Docker en una VM de Azure*
