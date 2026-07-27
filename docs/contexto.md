# Contexto de la solución — SentinelAPI

> API backend NestJS para administración multi-tenant (clientes, usuarios, roles, permisos y módulos), con autenticación JWT (access + refresh), auditoría en bitácora, correo y almacenamiento de archivos en AWS S3.

| Campo | Valor |
|---|---|
| **Nombre paquete** | `rondinesapi` (v2.0.0) |
| **Framework** | NestJS 11 + TypeORM + MySQL |
| **Puerto** | `PORT` (default `3001`) |
| **Swagger UI** | `/docs` |
| **Producción (proxy)** | `https://springtelecom.mx/sentinelAPI` |
| **Fecha de documentación** | 2026-07-27 |

---

## 1. Propósito

SentinelAPI concentra el núcleo de administración del ecosistema Sentinel / Spring Telecom:

- Autenticación y sesión (login, perfil, refresh, logout, recuperación/cambio de contraseña).
- Catálogo jerárquico de **clientes** (padre/hijos) con documentos fiscales y logotipo en S3.
- Gestión de **usuarios** multi-cliente con foto de perfil, roles y permisos asignables.
- Catálogos de **roles**, **permisos** y **módulos**.
- **Bitácora** de operaciones (éxito/error) por módulo.
- Carga genérica de archivos a **S3**.
- Envío de correo (recuperación de acceso) vía SMTP.

---

## 2. Stack técnico

| Capa | Tecnología |
|---|---|
| Runtime | Node.js + NestJS 11 |
| ORM / BD | TypeORM + MySQL (`mysql2`) |
| Auth | Passport JWT (`@nestjs/jwt`, `passport-jwt`) |
| Validación | `class-validator` + `class-transformer` + `ValidationPipe` global |
| Documentación | Swagger (`@nestjs/swagger`) en `/docs` |
| Archivos | Multer (memoria) + AWS SDK S3 |
| Correo | Nodemailer (Zoho SMTP) |
| Config | `@nestjs/config` + validación Joi |

### Variables de entorno requeridas (nombres)

| Variable | Uso |
|---|---|
| `DB_HOST`, `DB_PORT`, `DB_USER`, `DB_PASSWORD`, `DB_DATABASE`, `DB_TZ` | Conexión MySQL |
| `JWT_SECRET`, `JWT_EXPIRES_IN`, `JWT_REFRESH_EXPIRES_IN` | Tokens access/refresh |
| `JWT_CONFIRMACION` | TTL token de recuperación de contraseña |
| `AWS_REGION`, `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_S3_BUCKET` | S3 |
| `UPLOAD_MAX_SIZE` | Límite de tamaño de archivos (bytes) |
| `E_MAIL`, `E_MAIL_PASS`, `HOST`, `SMTP` | Correo |
| `PORT` | Puerto HTTP |

> `synchronize: false` en TypeORM. El esquema se gestiona fuera de la app.

---

## 3. Arquitectura modular

```
AppModule
├── AuthModule          → login, me, refresh, logout, recuperar/cambiar acceso
├── UsuariosModule      → CRUD usuarios + foto S3 + permisos
├── ClientesModule      → CRUD clientes + documentos/logotipo S3
├── RolesModule         → CRUD roles
├── PermisosModule      → CRUD permisos + agrupados por módulo
├── ModulosModule       → CRUD módulos
├── BitacoraModule      → consulta de auditoría (+ logger interno)
├── S3Module            → upload genérico
└── MailModule          → servicio interno de correo (sin endpoints públicos)
```

### Capas transversales

- **`JwtAuthGuard`**: valida Bearer JWT; solo acepta tokens con `type === 'access'`.
- **`RolesGuard` + `@Roles(...)`**: control de acceso por rol numérico.
- **`BitacoraLoggerService`**: registra CREATE/UPDATE/DELETE/acciones en tabla `Bitacora`.
- **`HttpStringResponseFilter`**: filtro global de respuestas HTTP.
- **`ValidationPipe`**: `whitelist`, `forbidNonWhitelisted`, `transform`.

---

## 4. Modelo de seguridad y multi-tenant

### Roles numéricos

| ID | Nombre (convención) | Alcance típico |
|---:|---|---|
| **1** | SuperAdministrador | Ve/crea/elimina recursos privilegiados; listados globales |
| **2** | Administrador | Acceso autenticado; datos filtrados por cliente e hijos |
| **3** | Usuario / operador | Acceso autenticado; alcance limitado al cliente |

### JWT

| Tipo | Uso | Payload relevante |
|---|---|---|
| **Access** | Header `Authorization: Bearer …` | `id`, `email`, `idCliente`, `rol`, `type: 'access'` |
| **Refresh** | Solo en body de `/login/refresh` y `/login/logout` | `id`, `type: 'refresh'`, `jti` |

- Access → `req.user = { userId, email, idCliente, rol }`.
- Refresh se persiste en `RefreshSessions` como **hash SHA-256** (nunca el token en claro).
- Login emite `{ token, refreshToken }`.
- Refresh **rota** la sesión (revoca anterior, emite nuevo par).
- Logout es **idempotente**.
- Cambio de contraseña revoca **todas** las refresh del usuario.

### Patrón de toggle de estatus

En clientes, usuarios, roles, permisos y módulos:

