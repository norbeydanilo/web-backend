# Construcción de una API REST con Node.js y Express
## Guía práctica: desde cero hasta Docker

**Computación en la nube**

---

## ¿Qué es una API REST?

Una **API** (Application Programming Interface) es un contrato que le permite a un programa hablar con otro. Una **API REST** es una forma específica de diseñar ese contrato usando el protocolo HTTP — el mismo que usa el navegador cuando carga una página web.

La diferencia es que una API no devuelve HTML para mostrar al usuario: devuelve **datos en formato JSON** para que otra aplicación los consuma. Cuando una app móvil muestra tus pedidos, cuando un dashboard carga estadísticas en tiempo real, cuando un servicio de pago procesa una transacción — todos están consumiendo APIs REST.

---

## Los 4 métodos HTTP

Una API REST usa los métodos HTTP para indicar la **intención** de cada operación:

| Método | Intención | Analogía |
|---|---|---|
| `GET` | Obtener datos | Leer un registro |
| `POST` | Crear algo nuevo | Insertar un registro |
| `PUT` | Reemplazar algo existente | Actualizar completo |
| `DELETE` | Eliminar algo | Borrar un registro |

Juntos cubren las cuatro operaciones básicas de cualquier sistema de datos, conocidas como **CRUD** (Create, Read, Update, Delete).

---

## Tu tarea: inventa tu propio dominio

Este es el momento de ser creativo. Tu API va a gestionar datos de **algo que tú elijas**. El código de esta guía usa `libros` como ejemplo de referencia — tú deberás cambiarlo por tu propio tema.

Algunas ideas:

| Tema | Entidad | Campos posibles |
|---|---|---|
| Videojuegos | juego | id, titulo, genero, año, plataforma |
| Restaurante | plato | id, nombre, precio, categoria, disponible |
| Mascotas en adopción | mascota | id, nombre, especie, edad, adoptado |
| Películas | pelicula | id, titulo, director, duracion, calificacion |
| Inventario de tienda | producto | id, nombre, precio, stock, categoria |
| Pacientes de clínica | paciente | id, nombre, edad, diagnostico, activo |
| Rutas de senderismo | ruta | id, nombre, distancia, dificultad, departamento |

Elige un tema, define tus 5 o 6 campos y ten eso claro antes de escribir código.

---

## Parte 1 — Configurar el proyecto

### 1.1 Crear la carpeta del proyecto

Si quieres desde la terminal: 

Abre una terminal y ejecuta:

```bash
mkdir mi-api 
cd mi-api
```

### 1.2 Inicializar el proyecto Node.js

```bash
npm init -y
```

Este comando crea el archivo `package.json` con valores por defecto. La bandera `-y` responde "sí" a todas las preguntas automáticamente. Puedes abrir el archivo generado para ver su contenido.

> **¿Para qué sirve `package.json`?**
> Es el archivo de configuración del proyecto. Registra el nombre, la versión, el punto de entrada (`main`) y todas las dependencias que instales. Es lo primero que cualquier desarrollador revisa cuando llega a un proyecto Node.js.

### 1.3 Instalar Express

**Express** es el framework web más popular de Node.js. Se encarga de recibir las peticiones HTTP, enrutarlas al código correcto y enviar las respuestas — sin que tengas que manejar eso manualmente.

```bash
npm install express
```

Verás que se crea la carpeta `node_modules/` y que `package.json` ahora lista Express como dependencia.

### 1.4 Crear el archivo `.gitignore`

Si usas Git, no querrás subir la carpeta `node_modules` (puede pesar cientos de MB). Crea el archivo `.gitignore`:

```
node_modules/
```

### 1.5 Estructura del proyecto

Al terminar este paso, tu carpeta debe verse así:

```
mi-api/
├── node_modules/       ← generado por npm install
├── .gitignore
├── package.json
└── index.js            ← lo crearás en el siguiente paso
```

---

## Parte 2 — Crear el servidor y los endpoints

Crea el archivo `index.js` en la raíz del proyecto.

