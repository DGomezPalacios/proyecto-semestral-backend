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

## ☁️ Despliegue en AWS

### Arquitectura de instancias

| Instancia | Rol | IP Pública | IP Privada |
|---|---|---|---|
| EC2-web | Frontend (nginx + React) | 3.83.173.99 | 10.0.12.224 |
| EC2-app | Backend (Docker + microservicios) | — | 10.0.131.198 |
| EC2-datos | Base de datos (MySQL 8.0) | — | 10.0.145.181 |

Todas las instancias pertenecen a la VPC **proyecto-semestral-vpc**. Solo EC2-web tiene IP pública; EC2-app y EC2-datos son accesibles únicamente desde dentro de la VPC.

### Conexión SSH

La clave privada requerida es `devops-front.pem`. Acceder a EC2-web directamente:

```bash
# Conectar a EC2-web (Frontend) — única instancia con IP pública
ssh -i devops-front.pem ec2-user@3.83.173.99
```

EC2-app y EC2-datos no tienen IP pública; acceder desde EC2-web como salto (jump host):

```bash
# Conectar a EC2-app (Backend) usando EC2-web como bastión
ssh -i devops-front.pem -J ec2-user@3.83.173.99 ec2-user@10.0.131.198

# Conectar a EC2-datos (MySQL) usando EC2-web como bastión
ssh -i devops-front.pem -J ec2-user@3.83.173.99 ec2-user@10.0.145.181
```

> Asegurarse de que el archivo `.pem` tenga permisos correctos antes de usarlo:
> ```bash
> chmod 400 devops-front.pem
> ```

### Levantar los microservicios en EC2-app

```bash
# 1. Conectar a EC2-app
ssh -i devops-front.pem -J ec2-user@3.83.173.99 ec2-user@10.0.131.198

# 2. Ir al directorio del proyecto
cd ~/ProyectoSemestral

# 3. Construir y levantar en segundo plano
docker-compose up -d --build

# 4. Verificar que los contenedores corren
docker-compose ps

# 5. Ver logs en tiempo real
docker-compose logs -f backend-despachos
docker-compose logs -f backend-ventas

# 6. Detener los servicios
docker-compose down
```

### Acceso a endpoints en AWS

El tráfico llega a EC2-web (IP pública) y nginx actúa como reverse proxy hacia EC2-app:

| Servicio | URL pública | Puerto interno en EC2-app |
|---|---|---|
| Frontend | http://3.83.173.99 | 80 |
| API Despachos | http://3.83.173.99/api-despachos | 8081 |
| API Ventas | http://3.83.173.99/api-ventas | 8082 |
| Swagger Despachos | http://3.83.173.99/api-despachos/swagger-ui.html | 8081 |
| Swagger Ventas | http://3.83.173.99/api-ventas/swagger-ui.html | 8082 |

Acceso directo a los puertos del backend (requiere abrir Security Group de EC2-app):

| Servicio | URL directa (desde dentro de la VPC) |
|---|---|
| backend-despachos | http://10.0.131.198:8081 |
| backend-ventas | http://10.0.131.198:8082 |

### Security Groups configurados

| Security Group | Instancia | Reglas de entrada |
|---|---|---|
| sg-web | EC2-web | 22 (SSH) desde cualquier IP, 80/443 (HTTP/HTTPS) desde cualquier IP |
| sg-app | EC2-app | 22 (SSH) desde sg-web, 8081/8082 desde sg-web |
| sg-datos | EC2-datos | 3306 (MySQL) desde sg-app únicamente |

### Flujo de conexión

```
Internet
   │
   ▼
EC2-web (3.83.173.99)     ← IP pública, nginx reverse proxy
   │  10.0.12.224
   │
   ▼ IP privada
EC2-app (10.0.131.198)    ← Docker + microservicios Spring Boot
   │
   ▼ IP privada
EC2-datos (10.0.145.181)  ← MySQL 8.0
```

### AWS vs ejecución local

| Aspecto | Local (docker-compose) | AWS |
|---|---|---|
| Acceso externo | Solo localhost | IP pública accesible desde cualquier lugar |
| Alta disponibilidad | No | Posible con Auto Scaling y ALB |
| Aislamiento de red | Red Docker interna | VPC con subredes públicas y privadas |
| Base de datos | Contenedor efímero | EC2 dedicado con volumen EBS persistente |
| Seguridad | Security groups de Docker | AWS Security Groups y NACLs |
| Escalabilidad | Limitada al host | Vertical y horizontal según necesidad |

## Seguridad

- Cambiar las credenciales de base de datos (`root123`) por secretos gestionados (Docker Secrets, Vault) en producción.
- En producción usar `spring.jpa.hibernate.ddl-auto=validate` o `none` para evitar migraciones automáticas no controladas.
- Nunca exponer el puerto 3306 de MySQL en entornos productivos.

## DevOps

El `Dockerfile` de cada microservicio sigue el patrón multi-stage:

1. **Stage `builder`** (`maven:3.8.1-openjdk-17`): descarga dependencias y empaqueta el JAR.
2. **Stage runtime** (`openjdk:17-slim`): copia solo el JAR final y lo ejecuta con usuario sin privilegios.

El `HEALTHCHECK` integrado permite que `docker-compose` y orquestadores como Kubernetes conozcan el estado real de la aplicación.
