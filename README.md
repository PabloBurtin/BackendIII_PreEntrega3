# ShipNow API — Pre-entrega Módulo 3: Manejo profesional de errores

API de logística (Node.js, Express, MongoDB/Mongoose) con arquitectura por capas **Controller → Service → Repository** y un módulo de mocking que genera datos de prueba (usuarios, repartidores, pedidos y entregas) con `@faker-js/faker`.

## Requisitos

- Node.js 20.19 o superior
- MongoDB corriendo en local (o una URI de MongoDB Atlas)

## Instalación

```bash
npm install
cp .env.example .env
```

Completa el `.env` con tus valores (`PORT`, `MONGODB_URI`, `NODE_ENV`).
Si falta alguna variable, la app no arranca y avisa cuál.

## Variables de entorno

El archivo `.env.example` lista las variables necesarias, sin valores.
Cópialo a `.env` y complétalo:

| Variable | Descripción | Validación |
|---|---|---|
| `PORT` | Puerto del servidor | Entero entre 1 y 65535 |
| `NODE_ENV` | Entorno de ejecución | `development`, `production` o `test` |
| `MONGODB_URI` | URI de conexión a MongoDB | Obligatoria |

Ejemplo de valores: `PORT=8080`, `NODE_ENV=development`,
`MONGODB_URI=mongodb://localhost:27017/shipnow`.

Las variables se validan al arrancar (`src/config/env.config.js`). Si falta alguna, `PORT` no es numérico o `NODE_ENV` tiene un valor no permitido, la app lanza un error descriptivo y no inicia. Es el único archivo que lee `process.env`.

## Ejecución

```bash
npm run dev     # desarrollo (nodemon)
npm start       # producción
```

## Reglas de la API

- **Capas:** Controller → Service → Repository. Solo los repositories y los modelos conocen Mongoose.
- **Campos controlados por el sistema:** el body de las peticiones no puede definir `role` (los usuarios se crean siempre con rol `USER`) ni `status` (el estado de un producto se deriva del stock: `AVAILABLE` si hay stock, `OUT_OF_STOCK` si no). Los campos no permitidos se ignoran.
- **Contraseñas:** se guardan hasheadas con bcrypt y los endpoints de usuarios nunca las devuelven. Los datos simulados de `/api/mocks` sí incluyen la contraseña de prueba (`Mock1234`), porque no se guardan en la base.
- **Constantes:** roles, estados, prioridades, entornos, colleciones y los valores `success`/`error` de las respuestas están centralizados en `src/constants/index.js`.
- **Formato de respuesta exitosa:** `{ "status": "success", "message": "...", "payload": ... }`.
  Los endpoints de `/api/mocks` devuelven los datos tal cual, sin envoltorio.
- **Formato de respuesta de error:** único para toda la API (ver "Manejo de errores").

## Manejo de errores

Todos los errores pasan por una capa común. Los **services** detectan el problema y
lanzan un error del dominio; los **controllers** no atrapan ni responden errores
(Express 5 envía las promesas rechazadas al middleware); la **única** respuesta de
error al cliente sale del middleware global.

```
src/
├── errors/
│   ├── error.dictionary.js   Códigos de error y diccionario (status + mensaje)
│   ├── app.error.js          Clase base AppError
│   └── domain.error.js       Errores del dominio (UserNotFoundError, etc.)
└── middlewares/
    └── error.middleware.js   notFoundHandler + errorHandler
```

### Formato único de error

```json
{
  "status": "error",
  "statusCode": 404,
  "code": "USER_NOT_FOUND",
  "message": "Usuario no encontrado",
  "details": { "id": "000000000000000000000000" }
}
```

`details` es opcional y da contexto (ids, valores recibidos, valores permitidos).

### Diccionario de errores

