# Chat: Microservicio de soporte con tickets en JavaScript y Docker

Este archivo contiene el resumen estructurado del chat sobre la creación de un microservicio de soporte con tickets usando **JavaScript**, **Node.js**, **Express**, **PostgreSQL** y **Docker**.

---

## Usuario

> Muéstrame como hacer el backend de soporte con tickets en js, solo como micro servicio

---

## Asistente

Voy a armarlo como microservicio REST independiente: Node.js + Express, base de datos simple con Prisma/SQLite para desarrollo, endpoints CRUD de tickets y una estructura lista para separar después.

---

## Usuario

> Que sea con Docker y sólo js, es una tarea, solo micro servicios

---

## Asistente

Perfecto. Lo dejo como tarea de microservicios: un único servicio de tickets, dockerizado, sin frontend, sin TypeScript y sin frameworks pesados. Usaré JavaScript puro con Express y PostgreSQL en Docker.

Sí. Para una tarea de **microservicios**, lo dejaría así: un microservicio independiente llamado `ticket-service`, hecho en **JavaScript**, con **Express**, **PostgreSQL** y **Docker**.

## 1. Estructura del proyecto

```txt
ticket-service/
│
├── src/
│   ├── server.js
│   ├── db.js
│   └── tickets.routes.js
│
├── db/
│   └── init.sql
│
├── Dockerfile
├── docker-compose.yml
├── package.json
└── .dockerignore
```

---

## 2. `package.json`

```json
{
  "name": "ticket-service",
  "version": "1.0.0",
  "description": "Microservicio de soporte con tickets",
  "main": "src/server.js",
  "scripts": {
    "start": "node src/server.js",
    "dev": "node --watch src/server.js"
  },
  "dependencies": {
    "express": "^4.18.3",
    "pg": "^8.11.3"
  }
}
```

Instalas dependencias con:

```bash
npm install
```

---

## 3. `docker-compose.yml`

Este archivo levanta:

- El microservicio `ticket-service`
- Una base de datos PostgreSQL exclusiva para este microservicio

```yml
services:
  ticket-service:
    build: .
    container_name: ticket-service
    ports:
      - "3000:3000"
    environment:
      PORT: 3000
      DB_HOST: ticket-db
      DB_PORT: 5432
      DB_USER: ticket_user
      DB_PASSWORD: ticket_password
      DB_NAME: ticket_db
    depends_on:
      - ticket-db

  ticket-db:
    image: postgres:16-alpine
    container_name: ticket-db
    environment:
      POSTGRES_USER: ticket_user
      POSTGRES_PASSWORD: ticket_password
      POSTGRES_DB: ticket_db
    ports:
      - "5433:5432"
    volumes:
      - ticket_data:/var/lib/postgresql/data
      - ./db/init.sql:/docker-entrypoint-initdb.d/init.sql

volumes:
  ticket_data:
```

---

## 4. `Dockerfile`

```dockerfile
FROM node:lts-alpine

WORKDIR /app

COPY package*.json ./

RUN npm install --omit=dev

COPY . .

EXPOSE 3000

CMD ["npm", "start"]
```

---

## 5. `.dockerignore`

```txt
node_modules
npm-debug.log
.git
.gitignore
```

---

## 6. `db/init.sql`

Aquí se crea la tabla de tickets.

```sql
CREATE TABLE IF NOT EXISTS tickets (
    id SERIAL PRIMARY KEY,
    title VARCHAR(150) NOT NULL,
    description TEXT NOT NULL,
    status VARCHAR(30) NOT NULL DEFAULT 'open',
    priority VARCHAR(30) NOT NULL DEFAULT 'medium',
    customer_email VARCHAR(150) NOT NULL,
    assigned_to VARCHAR(150),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

Estados posibles para la tarea:

```txt
open
in_progress
resolved
closed
```

Prioridades posibles:

```txt
low
medium
high
critical
```

---

## 7. `src/db.js`

```js
const { Pool } = require("pg");

