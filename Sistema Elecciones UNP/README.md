# Sistema Elecciones UNP

## Estructura

- `backend/`: API Spring Boot 3, Java 21 y MariaDB.
- `frontend/`: portal React con Vite.

## Base de datos

1. Inicia MySQL/MariaDB desde XAMPP.
2. Abre phpMyAdmin.
3. Importa `..\Base de Datos\01_schema.sql`.
4. El script crea la base `elecciones_unp`.

El backend usa por defecto:

```text
jdbc:mariadb://localhost:3306/elecciones_unp
usuario: root
contraseña: vacía
```

Se pueden cambiar con `DB_URL`, `DB_USERNAME` y `DB_PASSWORD`.

## Ejecución local

Backend:

```powershell
cd backend
mvn spring-boot:run
```

Frontend:

```powershell
cd frontend
npm install
npm run dev
```

La API de comprobación queda en `http://localhost:8080/api/health`.

Después de iniciar el frontend, el portal queda disponible en
`http://localhost:5173`. Tras iniciar sesión, el dashboard permite consultar
procesos electorales y docentes. Los roles `ADMIN` y `CEUNP` también ven el
formulario para crear procesos; las rutas siguen protegidas en el backend,
por lo que ocultar una opción en React no reemplaza la autorización del API.

El menú **Candidaturas** permite seleccionar un proceso, un cargo y una lista;
los usuarios administrativos pueden permitir categorías, crear listas y
registrar candidatos. El frontend consume las rutas de candidaturas existentes
y muestra los candidatos con su estado de validación.

El menú **Tachas** permite presentar una tacha indicando el docente denunciante,
el proceso, cargo, lista, candidato y motivo. Los roles `ADMIN` y `CEUNP` pueden
filtrar tachas y resolverlas como `FUNDADA` o `INFUNDADA`; una resolución
fundada excluye al candidato desde el backend.

## Autenticación inicial

El backend usa JWT sin estado. El endpoint público de inicio de sesión es:

```text
POST /api/auth/login
Content-Type: application/json

{
  "username": "usuario",
  "password": "contraseña"
}
```

Los demás endpoints requieren:

```text
Authorization: Bearer <token>
```

Configura un secreto propio antes de usar el sistema fuera de desarrollo:

```powershell
$env:JWT_SECRET = "un-secreto-largo-de-al-menos-32-bytes"
```

El esquema solo almacena hashes BCrypt en `usuario.password_hash`; no se deben guardar contraseñas en el código ni en el script SQL. El apartado **Usuarios**, visible para `ADMIN`, permite crear, listar, activar, desactivar y eliminar cuentas. Los roles `DOCENTE`, `PERSONERO` y `MIEMBRO_MESA` requieren asociar un docente. El sistema impide desactivar o eliminar al último administrador activo.

La API de usuarios es:

```text
GET    /api/auth/users
POST   /api/auth/users
PATCH  /api/auth/users/{id}/status
DELETE /api/auth/users/{id}
PATCH  /api/auth/password
```

Las rutas de administración de usuarios requieren un JWT con rol `ADMIN`.
La respuesta nunca incluye la contraseña ni su hash. La eliminación es física y no debe usarse para
cuentas que deban conservarse por auditoría; para esos casos se recomienda
desactivar la cuenta.

Cada usuario autenticado puede cambiar su contraseña enviando `passwordActual`
y `passwordNueva` a `/api/auth/password`. La contraseña nueva debe tener al
menos ocho caracteres.

## API electoral inicial

Con un token válido se pueden consultar:

```text
GET /api/docentes
GET /api/procesos
GET /api/procesos/{idProceso}/cargos
```

Las operaciones de administración requieren `ADMIN` o `CEUNP`:

```text
POST /api/docentes
POST /api/procesos
POST /api/procesos/{idProceso}/cargos
```

La gestión de candidaturas utiliza estas rutas:

```text
GET  /api/cargos/{idCargo}/listas
POST /api/cargos/{idCargo}/listas
POST /api/cargos/{idCargo}/categorias-permitidas
GET  /api/listas/{idLista}/candidatos
POST /api/listas/{idLista}/candidatos
```

Antes de registrar candidatos se deben configurar las categorías permitidas
para el cargo. El sistema solo acepta docentes activos cuya categoría esté
permitida y evita repetir al mismo docente dentro de una lista.

## Tachas

Las tachas se gestionan mediante:

```text
POST /api/tachas
GET  /api/tachas?estado=PENDIENTE
POST /api/tachas/{idTacha}/resolver
```

La presentación requiere un docente denunciante y un motivo. La resolución
solo admite `FUNDADA` o `INFUNDADA`, requiere `ADMIN` o `CEUNP`, y no permite
resolver dos veces la misma tacha. Si una tacha es fundada, el candidato pasa
automáticamente a estado `EXCLUIDO` dentro de la misma transacción.

El esquema actual registra el estado y la fecha de resolución. Todavía no
guarda el identificador del usuario que resolvió la tacha; ese dato debe
agregarse mediante una migración antes de habilitar auditoría jurídica
completa.

El backend valida, entre otras reglas:

- DNI de exactamente ocho dígitos y no duplicado.
- Fecha de inicio anterior a la fecha de fin.
- Quórum entre 0 y 100.
- Segunda vuelta vinculada a un proceso padre.
- Jurisdicción obligatoria para cargos de facultad o departamento.
- Ausencia de jurisdicción específica para cargos universitarios.

## Decisión sobre procedimientos almacenados

La lógica electoral debe vivir principalmente en Spring Boot:

- reglas de negocio, permisos y casos de uso en servicios Java;
- validación y transacciones con `@Transactional`;
- entidades, índices y restricciones en MariaDB;
- migraciones versionadas para cambios de esquema.

No se recomienda poner toda la lógica en procedimientos almacenados porque dificulta las pruebas, el mantenimiento y la portabilidad. Puede usarse SQL nativo o un procedimiento puntual para operaciones críticas, pero la emisión del voto debe tener una transacción de servicio que marque el padrón, inserte el voto anónimo y genere la constancia de manera consistente.