### 2.1 El servidor básico

```javascript
// index.js
const express = require('express');
const app = express();

// Middleware: le dice a Express que lea el cuerpo de las peticiones como JSON
app.use(express.json());

const PORT = process.env.PORT || 3000;

app.listen(PORT, () => {
  console.log(`API corriendo en http://localhost:${PORT}`);
});
```

> **¿Qué es un middleware?**
> Es una función que se ejecuta antes de que la petición llegue a tu endpoint. `express.json()` lee el cuerpo de las peticiones POST y PUT y lo convierte en un objeto JavaScript accesible como `req.body`. Sin esto, `req.body` siempre sería `undefined`.

Prueba que el servidor arranca:

```bash
node index.js
```

Debes ver: `API corriendo en http://localhost:3000`

Presiona `Ctrl+C` para detenerlo.

---

### 2.2 Los datos quemados

Justo debajo de `app.use(express.json())` y antes de `app.listen`, agrega tu colección de datos. Este array simula lo que más adelante sería una base de datos real.

El siguiente es el ejemplo de referencia con libros. **Tú deberás reemplazarlo con tu propio dominio:**

```javascript
// ─── DATOS DE EJEMPLO ────────────────────────────────────────────
// Reemplaza esto con tu propio tema y campos
let libros = [
  { id: 1, titulo: 'Cien años de soledad', autor: 'Gabriel García Márquez', año: 1967, disponible: true },
  { id: 2, titulo: 'El amor en los tiempos del cólera', autor: 'Gabriel García Márquez', año: 1985, disponible: false },
  { id: 3, titulo: 'La hojarasca', autor: 'Gabriel García Márquez', año: 1955, disponible: true },
];

// Contador para asignar IDs a nuevos registros
let nextId = 4;
// ─────────────────────────────────────────────────────────────────
```

---

### 2.3 GET — Obtener todos los registros

```javascript
// GET /libros
// Devuelve la lista completa
app.get('/libros', (req, res) => {
  res.json(libros);
});
```

---

### 2.4 GET — Obtener un registro por ID

```javascript
// GET /libros/:id
// :id es un parámetro dinámico en la URL
app.get('/libros/:id', (req, res) => {
  // req.params.id llega como string, parseInt lo convierte a número
  const id = parseInt(req.params.id);
  const libro = libros.find(l => l.id === id);

  if (!libro) {
    // 404 = Not Found
    return res.status(404).json({ error: `No se encontró el libro con id ${id}` });
  }

  res.json(libro);
});
```

---

### 2.5 POST — Crear un nuevo registro

```javascript
// POST /libros
// El cuerpo de la petición (req.body) debe ser un JSON con los campos del nuevo registro
app.post('/libros', (req, res) => {
  const { titulo, autor, año, disponible } = req.body;

  // Validación mínima: los campos obligatorios deben existir
  if (!titulo || !autor) {
    return res.status(400).json({ error: 'Los campos titulo y autor son obligatorios' });
  }

  const nuevoLibro = {
    id: nextId++,
    titulo,
    autor,
    año: año || null,
    disponible: disponible !== undefined ? disponible : true,
  };

  libros.push(nuevoLibro);

  // 201 = Created
  res.status(201).json(nuevoLibro);
});
```

> **Importante:** los datos quemados en este array viven en la memoria del proceso. Si reinicias el servidor, los cambios que hagas con POST, PUT y DELETE se pierden. Eso es intencional en esta guía — el objetivo es entender los métodos HTTP, no la persistencia de datos.

---

### 2.6 PUT — Actualizar un registro existente

```javascript
// PUT /libros/:id
// Reemplaza completamente el registro con los datos enviados en el body
app.put('/libros/:id', (req, res) => {
  const id = parseInt(req.params.id);
  const index = libros.findIndex(l => l.id === id);

  if (index === -1) {
    return res.status(404).json({ error: `No se encontró el libro con id ${id}` });
  }

  const { titulo, autor, año, disponible } = req.body;

  if (!titulo || !autor) {
    return res.status(400).json({ error: 'Los campos titulo y autor son obligatorios' });
  }

  // Conserva el id original, reemplaza el resto
  libros[index] = { id, titulo, autor, año, disponible };

  res.json(libros[index]);
});
```