| Código | HTTP | Mensaje |
|---|---|---|
| `USER_NOT_FOUND` | 404 | Usuario no encontrado |
| `PRODUCT_NOT_FOUND` | 404 | Producto no encontrado |
| `ORDER_NOT_FOUND` | 404 | Pedido no encontrado (reservado para futuros endpoints de pedidos) |
| `ROUTE_NOT_FOUND` | 404 | Ruta no encontrada |
| `EMAIL_REQUIRED` | 400 | El email es obligatorio |
| `EMAIL_ALREADY_EXISTS` | 409 | Ya existe un usuario con ese email |
| `INVALID_PASSWORD` | 400 | La contraseña no cumple los requisitos |
| `INVALID_STATUS` | 400 | Estado inválido |
| `INVALID_QUANTITY` | 400 | Cantidad inválida |
| `INVALID_ID` | 400 | El identificador no tiene un formato válido |
| `VALIDATION_ERROR` | 400 | Los datos enviados no son válidos |
| `INVALID_JSON` | 400 | El cuerpo de la petición no es un JSON válido |
| `MOCK_SEED_FAILED` | 500 | No se pudieron guardar los datos de prueba |
| `INTERNAL_ERROR` | 500 | Error interno del servidor |

El middleware también traduce errores de Mongoose (`CastError` → `INVALID_ID`,
`ValidationError` → `VALIDATION_ERROR`, índice duplicado → `EMAIL_ALREADY_EXISTS`)
y los de JSON mal formado. Los errores inesperados responden `INTERNAL_ERROR` sin
exponer detalles internos. Los 5xx se registran con su stack trace en consola; los
4xx, en una línea.

### Validaciones del módulo de mocks

| Caso | Respuesta |
|---|---|
| `qty` no numérica o decimal (`abc`, `2.5`) | 400 `INVALID_QUANTITY` (`reason`: debe ser entero) |
| `qty` negativa (`-5`) | 400 `INVALID_QUANTITY` (`reason`: no puede ser negativa) |
| `qty` fuera de rango (`0`, `1000`) | 400 `INVALID_QUANTITY` (mínimo 1, máximo 100) |
| `status` inexistente en `GET /api/mocks/orders?status=` | 400 `INVALID_STATUS` (`details.allowed` lista los válidos) |
| Falla de MongoDB durante `seed` | 500 `MOCK_SEED_FAILED` |

`GET /api/mocks/orders?qty=3&status=DELIVERED` genera pedidos con un estado concreto.

Si `seed` falla a mitad de la carga, se intenta eliminar lo ya insertado en esa ejecución (rollback por mejor esfuerzo) para no dejar datos huérfanos. Si el borrado también falla, se registra en consola.

## Endpoints de la API base

| Recurso | Rutas |
|---|---|
| Productos | `GET/POST /api/products`, `GET/PUT/DELETE /api/products/:id` |
| Usuarios | `GET/POST /api/users`, `GET/PUT/DELETE /api/users/:id` |
| Health | `GET /api/health` |

## Estructura del módulo de mocking

```
src/
├── mocks/                 Generadores puros (solo faker, sin base de datos)
│   ├── user.mocks.js
│   ├── order.mocks.js
│   └── delivery.mocks.js
├── services/mock.service.js         Valida qty, arma el dataset y las relaciones
├── repositories/mock.repository.js  Único que inserta en MongoDB
├── controllers/mock.controller.js
├── routes/mock.routes.js            Montado en /api/mocks
└── constants/index.js               Roles, estados, prioridades y MOCKING_PARAMETERS
```

## Endpoints de mocking

Parámetro `qty` (opcional): entero entre 1 y 100. Por defecto, 10.
Si es inválido responde `400` con `INVALID_QUANTITY` (ver "Manejo de errores").

### Datos simulados (NO se guardan en la base)

