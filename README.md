# 🗂️ Project Manager — Software Colombia

> Sistema **SaaS Multi-Tenant** de gestión de proyectos donde un usuario puede pertenecer a múltiples espacios de trabajo (*Workspaces*) con roles diferenciados por contexto (**Admin**, **Editor**, **Lector**). Implementado con Spring WebFlux en el backend y React 19 en el frontend, orquestado completamente con Docker.

---

## 🛠️ Stack Tecnológico

### Backend
| Capa | Tecnología |
|---|---|
| Lenguaje | Java 21 |
| Framework | Spring Boot 3.5 (WebFlux — Programación Reactiva) |
| Seguridad | Spring Security + JWT |
| Persistencia | R2DBC + PostgreSQL 16 (no bloqueante) |
| Build Tool | Gradle 8+ |
| Arquitectura | Hexagonal (Ports & Adapters) |

### Frontend
| Capa | Tecnología |
|---|---|
| Lenguaje | TypeScript ~6.0.2 |
| UI Library | React 19 |
| Bundler | Vite 8 |
| Estilos | Tailwind CSS v4 |
| HTTP Client | Axios |
| Routing | React Router DOM v7 |
| Estado global | React Context API |
| Arquitectura | Feature-based |

### Infraestructura
| Servicio | Tecnología |
|---|---|
| Base de datos | PostgreSQL 16 Alpine |
| Contenedores | Docker + Docker Compose |
| Servidor web | Nginx Alpine |

---

## 📐 Decisiones de Arquitectura

### Backend — Arquitectura Hexagonal

Separa completamente la lógica de negocio del dominio de los detalles de infraestructura, permitiendo cambiar la base de datos sin tocar el dominio y testear la lógica de forma aislada.

```
src/
├── application/         # Casos de uso (orquestación)
├── domain/              # Entidades, puertos e interfaces puras
└── infrastructure/
    ├── adapters/        # Implementaciones R2DBC por entidad
    ├── config/          # Seguridad, JWT, configuración global
    └── entrypoints/     # Handlers y routers funcionales (WebFlux)
```

### Frontend — Arquitectura Feature-based

Organiza el código por dominio de negocio, no por tipo de archivo. Cada feature es un módulo autónomo con sus propios componentes, servicios, hooks y tipos.

```
src/
├── app/routes/          # Routing + guards por fase de auth
├── core/                # Axios, tipos globales, storage
└── features/
    ├── auth/            # Login, AuthContext, JWT
    ├── workspace/       # Selector de workspace, roles
    └── projects/        # Dashboard, tabla, creación
```

### ¿Por qué WebFlux?

Permite manejar alta concurrencia con bajo consumo de hilos mediante programación reactiva (Project Reactor), ideal para APIs que deben escalar sin bloquear recursos.

### ¿Por qué dos tokens JWT?

El flujo de autenticación maneja dos tokens con responsabilidades distintas, lo que permite cambiar de workspace sin cerrar sesión:

```
POST /auth/login
  └─► preAuthToken (5 min) — identifica al usuario, lista sus workspaces

POST /auth/token  (preAuthToken + workspaceId)
  └─► accessToken (1 hora) — contiene workspaceId + role del contexto activo
```

### Seguridad stateless

Un filtro JWT personalizado intercepta cada request, valida el token criptográficamente y construye el `AuthPrincipal` con `userId`, `workspaceId` y `role`. Spring Security evalúa los permisos por ruta sin consultar la base de datos en cada request.

### Routing por fases de autenticación (Frontend)

| Fase | Token activo | Ruta |
|---|---|---|
| `unauthenticated` | ninguno | `/login` |
| `pre_authenticated` | preAuthToken | `/workspace` |
| `authenticated` | accessToken | `/dashboard` |

---

## ✅ Requisitos Previos

