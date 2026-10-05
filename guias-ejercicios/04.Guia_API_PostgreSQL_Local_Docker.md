# API REST con PostgreSQL y Sequelize
## Parte 1 y 2: Migración de la API y prueba local con Docker Compose

**Computación en la nube**

---

## ¿Qué vamos a hacer?

Hasta ahora la API de libros guardaba los datos en un array en memoria. Cuando se reiniciaba el servidor, todo volvía al estado inicial. En esta guía vamos a conectar la API a una base de datos PostgreSQL real para que los datos persistan.

Cubrimos dos etapas:

**Parte 1:** migrar el código de la API a una arquitectura por capas usando Sequelize como ORM y PostgreSQL como base de datos.

**Parte 2:** levantar la API y PostgreSQL juntos en local usando Docker Compose, con un solo comando.

En la siguiente guía veremos cómo desplegar esta misma solución en dos VMs de Azure.

## ¿Qué es Sequelize?

Sequelize es un ORM (Object-Relational Mapper): una librería que hace de puente entre el código JavaScript y la base de datos. En lugar de escribir SQL directamente:

```sql
-- Sin ORM
SELECT * FROM libros WHERE id = 1;
INSERT INTO libros (titulo, autor) VALUES ('...', '...');
```

Se usan métodos de JavaScript:

```javascript
// Con Sequelize
await Libro.findByPk(1);
await Libro.create({ titulo: '...', autor: '...' });
```

Además, Sequelize crea la tabla automáticamente al arrancar si no existe, lo que elimina la necesidad de crear el esquema manualmente.

---

## Parte 1: Migrar la API a PostgreSQL con Sequelize

### 1.1 Nueva estructura de carpetas

La API pasa de un solo archivo `index.js` a una arquitectura por capas. Cada capa tiene una responsabilidad específica:

```
mi-api/
├── src/
│   ├── config/
│   │   └── database.js        ← conexión a PostgreSQL
│   ├── models/
│   │   └── libro.js           ← define la tabla y los campos
│   ├── controllers/
│   │   └── libroController.js ← lógica de cada operación
│   ├── routes/
│   │   └── libroRoutes.js     ← asocia URLs a controllers
│   └── app.js                 ← configura Express
├── index.js                   ← punto de entrada
├── .env                       ← variables de entorno (no se sube a Git)
├── .env.example               ← plantilla de variables (sí se sube a Git)
├── .gitignore
├── Dockerfile
└── package.json
```

**¿Por qué esta separación?**

- `config/`: si cambia la contraseña o el host de la BD, se cambia en un solo lugar.
- `models/`: si se agregan campos a la tabla, se modifica solo el modelo.
- `controllers/`: cada función hace una sola cosa. Fácil de leer, fácil de probar.
- `routes/`: el mapeo de URLs queda separado de la lógica. Express queda limpio.

En proyectos pequeños todo puede ir en un archivo, pero separarlo desde el principio es un hábito que paga dividendos cuando el proyecto crece.

### 1.2 Instalar dependencias

```bash
npm install sequelize pg pg-hstore dotenv express
```

- `sequelize`: el ORM
- `pg`: el driver de PostgreSQL para Node.js
- `pg-hstore`: requerido internamente por Sequelize para manejar tipos de datos de PostgreSQL
- `dotenv`: carga las variables del archivo `.env` a `process.env`

### 1.3 Archivo `.env`

Crea el archivo `.env` en la raíz del proyecto. Este archivo nunca se sube a Git porque contiene credenciales.

```
DB_HOST=localhost
DB_PORT=5432
DB_NAME=libros_db
DB_USER=postgres
DB_PASSWORD=postgres123
PORT=3000
```

> En local, `DB_HOST=localhost` apunta a PostgreSQL corriendo en tu máquina o en Docker Compose. En las VMs, este valor cambiará a la IP privada de VM2 sin modificar ningún otro archivo.

### 1.4 Archivo `.env.example`

Crea este archivo como plantilla para que otros desarrolladores sepan qué variables necesitan configurar, sin revelar los valores reales:

```
DB_HOST=
DB_PORT=5432
DB_NAME=
DB_USER=
DB_PASSWORD=
PORT=3000
```

### 1.5 Actualizar `.gitignore`

```
node_modules/
.env
```

### 1.6 `src/config/database.js`

```javascript
const { Sequelize } = require('sequelize');

// Sequelize lee las variables de entorno para construir la conexión.
// En local vienen del archivo .env (cargado por dotenv en index.js).
// En Docker Compose o en la VM vienen de las variables del contenedor o del sistema.
const sequelize = new Sequelize(
  process.env.DB_NAME,
  process.env.DB_USER,
  process.env.DB_PASSWORD,
  {
    host: process.env.DB_HOST,
    port: process.env.DB_PORT || 5432,
    dialect: 'postgres',
    logging: false, // cambia a true si quieres ver el SQL que genera Sequelize
  }
);

module.exports = sequelize;
```

### 1.7 `src/models/libro.js`

El modelo define la estructura de la tabla. Sequelize se encarga de crearla en PostgreSQL si no existe.

```javascript
const { DataTypes } = require('sequelize');
const sequelize = require('../config/database');

const Libro = sequelize.define('Libro', {
  // isbn es la clave primaria y la proporciona el usuario al crear el libro.
  // Sequelize NO generará un id automático porque definimos la primaryKey aquí.
  isbn: {
    type: DataTypes.STRING,
    primaryKey: true,
    allowNull: false,
  },
  titulo: {
    type: DataTypes.STRING,
    allowNull: false,
  },
  autor: {
    type: DataTypes.STRING,
    allowNull: false,
  },
  año: {
    type: DataTypes.INTEGER,
    allowNull: true,
  },
  disponible: {
    type: DataTypes.BOOLEAN,
    defaultValue: true,
  },
}, {
  tableName: 'libros',
  timestamps: true,
});

module.exports = Libro;
```

> **Nota para cada estudiante:** el identificador introducido por el usuario no siempre se llama ISBN. Cada dominio tiene el suyo: `codigo` para un producto, `matricula` para un vehículo, `placa` para un juego, etc. Cambia el nombre del campo y el tipo según corresponda. Si el identificador es numérico, usa `DataTypes.INTEGER`; si es alfanumérico (como el ISBN), usa `DataTypes.STRING`.

### 1.8 `src/controllers/libroController.js`

Cada función exportada corresponde a una operación CRUD. Las funciones son `async` porque las consultas a la base de datos son operaciones que toman tiempo — hay que esperar la respuesta antes de continuar.