- Endpoint `PATCH …/estatus…` **sin body**.
- Si `estatus === 1` → pasa a `0`; si `0` → pasa a `1`.
- Clientes: el cambio aplica también a **hijos**.
- Módulos: al desactivar/activar se propaga a **permisos** del módulo (según lógica del servicio).

---

## 5. Funcionalidades implementadas por módulo

### 5.1 Autenticación (`/login`)

- Login con `userName` + `password` (usuario activo, email confirmado, cliente activo).
- Perfil autenticado `GET /login/me` (perfil + permisos activos).
- Rotación de refresh y logout.
- Recuperación de acceso por correo (link con token).
- Cambio de contraseña autenticado.

### 5.2 Usuarios (`/usuarios`)

- Alta/edición con **multipart** (`fotoPerfil` → S3 folder `usuarios`).
- Defaults al crear: `emailConfirmado = 1`, `estatus = 1`.
- Asignación de permisos (`permisosIds`).
- Listados (completo, por cliente, paginado) con filtro por rol/cliente.
- Toggle estatus.
- Cambio de contraseña propia (`:id` debe coincidir con el usuario del token).
- Eliminación.

### 5.3 Clientes (`/clientes`)

- Alta/edición multipart con documentos y logotipo → S3 folder `clientes`:
  - `constanciaSituacionFiscal`, `comprobanteDomicilio`, `actaConstitutiva` (PNG/JPG/PDF)
  - `logotipo` (PNG/JPG)
- Defaults: `idPadre = 1` si vacío/nulo; `estatus = 1` (no se solicita en Swagger de create).
- RFC único.
- Jerarquía padre/hijos (`spGetClientes`).
- Toggle estatus con cascada a hijos.
- Crear/eliminar restringidos a rol **1**.

### 5.4 Roles (`/roles`)

- CRUD + toggle estatus.
- Crear/eliminar solo rol **1**.
- Listados: si el usuario no es SuperAdmin, se oculta el rol id=1.

### 5.5 Permisos (`/permisos`)

- CRUD + toggle estatus.
- Listado agrupado por módulo del usuario autenticado.
- Crear/eliminar solo rol **1**.

### 5.6 Módulos (`/modulos`)

- CRUD + toggle estatus (con impacto en permisos asociados).
- Crear/eliminar solo rol **1**.

### 5.7 Bitácora (`/bitacora`)

- Consulta listado, paginado y detalle (solo lectura).
- Escritura interna desde servicios de negocio.

### 5.8 S3 (`/s3`)

- `POST /s3/upload` genérico (folder + idModule + file).
- Folders admitidos vía DTO: p. ej. `clientes`, `usuarios`, `operadores`, `vehiculos`, `pasajeros`.
- CRUD de usuarios/clientes también suben archivos sin pasar por este endpoint.

### 5.9 Mail

- Sin endpoints HTTP públicos.
- Usado internamente (recuperación de contraseña, plantilla Spring Telecom).

---

## 6. Entidades principales

| Entidad | Tabla | Responsabilidad |
|---|---|---|
| `Usuarios` | `Usuarios` | Credenciales, perfil, `idRol`, `idCliente`, `fotoPerfil`, `estatus` |
| `Clientes` | `Clientes` | Jerarquía `IdPadre`, RFC, dirección, docs/logotipo, `Estatus` |
| `Roles` | `Roles` | Catálogo de roles |
| `Modulos` | `Modulos` | Módulos del sistema |
| `Permisos` | `Permisos` | Permisos por módulo |
| `UsuariosPermisos` | `UsuariosPermisos` | Relación usuario ↔ permiso |
| `Bitacora` | `Bitacora` | Auditoría de operaciones |
| `RefreshSessions` | `RefreshSessions` | Sesiones refresh (jti, hash, expiración, revocación) |
| `CodigoAutenticacion` | `CodigoAutenticacion` | Códigos/OTP de autenticación |

### Carpetas S3 usadas por negocio

| Folder | Origen | Archivos |
|---|---|---|
| `usuarios` | CRUD usuarios | `fotoPerfil` |
| `clientes` | CRUD clientes | documentos fiscales + `logotipo` |

En update, si llega un archivo nuevo: se sube, se actualiza la URL en BD y se elimina el anterior en segundo plano.

---

## 7. Respuestas estándar

Los servicios de escritura suelen devolver `ApiCrudResponse`:

```json
{
  "status": "success",
  "message": "...",
  "data": { "id": 1, "nombre": "..." },
  "estatus": { "estatus": 1 }
}
```

Listados suelen devolver `ApiResponseCommon`:

```json
{
  "data": [ ... ],
  "meta": { "total": 100, "page": 1, "limit": 10 }
}
```

(La presencia exacta de `meta` depende del endpoint.)

---

## 8. Enums de módulos (bitácora)

IDs usados en auditoría (`EnumModulos`):

| ID | Módulo |
|---:|---|
| 0 | Bitácora |
| 1 | Clientes |
| 2 | Usuarios |
| 3 | Roles |
| 4 | Permisos |
| 5 | Módulos |
| 6 | UsuariosPermisos |

(Existen más IDs reservados para dominios futuros: operadores, vehículos, viajes, etc.)

---

## 9. Cómo correr

```bash
npm install
npm run start:dev
```

- API: `http://localhost:3001`
- Swagger: `http://localhost:3001/docs`

---

## 10. Documentos relacionados

- **[Contratos HTTP](./contratos.md)** — especificación de endpoints, request/response y reglas de acceso.