const pool = new Pool({
  host: process.env.DB_HOST,
  port: process.env.DB_PORT,
  user: process.env.DB_USER,
  password: process.env.DB_PASSWORD,
  database: process.env.DB_NAME
});

module.exports = pool;
```

---

## 8. `src/tickets.routes.js`

```js
const express = require("express");
const pool = require("./db");

const router = express.Router();

const validStatuses = ["open", "in_progress", "resolved", "closed"];
const validPriorities = ["low", "medium", "high", "critical"];

// Crear ticket
router.post("/", async (req, res) => {
  try {
    const {
      title,
      description,
      priority = "medium",
      customer_email
    } = req.body;

    if (!title || !description || !customer_email) {
      return res.status(400).json({
        message: "title, description y customer_email son obligatorios"
      });
    }

    if (!validPriorities.includes(priority)) {
      return res.status(400).json({
        message: "Prioridad inválida"
      });
    }

    const result = await pool.query(
      `INSERT INTO tickets 
       (title, description, priority, customer_email)
       VALUES ($1, $2, $3, $4)
       RETURNING *`,
      [title, description, priority, customer_email]
    );

    res.status(201).json(result.rows[0]);
  } catch (error) {
    console.error(error);
    res.status(500).json({
      message: "Error al crear ticket"
    });
  }
});

// Listar tickets
router.get("/", async (req, res) => {
  try {
    const { status, priority } = req.query;

    let query = "SELECT * FROM tickets WHERE 1=1";
    const values = [];

    if (status) {
      values.push(status);
      query += ` AND status = $${values.length}`;
    }

    if (priority) {
      values.push(priority);
      query += ` AND priority = $${values.length}`;
    }

    query += " ORDER BY created_at DESC";

    const result = await pool.query(query, values);

    res.json(result.rows);
  } catch (error) {
    console.error(error);
    res.status(500).json({
      message: "Error al obtener tickets"
    });
  }
});

// Obtener ticket por ID
router.get("/:id", async (req, res) => {
  try {
    const { id } = req.params;

    const result = await pool.query(
      "SELECT * FROM tickets WHERE id = $1",
      [id]
    );

    if (result.rows.length === 0) {
      return res.status(404).json({
        message: "Ticket no encontrado"
      });
    }

    res.json(result.rows[0]);
  } catch (error) {
    console.error(error);
    res.status(500).json({
      message: "Error al obtener ticket"
    });
  }
});

// Cambiar estado del ticket
router.patch("/:id/status", async (req, res) => {
  try {
    const { id } = req.params;
    const { status } = req.body;

    if (!validStatuses.includes(status)) {
      return res.status(400).json({
        message: "Estado inválido"
      });
    }

    const result = await pool.query(
      `UPDATE tickets 
       SET status = $1, updated_at = CURRENT_TIMESTAMP
       WHERE id = $2
       RETURNING *`,
      [status, id]
    );

    if (result.rows.length === 0) {
      return res.status(404).json({
        message: "Ticket no encontrado"
      });
    }

    res.json(result.rows[0]);
  } catch (error) {
    console.error(error);
    res.status(500).json({
      message: "Error al actualizar estado"
    });
  }
});

// Asignar ticket a un agente
router.patch("/:id/assign", async (req, res) => {
  try {
    const { id } = req.params;
    const { assigned_to } = req.body;

    if (!assigned_to) {
      return res.status(400).json({
        message: "assigned_to es obligatorio"
      });
    }

    const result = await pool.query(
      `UPDATE tickets 
       SET assigned_to = $1, updated_at = CURRENT_TIMESTAMP
       WHERE id = $2
       RETURNING *`,
      [assigned_to, id]
    );

    if (result.rows.length === 0) {
      return res.status(404).json({
        message: "Ticket no encontrado"
      });
    }

    res.json(result.rows[0]);
  } catch (error) {
    console.error(error);
    res.status(500).json({
      message: "Error al asignar ticket"
    });
  }
});

