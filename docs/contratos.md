# Contratos HTTP — SentinelAPI

> Contratos de las APIs implementadas. Complementa [contexto.md](./contexto.md).  
> Base URL local: `http://localhost:{PORT}` · Auth: `Authorization: Bearer <accessToken>` salvo endpoints públicos.

---

## Convenciones

| Concepto | Regla |
|---|---|
| Content-Type JSON | `application/json` |
| Multipart | `multipart/form-data` (campos texto + files) |
| Roles | `1` SuperAdmin · `2` Admin · `3` Usuario |
| Toggle estatus | Sin body: `1 ↔ 0` |
| Access JWT | Obligatorio en casi todos los endpoints protegidos |
| Refresh JWT | Solo body en `/login/refresh` y `/login/logout` (no sirve como Bearer) |
| Códigos comunes | `200/201` OK · `400` validación · `401` no auth · `403` rol · `404` no encontrado |

---

## 1. Autenticación — prefijo `/login`

Guards a nivel clase: ninguno (salvo endpoints marcados).

### `POST /login`

| | |
|---|---|
| **Auth** | Público |
| **Body** | `{ "userName": string, "password": string }` |
| **Response 200** | `{ "token": string, "refreshToken": string }` |
| **Reglas** | Usuario `estatus=1`, `emailConfirmado=1`, cliente activo. Actualiza `ultimoLogin`. |

### `GET /login/me`

| | |
|---|---|
| **Auth** | Bearer access |
| **Response 200** | Perfil del usuario + permisos activos (misma forma de perfil que el login histórico, sin tokens) |

### `POST /login/refresh`

| | |
|---|---|
| **Auth** | Público (usa refresh en body) |
| **Body** | `{ "refreshToken": string }` |
| **Response 200** | `{ "token": string, "refreshToken": string }` |
| **Reglas** | Valida JWT refresh + sesión en `RefreshSessions`. **Rota** sesión (revoca anterior). |

### `POST /login/logout`

| | |
|---|---|
| **Auth** | Público (usa refresh en body) |
| **Body** | `{ "refreshToken": string }` |
| **Response 200** | Éxito (idempotente aunque el token ya esté revocado) |

### `POST /login/usuario/recuperar/acceso`

| | |
|---|---|
| **Auth** | Público |
| **Body** | `{ "userName": string }` (según DTO de confirmación/recuperación) |
| **Efecto** | Envía correo con enlace de cambio de contraseña (token de confirmación) |

### `POST /login/cambiar/accesso`

| | |
|---|---|
| **Auth** | Bearer access |
| **Body** | DTO reset (`userName`, `password`, …) |
| **Efecto** | Actualiza hash de contraseña y **revoca todas** las refresh del usuario |

---

## 2. Usuarios — prefijo `/usuarios`

Guards: `JwtAuthGuard` + `RolesGuard`. Roles clase: autenticado (`@Roles()`).

### `POST /usuarios`

| | |
|---|---|
| **Content-Type** | `multipart/form-data` |
| **Campos** | `userName`, `passwordHash`, `nombre`, `apellidoPaterno`, `apellidoMaterno?`, `telefono?`, `idRol`, `idCliente`, `permisosIds` (JSON/CSV/array), file `fotoPerfil?` |
| **Defaults** | `emailConfirmado=1`, `estatus=1` (no se solicitan en Swagger) |
| **S3** | Folder `usuarios` |
| **Response** | `ApiCrudResponse` |

### `GET /usuarios/list`

Lista completa filtrada por rol/`idCliente` del token.

### `GET /usuarios/list/cliente/:id`

Usuarios del cliente `:id`.

### `GET /usuarios/:page/:limit`

Paginado según rol/cliente/hijos.

### `GET /usuarios/:id`

Detalle + permisos. Alcance según rol.

### `PATCH /usuarios/:id`

| | |
|---|---|
| **Content-Type** | `multipart/form-data` |
| **Campos** | Parciales del usuario + file `fotoPerfil?` |
| **S3** | Reemplaza foto; elimina anterior en background |

### `PATCH /usuarios/estatus/:id`

| | |
|---|---|
| **Body** | Ninguno |
| **Efecto** | Toggle `estatus` `1 ↔ 0` |

### `PATCH /usuarios/actualizar/contrasena/:id`