---

### 2.7 DELETE — Eliminar un registro

```javascript
// DELETE /libros/:id
app.delete('/libros/:id', (req, res) => {
  const id = parseInt(req.params.id);
  const index = libros.findIndex(l => l.id === id);

  if (index === -1) {
    return res.status(404).json({ error: `No se encontró el libro con id ${id}` });
  }

  const eliminado = libros.splice(index, 1)[0];

  // 200 con el objeto eliminado para confirmar qué se borró
  res.json({ mensaje: 'Registro eliminado', eliminado });
});
```

---

### 2.8 Archivo `index.js` completo

Para verificar que todo está en orden, el archivo completo queda así (con el ejemplo de libros — recuerda que tú usarás tu propio dominio):

```javascript
const express = require('express');
const app = express();

app.use(express.json());

// ─── DATOS ───────────────────────────────────────────────────────
let libros = [
  { id: 1, titulo: 'Cien años de soledad', autor: 'Gabriel García Márquez', año: 1967, disponible: true },
  { id: 2, titulo: 'El amor en los tiempos del cólera', autor: 'Gabriel García Márquez', año: 1985, disponible: false },
  { id: 3, titulo: 'La hojarasca', autor: 'Gabriel García Márquez', año: 1955, disponible: true },
];
let nextId = 4;
// ─────────────────────────────────────────────────────────────────

// GET todos
app.get('/libros', (req, res) => {
  res.json(libros);
});

// GET por id
app.get('/libros/:id', (req, res) => {
  const id = parseInt(req.params.id);
  const libro = libros.find(l => l.id === id);
  if (!libro) return res.status(404).json({ error: `No se encontró el libro con id ${id}` });
  res.json(libro);
});

// POST crear
app.post('/libros', (req, res) => {
  const { titulo, autor, año, disponible } = req.body;
  if (!titulo || !autor) return res.status(400).json({ error: 'titulo y autor son obligatorios' });
  const nuevo = { id: nextId++, titulo, autor, año: año || null, disponible: disponible !== undefined ? disponible : true };
  libros.push(nuevo);
  res.status(201).json(nuevo);
});

// PUT actualizar
app.put('/libros/:id', (req, res) => {
  const id = parseInt(req.params.id);
  const index = libros.findIndex(l => l.id === id);
  if (index === -1) return res.status(404).json({ error: `No se encontró el libro con id ${id}` });
  const { titulo, autor, año, disponible } = req.body;
  if (!titulo || !autor) return res.status(400).json({ error: 'titulo y autor son obligatorios' });
  libros[index] = { id, titulo, autor, año, disponible };
  res.json(libros[index]);
});

// DELETE eliminar
app.delete('/libros/:id', (req, res) => {
  const id = parseInt(req.params.id);
  const index = libros.findIndex(l => l.id === id);
  if (index === -1) return res.status(404).json({ error: `No se encontró el libro con id ${id}` });
  const eliminado = libros.splice(index, 1)[0];
  res.json({ mensaje: 'Registro eliminado', eliminado });
});

const PORT = process.env.PORT || 3000;
app.listen(PORT, () => console.log(`API corriendo en http://localhost:${PORT}`));
```

---

## Parte 3 — Probar con Postman en local

### 3.1 Instalar Postman

Descarga Postman desde [postman.com/downloads](https://www.postman.com/downloads/). Es gratuito. No es necesario crear una cuenta para usarlo de forma local.

### 3.2 Iniciar el servidor

```bash
node index.js
```

Deja esta terminal abierta mientras pruebas.

### 3.3 Crear una colección en Postman

1. Abre Postman
2. Haz clic en **New** → **Collection**
3. Nómbrala `Mi API - [tu tema]`

Dentro de esta colección crearás una request por cada operación.

---

### 3.4 Probar GET — Todos los registros

1. Haz clic en **New Request** dentro de tu colección
2. Método: **GET**
3. URL: `http://localhost:3000/libros`
4. Haz clic en **Send**

