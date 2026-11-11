# CRM ORM/ODM Lab

API REST de un CRM básico que combina un ORM (Sequelize + PostgreSQL) y un ODM (Mongoose + MongoDB).

## Stack

- Node.js 22, Express 5, CommonJS
- Sequelize + PostgreSQL 16 (`User`, `Company`, `Contact`)
- Mongoose + MongoDB 7 (`Activity`)
- Jest + Supertest
- GitHub Codespaces, Dev Containers, Docker Compose
- Supervisor (`npm run dev`)

## Arquitectura

```text
GitHub Codespace
│
├── app       Node.js 22  ──┬── Sequelize ──> postgres (PostgreSQL)
│                           └── Mongoose  ──> mongo    (MongoDB)
├── postgres
└── mongo
```

La aplicación se conecta por nombre de servicio (`postgres`, `mongo`). Las credenciales de desarrollo llegan como variables de entorno definidas en `.devcontainer/docker-compose.yml` (ver `.env.example`).

## Iniciar el Codespace

1. En GitHub: **Code → Codespaces → Create codespace on main**.
2. Espera a que se levanten los tres servicios (`app`, `postgres`, `mongo`). `postCreateCommand` ejecuta `npm install`.

## Instalar dependencias

```bash
npm install
```

## Seed y reset

```bash
npm run seed    # inserta datos deterministas (3 users, 4 companies, 8 contacts, 10 activities)
npm run reset   # elimina y recrea tablas/base de datos y vuelve a sembrar
```

## Iniciar la API

```bash
npm start       # node ./bin/www
npm run dev     # supervisor ./bin/www
```

Servidor en el puerto `3000` (variable `PORT`).

## Pruebas

```bash
npm test
```

Cada suite restablece PostgreSQL y MongoDB antes de ejecutarse y cierra las conexiones al terminar.

## Endpoints

| Método | Ruta | Descripción |
|---|---|---|
| GET | `/health` | Health check |
| GET | `/users` | Listar usuarios |
| GET | `/users/:id` | Obtener usuario |
| POST | `/users` | Crear usuario |
| PUT | `/users/:id` | Actualizar usuario |
| DELETE | `/users/:id` | Eliminar usuario |
| GET | `/companies` | Listar compañías (`?industry=`) |
| GET | `/companies/:id` | Obtener compañía |
| POST | `/companies` | Crear compañía |
| PUT | `/companies/:id` | Actualizar compañía |
| DELETE | `/companies/:id` | Eliminar compañía |
| GET | `/contacts` | Listar contactos |
| GET | `/contacts/:id` | Obtener contacto |
| POST | `/contacts` | Crear contacto |
| PUT | `/contacts/:id` | Actualizar contacto |
| DELETE | `/contacts/:id` | Eliminar contacto |
| GET | `/activities` | Listar actividades (`?type=`) |
| GET | `/activities/:id` | Obtener actividad |
| POST | `/activities` | Crear actividad |
| PUT | `/activities/:id` | Actualizar actividad |
| DELETE | `/activities/:id` | Eliminar actividad |

Los errores se devuelven como JSON: `{ "error": "Contact not found" }`.

## Respuestas

**1. Dos motores.**

Activity es adecuada para MongoDB porque las actividades pueden tener información diferente dependiendo de si son llamadas, correos o reuniones. Por otro lado, Company y Contact funcionan bien en PostgreSQL porque tienen una relación definida mediante llaves foráneas, lo que facilita mantener organizados sus datos.

**2. ORM vs ODM.**

Un ORM permite trabajar con tablas de una base de datos relacional mediante objetos de programación. Aqui usamos Sequelize con PostgreSQL. Un ODM permite trabajar con documentos, como hace Mongoose con MongoDB. La diferencia principal es que uno trabaja con tablas y relaciones, mientras que el otro utiliza documentos más
flexibles.

**3. Configuración por variables de entorno.**

Las variables se encuentran en `.devcontainer/docker-compose.yml`. Es mejor mantenerlas separadas del código JavaScript para evitar exponer credenciales y facilitar los cambios de configuración. PostgreSQL utiliza `DB_HOST=postgres` y MongoDB utiliza `MONGODB_URI=mongodb://mongo:27017/crm`. No se usa localhost porque las bases de datos están en contenedores diferentes y se comunican mediante los nombres de sus servicios.

**4. Asociaciones.**

En `models/sequelize/index.js` se define una relación de uno a muchos entre Company y Contact. La llave foránea es `companyId` y se encuentra en la tabla de contactos. El alias `as: 'contacts'` permite identificar la asociación cuando queremos recuperar los contactos pertenecientes a una compañía.

**5. Eager loading.**

Si consultamos primero la compañía y después sus contactos, necesitamos realizar dos consultas por separado. Con `include`, Sequelize puede recuperar la compañía junto con sus contactos mediante la asociación configurada. Para este caso prefiero `include` porque simplifica el código y evita hacer una consulta adicional desde el controlador.

**6. Instancia vs consulta.**

En `controllers/contacts.js`, primero buscamos el contacto con `findByPk()` y después usamos `contact.update()`. Esto nos dice si existe y tambien nos sirve para devolver fácilmente el registro actualizado. Con `Model.update()` podemos modificar registros directamente mediante una condición `where`, lo cual es útil para actualizaciones MUY grandes, aunque normalmente devuelve la cantidad de filas afectadas en lugar de una instancia actualizada.

**7. Esquema flexible.**

En `models/mongoose/activity.js`, metadata utiliza `mongoose.Schema.Types.Mixed`. Este tipo permite almacenar diferentes propiedades dependiendo de la actividad, como duración en una llamada o asistentes en una reunión. Su desventaja es que Mongoose no valida automáticamente los tipos de cada propiedad interna.

**8. Sin ref.**

Los campos `contactId` y `userId` son números que hacen referencia a registros guardados en PostgreSQL. No se utiliza `ref` ni `populate` porque esas funciones de Mongoose trabajan con modelos y documentos de MongoDB. Esto significa que MongoDB no comprueba automáticamente si esos usuarios o contactos existen, por lo que podrían quedar referencias inválidas si se eliminan en PostgreSQL.

**9. Documento actualizado.**

Antes de corregir el Reto 08, `findByIdAndUpdate()` devolvía el documento anterior a la modificación, aunque los cambios sí se guardaban. Para solucionarlo agregué la opción `new: true`, que devuelve el documento actualizado. También agregué `runValidators: true` para aplicar las validaciones del esquema a los campos modificados.

**10. Pruebas de comportamiento.**

Una ventaja de probar el comportamiento es que podemos comprobar si la API funciona correctamente sin depender de cómo esté escrito internamente el código. Esto permite cambiar la implementación sin tener que modificar las pruebas, siempre que las respuestas y funcionalidades sigan siendo correctas.

**11. Repetibilidad.**

En `tests/setup.js`, antes de cada suite se establecen las conexiones con PostgreSQL y MongoDB y se ejecuta `reset()` para restaurar los datos iniciales. Al terminar se cierran ambas conexiones. Esto evita que los cambios de una prueba afecten a otras y permite obtener resultados consistentes cada vez que ejecutamos `npm test`.

**12. Tu experiencia.**

El reto que más se me complicó fue el 08, porque al intentar actualizar las actividades apareció el error `ReferenceError: remove is not defined`. Al revisar `controllers/activities.js`, encontré que faltaba la función `remove`, que se estaba exportando al final del archivo. La restauré y volví a ejecutar Jest hasta conseguir que las nueve suites pasaran correctamente.

## Evidencia

![npm test con las 9 suites en verde]()