// Eliminar ticket
router.delete("/:id", async (req, res) => {
  try {
    const { id } = req.params;

    const result = await pool.query(
      "DELETE FROM tickets WHERE id = $1 RETURNING *",
      [id]
    );

    if (result.rows.length === 0) {
      return res.status(404).json({
        message: "Ticket no encontrado"
      });
    }

    res.json({
      message: "Ticket eliminado correctamente",
      ticket: result.rows[0]
    });
  } catch (error) {
    console.error(error);
    res.status(500).json({
      message: "Error al eliminar ticket"
    });
  }
});

module.exports = router;
```

---

## 9. `src/server.js`

```js
const express = require("express");
const ticketsRoutes = require("./tickets.routes");

const app = express();

app.use(express.json());

app.get("/", (req, res) => {
  res.json({
    service: "ticket-service",
    status: "running"
  });
});

app.get("/health", (req, res) => {
  res.json({
    status: "ok"
  });
});

app.use("/api/tickets", ticketsRoutes);

const PORT = process.env.PORT || 3000;

app.listen(PORT, () => {
  console.log(`Ticket service running on port ${PORT}`);
});
```

---

## 10. Levantar el microservicio

Desde la carpeta `ticket-service`:

```bash
docker compose up --build
```

Luego pruebas:

```txt
http://localhost:3000
```

Debería responder:

```json
{
  "service": "ticket-service",
  "status": "running"
}
```

---

## 11. Probar con Postman o Thunder Client

### Crear ticket

Método:

```txt
POST
```

URL:

```txt
http://localhost:3000/api/tickets
```

Body JSON:

```json
{
  "title": "No puedo iniciar sesión",
  "description": "El usuario indica que no puede acceder al sistema.",
  "priority": "high",
  "customer_email": "cliente@correo.com"
}
```

Respuesta esperada:

```json
{
  "id": 1,
  "title": "No puedo iniciar sesión",
  "description": "El usuario indica que no puede acceder al sistema.",
  "status": "open",
  "priority": "high",
  "customer_email": "cliente@correo.com",
  "assigned_to": null,
  "created_at": "...",
  "updated_at": "..."
}
```

---

### Listar tickets

```txt
GET http://localhost:3000/api/tickets
```

---

### Buscar ticket por ID

```txt
GET http://localhost:3000/api/tickets/1
```

---

### Cambiar estado

```txt
PATCH http://localhost:3000/api/tickets/1/status
```

Body:

```json
{
  "status": "in_progress"
}
```

---

### Asignar ticket

```txt
PATCH http://localhost:3000/api/tickets/1/assign
```

Body:

```json
{
  "assigned_to": "soporte1@empresa.com"
}
```

---

### Eliminar ticket

```txt
DELETE http://localhost:3000/api/tickets/1
```

---

## 12. Cómo explicarlo en la tarea

> Este proyecto corresponde a un microservicio de soporte encargado exclusivamente de la gestión de tickets. El servicio permite crear, listar, consultar, actualizar estado, asignar y eliminar tickets. Está desarrollado en JavaScript con Node.js y Express, utiliza PostgreSQL como base de datos y se ejecuta mediante Docker.
>
> La lógica del sistema se encuentra aislada en un único servicio independiente, por lo que podría comunicarse en el futuro con otros microservicios, como usuarios, notificaciones o autenticación.

---

## 13. Endpoints del microservicio

| Método | Ruta | Descripción |
|---|---|---|
| GET | `/health` | Verifica si el servicio está activo |
| POST | `/api/tickets` | Crea un ticket |
| GET | `/api/tickets` | Lista todos los tickets |
| GET | `/api/tickets/:id` | Obtiene un ticket específico |
| PATCH | `/api/tickets/:id/status` | Cambia el estado |
| PATCH | `/api/tickets/:id/assign` | Asigna el ticket |
| DELETE | `/api/tickets/:id` | Elimina un ticket |

Este queda como un microservicio limpio, dockerizado y hecho solamente con JavaScript.

---

## Usuario

> Muéstrame solo la función y como se vería en html y css

---

## Asistente

Claro. Como demo simple para consumir el microservicio de tickets, sería así:

## Función JS

```js
async function crearTicket(event) {
  event.preventDefault();

  const ticket = {
    title: document.getElementById("title").value,
    description: document.getElementById("description").value,
    priority: document.getElementById("priority").value,
    customer_email: document.getElementById("customer_email").value
  };

  try {
    const response = await fetch("http://localhost:3000/api/tickets", {
      method: "POST",
      headers: {
        "Content-Type": "application/json"
      },
      body: JSON.stringify(ticket)
    });

    const data = await response.json();

    if (!response.ok) {
      alert(data.message || "Error al crear ticket");
      return;
    }

    alert("Ticket creado correctamente");
    console.log(data);

  } catch (error) {
    console.error("Error:", error);
    alert("No se pudo conectar con el microservicio");
  }
}
```

---

## HTML

```html
<!DOCTYPE html>
<html lang="es">
<head>
  <meta charset="UTF-8">
  <title>Crear Ticket</title>
  <link rel="stylesheet" href="styles.css">