Deberías ver el array con los 3 registros iniciales en el panel de respuesta. El código de estado debe ser **200 OK**.

---

### 3.5 Probar GET — Un registro por ID

1. Método: **GET**
2. URL: `http://localhost:3000/libros/1`
3. **Send**

Respuesta esperada: el objeto con `id: 1`.

Prueba también con un ID que no existe, como `http://localhost:3000/libros/99`. Deberías recibir **404 Not Found** con el mensaje de error.

---

### 3.6 Probar POST — Crear un registro

1. Método: **POST**
2. URL: `http://localhost:3000/libros`
3. Ve a la pestaña **Body** → selecciona **raw** → en el menú desplegable a la derecha elige **JSON**
4. Escribe el cuerpo de la petición:

```json
{
  "titulo": "El otoño del patriarca",
  "autor": "Gabriel García Márquez",
  "año": 1975,
  "disponible": true
}
```

5. **Send**

Respuesta esperada: el objeto recién creado con `id: 4` y código **201 Created**.

Ejecuta de nuevo el GET de todos los registros y verifica que ya aparecen 4.

---

### 3.7 Probar PUT — Actualizar un registro

1. Método: **PUT**
2. URL: `http://localhost:3000/libros/2`
3. **Body** → **raw** → **JSON**:

```json
{
  "titulo": "El amor en los tiempos del cólera",
  "autor": "Gabriel García Márquez",
  "año": 1985,
  "disponible": true
}
```

4. **Send**

Respuesta esperada: el objeto actualizado. En este caso cambiamos `disponible` de `false` a `true`.

---

### 3.8 Probar DELETE — Eliminar un registro

1. Método: **DELETE**
2. URL: `http://localhost:3000/libros/1`
3. **Send** (no necesita body)

Respuesta esperada: el objeto eliminado confirmado con el mensaje.

Ejecuta el GET de todos y verifica que ya solo quedan 3 (o los que hayas creado con POST).

---

### 3.9 Tabla resumen de endpoints

| Método | URL | Body requerido | ¿Qué hace? | Código esperado |
|---|---|---|---|---|
| GET | `/libros` | No | Devuelve todos | 200 |
| GET | `/libros/:id` | No | Devuelve uno por ID | 200 / 404 |
| POST | `/libros` | Sí (JSON) | Crea uno nuevo | 201 |
| PUT | `/libros/:id` | Sí (JSON) | Actualiza completo | 200 / 404 |
| DELETE | `/libros/:id` | No | Elimina por ID | 200 / 404 |

---

## Parte 4 — Contenerizar la API con Docker

### 4.1 Crear el `Dockerfile`

```dockerfile
FROM node:20-alpine

WORKDIR /app

COPY package.json ./
RUN npm install --omit=dev

COPY index.js ./

EXPOSE 3000

CMD ["node", "index.js"]
```

### 4.2 Crear `.dockerignore`

Para que Docker no copie carpetas innecesarias al construir la imagen:

```
node_modules
```

### 4.3 Estructura final del proyecto

```
mi-api/
├── node_modules/
├── .dockerignore
├── .gitignore
├── Dockerfile
├── index.js
└── package.json
```

### 4.4 Construir la imagen

```bash
docker build -t mi-api:1.0 .
```

Verifica que se creó:

```bash
docker images
```

### 4.5 Ejecutar el contenedor

```bash
docker run -d -p 3000:3000 --name mi-api-container mi-api:1.0
```

Verifica que está corriendo:

```bash
docker ps
```

Verifica los logs:

```bash
docker logs mi-api-container
```

Debes ver: `API corriendo en http://localhost:3000`

---

## Parte 5 — Probar la API en Docker con Postman

**No necesitas cambiar nada en Postman.** La URL sigue siendo `http://localhost:3000` porque mapeaste el puerto 3000 del contenedor al 3000 de tu máquina con `-p 3000:3000`.

