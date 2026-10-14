# ShipNow — Pre-entrega Módulo 1 Backend 3

API de usuarios y productos hecha con Node.js, Express y MongoDB. El objetivo de esta práctica es separar las responsabilidades en Controller, Service y Repository, y validar la configuración antes de iniciar el servidor.

## Cómo ejecutar

Necesitás Node.js 20.19 o superior y MongoDB local o una conexión de Atlas.

1. Abrí una terminal en la carpeta del proyecto.
2. Instalá las dependencias con `npm install`.
3. Copiá `.env.example` a `.env`. 
4. Completá las tres variables. Para trabajar con una base local nueva:

```dotenv
PORT=
MONGODB_URI=
NODE_ENV=
```

5. Asegurate de que MongoDB esté iniciado. Si usás Atlas, colocá tu URI y habilitá la conexión desde tu IP.
6. Ejecutá `npm run dev` para desarrollo o `npm start`.
7. Consultá `http://localhost:8080/health` si usaste `PORT=8080`; si elegiste otro puerto, ajustá la URL.

El archivo `.env.example` tiene las claves vacías para completar. `.env` y `node_modules` están excluidos por `.gitignore`.&#x20;

Si falta `PORT`, `MONGODB_URI` o `NODE_ENV`, el programa muestra qué variable falta y termina. También valida el rango del puerto, el entorno y el prefijo de la URI. Si MongoDB no está disponible, no se abre el servidor HTTP.

## Estructura

```text
src/
  config/          Lectura y validación de variables de entorno
  constants/       Roles y estados con Object.freeze
  controllers/     Reciben req y responden con res
  services/        Validaciones y reglas de negocio
  repositories/    Consultas y conexión a MongoDB
  models/          Esquemas de Mongoose
  routes/          Rutas conectadas con los controllers
  middlewares/     Respuesta común para errores
  utils/           Error personalizado y hash de contraseñas
  app.js           Configuración de Express
  server.js        Conexión y arranque
```

El recorrido de una solicitud es **Router → Controller → Service → Repository → MongoDB**. Hay un archivo separado para usuarios y otro para productos en cada capa. Se usan funciones; no hay clases base, controladores genéricos ni inyección de dependencias.

### Qué hace cada capa

El **Controller** recibe el ID, el body o el filtro de la solicitud. Llama al Service y devuelve JSON con el código HTTP correspondiente. No importa modelos ni Mongoose. Cada función usa `try/catch`: si ocurre un error, el catch llama a `next(error)` para que el middleware de errores devuelva la respuesta correspondiente.

El **Service** decide si los datos son válidos. Por ejemplo, el stock debe ser un entero no negativo y el estado del producto se calcula a partir de ese stock. Cuando vale cero, el estado es `OUT_OF_STOCK`; cuando es mayor que cero, es `AVAILABLE`. Esa decisión pertenece al negocio, por eso no está en el Repository.

El **Repository** decide cómo consultar y guardar: aplica filtros, ordena los resultados, selecciona los campos que devuelve y usa los validadores del esquema al actualizar. En usuarios excluye la contraseña incluso al crear. No decide roles ni calcula estados. Esto evita que sea solo un pasamanos de `find()`.

Los modelos únicamente declaran esquemas e índices. La conexión está en `repositories/database.js` para mantener los detalles de MongoDB junto al acceso a datos. Dentro de `src/`, `process.env` solo se lee en `config/env.config.js`.

La consigna menciona dos ubicaciones para configuración y constantes. Se implementan en `config/env.config.js` y `constants/index.js`; `config/index.js` y `utils/constants.js` permiten importarlas desde las otras ubicaciones sin duplicarlas.

### Usuarios y contraseñas

Los usuarios tienen los campos `firstName`, `lastName`, `email` y `password`. El email se guarda en minúsculas y tiene un índice único. La contraseña se transforma en un hash con `scrypt`, una función incluida en Node.js, y una sal aleatoria. Esta tarea está aislada en `utils/password.js` para que el Service sea fácil de seguir.

Los roles están definidos como `USER_ROLES.USER` y `USER_ROLES.ADMIN`. El alta asigna `USER`; el body no admite cambiar el rol. Esta práctica no incorpora login ni permisos sobre el CRUD, así que está pensada para ejecutarse localmente como ejercicio.

## Endpoints

| Método | Productos            | Usuarios          | Resultado                  |
| ------ | -------------------- | ----------------- | -------------------------- |
| GET    | `/api/products`      | `/api/users`      | Listar                     |
| GET    | `/api/products/:pid` | `/api/users/:uid` | Buscar por ID              |
| POST   | `/api/products`      | `/api/users`      | Crear                      |
| PUT    | `/api/products/:pid` | `/api/users/:uid` | Actualizar campos enviados |
| DELETE | `/api/products/:pid` | `/api/users/:uid` | Eliminar                   |

Las listas devuelven todos los registros, sin paginación para mantener el alcance del módulo. Productos admite el filtro opcional `?status=AVAILABLE` o `?status=OUT_OF_STOCK`. Sin filtro se devuelven ambos estados.

PUT permite actualizar solamente los campos enviados. DELETE elimina definitivamente el registro. Los campos desconocidos, los cuerpos vacíos y los IDs inválidos generan un error 400.

## Ejemplos de solicitudes

Para POST y PUT, enviá un cuerpo JSON con el encabezado `Content-Type: application/json`. Los ejemplos usan el puerto 8080; si configurás otro `PORT`, reemplazalo en las URLs.

### Crear un producto

POST `http://localhost:8080/api/products`

```json
{
  "name": "Caja de envío",
  "description": "Caja mediana",
  "price": 1500,
  "stock": 10
}
```

Se devuelve 201 con el producto y su `_id`. Copiá ese ID para las siguientes consultas. No envíes `status`: se calcula en el Service.

PUT `http://localhost:8080/api/products/ID_DEL_PRODUCTO`

```json
{
  "stock": 0
}
```

El estado cambia a `OUT_OF_STOCK`. Podés comprobarlo con GET `/api/products?status=OUT_OF_STOCK`.

### Crear un usuario

POST `http://localhost:8080/api/users`

```json
{
  "firstName": "Ana",
  "lastName": "Pérez",
  "email": "ana@example.com",
  "password": "ejemplo123"
}
```

Se devuelve 201 y el rol `USER`, sin la contraseña. Repetir el email devuelve 409. La contraseña debe tener entre 8 y 128 caracteres.

### Respuestas

Las respuestas correctas del CRUD tienen la forma `{"status":"success","payload":...}`. Un error tiene `{"status":"error","message":"..."}`.

| Código | Uso                                            |
| ------ | ---------------------------------------------- |
| 200    | Consulta, actualización o eliminación correcta |
| 201    | Registro creado                                |
| 400    | Campos, ID o JSON inválidos                    |
| 404    | Registro o ruta inexistente                    |
| 409    | Email repetido                                 |
| 413    | Cuerpo de la solicitud demasiado grande        |
| 500    | Error interno                                  |