```javascript
const Libro = require('../models/libro');

// GET /libros — devuelve todos los registros
const obtenerTodos = async (req, res) => {
  try {
    const libros = await Libro.findAll();
    res.json(libros);
  } catch (error) {
    res.status(500).json({ error: 'Error al obtener los libros', detalle: error.message });
  }
};

// GET /libros/:isbn — devuelve un registro por su ISBN
const obtenerUno = async (req, res) => {
  try {
    const libro = await Libro.findByPk(req.params.isbn);
    if (!libro) {
      return res.status(404).json({ error: `No se encontró el libro con ISBN ${req.params.isbn}` });
    }
    res.json(libro);
  } catch (error) {
    res.status(500).json({ error: 'Error al obtener el libro', detalle: error.message });
  }
};

// POST /libros — crea un nuevo registro con el ISBN proporcionado por el cliente
const crear = async (req, res) => {
  try {
    const { isbn, titulo, autor, año, disponible } = req.body;
    if (!isbn) {
      return res.status(400).json({ error: 'isbn es obligatorio' });
    }
    if (!titulo || !autor) {
      return res.status(400).json({ error: 'titulo y autor son obligatorios' });
    }
    // Si ya existe un libro con ese ISBN, Sequelize lanzará un error de clave duplicada
    const nuevo = await Libro.create({ isbn, titulo, autor, año, disponible });
    res.status(201).json(nuevo);
  } catch (error) {
    if (error.name === 'SequelizeUniqueConstraintError') {
      return res.status(409).json({ error: `Ya existe un libro con ISBN ${req.body.isbn}` });
    }
    res.status(500).json({ error: 'Error al crear el libro', detalle: error.message });
  }
};

// PUT /libros/:isbn — actualiza los campos de un registro existente (no el ISBN)
const actualizar = async (req, res) => {
  try {
    const libro = await Libro.findByPk(req.params.isbn);
    if (!libro) {
      return res.status(404).json({ error: `No se encontró el libro con ISBN ${req.params.isbn}` });
    }
    const { titulo, autor, año, disponible } = req.body;
    if (!titulo || !autor) {
      return res.status(400).json({ error: 'titulo y autor son obligatorios' });
    }
    // El ISBN no se actualiza: es la clave primaria y no debe cambiar
    await libro.update({ titulo, autor, año, disponible });
    res.json(libro);
  } catch (error) {
    res.status(500).json({ error: 'Error al actualizar el libro', detalle: error.message });
  }
};

// DELETE /libros/:isbn — elimina un registro por su ISBN
const eliminar = async (req, res) => {
  try {
    const libro = await Libro.findByPk(req.params.isbn);
    if (!libro) {
      return res.status(404).json({ error: `No se encontró el libro con ISBN ${req.params.isbn}` });
    }
    await libro.destroy();
    res.json({ mensaje: 'Libro eliminado', eliminado: libro });
  } catch (error) {
    res.status(500).json({ error: 'Error al eliminar el libro', detalle: error.message });
  }
};

module.exports = { obtenerTodos, obtenerUno, crear, actualizar, eliminar };
```

### 1.9 `src/routes/libroRoutes.js`

Las rutas asocian cada URL y método HTTP a su función del controller. El router no sabe nada de la lógica: solo conecta.

```javascript
const express = require('express');
const router = express.Router();
const {
  obtenerTodos,
  obtenerUno,
  crear,
  actualizar,
  eliminar,
} = require('../controllers/libroController');

router.get('/',          obtenerTodos);
router.get('/:isbn',     obtenerUno);
router.post('/',         crear);
router.put('/:isbn',     actualizar);
router.delete('/:isbn',  eliminar);

module.exports = router;
```

### 1.10 `src/app.js`

Express queda limpio: solo configura middlewares y registra las rutas.

```javascript
const express = require('express');
const app = express();

app.use(express.json());

// Rutas
const libroRoutes = require('./routes/libroRoutes');
app.use('/libros', libroRoutes);

// Para agregar más entidades en el futuro:
// const otraRoutes = require('./routes/otraRoutes');
// app.use('/otra', otraRoutes);

module.exports = app;
```

### 1.11 `index.js`

El punto de entrada hace tres cosas en orden: cargar las variables de entorno, verificar la conexión a la BD y arrancar el servidor.

```javascript
require('dotenv').config();

const app      = require('./src/app');
const sequelize = require('./src/config/database');

const PORT = process.env.PORT || 3000;

async function iniciar() {
  try {
    // 1. Verifica que la conexión a PostgreSQL funciona
    await sequelize.authenticate();
    console.log('Conexión a la base de datos establecida.');

    // 2. Sincroniza los modelos con la base de datos:
    //    - Si la tabla no existe, la crea.
    //    - Si el modelo cambió (nuevos campos), altera la tabla.
    //    - No borra datos existentes.
    await sequelize.sync({ alter: true });
    console.log('Modelos sincronizados con la base de datos.');

    // 3. Arranca el servidor HTTP
    app.listen(PORT, () => {
      console.log(`API corriendo en http://localhost:${PORT}`);
    });
  } catch (error) {
    console.error('Error al iniciar la aplicación:', error.message);
    process.exit(1); // sale con código de error para que Docker pueda reiniciar el contenedor
  }
}

iniciar();
```

### 1.12 `Dockerfile` actualizado

El Dockerfile cambia para copiar la carpeta `src/`:

```dockerfile
FROM node:20-alpine

WORKDIR /app

COPY package.json ./
RUN npm install --omit=dev

COPY index.js ./
COPY src/ ./src/

EXPOSE 3000