| | |
|---|---|
| **Body** | `{ passwordActual, passwordNueva, passwordNuevaConfirmacion }` |
| **Regla** | `:id` debe ser igual a `req.user.userId` → si no, `403` |

### `DELETE /usuarios/:id`

Elimina / deshabilita usuario según lógica del servicio.

---

## 3. Clientes — prefijo `/clientes`

Guards: JWT + Roles. Roles clase: `@Roles(1,2,3)`.

### `POST /clientes` — solo rol **1**

| | |
|---|---|
| **Content-Type** | `multipart/form-data` |
| **Required** | `rfc`, `tipoPersona` (`1` Física / `2` Moral) |
| **Opcionales texto** | `idPadre`, `nombre`, `apellidoPaterno`, `apellidoMaterno`, `telefono`, `correo`, `sitioWeb`, dirección (`estado`, `municipio`, `colonia`, `calle`, `entreCalles`, `numeroExterior`, `numeroInterior`, `cp`), encargado (`nombreEncargado`, `telefonoEncargado`, `correoEncargado`) |
| **Files** | `constanciaSituacionFiscal?`, `comprobanteDomicilio?`, `actaConstitutiva?` (PNG/JPG/PDF), `logotipo?` (PNG/JPG) |
| **Defaults código** | `idPadre = 1` si nulo/vacío · `estatus = 1` (no se pide en Swagger) |
| **S3** | Folder `clientes` |
| **Errores** | RFC duplicado → `400` |

### `GET /clientes/list`

Lista según rol/`idCliente`.

### `GET /clientes/list/:cliente`

Lista filtrada por id de cliente.

### `GET /clientes/:page/:limit`

Paginado.

### `GET /clientes/:id`

Detalle de un cliente.

### `PATCH /clientes/:id`

| | |
|---|---|
| **Content-Type** | `multipart/form-data` |
| **Campos** | Parciales + mismos files opcionales que create |
| **S3** | Reemplazo de URL + borrado del archivo anterior |

### `PATCH /clientes/estatus/:id`

| | |
|---|---|
| **Body** | Ninguno |
| **Efecto** | Toggle `1 ↔ 0` en cliente **e hijos** |

### `DELETE /clientes/:id` — solo rol **1**

Eliminación lógica / deshabilitado según servicio (incluye hijos vía SP).

---

## 4. Roles — prefijo `/roles`

Guards: JWT + Roles. Roles clase: `@Roles(1,2,3)`.

| Método | Ruta | Roles extra | Body | Notas |
|---|---|---|---|---|
| `POST` | `/roles` | **1** | `{ nombre, descripcion?, estatus? }` | Default `estatus=1` |
| `GET` | `/roles/:page/:limit` | 1,2,3 | — | Si rol ≠ 1, oculta rol id=1 |
| `GET` | `/roles/list` | 1,2,3 | — | Activos; oculta id=1 si rol ≠ 1 |
| `GET` | `/roles/:id` | 1,2,3 | — | Detalle |
| `PUT` | `/roles/:id` | 1,2,3 | Partial | Actualiza |
| `PATCH` | `/roles/estatus/:id` | 1,2,3 | **ninguno** | Toggle `1 ↔ 0` |
| `DELETE` | `/roles/:id` | **1** | — | Elimina / deshabilita |

---

## 5. Permisos — prefijo `/permisos`

Guards: JWT + Roles. Roles clase: `@Roles(1,2,3)`.

| Método | Ruta | Roles extra | Body | Notas |
|---|---|---|---|---|
| `POST` | `/permisos` | **1** | `{ nombre, descripcion?, idModulo, estatus? }` | Default `estatus=1` |
| `GET` | `/permisos/:page/:limit` | 1,2,3 | — | Paginado |
| `GET` | `/permisos/list` | 1,2,3 | — | Lista |
| `GET` | `/permisos/permisosAgrupados` | 1,2,3 | — | Permisos del usuario del token, agrupados por módulo |
| `GET` | `/permisos/:id` | 1,2,3 | — | Detalle |
| `PUT` | `/permisos/:id` | 1,2,3 | Partial | Actualiza |
| `PATCH` | `/permisos/:id/estatus` | 1,2,3 | **ninguno** | Toggle `1 ↔ 0` |
| `DELETE` | `/permisos/:id` | **1** | — | Elimina |

