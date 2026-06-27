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

Todas las instancias pertenecen a la VPC **proyecto-semestral-vpc** (`10.0.0.0/16`). Solo EC2-web tiene IP pública; EC2-app y EC2-datos son accesibles únicamente desde dentro de la VPC.

### Diagrama de red

```
          Internet
             │
             ▼
  ┌────────────────────────────┐
  │  EC2-web  3.83.173.99:80   │  ← nginx + React (IP pública)
  │  privada: 10.0.12.224      │
  └─────────────┬──────────────┘
                │ VPC  10.0.0.0/16
                ▼
  ┌────────────────────────────┐
  │  EC2-app  10.0.131.198     │  ← Spring Boot Despachos :8081
  │                            │     Spring Boot Ventas    :8082
  └─────────────┬──────────────┘
                │ VPC interna
                ▼
  ┌────────────────────────────┐
  │  EC2-datos 10.0.145.181    │  ← MySQL 8.0 :3306
  └────────────────────────────┘
```

### Imágenes Docker Hub

Las imágenes publicadas en Docker Hub se usan para el despliegue en EC2-app y EC2-web:

| Servicio | Imagen |
|---|---|
| Backend Despachos | `dgomezpalacios/proyecto-semestral-backend:latest` |
| Backend Ventas | `dgomezpalacios/proyecto-semestral-backend:latest` |

```bash
# Descargar imagen manualmente
docker pull dgomezpalacios/proyecto-semestral-backend:latest

# Ejecutar directamente desde Docker Hub (sin build)
docker run -d -p 8081:8080 dgomezpalacios/proyecto-semestral-backend:latest
```

### Conexión SSH

La clave privada requerida es `devops-front.pem`.

```bash
# Permisos correctos antes de conectar (Linux/Mac)
chmod 400 devops-front.pem

# Conectar a EC2-web (única instancia con IP pública)
ssh -i devops-front.pem ec2-user@3.83.173.99

# Conectar a EC2-app usando EC2-web como bastión
ssh -i devops-front.pem -J ec2-user@3.83.173.99 ec2-user@10.0.131.198

# Conectar a EC2-datos usando EC2-web como bastión
ssh -i devops-front.pem -J ec2-user@3.83.173.99 ec2-user@10.0.145.181
```

### Levantar los microservicios en EC2-app

```bash
# 1. Conectar a EC2-app
ssh -i devops-front.pem -J ec2-user@3.83.173.99 ec2-user@10.0.131.198

# 2. Ir al directorio del proyecto
cd ~/ProyectoSemestral

# 3. Levantar en segundo plano (usa imágenes de Docker Hub)
docker-compose up -d

# 4. Verificar que los contenedores corren
docker-compose ps

# 5. Ver logs en tiempo real
docker-compose logs -f backend-despachos
docker-compose logs -f backend-ventas

# 6. Detener los servicios
docker-compose down
```

### Endpoints en AWS

| Servicio | URL pública | Puerto en EC2-app |
|---|---|---|
| Frontend | http://3.83.173.99:80 | — |
| API Despachos | http://3.83.173.99:8081 | 8081 |
| API Ventas | http://3.83.173.99:8082 | 8082 |
| Swagger Despachos | http://3.83.173.99:8081/swagger-ui.html | 8081 |
| Swagger Ventas | http://3.83.173.99:8082/swagger-ui.html | 8082 |
| MySQL | — (solo VPC interna) | 3306 |

Acceso interno desde EC2-web hacia EC2-app:

| Servicio | URL interna |
|---|---|
| backend-despachos | http://10.0.131.198:8081 |
| backend-ventas | http://10.0.131.198:8082 |
| MySQL | mysql://10.0.145.181:3306 |

### Security Groups configurados

| Security Group | Instancia | Reglas de entrada |
|---|---|---|
| sg-web | EC2-web | 22 SSH desde cualquier IP · 80/443 HTTP/HTTPS desde cualquier IP |
| sg-app | EC2-app | 22 SSH desde sg-web · 8081/8082 desde sg-web |
| sg-datos | EC2-datos | 3306 MySQL desde sg-app únicamente |

### AWS vs ejecución local

| Aspecto | Local (docker-compose) | AWS |
|---|---|---|
| Acceso externo | Solo localhost | IP pública accesible desde cualquier lugar |
| Alta disponibilidad | No | Posible con Auto Scaling y ALB |
| Aislamiento de red | Red Docker interna | VPC `10.0.0.0/16` con subredes públicas y privadas |
| Base de datos | Contenedor efímero | EC2 dedicado con volumen EBS persistente |
| Seguridad | Sin Security Groups | AWS Security Groups y NACLs por instancia |
| Escalabilidad | Limitada al host | Vertical y horizontal según necesidad |
| Imágenes | Build local | Docker Hub (`dgomezpalacios/`) |

