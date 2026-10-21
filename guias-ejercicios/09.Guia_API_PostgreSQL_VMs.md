# API REST con PostgreSQL en dos VMs de Azure
## Parte 3: API en VM1 (Docker) y PostgreSQL en VM2

**Computación en la nube**

---

## Requisitos previos

Antes de continuar, asegúrate de haber completado la Guía anterior (Partes 1 y 2):

- La API ya está migrada a Sequelize y PostgreSQL.
- El `Dockerfile` está actualizado con la carpeta `src/`.
- La API funciona correctamente con Docker Compose en local.

En esta guía vamos a desplegar esa misma API en dos VMs de Azure: la API en un contenedor Docker en VM1 y PostgreSQL instalado directamente en VM2, comunicadas por la red privada interna de Azure.

---

## Parte 3: Dos VMs en Azure

### 3.1 Cómo se comunican dos VMs: VNet y red privada

Cuando dos VMs se crean en el mismo grupo de recursos en Azure, Azure las conecta automáticamente a la misma **red virtual (VNet)**, igual que cuando conectas dos computadores al mismo router de tu casa. Dentro de esa red, cada VM tiene una **IP privada** (del rango `10.0.x.x`) y pueden hablarse entre sí directamente.

La arquitectura que vamos a construir es:

```
Internet
    |
    |  puerto 22 (SSH) y puerto 3000 (API)
    |
[ VM1: API en Docker ]  ← IP pública + IP privada (ej: 10.0.0.4)
    |
    |  puerto 5432 (PostgreSQL)
    |  solo tráfico interno de la VNet
    |
[ VM2: PostgreSQL ]     ← solo IP privada (ej: 10.0.0.5)
                          sin IP pública: nadie en internet puede llegar aquí
```

VM2 no tiene IP pública, lo que significa que nadie en internet puede intentar conectarse directamente a la base de datos. Solo VM1 puede hablar con VM2, y solo por el puerto 5432. Eso es seguridad en capas aplicada de forma concreta.

Para conectarnos a VM2 desde nuestro equipo, usamos VM1 como puente: primero entramos a VM1, y desde ahí hacemos SSH a VM2 por su IP privada.

### 3.2 Crear VM1 (ya existe)

Si ya tienes la VM de las guías anteriores con la API corriendo en Docker, puedes reutilizarla. Solo asegúrate de que tiene abierto el puerto 3000 en su NSG.

Si no la tienes, créala siguiendo la guía de VMs anterior.

Anota la **IP privada de VM1**: está en el panel de Información general de la VM, en la sección Redes.

### 3.3 Crear VM2 (base de datos)

1. En el portal de Azure, crea una nueva VM con estos valores:

| Campo | Valor |
|---|---|
| **Grupo de recursos** | El mismo de VM1: `rg-taller-vm` |
| **Nombre** | `vm-bd-postgres` |
| **Región** | La misma que VM1 |
| **Imagen** | Ubuntu Server 22.04 LTS |
| **Tamaño** | El mismo tamaño elegible que usaste en VM1 |
| **Tipo de autenticación** | Clave pública SSH |
| **Nombre de usuario** | `azureuser` |
| **Origen de clave pública SSH** | Usar clave pública existente |
| **Clave pública SSH** | Pega el contenido de `~/.ssh/vm-taller-key.pub` |

2. En la pestaña **Redes**, configura así:

| Campo | Valor |
|---|---|
| **IP pública** | Ninguna |
| **Puertos de entrada públicos** | Ninguno |

> VM2 no necesita IP pública. Solo recibirá conexiones desde VM1, que está en la misma VNet.

3. Crea la VM y espera a que esté en estado "En ejecución".

4. Anota la **IP privada de VM2**: Información general → sección Redes → Dirección IP privada. Tendrá un formato como `10.0.0.5`.

### 3.4 Abrir el puerto 5432 en el NSG de VM2

VM2 necesita aceptar conexiones de PostgreSQL pero solo desde VM1. En el portal:

1. Abre VM2 → **Redes**
2. Haz clic en **Crear ACL del puerto** → **Regla de puerto de entrada**

| Campo | Valor |
|---|---|
| **Origen** | Dirección IP |
| **Intervalos de direcciones IP de origen** | IP privada de VM1 (ej: `10.0.0.4`) |
| **Intervalos de puertos de destino** | `5432` |
| **Protocolo** | TCP |
| **Acción** | Permitir |
| **Prioridad** | `310` |
| **Nombre** | `allow-postgres-from-vm1` |

3. Haz clic en **Agregar**

Esto permite que solo VM1 pueda conectarse al puerto 5432 de VM2. Cualquier intento desde fuera de la VNet es bloqueado.

### 3.5 Conectarse a VM2 usando VM1 como puente

VM2 no tiene IP pública, así que no podemos conectarnos directamente desde nuestro equipo. La forma de administrarla es entrar primero a VM1 y desde ahí saltar a VM2.

**Paso 1:** Copia la llave privada a VM1 para poder usarla desde ahí:

