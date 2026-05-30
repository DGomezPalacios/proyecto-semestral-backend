# Backend — Proyecto Semestral DevOps

Arquitectura de microservicios Spring Boot para el sistema de Despachos y Ventas.

## Arquitectura

```
proyecto-semestral-backend/
├── Dockerfile               # Microservicio Despachos
├── .dockerignore
├── pom.xml                  # Spring Boot Despachos (Springboot-API-REST-DESPACHO)
├── src/                     # Código fuente Despachos
└── back-ventas/
    ├── Dockerfile           # Microservicio Ventas
    ├── .dockerignore
    ├── pom.xml              # Spring Boot Ventas (Springboot-API-REST)
    └── src/                 # Código fuente Ventas
```

Cada microservicio es independiente, tiene su propia base de datos MySQL y se despliega como contenedor separado.

## Tecnologías

| Tecnología | Versión | Uso |
|---|---|---|
| Spring Boot | 3.4.4 | Framework principal |
| Java | 17 | Lenguaje |
| Maven | 3.8.1 | Build tool |
| Spring Data JPA | 3.4.3 | Persistencia ORM |
| MySQL Connector/J | latest | Driver de base de datos |
| Springdoc OpenAPI | 2.7.0 | Documentación Swagger UI |
| Lombok | 1.18.36 | Reducción de boilerplate |
| MySQL | 8.0 | Base de datos relacional |

## Iniciar con docker-compose

Ejecutar desde la raíz del proyecto (donde está `docker-compose.yml`):

```bash
# Primera vez: construir imágenes y levantar
docker-compose up --build

# Levantar en segundo plano
docker-compose up -d --build

# Ver logs de un servicio específico
docker-compose logs -f backend-despachos
docker-compose logs -f backend-ventas

# Detener todos los servicios
docker-compose down

# Detener y limpiar volúmenes (borra datos de MySQL)
docker-compose down -v
```

## Puertos y endpoints

| Servicio | Puerto externo | Puerto interno | Base de datos |
|---|---|---|---|
| backend-despachos | 8081 | 8080 | despachos_db |
| backend-ventas | 8082 | 8080 | ventas_db |
| database (MySQL) | 3306 | 3306 | — |

### URLs de la API

| Servicio | Swagger UI | API Base |
|---|---|---|
| Despachos | http://localhost:8081/swagger-ui.html | http://localhost:8081/api |
| Ventas | http://localhost:8082/swagger-ui.html | http://localhost:8082/api |

## Variables de entorno

Configuradas en `docker-compose.yml`:

| Variable | Descripción |
|---|---|
| `SPRING_DATASOURCE_URL` | URL JDBC de conexión a MySQL |
| `SPRING_DATASOURCE_USERNAME` | Usuario de base de datos |
| `SPRING_DATASOURCE_PASSWORD` | Contraseña de base de datos |
| `SPRING_JPA_HIBERNATE_DDL_AUTO` | Estrategia DDL (`update` en dev) |

## Iniciar en local (sin Docker)

Requiere MySQL local corriendo en puerto 3306.

```bash
# Microservicio Despachos
cd proyecto-semestral-backend
./mvnw spring-boot:run

# Microservicio Ventas
cd proyecto-semestral-backend/back-ventas
./mvnw spring-boot:run
```

## Best practices aplicadas

- **Multi-stage build**: imagen final basada en `openjdk:17-slim`, sin Maven en runtime.
- **Usuario no-root**: `appuser` sin privilegios de sistema operativo.
- **HEALTHCHECK**: Docker verifica que la aplicación responde antes de marcarla como `healthy`.
- **`dependency:go-offline`**: cachea dependencias Maven en la capa de build para rebuilds rápidos.
- **`-DskipTests`**: los tests se ejecutan en CI/CD, no en la imagen de producción.
- **`restart: unless-stopped`**: los contenedores se reinician automáticamente tras fallos.

## Seguridad

- Cambiar las credenciales de base de datos (`root123`) por secretos gestionados (Docker Secrets, Vault) en producción.
- En producción usar `spring.jpa.hibernate.ddl-auto=validate` o `none` para evitar migraciones automáticas no controladas.
- Nunca exponer el puerto 3306 de MySQL en entornos productivos.

## DevOps

El `Dockerfile` de cada microservicio sigue el patrón multi-stage:

1. **Stage `builder`** (`maven:3.8.1-openjdk-17`): descarga dependencias y empaqueta el JAR.
2. **Stage runtime** (`openjdk:17-slim`): copia solo el JAR final y lo ejecuta con usuario sin privilegios.

El `HEALTHCHECK` integrado permite que `docker-compose` y orquestadores como Kubernetes conozcan el estado real de la aplicación.
