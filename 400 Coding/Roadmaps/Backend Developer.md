## 🌱 **Nivel Básico (Fundamentos)**
### 1. **Java Core**
   - **Proyecto**: 
     - `Calculadora CLI`: Usa POO para operaciones básicas (+ pruebas unitarias con JUnit).
     - `Gestor de Tareas`: Lista de tareas en memoria (sin base de datos).

### 2. **Spring Boot Basics**
   - **Proyecto**: 
     - `API de Libros`: 
       - Endpoints CRUD para libros (`/books`). 
       - Usa Spring Data JPA + H2 (base de datos en memoria).
     - `Weather API Proxy`: Consume una API pública (ej.: OpenWeatherMap) y expón datos simplificados.

---

## 🚧 **Nivel Intermedio (Bases de Datos + Seguridad)**
### 3. **SQL + Spring Data JPA**
   - **Proyecto**: 
     - `Blog API`: 
       - Posts, comentarios (relaciones `@OneToMany`). 
       - Usa PostgreSQL.
     - `Sistema de Reservas`: 
       - Tablas para usuarios, reservas, recursos (ej.: salas). 
       - Consultas con `@Query` personalizadas.

### 4. **Seguridad (Spring Security + JWT)**
   - **Proyecto**: 
     - `Auth API`: 
       - Registro/login con JWT. 
       - Roles (USER, ADMIN) y permisos (`@PreAuthorize`).
     - `API con OAuth2`: Integra login con Google/GitHub.

---

## 🛠️ **Nivel Avanzado (Arquitectura + Rendimiento)**
### 5. **Arquitectura Limpia/Hexagonal**
   - **Proyecto**: 
     - `E-commerce Modular`: 
       - Capas separadas (dominio, aplicación, infraestructura). 
       - Usa DTOs y mapeadores (MapStruct).

### 6. **Caché + Concurrencia**
   - **Proyecto**: 
     - `API de Productos con Redis`: 
       - Cachea respuestas de productos populares. 
       - Invalida caché al actualizar datos.
     - `Contador de Visitas`: Usa `@Async` para registrar visitas en segundo plano.

### 7. **Mensajería (RabbitMQ/Kafka)**
   - **Proyecto**: 
     - `Notificaciones en Tiempo Real`: 
       - Envía emails (simulados) al crear una orden vía RabbitMQ.
     - `Procesador de Archivos`: Sube un CSV, procesa en lote con Spring Batch.

---

## ☁️ **Nivel Experto (Cloud + DevOps)**
### 8. **Docker + Kubernetes**
   - **Proyecto**: 
     - `Dockerizar el Blog API`: 
       - Crea un `Dockerfile` y usa `docker-compose` para PostgreSQL.
     - `Despliegue en Minikube`: 
       - Configura pods y servicios en Kubernetes local.

### 9. **Cloud (AWS/Azure)**
   - **Proyecto**: 
     - `File Storage API`: 
       - Sube archivos a AWS S3/Azure Blob Storage. 
       - Usa SDK de Spring Cloud AWS/Azure.

---

## 🧪 **Proyectos Integrales (Final Roadmap)**
1. **`Sistema de Pedidos`**:
   - Microservicios: 
     - `Orders` (Spring Boot + JPA). 
     - `Payments` (Mock de pasarela de pago). 
     - Comunicación vía Kafka.
   - Docker + Kubernetes.

2. **`Clone de Twitter (Backend)`**:
   - Tweets, followers, timelines. 
   - Cachea feeds con Redis. 
   - Autenticación JWT + OAuth2.

3. **`Monitor de Salud API`**:
   - Endpoints para métricas (CPU, memoria). 
   - Usa Spring Actuator + Prometheus/Grafana.