CMD ["node", "index.js"]
```

### 1.13 Siguiente paso

Con todos los archivos creados, el siguiente paso es levantar la API junto con PostgreSQL usando Docker Compose. No es necesario tener PostgreSQL instalado en tu equipo: Docker Compose se encarga de todo.

---

## Parte 2: Docker Compose local (API + PostgreSQL)

Con Docker Compose levantamos los dos servicios con un solo comando. La API se conecta a PostgreSQL usando el nombre del servicio (`db`) como host, igual que hicimos con nginx en la guía de Docker.

### 2.1 `docker-compose.yml`

```yaml
services:

  # Servicio 1: la API de Node.js
  api:
    build: .
    ports:
      - "3000:3000"
    environment:
      - DB_HOST=db          # nombre del servicio de base de datos
      - DB_PORT=5432
      - DB_NAME=libros_db
      - DB_USER=postgres
      - DB_PASSWORD=postgres123
      - PORT=3000
    depends_on:
      db:
        condition: service_healthy  # espera a que PostgreSQL esté listo
    restart: unless-stopped

  # Servicio 2: PostgreSQL
  db:
    image: postgres:16-alpine
    environment:
      - POSTGRES_DB=libros_db
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=postgres123
    volumes:
      - postgres_data:/var/lib/postgresql/data  # los datos persisten aunque el contenedor se reinicie
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 5s
      retries: 5
    restart: unless-stopped

volumes:
  postgres_data:  # volumen nombrado gestionado por Docker
```

> **`depends_on` con `condition: service_healthy`:** sin esto, la API intentaría conectarse a PostgreSQL antes de que esté listo para aceptar conexiones y fallaría. El `healthcheck` le pregunta a PostgreSQL cada 5 segundos si está listo. Solo cuando responde correctamente, Docker levanta la API.

> **El volumen `postgres_data`:** los datos de PostgreSQL se guardan aquí. Si se hace `docker compose down` y luego `docker compose up`, los datos siguen ahí. Para borrar los datos también: `docker compose down -v`.

### 2.2 Levantar los servicios

```bash
docker compose up -d
```

Ver los logs para confirmar que todo arrancó:

```bash
docker compose logs -f api
```

Debes ver:

```
Conexión a la base de datos establecida.
Modelos sincronizados con la base de datos.
API corriendo en http://localhost:3000
```

### 2.3 Verificar que los datos persisten

1. Crea un libro con POST en Postman
2. Ejecuta `docker compose down` para detener y eliminar los contenedores
3. Ejecuta `docker compose up -d` para volver a levantarlos
4. Haz GET a `/libros` — el libro sigue ahí

Eso es la diferencia respecto al array en memoria.

### 2.3 Ejemplo de body para Postman (POST)

Al crear un libro el body JSON debe incluir el `isbn`. Sin él, la API devuelve un error 400:

```json
{
  "isbn": "978-0-06-112008-4",
  "titulo": "Cien años de soledad",
  "autor": "Gabriel García Márquez",
  "año": 1967,
  "disponible": true
}
```

Para GET, PUT y DELETE, el ISBN va en la URL:

```
GET    http://localhost:3000/libros/978-0-06-112008-4
PUT    http://localhost:3000/libros/978-0-06-112008-4
DELETE http://localhost:3000/libros/978-0-06-112008-4
```

Si intentas crear dos libros con el mismo ISBN, la API devuelve **409 Conflict**.

### 2.4 Comandos útiles

```bash
# Ver el estado de los servicios
docker compose ps

# Entrar a la terminal del contenedor de PostgreSQL
docker compose exec db psql -U postgres -d libros_db

# Ver las tablas creadas por Sequelize
# (dentro del psql anterior)
\dt
SELECT * FROM libros;
\q

# Detener sin borrar datos
docker compose stop

# Detener y borrar contenedores (los datos persisten en el volumen)
docker compose down

# Detener, borrar contenedores Y borrar datos
docker compose down -v
```

---

---

*Computación en la nube · API REST con PostgreSQL y Sequelize: Partes 1 y 2*
