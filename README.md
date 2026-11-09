# JustLift API 

Backend REST para la aplicación Android [JustLift](https://github.com/FernadoOXD/JustLift).
Es una API REST implementada con [json-server](https://github.com/typicode/json-server) y
[json-server-auth](https://github.com/jeremyben/json-server-auth) que provee autenticación mediante tokens JWT y un recurso protegido (`rutinas`) con aislamiento de datos por usuario.

>  **API REST en la Nube (Producción):** `https://justlift-backend.onrender.com/`  
>  **Nota:** Entorno configurado para fines académicos y de desarrollo.

---

## Contenido

- [Requisitos](#requisitos)
- [Instalación y ejecución local](#instalación-y-ejecución-local)
- [Despliegue en la Nube (Render)](#despliegue-en-la-nube-render)
- [Estructura del proyecto](#estructura-del-proyecto)
- [Endpoints](#endpoints)
- [Autenticación](#autenticación)
- [Permisos y Seguridad](#permisos-y-seguridad)
- [Ejemplos con curl](#ejemplos-con-curl)
- [Conectar la app Android](#conectar-la-app-android)
- [Restablecer los datos](#restablecer-los-datos)
- [Solución de problemas](#solución-de-problemas)
- [Limitaciones](#limitaciones)

---

## Requisitos

- [Node.js](https://nodejs.org) LTS v18+ (incluye `npm`).

---

## Instalación y ejecución local

```bash
git clone https://github.com/FernadoOXD/JustLift_Backend.git
cd JustLift_Backend
npm install
npm start
```

El servidor local iniciará escuchando en **http://localhost:3000**.

| Comando | Descripción |
|---|---|
| `npm start` | Inicia el servidor escuchando en `0.0.0.0` y usando el puerto configurado o el puerto 3000 |
| `npm run reset` | Restablece `db.json` al estado inicial (limpia usuarios y rutinas de prueba) |

> **Nota:** Se utiliza `json-server@0.17.4` debido a incompatibilidad de `json-server-auth` con las versiones 1.x de json-server.

---

## Despliegue en la Nube (Render)

El backend está configurado para desplegarse automáticamente en **Render**:

- **Host Binding:** Escucha explícitamente en la interfaz `--host 0.0.0.0`.
- **Port Binding:** Lee dinámicamente la variable de entorno `--port $PORT` inyectada por el contenedor de Render.
- **URL Pública:** `https://justlift-backend.onrender.com`

---

## Estructura del proyecto

```text
JustLift_Backend/
├── db.template.json   # Base de datos plantilla de origen
├── db.json            # Base de datos activa (generada localmente / ignorada en git)
├── routes.json        # Configuración de permisos y alias de rutas
├── package.json       # Configuración de dependencias y scripts de inicio
└── README.md          # Documentación general de la API
```

### Configuración del `package.json`

```json
{
  "name": "justlift-backend",
  "version": "1.0.0",
  "private": true,
  "scripts": {
    "prepare-db": "node -e \"const fs=require('fs');if(!fs.existsSync('db.json')){fs.copyFileSync('db.template.json','db.json')}\"",
    "start": "npm run prepare-db && json-server-auth db.json --host 0.0.0.0 --port ${PORT:-3000} --routes routes.json",
    "reset": "node -e \"require('fs').copyFileSync('db.template.json','db.json')\""
  },
  "dependencies": {
    "json-server": "0.17.4",
    "json-server-auth": "^2.1.0"
  }
}
```

---

## Endpoints

| Método | Ruta | Descripción | Requiere Bearer Token |
|---|---|---|---|
| POST | `/register` | Registra un usuario y devuelve token + datos de usuario | No |
| POST | `/login` | Inicia sesión y devuelve token + datos de usuario | No |
| GET | `/rutinas` | Lista las rutinas creadas | Sí |
| GET | `/rutinas/{id}` | Obtiene los detalles de una rutina específica | Sí |
| POST | `/rutinas` | Crea una rutina asignada al `userId` autenticado | Sí |
| PUT / PATCH | `/rutinas/{id}` | Modifica una rutina (solo si pertenece al usuario) | Sí |
| DELETE | `/rutinas/{id}` | Elimina una rutina (solo si pertenece al usuario) | Sí |

### Estructura de la Entidad `Rutina`

```json
{
  "id": 1,
  "userId": 1,
  "nombre": "Hipertrofia de pecho",
  "grupoMuscular": "Pecho",
  "series": 4,
  "repeticiones": 10,
  "activa": true
}
```

---

## Autenticación

El endpoint `POST /register` acepta `email`, `password` y `name`. Las contraseñas se almacenan con cifrado bcrypt.

### Respuesta en `/register` y `/login`

```json
{
  "accessToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "user": {
    "id": 1,
    "email": "ana@test.com",
    "name": "Ana"
  }
}
```

- El token de acceso expira en **1 hora**; peticiones posteriores sin token válido devolverán HTTP `401 Unauthorized`.
- Las peticiones protegidas deben incluir la cabecera: `Authorization: Bearer <accessToken>`.

---

## Permisos y Seguridad

Configuración en `routes.json`:

```json
{
  "users": 600,
  "rutinas": 640
}
```

- **`rutinas` (`640`):** El usuario creador (`userId`) tiene permisos de lectura y escritura; otros usuarios autenticados tienen acceso de lectura; usuarios no autenticados no tienen acceso (`401`).
- **`users` (`600`):** Cada usuario solo puede consultar o modificar sus propios datos de cuenta.

---

## Ejemplos con curl

> **Nota para PowerShell:** Usa `curl.exe` en lugar de `curl` para evitar conflictos con los alias de PowerShell.

### 1. Registrar usuario

```bash
curl.exe -X POST http://localhost:3000/register `
  -H "Content-Type: application/json" `
  -d '{"email":"ana@test.com","password":"12345678","name":"Ana"}'
```

### 2. Iniciar sesión

```bash
curl.exe -X POST http://localhost:3000/login `
  -H "Content-Type: application/json" `
  -d '{"email":"ana@test.com","password":"12345678"}'
```

### 3. Crear una rutina (requiere token JWT)

```bash
curl.exe -X POST http://localhost:3000/rutinas `
  -H "Authorization: Bearer TOKEN_AQUI" `
  -H "Content-Type: application/json" `
  -d '{"userId":1,"nombre":"Sentadillas","grupoMuscular":"Pierna","series":4,"repeticiones":12,"activa":true}'
```

### 4. Consultar rutinas

```bash
curl.exe http://localhost:3000/rutinas -H "Authorization: Bearer TOKEN_AQUI"
```

---

## Conectar la app Android

### Opción A: API en Producción (Render)
Modifica la URL en `AppContainer.kt` del proyecto Android:

```kotlin
.baseUrl("https://justlift-backend.onrender.com/")
```

### Opción B: Entorno Local mediante USB
1. Inicia la API localmente en tu PC (`npm start`).
2. Configura Retrofit en la app Android hacia `http://localhost:3000/`.
3. Crea el túnel de comunicación por USB en PowerShell:

```powershell
& "$env:LOCALAPPDATA\Android\Sdk\platform-tools\adb.exe" reverse tcp:3000 tcp:3000
```

---

## Restablecer los datos

Para volver la base de datos a su estado limpio inicial:

```bash
npm run reset
```

---

## Solución de problemas

| Síntoma | Causa probable | Solución |
|---|---|---|
| `401 Missing authorization header` | Petición a endpoint protegido sin token | Adjuntar el encabezado `Authorization: Bearer <token>` |
| `401 jwt expired` | El token expiró (duración: 1h) | Volver a iniciar sesión (`POST /login`) para renovar el token |
| `No open ports detected on 0.0.0.0` (Render) | Falta la bandera `--host 0.0.0.0` en el script | Verificar que `package.json` use `--host 0.0.0.0 --port $PORT` |
| Respuesta lenta en la primera petición (Render) | Contenedor en reposo por inactividad (Free Tier) | Esperar 30 segundos para el arranque en frío del servidor |

---

## Autor

- **Nombre:** Fernando Reyes Gómez
- **Repositorio App:** [JustLift](https://github.com/FernadoOXD/JustLift)
- **Repositorio Backend:** [JustLift_Backend](https://github.com/FernadoOXD/JustLift_Backend)