- [Docker](https://www.docker.com/) instalado y corriendo
- [Docker Compose](https://docs.docker.com/compose/) v2+

---

## 🚀 Ejecución con Docker

> Con un solo comando toda la aplicación queda funcional: base de datos con datos precargados + backend + frontend.

### 1. Clonar el repositorio completo

```bash
git clone --recurse-submodules https://github.com/Jaiderin123/software-colombia-technical-test.git
cd software-colombia-technical-test
```

> El flag `--recurse-submodules` descarga automáticamente el backend y el frontend.

### 2. Levantar todos los servicios

```bash
docker compose up --build
```

> La primera vez tarda unos minutos mientras descarga las imágenes y compila el proyecto.

### 3. Verificar que está corriendo

```
✅ postgres-db  → localhost:5432
✅ backend      → localhost:8080
✅ frontend     → localhost:80
```

Abrir en el navegador: **http://localhost**

### 4. Detener los servicios

```bash
docker compose down
```

> ⚠️ Al bajar los contenedores la base de datos se reinicializa con los datos del script `init.sql` al volver a levantarlos. Esto garantiza un entorno limpio y reproducible.

---

## 🔐 Usuarios Precargados

Todos los usuarios tienen la misma contraseña: **`admin123`**

| Email | Nombre | Workspace Alfa | Workspace Beta |
|---|---|---|---|
| `test@software-colombia.com` | Usuario Prueba | ADMIN | LECTOR |
| `carlos.dev@software-colombia.com` | Carlos Developer | EDITOR | ADMIN |
| `laura.qa@software-colombia.com` | Laura QA | LECTOR | EDITOR |
| `andres.pm@software-colombia.com` | Andres PM | ADMIN | ADMIN |
| `sofia.design@software-colombia.com` | Sofia UI/UX | EDITOR | LECTOR |

**Usuario recomendado para probar:**
```
Email:    test@software-colombia.com
Password: admin123
```

---

## 🗺️ Flujo de la Aplicación

```
/login
  └─► Ingresar credenciales
        └─► /workspace
              └─► Seleccionar workspace (Alfa o Beta)
                    └─► /dashboard
                          ├─► Ver lista de proyectos
                          ├─► Crear proyecto (solo ADMIN / EDITOR)
                          ├─► Cambiar workspace (sin cerrar sesión)
                          └─► Cerrar sesión → /login
```

### Comportamiento por rol

| Rol | Ver proyectos | Crear proyectos |
|---|---|---|
| ADMIN | ✅ | ✅ |
| EDITOR | ✅ | ✅ |
| LECTOR | ✅ | ❌ |

**Flujo de prueba recomendado:**
1. Login con `test@software-colombia.com` / `admin123`
2. Seleccionar **Workspace Alfa** → rol ADMIN → crear un proyecto
3. Click **Cambiar Workspace** → seleccionar **Workspace Beta** → rol LECTOR → sin botón crear
4. **Cerrar sesión** → vuelve al login

---

## 📡 Endpoints del Backend

Base URL: `http://localhost:8080/api/v1`

| Método | Ruta | Auth | Descripción |
|---|---|---|---|
| POST | `/auth/login` | No | Credenciales → user + workspaces + `preAuthToken` |
| POST | `/auth/token?workspaceRequest_Id={id}` | Bearer preAuth | Intercambio → `accessToken` con rol y workspaceId |
| GET | `/project/` | Bearer access | Lista proyectos del workspace activo |
| POST | `/project/` | Bearer access | Crea proyecto (403 si LECTOR) |

---

## 🧪 Colección de Postman

Para probar los endpoints del backend sin configuración manual:

1. Abrir **Postman**
2. Click en **Import**
3. Seleccionar `backend/backend-postman-collection.json`
4. La colección incluye scripts que guardan el `preAuthToken` y el `accessToken` automáticamente entre requests

---

## 🧱 Principios Aplicados

- **SOLID** — cada clase y módulo tiene una responsabilidad única y clara
- **DRY** — lógica centralizada: auth en `AuthContext`, seguridad en `SecurityProperties`, estilos en clases globales
- **KISS** — flujos simples y directos sin complejidad innecesaria
- **Clean Code** — nombres descriptivos, métodos cortos, separación estricta de capas
- **Stateless** — sin sesiones en servidor; toda la información de contexto viaja en el JWT

---

## 📁 Estructura del Monorepo

```
software-colombia-technical-test/
├── docker-compose.yml           # Orquesta BD + Backend + Frontend
├── README.md
├── backend/                     # Spring WebFlux + Arquitectura Hexagonal
│   ├── Dockerfile
│   ├── local-env/postgres/
│   │   └── init.sql             # Script con datos precargados
│   └── src/
│       └── main/java/
│           ├── application/     # Casos de uso
│           ├── domain/          # Entidades y puertos
│           └── infrastructure/  # Adapters, security, entrypoints
└── frontend/                    # React 19 + Feature-based
    ├── Dockerfile
    ├── nginx.conf
    └── src/
        ├── app/routes/          # Routing + guards
        ├── core/                # Axios, storage, tipos
        └── features/
            ├── auth/            # Login, AuthContext
            ├── workspace/       # Selector de workspace
            └── projects/        # Dashboard, tabla, modal
```

---

<div align="center">
  <sub>Desarrollado por <strong>Jaider Betancur</strong> ( <strong>torrezjaider10@gmail.com</strong> ) como prueba técnica para Software Colombia Servicios Informáticos S.A.S.</sub>
</div>