## Seguridad

- Cambiar las credenciales de base de datos (`root123`) por secretos gestionados (Docker Secrets, Vault) en producción.
- En producción usar `spring.jpa.hibernate.ddl-auto=validate` o `none` para evitar migraciones automáticas no controladas.
- Nunca exponer el puerto 3306 de MySQL en entornos productivos.

## DevOps

El `Dockerfile` de cada microservicio sigue el patrón multi-stage:

1. **Stage `builder`** (`maven:3.8.1-openjdk-17`): descarga dependencias y empaqueta el JAR.
2. **Stage runtime** (`openjdk:17-slim`): copia solo el JAR final y lo ejecuta con usuario sin privilegios.

El `HEALTHCHECK` integrado permite que `docker-compose` y orquestadores como Kubernetes conozcan el estado real de la aplicación.

## ☁️ EP3 — Despliegue en AWS ECS Fargate

### Arquitectura EP3
Los microservicios migraron de EC2 con Docker Compose a AWS ECS Fargate con orquestación serverless. Cada microservicio tiene su propio servicio ECS independiente.

- **Orquestador:** AWS ECS Fargate
- **Registry:** Amazon ECR (3 repositorios privados)
- **Load Balancer:** Application Load Balancer (ALB)
- **Base de datos:** EC2-datos MySQL 8.0 (IP privada: 10.0.145.181)

### Recursos AWS

| Recurso | Despachos | Ventas |
|---------|-----------|--------|
| Repositorio ECR | proyecto-semestral-backend-despachos | proyecto-semestral-backend-ventas |
| Task Definition | despachos-task:1 | ventas-task:1 |
| Servicio ECS | despachos-service | ventas-service |
| Target Group | despachos-tg (puerto 8080) | ventas-tg (puerto 8080) |
| CPU | 512 | 512 |
| Memoria | 1024 MB | 1024 MB |

### Rutas ALB
| Ruta | Destino |
|------|---------|
| /api/despachos* | despachos-service |
| /api/ventas* | ventas-service |

### Variables de entorno en ECS
| Variable | Despachos | Ventas |
|----------|-----------|--------|
| SPRING_DATASOURCE_URL | jdbc:mysql://10.0.145.181:3306/despachos_db | jdbc:mysql://10.0.145.181:3306/ventas_db |
| SPRING_DATASOURCE_USERNAME | root | root |
| SPRING_JPA_HIBERNATE_DDL_AUTO | update | update |

### Pipeline CI/CD (EP3)
El pipeline se dispara automáticamente en cada push a la rama deploy y procesa ambos microservicios en el mismo workflow:

1. Build Despachos — docker build -f Dockerfile .
2. Push Despachos — sube imagen a ECR con tag :latest y :<commit-sha>
3. Build Ventas — docker build -f back-ventas/Dockerfile ./back-ventas
4. Push Ventas — sube imagen a ECR con tag :latest y :<commit-sha>
5. Deploy Despachos — aws ecs update-service --force-new-deployment
6. Deploy Ventas — aws ecs update-service --force-new-deployment

#### Secrets requeridos en GitHub
| Secret | Descripción |
|--------|-------------|
| AWS_ACCESS_KEY_ID | Credencial AWS |
| AWS_SECRET_ACCESS_KEY | Credencial AWS |
| AWS_SESSION_TOKEN | Token de sesión AWS Academy |

### Cambios en Dockerfile para ECS
La imagen base fue actualizada de openjdk:17-slim (deprecada) a eclipse-temurin:17-jre-alpine.
El comando de creación de usuario fue actualizado de groupadd/useradd a addgroup/adduser para compatibilidad con Alpine Linux.

### Autoscaling ECS
| Parámetro | Valor |
|-----------|-------|
| Tipo | Target Tracking Scaling |
| Métrica | ECSServiceAverageCPUUtilization |
| Umbral | 50% CPU |
| Mínimo tasks | 1 |
| Máximo tasks | 3 |
| Cooldown | 60 segundos |

### Logs
Los logs se envían automáticamente a CloudWatch Logs:
- Despachos: /ecs/despachos
- Ventas: /ecs/ventas

## 👥 Equipo
- Daniela Gómez Palacios
- Berta Soto Jerez

**Curso:** ISY1101 — Introducción a Herramientas DevOps  
**Profesor:** Álvaro Mellado Pimentel  
**Instituto:** DuocUC — 2025