```bash
# Desde tu equipo local (Mac/Linux)
scp -i ~/.ssh/vm-taller-key ~/.ssh/vm-taller-key \
  azureuser@[IP-PÚBLICA-VM1]:~/.ssh/vm-taller-key

# Windows PowerShell
scp -i "$HOME\.ssh\vm-taller-key" "$HOME\.ssh\vm-taller-key" `
  azureuser@[IP-PÚBLICA-VM1]:~/.ssh/vm-taller-key
```

**Paso 2:** Conéctate a VM1:

```bash
# Mac/Linux
ssh -i ~/.ssh/vm-taller-key azureuser@[IP-PÚBLICA-VM1]

# Windows PowerShell
ssh -i "$HOME\.ssh\vm-taller-key" azureuser@[IP-PÚBLICA-VM1]
```

**Paso 3:** Desde dentro de VM1, conéctate a VM2 usando su IP privada:

```bash
chmod 400 ~/.ssh/vm-taller-key
ssh -i ~/.ssh/vm-taller-key azureuser@[IP-PRIVADA-VM2]
```

Ahora estás dentro de VM2. La terminal muestra `azureuser@vm-bd-postgres:~$`.

### 3.6 Instalar PostgreSQL en VM2

Ejecuta estos comandos dentro de VM2:

```bash
# Actualizar paquetes
sudo apt update && sudo apt upgrade -y

# Instalar PostgreSQL
sudo apt install -y postgresql postgresql-contrib

# Verificar que está corriendo
sudo systemctl status postgresql
```

Debes ver `Active: active (running)`.

### 3.7 Configurar PostgreSQL en VM2

**Paso 1:** Crear la base de datos y el usuario que usará la API:

```bash
sudo -u postgres psql << 'SQL'
CREATE DATABASE libros_db;
CREATE USER api_user WITH PASSWORD 'contraseña_segura_aqui';
GRANT ALL PRIVILEGES ON DATABASE libros_db TO api_user;
\q
SQL
```

> Cambia `contraseña_segura_aqui` por una contraseña real. Anótala porque la necesitarás en la configuración de la API.

**Paso 1B:** Otorgar permisos sobre el esquema `public` dentro de la base de datos.

A partir de PostgreSQL 15, los permisos sobre el esquema `public` ya no se otorgan automáticamente a usuarios nuevos. Sin este paso, Sequelize se conecta correctamente pero falla al intentar crear las tablas con `sync()`, mostrando el error `permission denied for schema public`.

```bash
sudo -u postgres psql -d libros_db -c "GRANT ALL ON SCHEMA public TO api_user;"
```

**Paso 2:** Ver qué versión de PostgreSQL se instaló:

```bash
ls /etc/postgresql/
```

Mostrará un número como `14` o `16`. Úsalo en los siguientes pasos.

**Paso 3:** Permitir que PostgreSQL acepte conexiones desde otras IPs (por defecto solo escucha en localhost):

```bash
# Reemplaza [VERSION] con el número que viste en el paso anterior
sudo nano /etc/postgresql/[VERSION]/main/postgresql.conf
```

Busca la línea `#listen_addresses = 'localhost'` y cámbiala a:

```
listen_addresses = '*'
```

Guarda con `Ctrl+O` + Enter y sal con `Ctrl+X`.

**Paso 4:** Configurar qué IPs pueden conectarse y con qué usuario:

```bash
sudo nano /etc/postgresql/[VERSION]/main/pg_hba.conf
```

Al final del archivo agrega esta línea:

```
host    libros_db    api_user    10.0.0.0/24    scram-sha-256
```

Esto permite que cualquier IP del rango `10.0.0.0/24` (toda la VNet) se conecte a `libros_db` con el usuario `api_user`. Si quieres restringir solo a VM1, reemplaza `10.0.0.0/24` por la IP privada exacta de VM1 con `/32` (ej: `10.0.0.4/32`).

Guarda y sal.

**Paso 5:** Reiniciar PostgreSQL para aplicar los cambios:

```bash
sudo systemctl restart postgresql
```

**Paso 6:** Verificar que PostgreSQL escucha en todas las interfaces:

```bash
sudo ss -tlnp | grep 5432
```

Debes ver `0.0.0.0:5432` en la salida.

**Paso 7:** Sal de VM2 y regresa a VM1:

```bash
exit
```

Ahora estás de nuevo en VM1.

### 3.8 Verificar la conexión desde VM1 a VM2

Desde VM1, prueba que puedes conectarte a PostgreSQL en VM2:

```bash
# Instalar el cliente de PostgreSQL en VM1 para hacer la prueba
sudo apt install -y postgresql-client

# Probar la conexión a VM2
psql -h [IP-PRIVADA-VM2] -U api_user -d libros_db
```

Ingresa la contraseña cuando la pida. Si ves el prompt `libros_db=>`, la conexión entre VMs funciona. Escribe `\q` para salir.

### 3.9 Desplegar la API en VM1 apuntando a VM2

**Paso 1:** Copia los archivos de la API desde tu equipo a VM1. Desde tu terminal local:

```bash
# Mac/Linux: copiar toda la carpeta (excepto node_modules)
scp -i ~/.ssh/vm-taller-key -r \
  index.js src/ package.json Dockerfile \
  azureuser@[IP-PÚBLICA-VM1]:~/mi-api/

# Windows PowerShell
scp -i "$HOME\.ssh\vm-taller-key" -r `
  index.js src/ package.json Dockerfile `
  azureuser@[IP-PÚBLICA-VM1]:~/mi-api/
```

**Paso 2:** Conéctate a VM1 y construye la imagen:

```bash
ssh -i ~/.ssh/vm-taller-key azureuser@[IP-PÚBLICA-VM1]
cd ~/mi-api
docker build -t mi-api:2.0 .
```

**Paso 3:** Corre el contenedor con las variables de entorno apuntando a VM2:

```bash
docker run -d -p 3000:3000 \
  -e DB_HOST=[IP-PRIVADA-VM2] \
  -e DB_PORT=5432 \
  -e DB_NAME=libros_db \
  -e DB_USER=api_user \
  -e DB_PASSWORD=contraseña_segura_aqui \
  -e PORT=3000 \
  --name mi-api-container \
  --restart unless-stopped \
  mi-api:2.0
```

**Paso 4:** Verificar los logs:

```bash
docker logs mi-api-container
```

Debes ver:

```
Conexión a la base de datos establecida.
Modelos sincronizados con la base de datos.
API corriendo en http://localhost:3000
```

Si aparece `Error al iniciar la aplicación`, revisar: la IP privada de VM2, la contraseña, y que el NSG de VM2 tenga la regla del puerto 5432.

### 3.10 Probar con Postman

La URL base es la IP pública de VM1 en el puerto 3000:

```
http://[IP-PÚBLICA-VM1]:3000
```

| Método | URL | Qué probar |
|---|---|---|
| GET | `/libros` | Devuelve `[]` inicialmente (BD vacía) |
| POST | `/libros` | Crea un libro — el body JSON debe incluir el `isbn` |
| GET | `/libros` | Ahora aparece el libro creado |
| GET | `/libros/978-0-06-112008-4` | Devuelve el libro con ese ISBN |
| PUT | `/libros/978-0-06-112008-4` | Actualiza titulo, autor, año o disponible (no el ISBN) |
| DELETE | `/libros/978-0-06-112008-4` | Elimina el libro con ese ISBN |

Para confirmar que los datos persisten: reinicia el contenedor en VM1 y vuelve a hacer GET.

```bash
docker restart mi-api-container
```

Los libros siguen ahí porque están en PostgreSQL en VM2, no en memoria. El ISBN que usaste al crear cada libro es la clave con la que los recuperas.

---

## Resumen: lo que cambió entre versiones

| | Array en memoria | Docker Compose local | Dos VMs en Azure |
|---|---|---|---|
| Dónde viven los datos | RAM del proceso | Volumen Docker en tu equipo | Disco de VM2 |
| Persisten al reiniciar | No | Sí | Sí |
| Accesible desde internet | No | No | Sí |
| BD separada de la API | No | Sí (contenedor) | Sí (VM distinta) |
| BD accesible desde internet | N/A | No | No (sin IP pública) |

---

## Conexión con lo que viene

En el siguiente paso del curso veremos **Azure Database for PostgreSQL Flexible Server**: el mismo PostgreSQL que instalamos manualmente en VM2, pero gestionado completamente por Azure. Azure se encarga de los parches, los backups, la alta disponibilidad y el escalado. La API no cambiará: solo se actualiza `DB_HOST` con el endpoint del servicio gestionado.

Eso es exactamente la diferencia entre IaaS y PaaS aplicada a bases de datos.

---

## Solución de problemas frecuentes

| Problema | Causa | Solución |
|---|---|---|
| `Error: connect ECONNREFUSED` | PostgreSQL no está corriendo o no acepta conexiones externas | En VM2: verificar `sudo systemctl status postgresql` y que `listen_addresses = '*'` en `postgresql.conf` |
| `permission denied for schema public` | PostgreSQL 15+ revocó los permisos del esquema `public` por defecto | En VM2 ejecutar: `sudo -u postgres psql -d libros_db -c "GRANT ALL ON SCHEMA public TO api_user;"` |
| `Error: password authentication failed` | Credenciales incorrectas | Verificar que `DB_USER`, `DB_PASSWORD` y `DB_NAME` coincidan con lo configurado en PostgreSQL |
| `Error: pg_hba.conf rejects connection` | La IP de VM1 no está en `pg_hba.conf` de VM2 | Agregar la entrada correcta en `pg_hba.conf` y reiniciar PostgreSQL |
| El contenedor en VM1 aparece como `Exited` | Error al conectarse a la BD al arrancar | Revisar `docker logs mi-api-container` para el error exacto |
| POST devuelve 500 | El modelo no se sincronizó con la BD | Verificar que `sequelize.sync()` completó sin error en los logs |
| `relation "libros" does not exist` | Sequelize no creó la tabla | El `sync()` falló silenciosamente; verificar los logs de inicio del contenedor |

---

*Computación en la nube · API REST con PostgreSQL y Sequelize*
