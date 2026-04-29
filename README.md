# Template Login API (.NET 8)

Microservicio de autenticacion y gestion de usuarios para usar como base de proyectos.
Incluye registro, login con JWT, manejo de refresh token, cambio y recuperacion de contrasena, verificacion de email y procesamiento asincrono de eventos de notificacion.

## Que hace este microservicio

- Alta de usuarios con validaciones de negocio.
- Autenticacion por email y password.
- Emision de `access token` (JWT) y `refresh token`.
- Renovacion de sesion con refresh token.
- Logout invalidando refresh token.
- Cambio de contrasena para usuarios autenticados.
- Flujo de recuperacion de contrasena por codigo enviado por email.
- Verificacion de cuenta por codigo de email y reenvio de codigo.
- Recepcion de eventos de notificacion para procesarlos en background.

## Arquitectura

La solucion esta organizada por capas:

- `LoginMS`: API HTTP, configuracion, DI, autenticacion, CORS y Swagger.
- `Application`: casos de uso, DTOs, validadores e interfaces.
- `Domain`: entidades del dominio (`User`, `RefreshToken`, `PasswordResetToken`, etc.).
- `Infrastructure`: persistencia con EF Core, implementaciones de queries/commands y servicios tecnicos (JWT, email, dispatcher de notificaciones).

Patrones usados:

- Separacion por capas (estilo Clean Architecture).
- CQRS para operaciones de lectura/escritura.
- Validacion de requests con FluentValidation.
- Hosted Service para procesamiento asincrono de notificaciones.

## Endpoints principales

Base path: `api/v1`

Autenticacion (`AuthController`):

- `POST /api/v1/Auth/Login`
- `POST /api/v1/Auth/Logout`
- `POST /api/v1/Auth/RefreshToken`
- `POST /api/v1/Auth/ChangePassword` (requiere JWT)
- `POST /api/v1/Auth/PasswordResetRequest`
- `POST /api/v1/Auth/PasswordResetConfirm`
- `POST /api/v1/Auth/VerifyEmail`
- `POST /api/v1/Auth/ResendVerificationEmail`

Usuarios (`UserController`):

- `POST /api/v1/User` (registro)
- `PUT /api/v1/User/{id}` (requiere JWT)
- `PATCH /api/v1/User/RemoveImage/{id}` (requiere JWT)
- `GET /api/v1/User/{id}`

Notificaciones (`NotificationController`):

- `POST /api/v1/notifications/events` (encola evento para procesamiento asincrono, responde `202 Accepted`)

## Flujo funcional (resumen)

1. Usuario se registra.
2. El sistema puede enviar codigo de verificacion de email.
3. Usuario hace login y obtiene `access token` + `refresh token`.
4. Con `access token` consume endpoints protegidos.
5. Cuando expira el `access token`, usa `RefreshToken` para renovarlo.
6. Puede hacer `Logout` para invalidar el refresh token.
7. Si olvida su clave, usa `PasswordResetRequest` + `PasswordResetConfirm`.

## Validaciones de seguridad destacadas

- Password con politica minima (longitud, mayuscula, minuscula, numero y simbolo).
- Regla para evitar reutilizar la misma password al cambiarla.
- Control de expiracion de JWT.
- Configuracion de lockout por intentos fallidos.

## Configuracion

Archivo base de desarrollo: `LoginMS/appsettings.Development.json`

Claves relevantes:

- `ConnectionString`
- `SaltSettings`
- `Argon2Settings`
- `JwtSettings:TokenExpirationMinutes`
- `RefreshTokenSettings:Lenght`
- `RefreshTokenSettings:LifeTimeInMinutes`
- `RefreshTokenSettings:IdleTimeoutMinutes`
- `LockoutSettings`
- `EmailSettings:SmtpServer`
- `EmailSettings:SmtpPort`
- `EmailSettings:SenderEmail`

Secretos obligatorios (no commitear):

- `JwtSettings:key`
- `EmailSettings:SenderPassword`

## Requisitos

- .NET SDK 8+
- SQL Server o LocalDB
- (Opcional) herramienta EF Core CLI para migraciones:

```bash
dotnet tool install --global dotnet-ef
```

## Puesta en marcha local

1. Restaurar dependencias:

```bash
dotnet restore LoginMS.sln
```

2. Configurar secretos:

```bash
dotnet user-secrets --project LoginMS set "JwtSettings:key" "TU_CLAVE_JWT_SEGURA"
dotnet user-secrets --project LoginMS set "EmailSettings:SenderPassword" "TU_PASSWORD_O_APP_PASSWORD_SMTP"
```

3. Aplicar migraciones:

```bash
dotnet ef database update --project Infrastructure --startup-project LoginMS
```

4. Ejecutar la API:

```bash
dotnet run --project LoginMS
```

5. Abrir Swagger en la URL del perfil local (ver `LoginMS/Properties/launchSettings.json`).

## Ejemplos minimos de payloads

Registro:

```json
{
  "firstName": "Juan",
  "lastName": "Perez",
  "email": "juan@example.com",
  "dni": "12345678",
  "password": "Password123!"
}
```

Login:

```json
{
  "email": "juan@example.com",
  "password": "Password123!"
}
```

Refresh token:

```json
{
  "expiredAccessToken": "JWT_EXPIRADO",
  "refreshToken": "REFRESH_TOKEN"
}
```

Evento de notificacion:

```json
{
  "userId": 1,
  "eventType": "ReservationCreated",
  "payload": {
    "reservationId": 1001
  }
}
```