</head>
<body>

  <div class="ticket-container">
    <h1>Soporte</h1>
    <p>Crear nuevo ticket</p>

    <form onsubmit="crearTicket(event)">
      <label for="title">Título</label>
      <input 
        type="text" 
        id="title" 
        placeholder="Ej: No puedo iniciar sesión"
        required
      >

      <label for="description">Descripción</label>
      <textarea 
        id="description" 
        placeholder="Describe el problema"
        required
      ></textarea>

      <label for="priority">Prioridad</label>
      <select id="priority">
        <option value="low">Baja</option>
        <option value="medium" selected>Media</option>
        <option value="high">Alta</option>
        <option value="critical">Crítica</option>
      </select>

      <label for="customer_email">Correo del cliente</label>
      <input 
        type="email" 
        id="customer_email" 
        placeholder="cliente@correo.com"
        required
      >

      <button type="submit">Crear ticket</button>
    </form>
  </div>

  <script src="app.js"></script>
</body>
</html>
```

---

## CSS

```css
body {
  margin: 0;
  font-family: Arial, sans-serif;
  background: #f4f6f8;
  display: flex;
  justify-content: center;
  align-items: center;
  min-height: 100vh;
}

.ticket-container {
  background: white;
  width: 400px;
  padding: 30px;
  border-radius: 12px;
  box-shadow: 0 4px 20px rgba(0, 0, 0, 0.1);
}

.ticket-container h1 {
  margin: 0;
  color: #222;
}

.ticket-container p {
  color: #666;
  margin-bottom: 25px;
}

form {
  display: flex;
  flex-direction: column;
}

label {
  margin-bottom: 6px;
  font-weight: bold;
  color: #333;
}

input,
textarea,
select {
  margin-bottom: 18px;
  padding: 12px;
  border: 1px solid #ccc;
  border-radius: 8px;
  font-size: 15px;
}

textarea {
  min-height: 100px;
  resize: none;
}

button {
  padding: 12px;
  background: #2563eb;
  color: white;
  border: none;
  border-radius: 8px;
  font-size: 16px;
  cursor: pointer;
}

button:hover {
  background: #1d4ed8;
}
```

---

## Vista aproximada

```txt
+--------------------------------+
| Soporte                        |
| Crear nuevo ticket             |
|                                |
| Título                         |
| [ No puedo iniciar sesión    ] |
|                                |
| Descripción                    |
| [ Describe el problema       ] |
|                                |
| Prioridad                      |
| [ Media                    v ] |
|                                |
| Correo del cliente             |
| [ cliente@correo.com         ] |
|                                |
| [ Crear ticket ]               |
+--------------------------------+
```

---

## Notas técnicas

- El backend corresponde a un microservicio independiente.
- El frontend mostrado es solo una vista de ejemplo para consumir el endpoint `POST /api/tickets`.
- La base de datos queda encapsulada dentro del `docker-compose.yml`.
- El microservicio puede crecer después con otros servicios, por ejemplo:
  - `auth-service`
  - `user-service`
  - `notification-service`
  - `report-service`