---

## 6. Módulos — prefijo `/modulos`

Guards: JWT + Roles. Roles clase: `@Roles(1,2,3)`.

| Método | Ruta | Roles extra | Body | Notas |
|---|---|---|---|---|
| `POST` | `/modulos` | **1** | `{ nombre, descripcion, estatus? }` | Default `estatus=1` |
| `GET` | `/modulos/list` | 1,2,3 | — | Lista |
| `GET` | `/modulos/:page/:limit` | 1,2,3 | — | Paginado |
| `GET` | `/modulos/:id` | 1,2,3 | — | Detalle |
| `PUT` | `/modulos/:id` | 1,2,3 | Partial | Actualiza |
| `PATCH` | `/modulos/:id/estatus` | 1,2,3 | **ninguno** | Toggle `1 ↔ 0` (+ impacto en permisos) |
| `DELETE` | `/modulos/:id` | **1** | — | Elimina / deshabilita |

---

## 7. Bitácora — prefijo `/bitacora`

Guards: JWT + Roles. Roles clase: `@Roles(1,2,3)`. Solo lectura.

| Método | Ruta | Notas |
|---|---|---|
| `GET` | `/bitacora/list` | Lista (marcado obsoleto en código) |
| `GET` | `/bitacora/:page/:limit` | Paginado filtrado por rol/cliente |
| `GET` | `/bitacora/:id` | Detalle |

---

## 8. S3 — prefijo `/s3`

Guards: JWT + Roles. Roles clase: `@Roles(1,2,3)`.

### `POST /s3/upload`

| | |
|---|---|
| **Content-Type** | `multipart/form-data` |
| **Campos** | file `file` (PNG/JPG/PDF, máx. ~10 MB), `folder` (string), `idModule` (number) |
| **Response** | `{ url: string, … }` (según servicio) |
| **Efecto** | Sube a bucket, registra bitácora |

---

## 9. Mail

Sin contratos HTTP públicos. Consumo interno desde autenticación (recuperación de acceso).

---

## 10. Matriz de contratos (resumen)

| Prefijo | # endpoints | JWT | Multipart | Solo rol 1 |
|---|---:|---|---|---|
| `/login` | 6 | 2 (`/me`, `/cambiar/accesso`) | — | — |
| `/usuarios` | 9 | todos | create + update | — |
| `/clientes` | 8 | todos | create + update | POST, DELETE |
| `/roles` | 7 | todos | — | POST, DELETE |
| `/permisos` | 8 | todos | — | POST, DELETE |
| `/modulos` | 7 | todos | — | POST, DELETE |
| `/bitacora` | 3 | todos | — | — |
| `/s3` | 1 | todos | upload | — |
| **Total** | **~49** | | | |

---

## 11. Ejemplos mínimos

### Login

```http
POST /login
Content-Type: application/json

{
  "userName": "admin@ejemplo.com",
  "password": "P@ssword123"
}
```

```json
{
  "token": "eyJ...",
  "refreshToken": "eyJ..."
}
```

### Refresh

```http
POST /login/refresh
Content-Type: application/json

{ "refreshToken": "eyJ..." }
```

### Toggle estatus (sin body)

```http
PATCH /clientes/estatus/5
Authorization: Bearer <accessToken>
```

### Crear cliente (multipart)

```http
POST /clientes
Authorization: Bearer <accessToken>
Content-Type: multipart/form-data

rfc=XAXX010101000
tipoPersona=1
nombre=Juan
apellidoPaterno=Pérez
logotipo=<file>
constanciaSituacionFiscal=<file>
```

---

## 12. Esquema Clientes (referencia BD)

Columnas alineadas con la entidad TypeORM `Clientes`:

`Id`, `IdPadre`, `RFC` (unique), `TipoPersona`, `Nombre`, `ApellidoMaterno`, `ApellidoPaterno`, `Telefono`, `Correo`, `SitioWeb`, dirección (`Estado`…`CP`), encargado, `ConstanciaSituacionFiscal`, `ComprobanteDomicilio`, `ActaConstitutiva`, `Logotipo`, `Estatus` (default 1), `FechaActualizacion`, `FechaCreacion`. FK `IdPadre` → `Clientes.Id`.