Repite exactamente los mismos pasos de la Parte 3:

1. GET todos → `http://localhost:3000/libros`
2. GET por ID → `http://localhost:3000/libros/1`
3. POST crear → `http://localhost:3000/libros` con body JSON
4. PUT actualizar → `http://localhost:3000/libros/2` con body JSON
5. DELETE eliminar → `http://localhost:3000/libros/1`

> **Diferencia importante:** si haces cambios en `index.js` mientras el contenedor está corriendo, esos cambios no se reflejan automáticamente. Debes reconstruir la imagen y reiniciar el contenedor:
>
> ```bash
> docker stop mi-api-container
> docker rm mi-api-container
> docker build -t mi-api:1.0 .
> docker run -d -p 3000:3000 --name mi-api-container mi-api:1.0
> ```

---

## Parte 6 — Agregar un script de inicio a `package.json`

Para no tener que escribir `node index.js` cada vez, agrega un script en `package.json`:

```json
{
  "scripts": {
    "start": "node index.js",
    "dev": "node --watch index.js"
  }
}
```

Ahora puedes iniciar el servidor con:

```bash
# En producción / Docker
npm start

# En desarrollo (recarga automática al guardar)
npm run dev
```

> `node --watch` es una función nativa de Node.js 18+ que reinicia el proceso automáticamente cuando detecta cambios en los archivos. Es útil en desarrollo para no tener que detener y reiniciar el servidor manualmente después de cada cambio.

---

## Solución de problemas frecuentes

| Problema | Causa probable | Solución |
|---|---|---|
| `Cannot find module 'express'` | `npm install` no se ejecutó | Ejecuta `npm install` en la carpeta del proyecto |
| Puerto 3000 ya está en uso | Otra app (o el contenedor anterior) usa el puerto | Cambia el puerto: `node index.js` con `PORT=3001 node index.js` |
| Postman devuelve `Could not send request` | El servidor no está corriendo | Ejecuta `node index.js` o verifica que el contenedor esté activo |
| `req.body` es `undefined` | Falta el middleware `express.json()` | Asegúrate de tener `app.use(express.json())` antes de las rutas |
| POST devuelve 400 aunque enviaste datos | El `Content-Type` no es `application/json` | En Postman, Body → raw → JSON (el menú a la derecha) |
| El contenedor no refleja los cambios | La imagen no fue reconstruida | `docker stop` → `docker rm` → `docker build` → `docker run` |

---

## Códigos de estado HTTP más comunes

| Código | Nombre | Cuándo usarlo |
|---|---|---|
| 200 | OK | Operación exitosa (GET, PUT, DELETE) |
| 201 | Created | Recurso creado exitosamente (POST) |
| 400 | Bad Request | El cliente envió datos inválidos o incompletos |
| 404 | Not Found | El recurso solicitado no existe |
| 500 | Internal Server Error | Error inesperado en el servidor |

---

## Lista de verificación para el entregable

Antes de entregar, verifica que tu API cumple con todo:

- [ ] El proyecto tiene su propio dominio (no libros ni el ejemplo de referencia)
- [ ] `package.json` fue generado con `npm init`
- [ ] Express está instalado como dependencia (`npm install express`)
- [ ] Los datos quemados tienen al menos 3 registros con mínimo 4 campos cada uno
- [ ] `GET /[entidad]` devuelve todos los registros — código 200
- [ ] `GET /[entidad]/:id` devuelve uno — código 200 o 404
- [ ] `POST /[entidad]` crea un registro — código 201
- [ ] `PUT /[entidad]/:id` actualiza un registro — código 200 o 404
- [ ] `DELETE /[entidad]/:id` elimina un registro — código 200 o 404
- [ ] Todos los endpoints responden a peticiones desde Postman en local
- [ ] El `Dockerfile` está creado y la imagen construye sin errores
- [ ] El contenedor corre y los mismos endpoints responden en Postman sobre Docker

---

*Computación en la nube · API REST con Node.js y Docker*