| Método | Ruta | Devuelve |
|---|---|---|
| GET | `/api/mocks/users?qty=2` | Usuarios con rol `USER` o `COURIER` |
| GET | `/api/mocks/couriers?qty=2` | Repartidores (rol `COURIER`) |
| GET | `/api/mocks/orders?qty=3` | Pedidos con estados y prioridades válidos. `status`(opcional) fuerza el estado de todos los pedidos |
| GET | `/api/mocks/deliveries?qty=5` | Entregas asociadas a pedidos y repartidores |

### Carga de datos de prueba (SÍ se guardan en MongoDB)

| Método | Ruta | Qué hace |
|---|---|---|
| POST | `/api/mocks/seed/users?qty=10` | Inserta `qty` usuarios |
| POST | `/api/mocks/seed?qty=10` | Inserta el conjunto completo: clientes, repartidores, pedidos y entregas |

Respuestas de ejemplo:

```json
{ "insertados": 10, "coleccion": "usuarios" }
```

```json
{ "insertados": { "usuarios": 10, "repartidores": 4, "pedidos": 10, "entregas": 7 } }
```

## Datos que se generan y sus relaciones

- **Usuarios:** `name`, `email` (único, dominio `test.com`), `password`, `role`.
  Roles tomados de `USER_ROLES`.
- **Repartidores:** usuarios con rol `COURIER`. En `seed` y en los `GET` de pedidos y entregas se generan `ceil(qty / 3)` (mínimo 1). En
  `GET /api/mocks/couriers?qty=N` se devuelven exactamente `N`.
- **Pedidos:** `customer` (referencia a un usuario con rol `USER`), `items`,
  `deliveryAddress`, `total` (suma de los items), `status` (`ORDER_STATUS`),
  `priority` (`ORDER_PRIORITY`).
- **Entregas:** `order` (referencia a un pedido), `courier`, `status`
  (`DELIVERY_STATUS`), `assignedAt`, `deliveredAt`.

Reglas de coherencia entre pedido y entrega:

| Estado del pedido | Entrega |
|---|---|
| `CREATED` | `PENDING`, sin repartidor |
| `ASSIGNED` / `PICKED_UP` | `ASSIGNED`, con repartidor (`COURIER`) |
| `IN_TRANSIT` | `IN_TRANSIT`, con repartidor |
| `DELIVERED` | `DELIVERED`, con repartidor y `deliveredAt` posterior a `assignedAt` |
| `CANCELLED` | No tiene entrega |

Cada pedido con entrega guarda en `delivery` el `_id` de esa entrega.

Todos los usuarios mock tienen la contraseña `Mock1234` (definida en
`MOCKING_PARAMETERS`). En la base se guarda **hasheada con bcrypt**.

## Cómo probar

```bash
curl "localhost:8080/api/mocks/users?qty=2"
curl "localhost:8080/api/mocks/orders?qty=3"
curl "localhost:8080/api/mocks/deliveries?qty=5"
curl -i "localhost:8080/api/mocks/users?qty=abc"     # 400
curl -X POST "localhost:8080/api/mocks/seed/users?qty=10"
curl -X POST "localhost:8080/api/mocks/seed?qty=10"
curl -i "localhost:8080/api/mocks/users?qty=-5"                 # 400 INVALID_QUANTITY
curl -i "localhost:8080/api/mocks/orders?qty=3&status=FOO"      # 400 INVALID_STATUS
curl -i localhost:8080/api/users/000000000000000000000000       # 404 USER_NOT_FOUND
curl -i localhost:8080/api/users/abc                            # 400 INVALID_ID
curl -i localhost:8080/api/nada                                 # 404 ROUTE_NOT_FOUND
```

Cada ejecución de `seed` agrega datos nuevos (no borra los anteriores).

## Limpiar los datos de prueba (mongosh)

```javascript
db.users.deleteMany({ email: /@test\.com$/ })
db.orders.deleteMany({})
db.deliveries.deleteMany({})
```

> `orders` y `deliveries` se vacían completas: úsalo solo en una base de pruebas.